# Personal Mental Models

> An agentic representation of how *you* think. Bring your own perspective — review style, ranked values, ownership map, vocabulary — to any codebase you touch with Claude Code.

When agents review or build code on your behalf, they default to generic LLM norms. This plugin replaces that default with **you** — a structured snapshot of how you actually review code, derived from your own PR review history.

---

## 60 seconds to your own model

Install the plugin once:

```
/plugin marketplace add antonio-rmrz/personal-mental-models
/plugin install personal-mental-models
```

Generate your model:

```
/create-personal-mental-model <your-github-handle> <a-repo-where-you-review-prs>
```

That's it. Claude reads your last 6 months of PR reviews on the target repo and writes a working `expertise.yaml` plus three sibling slash commands to `.claude/commands/mental-model/<your-handle>/`. Open the YAML. That's you, structured.

> 💡 **Best first run:** try it on yourself, against a repo where you've left at least 30 PR reviews. The output is more interesting when the data includes you.

---

## What you get

After running `/create-personal-mental-model jane octocat/spoon-knife`, you'll have:

```
.claude/commands/mental-model/jane/
├── expertise.yaml          ← the model — 8 sections, ~150–300 lines
├── question.md             ← /mental-model:jane:question — read-only Q&A
├── plan.md                 ← /mental-model:jane:plan — analyze a task through her lens
├── self-improve.md         ← /mental-model:jane:self-improve — refresh from new PRs
└── evals/
    └── evals.json          ← 3 scenarios for the question command
```

The eight sections of `expertise.yaml`:

| Section | What it captures |
|---|---|
| `overview` | role, philosophy, approval style |
| `ownership_map` | three tiers — primary / active / peripheral, derived from where their inline comments cluster |
| `values` | ranked principles, each with verbatim PR-cited evidence |
| `red_flags` | what reliably triggers CHANGES_REQUESTED |
| `green_flags` | what earns fast approvals |
| `review_vocabulary` | decoder ring — *"i'm uncomfortable with that"* → hard block, etc. |
| `domain_opinions` | strong architectural stances |
| `gotchas` | non-obvious quirks of their review style |

See [`examples/synthetic-frontend-reviewer.yaml`](examples/synthetic-frontend-reviewer.yaml) for a worked example.

---

## Try it on yourself

The fastest way to feel whether your model actually captures you is to interrogate it. Each prompt below is a small validation test — if the answer doesn't ring true, your evidence window was too narrow and you should re-run with more `months_back`.

In Claude Code:

```
/mental-model:<your-handle>:question  <the prompt>
```

In any other AI tool: drop your `expertise.yaml` into context and ask the same question directly.

### 🪞 Sanity check

> *"what does my model say my top 3 values are? cite the PR evidence for each."*

If you don't recognize the values listed back at you, the data is too thin — re-run with a wider PR window or a more review-heavy repo.

### 🔮 Surprise me

> *"what's one pattern in my reviews i probably don't realize i do?"*

This pulls from the `gotchas` section — non-obvious quirks that even careful self-reflection misses. If the answer is generic ("you care about code quality"), the model didn't capture anything distinctive.

### ⚖️ Predict the verdict

> *"i'm about to open a PR that bundles two unrelated bug fixes, has no tests, and renames a public function. predict my verdict and tell me which sections of my model trigger."*

Compare with your gut. If your model says "lgtm" and you'd actually leave twelve inline comments, the evidence threshold is set wrong. If it correctly identifies which `red_flags` fire and in what order, the predictive power is real.

### 🗝️ Decode your own voice

> *"what do i actually mean when i say 'i'd be careful here'? blocking, soft block, or nit?"*

The decoder-ring test. Replace the phrase with one you've actually used. A working model knows that polite phrasing doesn't soften severity — *"i'd be careful"*, *"i'd love it if"*, and *"have we considered..."* often mean **block**, not nit.

### ✍️ Self-portrait in your own voice

> *"describe me as a reviewer in one paragraph, in my voice."*

Read the answer out loud. If it sounds like you — your specific phrasing, your actual emphases — the model preserved voice. If it sounds like a generic "thoughtful senior engineer," the prompt sterilized the data and you should re-run.

---

> 💬 If one of these gives a great answer, screenshot it and share. It's the most efficient demo this idea has.

---

## Why "personal" mental models

Codebase mental models capture file paths and patterns — *facts about the project*. They make agents efficient at navigating code.

A *personal* mental model captures something different: how a specific human weighs decisions, what they care about, what they've said no to before. That's the part agents can't infer from source.

When you can't be in the room, your agent should bring **your** perspective, not a generic LLM consensus.

For a deeper writeup of the design intent, see [`docs/how-it-works.md`](docs/how-it-works.md).

---

## A note on modeling other people

The data this tool reads (PR reviews on public repos) is already public. But synthesizing someone's review style into a structured artifact feels different from quoting their individual comments — be thoughtful. **If you're modeling a colleague, ask them first.** The most fun first run is on yourself.

---

## How it works

`/create-personal-mental-model` does five things:

1. Calls `gh api search/issues` to find every PR in the target repo where the handle has authored a review.
2. Pulls every review, inline comment, and issue comment by that user across those PRs (verbatim, not paraphrased).
3. Clusters the comments into the eight reviewer-native sections, with PR-cited evidence quotes.
4. Writes the directory: `expertise.yaml` plus three sibling slash commands under `mental-model/<slug>/`.
5. Prints next steps — try `/mental-model:<slug>:question`, run `self-improve` after another month of reviews, share with your team.

---

## Requirements

- Claude Code with plugin support
- `gh` CLI authenticated against an account that can read the target repo (private repos work too)
- The reviewer should have ≥ 30 PR reviews in the target repo — fewer than that and the patterns are too thin to generalize

---

## Status

v0.1 — generates one kind of personal mental model (reviewer). The `personal-mental-models` umbrella is set up to host more flavors over time (style guides, decision archives, "what would $person say" models). PRs and ideas welcome.

---

## License

MIT — see [LICENSE](LICENSE). Generate freely; share what you learn.
