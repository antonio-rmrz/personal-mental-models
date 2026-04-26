---
name: create-personal-mental-model
allowed-tools: Bash, Read, Write, Glob, Grep, TodoWrite
description: Generate a "personal mental model" for one human reviewer by traversing every PR review they've authored on a target repo. Writes a finished mental-model directory (expertise.yaml + 3 sibling slash commands + evals) directly to .claude/commands/mental-model/<slug>/, ready to use immediately.
argument-hint: <github-handle> <target-owner/repo> [months_back=6] [--global]
---

# Create Personal Mental Model

Generate a structured profile of how one specific human reviews code on GitHub, by reading their actual review history on a target repo. The output is a finished `.claude/commands/mental-model/<slug>/` directory — drop-in usable in any Claude Code project.

This is a **personal** mental model — it captures a person, not a codebase. Pair it with codebase-domain mental models if you have them; they're orthogonal.

---

## Variables

```
HANDLE:        $1                                   # e.g. octocat
TARGET_REPO:   $2                                   # e.g. facebook/react
MONTHS_BACK:   ${3:-6}                              # PR window (default 6)
GLOBAL:        $4 == "--global"                     # if set, write to ~/.claude/ instead of project-local
SLUG:          lowercased + hyphenated form of HANDLE
OUT_DIR:       <cwd>/.claude/commands/mental-model/${SLUG}/
                 (or  ~/.claude/commands/mental-model/${SLUG}/  when --global)
```

---

## Workflow

### Step 1 — Validate inputs

Fail fast and ask the user if any are missing or malformed:

- `HANDLE` non-empty, no `@`, no spaces
- `TARGET_REPO` matches `owner/repo`
- `MONTHS_BACK` is a positive integer
- `gh auth status` shows an account that can read `TARGET_REPO`. If 404 on a private repo, surface that clearly — do **not** proceed silently against a missing repo.

If `HANDLE` matches the active `gh` user (`gh api user --jq .login`), print a friendly:

> 💡 Best first run — you're modeling yourself. The output is most interesting when the data includes you.

### Step 2 — Find every PR the user has reviewed

Compute the cutoff date:

```bash
SINCE=$(date -v-${MONTHS_BACK}m +%Y-%m-%d 2>/dev/null || date -d "${MONTHS_BACK} months ago" +%Y-%m-%d)
```

Discover candidate PRs in one paginated call:

```bash
gh api -X GET search/issues \
  --paginate \
  -f q="repo:${TARGET_REPO} is:pr reviewed-by:${HANDLE} updated:>=${SINCE}" \
  --jq '.items[] | {number, title, state, updated_at, html_url}'
```

If the result is fewer than ~30 PRs, **warn the user**: the model will be thin. Offer to widen `MONTHS_BACK` or pick a repo with more reviews. Don't bail automatically — let them choose.

**Aim for ≥ 80 PRs of coverage when available.** Patterns that appear once aren't patterns.

### Step 3 — Pull every comment & review by that user, per PR

For each PR `$N` from Step 2, fetch three streams and filter to `HANDLE`:

```bash
gh api repos/${TARGET_REPO}/pulls/$N/reviews     --jq ".[] | select(.user.login==\"${HANDLE}\") | {pr:$N, kind:\"review\",  state, body, submitted_at}"
gh api repos/${TARGET_REPO}/pulls/$N/comments    --jq ".[] | select(.user.login==\"${HANDLE}\") | {pr:$N, kind:\"inline\",  path, body, created_at}"
gh api repos/${TARGET_REPO}/issues/$N/comments   --jq ".[] | select(.user.login==\"${HANDLE}\") | {pr:$N, kind:\"comment\", body, created_at}"
```

Collect into a single in-memory list. For each item, capture: PR number, PR title, review state (APPROVED / CHANGES_REQUESTED / COMMENTED), file/path for inline comments, and **verbatim body text** (do not paraphrase — quotes are the evidence layer).

> Cache raw output in `/tmp/personal-mental-model-${SLUG}-raw.json` so you can re-cluster without re-fetching. A heavy reviewer over 6 months yields 200–800 comments; calls run in parallel where possible.

