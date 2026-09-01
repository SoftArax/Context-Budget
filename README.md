# Context Budget

A [ZCode](https://zcode.dev) skill that minimizes **input-token cost** during coding, debugging, review, and investigation by packing many actions into each turn and hard-capping the bytes any tool may return — without skipping required correctness checks.

**A/B-tested on a full static-website build (8 pages, 9 assets, self-check script): ~50% fewer input tokens with the skill than without it.**

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
| The Turn Rule | Speculative chains: predict the whole action sequence, batch every independent call (2, 5, 10, 20 files) into one message; on the first surprise fall back to single-action mode. Forbidden between actions: status updates, restating the plan, thinking out loud. |
| Hard Output Caps | Reads ≤ ~150 lines (300 = defect), `rg -m 20` / `head -n 40` search, `git diff --stat` first, `2>&1 \| tail -n 30` by default, never full logs/lockfiles/minified/JSON dumps, no result over ~10k tokens. |
| Delegate Broad Exploration | If honest discovery would pull >~3 files / ~400 lines into the main thread, send a search subagent that burns its own throwaway context and returns a <300-word conclusion. Delegate facts, not content you will edit. |
| Complete Work In Your Head | Ship finished code in one pass — imports, error handling, tests in the same Write. One patch per file. Never re-read what you just wrote. Fold all fixes implied by new results into the next turn. |
| Verify Once, Quietly | One verification turn after all writes, capped output. Never re-run a passing command. |
| Session Hygiene | Reuse, never re-fetch. `path:line` references instead of quoted content. Two failed attempts = stop patching, re-derive from scratch. Offload long-horizon state to a notes file that survives compaction. |
| Non-Negotiables | Density never overrides correctness: no guessing file contents, no skipped tests. An under-read that forces a second read costs more than the right-sized read. |

## Installation

Copy the skill folder into your ZCode skills directory:

```bash
# user-level (available in every workspace)
cp -r Context-Budget ~/.zcode/skills/context-budget
```

or clone and copy:

```bash
git clone https://github.com/<your-username>/Context-Budget.git
cp -r Context-Budget ~/.zcode/skills/context-budget
```

The skill consists of:

- `SKILL.md` — the full protocol (name + description frontmatter, 9 sections);
- `agents/openai.yaml` — UI metadata (display name, short description, default prompt).

No dependencies, no scripts — it is pure instructions the model follows.

## A/B test methodology

The same large task — build a complete 8-page static analytics-SaaS website (HTML + CSS + vanilla JS, 4 stylesheets, 5 scripts, self-check `tools/check.py`, README) — was given to two identical agents in isolated folders: one with the skill loaded, one without. Metrics were computed from model-I/O transcripts (requests and token usage per API call), not from self-reports.

**Result: ~50% fewer input tokens with the skill.** Fewer turns (batched writes/searches instead of one-small-step-per-turn) and smaller per-turn results (capped reads, bounded search output) compound: every avoided line is re-billed by every later turn it would have survived into.

## Notes and honest limitations

- The skill merges **model/tool round trips**, not client-level HTTP requests: the harness decides whether parallel tool calls ship as one or several API calls.
- Caps are tuned for cost; on genuinely huge files use targeted `offset/limit` reads or shell extraction instead of whole-file reads.
- Density never overrides correctness — the protocol explicitly allows extra turns or larger reads when verification genuinely needs them.

## License

[MIT](LICENSE)
