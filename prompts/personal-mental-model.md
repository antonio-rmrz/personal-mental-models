# Personal Mental Model — Prompt Pack

Generate a personal mental model in any AI tool. No plugin install, no editor lock-in. Works in ChatGPT, Claude.ai, Cursor chat, Copilot Chat, Continue, JetBrains AI Assistant — anywhere with a chat box.

---

## How it works

Two steps:

1. **Run a small shell script locally** to fetch one reviewer's PR history. Saves a `reviews.json` file.
2. **Paste the prompt below + the JSON** into your AI tool. It returns a complete `expertise.yaml`.

The shell script does deterministic data collection (`gh api` calls). The AI does the synthesis. Clean separation.

---

## Step 1 — Data collection

Save the script below as `collect.sh` somewhere you can run it (any local terminal). You need:

- `gh` CLI installed and authenticated against an account that can read the target repo
- `jq` (most package managers; `brew install jq` on macOS)

```bash
#!/usr/bin/env bash
# collect.sh — fetch one reviewer's PR review history
# Usage: ./collect.sh <github-handle> <owner/repo> [months_back=6]

HANDLE="$1"
REPO="$2"
MONTHS="${3:-6}"

if [ -z "$HANDLE" ] || [ -z "$REPO" ]; then
  echo "Usage: $0 <github-handle> <owner/repo> [months_back=6]" >&2
  exit 1
fi

SINCE=$(date -v-${MONTHS}m +%Y-%m-%d 2>/dev/null \
     || date -d "${MONTHS} months ago" +%Y-%m-%d)

echo "Finding PRs reviewed by @$HANDLE in $REPO since $SINCE..."

PR_NUMBERS=$(gh api -X GET search/issues --paginate \
  -f q="repo:$REPO is:pr reviewed-by:$HANDLE updated:>=$SINCE" \
  --jq '.items[].number' | tr '\n' ' ')

if [ -z "$PR_NUMBERS" ]; then
  echo "No PRs found. Try widening months_back or pick a repo where @$HANDLE reviews more." >&2
  exit 1
fi

PR_COUNT=$(echo $PR_NUMBERS | wc -w | tr -d ' ')
echo "Found $PR_COUNT PRs. Pulling reviews + comments (this can take a minute)..."

(for N in $PR_NUMBERS; do
  gh api "repos/$REPO/pulls/$N/reviews"   --jq ".[] | select(.user.login==\"$HANDLE\") | {pr: $N, stream: \"review\",  state, body, submitted_at}" 2>/dev/null
  gh api "repos/$REPO/pulls/$N/comments"  --jq ".[] | select(.user.login==\"$HANDLE\") | {pr: $N, stream: \"inline\",  path, body, created_at}" 2>/dev/null
  gh api "repos/$REPO/issues/$N/comments" --jq ".[] | select(.user.login==\"$HANDLE\") | {pr: $N, stream: \"comment\", body, created_at}" 2>/dev/null
done) | jq -s '.' > reviews.json

ITEM_COUNT=$(jq 'length' reviews.json)
echo "Done. Wrote $ITEM_COUNT review items from $PR_COUNT PRs to reviews.json"
echo "If $ITEM_COUNT < 60, the resulting model will be thin — consider widening the window."
```

Run it:

```
chmod +x collect.sh
./collect.sh <your-github-handle> <owner/repo>
```

You'll get a `reviews.json` file. Open it — sanity-check that there are at least ~60 items. Fewer than that and the patterns are too sparse to generalize.

---

## Step 2 — Synthesis prompt

Open your AI tool of choice. Paste the prompt below. Replace the three placeholders (`{HANDLE}`, `{TARGET_REPO}`, `{REVIEWS_JSON}`) — for the JSON, either paste the file contents inline, or upload `reviews.json` as a project file if your tool supports it (Claude.ai, ChatGPT projects, Cursor docs, etc.).

