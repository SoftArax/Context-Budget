---
name: context-budget
description: Minimize input-token cost across task types — coding, debugging, review, investigation — by classifying the job once, packing many actions into each turn, and hard-capping the bytes any tool may return, without skipping required correctness checks.
---

# Context Budget

## Why Input Tokens Explode

The transcript is re-sent as input on every turn and re-billed. Three things grow it permanently, because nothing is ever removed until compaction:

1. **Tool results** — every file read, every command's output stays in context for the rest of the session.
2. **Your own messages** — every narration line, restated plan, pasted snippet, and verbose answer is re-billed on every later turn.
3. **The instructions themselves** — one more reason to never pad this protocol with ceremony.

Extra turns and extra bytes compound: each new turn re-bills everything already in context, and each bloated result is re-billed by every later turn. So both levers are always mandatory:

- **Fewer turns** — pack as many independent actions as possible into each turn.
- **Fewer bytes per turn** — hard-cap what any tool may return.

A turn you saved by stuffing 5,000 unnecessary lines into context is a loss, not a win. Never trade one lever for the other.

## Step 0: Classify the Job, Then Apply Its Profile

Classify once, mentally, before the first action — never as an extra turn or an announced plan:

- **Build/edit session** (new feature, refactor, many files): full protocol — batched writes, caps, delegation, one verification turn.
- **Bugfix** (few files, a failing test): one search turn → one fix turn → one verify turn. Nothing in between.
- **Lookup / quiz / review-only** (questions, facts, report): batched search turns, answer once, stop. No edits, no notes file, no verification pass. Never one question per turn: if one grep's result answers four of twelve questions, claim all four and move on. Never re-verify a fact the transcript already contains.
- **Vast unknown codebase**: same as lookup, delegation becomes likely (see threshold).

The protocol never adds turns to a small job. Sections that don't fit the profile are skipped, not applied "just in case" — a skipped ceremony costs zero tokens; an applied one costs a turn plus its re-billing for the rest of the session.

## The Turn Rule: Maximum Actions, Zero Ceremony

A turn is one assistant message, and it may contain many tool calls. A turn ends only when the next action **depends on this turn's results** — never when one item of a list is done.

**Speculative chains.** For any task with predictable steps, silently predict the whole action sequence up front, then execute the chain as batched calls. Continue the chain while each next action is confirmed by previous results; on the first surprise — an unexpected result, a failed assumption, a missing file — fall back to single-action mode and re-plan. Never keep firing batched actions on stale assumptions.

- **Creating files?** All of them, this turn: 2, 5, 10, 20 files — every Write issued together in one message. Never one file per turn, never a pause to "check" a file you just wrote.
- **Searching?** Every query you will need in one turn: all candidate spellings, all directories, definitions plus callers plus imports plus tests — as one chained shell command or one parallel batch.
- **Editing N files?** All N in one turn, one patch per file, issued together. Ten small scripts = ten Write calls in one message, not ten turns.
- **Reading several files?** All reads in one turn, each capped.
- **Answering a question set?** Group questions by likely location; one batched search per location, not per question.

Explicitly forbidden between actions: status updates ("now I'll..."), restating the plan, thinking-out-loud messages, and any tool call whose only purpose is to announce or confirm.

**Loop guard.** If you catch yourself repeating an equivalent action (same search, same read, same fix attempt), stop: the plan is wrong, not the repetition. Re-derive the approach.

## Hard Output Caps

These caps are binding. A result larger than its cap is a defect even when the command "worked".

- **File reads:** default ceiling ~150 lines per read. Take the exact region the planned edit needs plus minimal glue (signature, imports, immediate call sites). Above ~300 lines is a defect — narrow the range or justify it because the whole file is genuinely being rewritten.
- **Search output:** always bounded — `rg -n` with `-m 20`, or pipe through `head -n 40`. A bare recursive grep across a repository is a defect.
- **Diffs:** `git diff --stat` first; a unified diff only for the few files you are about to edit, `--unified=2`.
- **Command output:** append `2>&1 | tail -n 30` by default. Ask for the failing lines, not the transcript.
- **Never into context:** full logs, build output, lockfiles, generated or minified files, directory listings over ~50 entries, binary/base64 data, JSON dumps over ~100 lines (select fields with `jq` instead). No single result over ~10k tokens, ever.