### Step 4 — Cluster into the eight reviewer-native sections

Read every captured comment and group into the **eight sections** below. Shape is fixed; contents are derived from what the data actually shows. Don't invent — patterns must be backed by **≥ 2 verbatim PR citations** to qualify.

1. **`overview`** — name, github handle, role (one line), `core_review_philosophy` (one paragraph synthesizing the dominant themes), `review_intensity` (one line), `approval_style` (how they typically signal approve vs block).

2. **`ownership_map`** — three tiers, derived from where their inline comments cluster (group inline-comment paths by directory, count occurrences):
   - `primary_owner`: dirs with ≥ 20 inline comments — deep technical depth
   - `active_reviewer`: dirs with 5–20 inline comments — architectural focus
   - `peripheral`: dirs with < 5 inline comments — light/rubber-stamp
   For each path: `depth` (one of: deep, architectural, moderate, light, rubber-stamp) and a one-line `notes`.

3. **`values`** — RANKED 1..N, target ~7 entries. Principles they enforce, ordered by `frequency × severity`. Each entry: `rank`, `name` (snake_case), `description`, `evidence` (list of `PR# + verbatim quote`). Drop any candidate value with fewer than 3 cited instances.

4. **`red_flags`** — ~8 entries, table-form. Patterns that reliably trigger CHANGES_REQUESTED. Columns: `flag`, `severity` (blocking / soft_block / change_requested / immediate_correction), `trigger`, `example` (PR# + quote).

5. **`green_flags`** — ~6 entries. What earns fast approvals or explicit praise (❤️, "lgtm", etc.). Columns: `flag`, `signal`, `example`.

6. **`review_vocabulary`** — the decoder ring. Four buckets: `blocking_phrases`, `soft_blocking_phrases`, `approval_phrases`, `neutral_phrases`. Each row: `phrase` (verbatim — preserve casing, contractions, politeness markers), `meaning`, `example` PR#. **This is the most useful section** — be precise about surface form.

7. **`domain_opinions`** — ~8 entries. Strong architectural stances. Columns: `topic`, `stance`, `evidence` (PR#s).

8. **`gotchas`** — numbered list, ~10–12 items. Non-obvious behaviors of their review style. E.g. *"approvals with outstanding comments are not final"*, *"naming alone can be the sole blocker"*.

Reference shape: see `examples/synthetic-frontend-reviewer.yaml` in the personal-mental-models plugin repo.

### Step 5 — Write the directory

Create `OUT_DIR` (mkdir -p). Write five files. The `expertise.yaml` is the analyzed content from Step 4. The other four are boilerplate — write them verbatim from the templates below, substituting `{{SLUG}}`, `{{HANDLE}}`, `{{TARGET_REPO}}`, `{{TITLE}}`.

#### `OUT_DIR/expertise.yaml`

The full eight-section YAML produced in Step 4. Validate with:

```bash
python3 -c "import yaml; yaml.safe_load(open('OUT_DIR/expertise.yaml')); print('YAML valid')"
```

Enforce ≤ 1000 lines. If over, trim oldest evidence quotes (not the patterns themselves).

#### `OUT_DIR/question.md`

```markdown
---
name: mental-model:{{SLUG}}:question
allowed-tools: Read, Bash, TodoWrite
description: Read-only Q&A against the {{TITLE}} personal mental model. Decode their review vocabulary, predict approval verdicts, surface the values and red_flags relevant to a given task.
argument-hint: [question]
---

# {{TITLE}} — Q&A

Variables:
- USER_QUESTION: $1
- EXPERTISE_PATH: .claude/commands/mental-model/{{SLUG}}/expertise.yaml

Workflow:
1. Read EXPERTISE_PATH.
2. Identify the relevant sections (ownership_map, values, red_flags, review_vocabulary, etc.) given USER_QUESTION.
3. Answer directly, citing PR evidence from the expertise file. Decode any vocabulary phrases. Predict the likely outcome (LGTM / CHANGES_REQUESTED / hard block) when applicable.

**IMPORTANT:** Read-only. Do not write, edit, or create any files.
```

#### `OUT_DIR/plan.md`

```markdown
---
name: mental-model:{{SLUG}}:plan
allowed-tools: Read, TodoWrite, Grep, Glob
description: Analyze a task through the {{TITLE}} reviewer's lens. Surface red_flags, ownership-map relevance, and predicted verdict before you write the code.
argument-hint: [task description]
---

# {{TITLE}} — Reviewer-Lens Plan

Variables:
- USER_TASK: $1
- EXPERTISE_PATH: .claude/commands/mental-model/{{SLUG}}/expertise.yaml

Workflow:
1. Read EXPERTISE_PATH.
2. Identify which `ownership_map` tiers the task touches.
3. Surface the top-3 `values` and any `red_flags` likely to trigger.
4. Decode any `review_vocabulary` phrases the reviewer would probably use on this PR.
5. Return a structured response:

   - **Tier touched:** primary_owner / active_reviewer / peripheral
   - **Constraints to honor:** top values + red_flags relevant to the task
   - **Predicted verdict:** LGTM / inline-comments / CHANGES_REQUESTED / hard block
   - **Vocabulary to expect:** likely phrases this reviewer will use
   - **One-paragraph plan:** how to approach the task to maximize LGTM odds

**IMPORTANT:** Read-only. Do not write, edit, or create any code files.
```

#### `OUT_DIR/self-improve.md`

```markdown
---
name: mental-model:{{SLUG}}:self-improve
allowed-tools: Read, Bash, Edit, Write, TodoWrite
description: Refresh the {{TITLE}} personal mental model from recent PR review history on {{TARGET_REPO}} via gh api. Validates the model against new data, adds emergent patterns, prunes stale evidence.
argument-hint: [pr_range_start (int)] [pr_range_end (int)]
---

# {{TITLE}} — Self-Improve

Variables:
- HANDLE: {{HANDLE}}
- TARGET_REPO: {{TARGET_REPO}}
- EXPERTISE_PATH: .claude/commands/mental-model/{{SLUG}}/expertise.yaml
- PR_START: $1 (default: latest PR# already cited in expertise + 1)
- PR_END: $2 (default: most recent PR in repo)
- MAX_LINES: 1000

Workflow:
1. **Read** EXPERTISE_PATH to understand existing patterns and evidence.
2. **Fetch** new reviews from PR_START to PR_END, filtered to HANDLE:
   - `gh api repos/${TARGET_REPO}/pulls/${N}/reviews --jq '.[] | select(.user.login=="${HANDLE}")'`
   - `gh api repos/${TARGET_REPO}/pulls/${N}/comments --jq '.[] | select(.user.login=="${HANDLE}")'`
   - `gh api repos/${TARGET_REPO}/issues/${N}/comments --jq '.[] | select(.user.login=="${HANDLE}")'`
3. **Diff** new patterns against existing sections:
   - New blocking phrases not in `review_vocabulary` → add
   - New red_flags appearing ≥ 2 times → add
   - Shifts in value-ranking frequency → re-rank
   - New directories with ≥ 3 inline comments → add to `ownership_map`
4. **Add** new PR evidence to existing entries — cite PR numbers.
5. **Do not remove** evidence from old PRs without strong counter-evidence (the reviewer hasn't repeated the pattern in N PRs AND has explicitly contradicted it).
6. **Enforce** line limit: `wc -l ${EXPERTISE_PATH}` ≤ MAX_LINES. If over, trim oldest weakest evidence (not the patterns themselves).
7. **Validate**: `python3 -c "import yaml; yaml.safe_load(open('${EXPERTISE_PATH}')); print('YAML valid')"`

Report: PRs scanned, new patterns found, updates made, final line count.
```

#### `OUT_DIR/evals/evals.json`

```json
{
  "name": "{{SLUG}}-question-evals",
  "description": "Three scenarios validating the {{TITLE}} mental model's question command",
  "evals": [
    {
      "id": 1,
      "name": "scope_check",
      "prompt": "I'm bundling an unrelated bug fix into a feature PR. Would {{TITLE}} approve?",
      "expected_behavior": "References scope-discipline value (or equivalent in this reviewer's model). Cites at least one PR# from the expertise file. Recommends splitting the bug fix into a separate PR.",
      "assertions": [
        "Does NOT create or modify any files",
        "Cites at least one PR# from the expertise file as evidence",
        "Predicts CHANGES_REQUESTED or hard block",
        "Recommends a concrete next action (split, or address bundled change separately)"
      ]
    },
    {
      "id": 2,
      "name": "vocabulary_decode",
      "prompt": "{{TITLE}} commented \"i'd be careful here\" on my PR. How serious is this?",
      "expected_behavior": "Looks up the phrase (or closest equivalent) in review_vocabulary. Classifies its severity correctly. Notes that polite phrasing does not soften the meaning.",
      "assertions": [
        "Does NOT create or modify any files",
        "References the review_vocabulary section",
        "Classifies the phrase severity (blocking / soft_block / approval / neutral)",
        "Notes that politeness markers do not change severity"
      ]
    },
    {
      "id": 3,
      "name": "ownership_map_lookup",
      "prompt": "I'm changing a file in a directory I'm unsure {{TITLE}} reviews. How do I check?",
      "expected_behavior": "Reads ownership_map. Reports which tier (primary_owner / active_reviewer / peripheral) the directory falls under, or notes if it's not in the map. Predicts review depth.",
      "assertions": [
        "Does NOT create or modify any files",
        "References the ownership_map section",
        "Returns a tier classification (or 'not in ownership_map')",
        "Predicts review depth (deep / architectural / moderate / light / rubber-stamp)"
      ]
    }
  ]
}
```

### Step 6 — Validate everything

```bash
python3 -c "import yaml; yaml.safe_load(open('${OUT_DIR}/expertise.yaml')); print('expertise YAML valid')"
python3 -c "import json; json.load(open('${OUT_DIR}/evals/evals.json')); print('evals JSON valid')"
wc -l ${OUT_DIR}/expertise.yaml
ls ${OUT_DIR}
```

Fix any syntax errors before reporting success. If line count > 1000, trim least-critical evidence quotes (not patterns).

### Step 7 — Report

```
Personal Mental Model Generated

Reviewer:    @${HANDLE}
Target repo: ${TARGET_REPO}
Window:      last ${MONTHS_BACK} months
PRs scanned: N
Comments analyzed: M

Output: ${OUT_DIR}
  ├── expertise.yaml          (L lines)
  ├── question.md
  ├── plan.md
  ├── self-improve.md
  └── evals/evals.json        (3 scenarios)

Sections populated:
  overview            ✓
  ownership_map       T tiers, P paths
  values              N ranked
  red_flags           N entries
  green_flags         N entries
  review_vocabulary   N phrases (blocking/soft/approval/neutral)
  domain_opinions     N entries
  gotchas             N entries

Next:
  /mental-model:${SLUG}:question  "would they approve adding a magic number?"
  /mental-model:${SLUG}:plan      "refactor the auth flow"
  /mental-model:${SLUG}:self-improve   # re-run after another month of new PRs

If you modeled someone other than yourself, share the YAML with them — it's
their persona, not yours. Treat consent as a feature, not an obstacle.
```

---

## Rules

- **Quotes are verbatim.** Every cited phrase must appear in the source data exactly as written, with the PR number that contained it.
- **Evidence threshold.** No `value`, `red_flag`, or `green_flag` ships with fewer than 2 PR citations. Drop weak ones rather than padding.
- **Preserve voice.** In `review_vocabulary`, keep the reviewer's actual phrasing — lowercase, contractions, politeness markers ("i'm sorry, but…"). Sterilizing the voice destroys the decoder ring.
- **No fabrication.** If a section ends up thin, ship it thin and note "fewer than the suggested entries — only N patterns met the evidence threshold".
- **Write the directory directly.** No issue-body intermediate, no dependency on `/implement` or `/planing`. The output is immediately usable as a Claude Code mental model.
