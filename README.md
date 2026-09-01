# Context Budget

[English](README.md) | [Русский](README.ru-RU.md)

A skill that minimizes **input-token cost** during coding, debugging, review, and investigation by packing many actions into each turn and hard-capping the bytes any tool may return — without skipping required correctness checks.

**A/B-tested across three job types — a full 8-page website build, a bugfix in a small CLI, and a 12-question audit of a 132-file monorepo: 31–54% fewer tokens and 23–60% less wall-clock time with the skill.**

## Why input tokens explode

The transcript is re-sent as input on every turn and re-billed. Three things grow it permanently, because nothing is ever removed until compaction:

1. **Tool results** — every file read, every command's output stays in context for the rest of the session.
2. **Your own messages** — every narration line, restated plan, pasted snippet, and verbose answer is re-billed on every later turn.
3. **The instructions themselves** — one more reason the skill stays dense.

Extra turns and extra bytes compound: each new turn re-bills everything already in context, and each bloated result is re-billed by every later turn. So the skill enforces **both** levers at once:

- **Fewer turns** — pack as many independent actions as possible into each turn (speculative multi-action chains, all files written in one message).
- **Fewer bytes per turn** — hard-cap what any tool may return (~150-line reads, bounded search, `tail -n 30` command output).

## What the skill does

| Section | Rule |
|---|---|
| The Cost Model | Cost ≈ (bytes in context) × (turns remaining). Both factors must be attacked; neither may be traded for the other. |
| Turn Budgets by Job Type | Lookup 1–2 tool turns, review 2–3, bugfix 3, feature 4–6, unknown huge repo +1. Plus the no-regression rule: any rule that would add a turn the job does not need is skipped, and the protocol is read once, never re-read. |
| The Turn Rule | Speculative chains: predict the whole sequence, fire it batched (all Writes together, all reads together); for a question set put every symbol in one `rg` alternation. On the first surprise drop to single-action mode. Forbidden between actions: status updates, restating the plan, thinking out loud. |
| Hard Output Caps | A search hit is already the answer — re-opening the file to confirm is a defect. Reads ≤ ~150 lines (300 = defect), `rg -m 20` / `head -n 40`, `git diff --stat` first, `2>&1 \| tail -n 30`, never full logs/lockfiles/minified/JSON dumps, no result over ~10k tokens. |
| Delegate Only Above the Threshold | A subagent re-bills its own prompt on every request, so delegate only above ~2,000 lines / ~15 files or when the target is unknown. Never two agents on one question, never delegate what one `rg` answers. Delegate facts, not content you will edit. |
| Finish The Thinking, Then Emit | One pass per file: imports, error handling, tests in the same Write. Never re-read what you just wrote. Fold every fix implied by new results into one next turn. |
| Verify Once | Edits: one capped verification turn, no unrequested smoke checks. Lookups: the answer is the verification. |
| Session Hygiene | Reuse, never re-fetch. `path:line` instead of quoted content. Two failed attempts = re-derive, don't patch again. Long sessions: one notes file that survives compaction. |
| Non-Negotiables | Density never overrides correctness: no guessed file contents, no skipped tests, and an under-read that forces a second read costs more than the right-sized read. |

## Installation

The skill is a plain `SKILL.md` (plus optional UI metadata) — no dependencies, no scripts. Native **Agent Skills** support exists in ZCode, Claude Code, and Codex CLI. For rules-based agents, install it as an always-on rule instead.

### ZCode

```bash
git clone https://github.com/SoftArax/Context-Budget.git
cp -r Context-Budget ~/.zcode/skills/context-budget
```

User-level (every workspace). For a single project, copy into `<project>/.zcode/skills/context-budget` instead.

### Claude Code

```bash
git clone https://github.com/SoftArax/Context-Budget.git
cp -r Context-Budget ~/.claude/skills/context-budget
```

Same format (`SKILL.md` with `name`/`description` frontmatter) — Claude Code picks it up automatically and applies it when relevant; you can also invoke it explicitly as `/context-budget`. For a single project, copy into `<project>/.claude/skills/context-budget`.

### Codex CLI

```bash
git clone https://github.com/SoftArax/Context-Budget.git
cp -r Context-Budget ~/.codex/skills/context-budget
```

`agents/openai.yaml` supplies the UI metadata (display name, short description, default prompt).

### Cursor

Agent Skills are not supported; install as an always-on Project Rule:

```bash
git clone https://github.com/SoftArax/Context-Budget.git
mkdir -p .cursor/rules
{ printf -- '---\ndescription: Minimize input-token cost: many actions per turn, hard-capped tool output\nglobs: \nalwaysApply: true\n---\n\n'; tail -n +5 Context-Budget/SKILL.md; } > .cursor/rules/context-budget.mdc
```

Commit the rule to the repo so the whole team gets it.

### Windsurf

```bash
git clone https://github.com/SoftArax/Context-Budget.git
mkdir -p .windsurf/rules
tail -n +5 Context-Budget/SKILL.md > .windsurf/rules/context-budget.md
```

### Gemini CLI

Append the skill body to your context file:

```bash
git clone https://github.com/SoftArax/Context-Budget.git
tail -n +5 Context-Budget/SKILL.md >> ~/.gemini/GEMINI.md
```

### Cline / Roo Code

```bash
git clone https://github.com/SoftArax/Context-Budget.git
mkdir -p .clinerules   # Roo Code: .roo/rules/
tail -n +5 Context-Budget/SKILL.md > .clinerules/10-context-budget.md
```

### Anything that reads AGENTS.md

Paste the body of `SKILL.md` (everything after the frontmatter) into your `AGENTS.md` — it is written agent-agnostically and works wherever the model can read project instructions.

> Note: the subagent-delegation section only applies in agents that have subagents. Everywhere else that section is simply inert.

## A/B test results

Each task was given to two identical agents running the same model in byte-identical copies of the same fixture, in parallel: one with the skill loaded, one without. Token and tool counts come from the harness usage records, not self-reports. Both sides produced correct results in every run (tests passing / all 12 answers right).

| Task | Tokens with / without | Tool calls | Time |
|---|---|---|---|
| Bugfix — failing test in a 9-file Node CLI | **127.8k / 279.4k** (−54%) | 9 / 19 (−53%) | 67s / 167s (−60%) |
| Audit — 12 questions across a 132-file monorepo | **162.7k / 235.0k** (−31%) | 7 / 10 (−30%) | 92s / 120s (−23%) |
| Build — complete 8-page static website | ~−50% input tokens | fewer turns | — |

**Why it works.** Cost ≈ (bytes in context) × (turns remaining), so both factors are attacked at once: batched actions collapse turns, and hard caps keep each turn's residue small. Every line kept out of context is a line that is not re-billed by every later turn.

**Why earlier versions regressed.** A first draft won on build tasks but lost badly on audits (+72% tokens): it delegated a scan a single `rg` could answer, and re-opened files it had already found. That is what the per-job turn budgets, the "a search hit is already the answer" cap, and the delegation threshold (~2,000 lines / ~15 files) exist to prevent. The protocol is explicitly forbidden from making any job cost more than it would without it.

## Notes and honest limitations

- The skill merges **model/tool round trips**, not client-level HTTP requests: the harness decides whether parallel tool calls ship as one or several API calls.
- Caps are tuned for cost; on genuinely huge files use targeted `offset/limit` reads or shell extraction instead of whole-file reads.
- Density never overrides correctness — the protocol explicitly allows extra turns or larger reads when verification genuinely needs them.

## License

[MIT](LICENSE)