Rough token math: one source line is ~10 tokens, so the 150-line read cap is ~1.5k tokens — re-billed on every later turn.

## Delegate Broad Exploration — With a Real Threshold

A subagent is not free: its own system prompt, tool definitions, and instructions are re-billed on every one of its requests, often tens of thousands of tokens for a handful of lines back. Delegation is a trade, and the trade has a floor.

- **Delegate only when** honest main-thread discovery would pull more than ~2,000 lines or ~15 files into context, or the search is open-ended (unknown vocabulary, unknown location, large repo). Below that floor, batch bounded reads yourself — several capped Read calls or one chained `rg` — it is strictly cheaper.
- **Never parallel-delegate for the same question.** Two subagents on one question re-scan the same ground and multiply overhead. Parallel agents only when their targets provably do not overlap.
- The delegation prompt states the exact question, the answer format, and the budget: "return: paths with line numbers, one-line role of each, and the direct answer — no code, no file dumps, under 300 words."
- Delegate when the result is a **fact or location**. Do the work yourself when the result is **content you will edit** — a summary cannot substitute for the actual lines you must patch.

## Complete Work In Your Head, Then Emit

(Build/edit profile.) Planning is free; intermediate turns are not. Before the first write call, decide everything: every file's final content, imports, signatures, call sites, config, tests. Do not start writing to "see how it looks", and do not write half a design to discover the rest.

- **Ship finished code in one pass.** A file is written once, complete and compiling: imports, error handling, types, exports, and its tests in the same turn. Stubs, `TODO`s, and placeholders are defects — they are scheduled future turns.
- **One patch per file.** Collect all changes to a file into a single Edit/Write. Returning to a file you already finished for a predictable follow-up is a defect; the follow-up belonged in the first patch.
- **Do not re-read what you just wrote.** The tool result already confirmed success. Re-reading re-adds the whole file to context and costs a turn.
- **When results change the plan** (a real failed test, a missing dependency), fold everything the new information implies into the next turn: all fixes, all files, one turn. Fixing one symptom per turn is the most expensive habit in this document.

## Verify Once, Quietly

- One verification turn after all writes: focused diff + narrowest relevant test in one command. Never per file, never per hunk, never re-run a command that already passed.
- Cap verification output too: failures and their surrounding lines, not full logs.
- **Lookup/review-only profile:** the final answer is the verification. One cross-check grep only for an answer you are genuinely unsure of; checking every answer re-bills certainty you already own.
- Do not broaden a passing check "for confidence". Broaden only after a failure or a concrete cross-module dependency.

## Session Hygiene

- **Reuse, never re-fetch.** Anything already in the transcript is free. Re-reading an unchanged file or repeating a successful search is a defect.
- **Hold references, not content.** Store facts as `path:line` pointers and load the actual lines only at the moment of editing.
- **Your text is context too.** Billed on every later turn. No narration between calls, no restating requests/plans/code/logs, final answer once: outcome, changed files, verification, blockers. Prefer `path:line` over quoting.
- **Two failed attempts = stop patching.** After two corrections of the same problem fail, the transcript is polluted with failed approaches; re-derive the cause from scratch (fresh look, smallest repro) instead of piling on attempt three.
- **Offload long-horizon state to disk.** For tasks spanning many turns, maintain one notes/plan file updated in place (objective, decisions, changed files, pending steps). It survives compaction; after compaction re-read only that file, never the old transcript. Before compaction, ensure the checkpoint preserves the full list of changed files and the exact verification commands.
- **Stay cache-friendly.** Treat the conversation as append-only: avoid re-editing the same file or instructions repeatedly; never let a single tool result exceed ~10k tokens.
- **No ceremony.** No todo lists on small tasks; on long ones a few outcome-level items updated in place. Ask a question only when the answer changes the implementation.

## Non-Negotiables

Density never overrides correctness: do not guess file contents instead of reading; do not skip required tests, type checks, or builds; do not suppress errors; do not shrink a read so far that you must guess — an under-read that forces a second read costs more than the right-sized read would have. When correctness genuinely needs another turn or a larger read, take it — and pack everything else outstanding into that same turn.

## Response Format

Final answer only, once, compact: what was done (1–2 sentences), changed files as `path:line` list, verification result (command + pass/fail), blockers or open questions if any. Lookup tasks: answers only, one line each. No restated task, no narration of steps already taken, no code blocks unless the user must copy or inspect them.