```
You are generating a "personal mental model" for one human reviewer based on their PR review history. The output is a structured YAML file capturing how they review code — values, ownership, vocabulary, red/green flags, gotchas. This is a reviewer model (a person), not a codebase model (a project).

INPUTS
- reviewer_handle: {HANDLE}
- target_repo: {TARGET_REPO}
- review_data: an array of comment / review / inline-comment objects from GitHub. Paste from reviews.json:

{REVIEWS_JSON}

TASK

Cluster the review data into the eight reviewer-native sections below and produce a single YAML document. Shape is fixed; contents are derived from what the data actually shows. Don't invent — patterns must be backed by ≥ 2 verbatim PR citations to qualify.

1. overview
   name (synthesize from handle if no real name in data), github (the handle),
   role (one line — infer from what they review), core_review_philosophy
   (one paragraph synthesizing dominant themes), review_intensity (one line),
   approval_style (how they signal approve vs block).

2. ownership_map
   Three tiers, derived from where their inline comments cluster (group inline-
   comment paths by directory, count occurrences):
     - primary_owner: dirs with ≥ 20 inline comments — deep technical depth
     - active_reviewer: dirs with 5–20 inline comments — architectural focus
     - peripheral: dirs with < 5 inline comments — light/rubber-stamp
   For each path: depth (deep / architectural / moderate / light / rubber-stamp)
   and one-line notes.

3. values
   RANKED 1..N, target ~7 entries. Principles they enforce, ordered by
   frequency × severity. Each entry: rank, name (snake_case), description,
   evidence (list of PR# + verbatim quote). Drop any candidate value with
   fewer than 3 cited instances.

4. red_flags
   ~8 entries. Patterns that reliably trigger CHANGES_REQUESTED. Columns:
   flag, severity (blocking / soft_block / change_requested /
   immediate_correction), trigger, example (PR# + quote).

5. green_flags
   ~6 entries. What earns fast approvals or explicit praise (❤️, "lgtm",
   etc.). Columns: flag, signal, example.

6. review_vocabulary
   Decoder ring. Four buckets: blocking_phrases, soft_blocking_phrases,
   approval_phrases, neutral_phrases. Each row: phrase (verbatim — preserve
   casing, contractions, politeness markers), meaning, example PR#.
   This is the most useful section — be precise about surface form.

7. domain_opinions
   ~8 entries. Strong architectural stances. Columns: topic, stance,
   evidence (PR#s).

8. gotchas
   ~10–12 numbered items. Non-obvious behaviors of their review style.

RULES

- Quotes are verbatim. Every cited phrase must appear in the source data exactly as written, with the PR number that contained it.
- Evidence threshold: no value, red_flag, or green_flag with fewer than 2 PR citations. Drop weak ones rather than padding.
- Preserve voice. Keep the reviewer's actual phrasing — lowercase, contractions, politeness markers ("i'm sorry, but…"). Sterilizing destroys the decoder ring.
- No fabrication. If a section ends up thin, ship it thin and note "fewer than the suggested entries — only N patterns met the evidence threshold".

OUTPUT

Return a single YAML document, valid as expertise.yaml, ready to save. No commentary before or after — just the YAML.
```

---

## Tips per AI tool

- **ChatGPT (GPT-4 / GPT-4o)** — handles ~150 review items comfortably in one shot. For very heavy reviewers (300+ items), split into two passes (months 1–3 and 4–6) and ask the AI to merge the YAMLs in a third pass.
- **Claude.ai web** — upload `reviews.json` as a project file rather than pasting. Keeps the prompt clean. Claude handles the full data set well even for heavy reviewers.
- **Cursor / Continue chat** — paste directly. Both have ample context windows.
- **GitHub Copilot Chat** — smaller context. Pre-summarize your `reviews.json` (e.g. `jq '.[].body' reviews.json | head -200`) before pasting, or use a different tool for synthesis and load the resulting YAML into Copilot's instructions afterward.

---

## Save the output

Save the AI's response as `expertise.yaml` somewhere convenient. To use it:

- **Claude Code:** create `.claude/commands/mental-model/<your-slug>/expertise.yaml` and add the three sibling commands manually (templates in [`commands/create-personal-mental-model.md`](../commands/create-personal-mental-model.md)). Or just install the plugin — it does this for you.
- **Cursor / others:** see the table in the [main README](../README.md#use-it-anywhere).

---

## Privacy & consent

The data flowing through the prompt is your own PR reviews on a repo you have access to. If you're modeling someone other than yourself, ask them first — synthesizing a structured persona is different from quoting individual comments. The most fun first run is on yourself.
