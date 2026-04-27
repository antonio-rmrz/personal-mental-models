# Personal Mental Models

> A structured snapshot of how *you* review code — your values, ownership map, vocabulary, red/green flags. Generate it once. Load it into whatever AI tool you use.

When agents review or build on your behalf, they default to generic LLM norms — a flattened consensus of "how the internet reviews code." That's wrong for the moments that matter. Your agent should bring **you**, not the median GitHub reviewer.

This repo gives you:

- 🧠 **The format** — eight reviewer-native YAML sections (`values`, `ownership_map`, `review_vocabulary`, `red_flags`, `gotchas`, …)
- ⚙️ **Multiple ways to generate one** — Claude Code plugin, copy-pasteable prompt pack, more on the way
- 🌐 **Cross-tool guidance** — load the same YAML into Claude Code, Cursor, Copilot, Continue, ChatGPT, JetBrains AI

---

## What's in the model

The artifact is a single `expertise.yaml` file. Eight sections:

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

See [`examples/synthetic-frontend-reviewer.yaml`](examples/synthetic-frontend-reviewer.yaml) for a worked example (~270 lines, fictional persona "Maya," teaches the format).

---

## Generate

Pick whichever fits your tooling. Every path produces the same `expertise.yaml`.

### 🔌 Claude Code plugin — recommended if you use Claude Code

```
/plugin marketplace add antonio-rmrz/personal-mental-models
/plugin install personal-mental-models
/create-personal-mental-model <your-handle> <a-repo-where-you-review>
```

Writes a finished `.claude/commands/mental-model/<you>/` directory with `expertise.yaml` plus three sibling slash commands (`question`, `plan`, `self-improve`).

### 📋 Prompt pack — works in any chat AI

Open [`prompts/personal-mental-model.md`](prompts/personal-mental-model.md). It walks you through:

1. Running a small shell script to fetch your PR review history (`gh api`, ~30 seconds)
2. Pasting the resulting `reviews.json` plus the synthesis prompt into ChatGPT, Claude.ai, Cursor chat, Copilot Chat, or anywhere else with a chat box

Output: the same `expertise.yaml` the plugin would generate. No install, no editor required.

### 🛠️ CLI — coming soon

`pipx install personal-mental-models` — bring-your-own-LLM-key (Anthropic / OpenAI / local Ollama). Same prompt, same output, no editor required. v0.2.

### ⚙️ GitHub Action — coming soon

Drop a workflow into your repo to refresh your model on a monthly cron and commit the YAML alongside your code. Best for teams who want models to live next to source. v0.3.

---

## Use it anywhere

Once you have an `expertise.yaml`, here's how to load it into the tool you actually use:

| Tool | How to load it |
|---|---|
| **Claude Code** | drop the file in `.claude/commands/mental-model/<you>/` (the plugin does this for you) |
| **Cursor** | save as `.cursor/rules/<you>.md` with a one-line wrapper, or paste into `.cursorrules` |
| **GitHub Copilot** | summarize into `.github/copilot-instructions.md` (Copilot's context handling is shallower; YAML structure may need flattening into prose) |
| **Continue.dev** | reference from `.continue/config.yaml` as a custom context source |
| **ChatGPT / Claude.ai web** | upload as a project file, or paste into custom instructions |
| **JetBrains AI Assistant** | paste into workspace custom instructions |

The pattern is the same everywhere: get the YAML into the model's context window, then ask questions or request reviews against it.

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

## Requirements

- `gh` CLI authenticated against an account that can read the target repo (private repos work too)
- The reviewer should have ≥ 30 PR reviews in the target repo — fewer than that and the patterns are too thin to generalize
- For the plugin path: Claude Code with plugin support
- For the prompt-pack path: any chat AI you already use

---

## Status

**v0.1** — generates one kind of personal mental model (reviewer). Two delivery channels live (Claude Code plugin + prompt pack), CLI and GitHub Action on the roadmap. PRs and ideas welcome.

---

## License

MIT — see [LICENSE](LICENSE). Generate freely; share what you learn.
