# How personal mental models work

## The gap they fill

Agents that touch your codebase default to generic LLM norms — a flattened consensus of "how people on the internet review code." That's fine for routine work. It's wrong for the moments that matter.

The moments that matter are when an agent is acting *on your behalf* — proposing a refactor, suggesting a fix, drafting a review comment. In those moments, the agent should bring **your** perspective. Not the median GitHub reviewer's.

Codebase mental models (the older, more familiar kind) don't fill this gap. They tell an agent where files live and what patterns the project uses. Useful — and orthogonal to the question of how *you*, specifically, weigh decisions.

A **personal mental model** captures the orthogonal half: how a specific human reviews. What they enforce. What they let slide. What words they use when they're really blocking versus mildly annoyed.

---

## What's in the model

Eight sections, designed to be reviewer-native rather than codebase-native:

1. **`overview`** — role, philosophy, approval style. The one-paragraph framing.
2. **`ownership_map`** — directories you actually review, in three tiers. Derived from where your inline comments cluster, not where you say they should.
3. **`values`** — ranked principles, each with ≥ 2 verbatim PR-cited evidence quotes. Ranking is by `frequency × severity`, not self-report.
4. **`red_flags`** — patterns that reliably trigger CHANGES_REQUESTED. Severity-tagged.
5. **`green_flags`** — what earns fast approvals or a heart emoji.
6. **`review_vocabulary`** — the decoder ring. *"i'd be careful here"* and *"have we considered..."* mean different things to different reviewers; this section captures yours.
7. **`domain_opinions`** — strong architectural stances with cited evidence.
8. **`gotchas`** — non-obvious behaviors of your review style. Things only a careful reader of your last 80 PRs would notice.

---

## How the model is built

`/create-personal-mental-model <handle> <repo>` makes five `gh api` calls per relevant PR:

```
gh api search/issues   q="repo:$REPO is:pr reviewed-by:$HANDLE updated:>=$SINCE"
gh api repos/$REPO/pulls/$N/reviews     # filtered to $HANDLE
gh api repos/$REPO/pulls/$N/comments    # inline code comments
gh api repos/$REPO/issues/$N/comments   # issue-level PR comments
```

Then it clusters the verbatim text into the eight sections, applying these rules:

- **Quotes are verbatim.** No paraphrasing. Every cited phrase appears in the source data exactly as written.
- **Evidence threshold.** No `value`, `red_flag`, or `green_flag` ships with fewer than 2 PR citations. Patterns that appear once aren't patterns.
- **Voice preserved.** `review_vocabulary` keeps the reviewer's actual phrasing — lowercase, contractions, politeness markers. Sterilizing the voice destroys the decoder ring.
- **No fabrication.** If a section ends up thin, it ships thin and notes that fewer than the suggested entries met the threshold.

---

## Using the model

Once the directory is written, four slash commands are live:

| Command | What it does |
|---|---|
| `/mental-model:<slug>:question <q>` | Read-only Q&A against the model — *"would Maya approve a PR that adds a magic number?"* |
| `/mental-model:<slug>:plan <task>` | Analyze a task through this reviewer's lens — surface red_flags, ownership relevance, predicted verdict |
| `/mental-model:<slug>:self-improve` | Refresh the model from new PR reviews via `gh api` |

The plugin doesn't ship a `plan_build_improve` chain — that's harness-specific. If you have a `/implement` or `/planing` of your own, wire one up; the pattern is in the existing slash commands.

---

## What good looks like

A useful personal mental model:

- **Predicts review verdicts before you submit.** If your model says you'd block on a magic-number-in-a-design-system, the agent should refuse to add one in a PR you'll see.
- **Decodes vocabulary correctly.** *"i'd be careful here"* should resolve to "soft block, expects justification" not "minor nit."
- **Distinguishes you from generic.** A reviewer model that reads like the LLM's default voice is a failed model. The text should sound like the actual person, not like a code-review FAQ.

If your generated model fails any of these, run `self-improve` — it usually means the data window was too narrow or the reviewer's style is still settling.

---

## What this is not

- **Not a replacement for dialogue.** When you can be in the room, be in the room. The model is for when you can't.
- **Not a finished system.** v0.1. The taxonomy of sections, the evidence threshold, the vocabulary classification — all open for revision based on what people generate.
- **Not a way to model someone without their knowledge.** PR reviews are technically public, but synthesizing a structured persona of someone is a different artifact. Ask first.
