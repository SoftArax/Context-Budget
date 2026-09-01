---
name: context-budget
description: Minimize input-token cost and wall-clock time on any task — building, fixing, reviewing, or answering questions about code — by choosing a turn budget for the job, packing every independent action into one turn, and hard-capping the bytes any tool may return.
---

# Context Budget

<!-- version: 1.03 -->

## The Cost Model

The transcript is re-sent as input and re-billed on **every** turn. So one line pulled into context at turn 2 is paid for again at turns 3, 4, 5… and one extra turn re-bills everything accumulated so far.

Cost ≈ (bytes in context) × (turns remaining). Both factors are yours to control, and neither may be traded for the other:

- **Fewer turns** — every independent action goes in the same turn.
- **Fewer bytes per turn** — hard-cap what any tool returns.

Three things grow context permanently, since nothing leaves until compaction: tool results, your own messages (narration is re-billed forever), and instructions. A turn you saved by dumping 5,000 unneeded lines is a loss.

## Turn Budgets by Job Type

Classify the job silently on the first read of the request — never as a separate turn, never announced. Then hold to its budget of **tool turns** (turns containing tool calls):

| Job | Budget | Shape |
|---|---|---|
| **Fact lookup / quiz / "where is X"** | 1–2 | one batched search command answering *all* questions at once → answer |
| **Review / audit (no edits)** | 2–3 | batched search + capped reads of only the hit regions → findings |
| **Bugfix (failing test, few files)** | 3 | reproduce + read implicated files (one turn) → one patch turn → one verify turn |
| **Feature / refactor (many files)** | 4–6 | orient (batched) → design in your head → all writes in one turn → one verify turn |
| **Unknown huge codebase** | +1 | one delegated exploration, then the budget above |

Over budget means the plan was wrong, not that the budget was. Under budget is always allowed.

**Coverage outranks caps.** When the task is exhaustive by definition — "find all defects", "list every caller", "audit the module" — the set of files in scope is part of the requirement. Enumerate that set once (one bounded listing), and look at every file in it at least once. Skipping a file because it looks unused or unimportant is not economy, it is a wrong answer: caps limit *how much* you read from a file, never *whether* you read it.

**The no-regression rule.** This protocol may never make a job cost *more* than it would without it. If applying any rule here would add a turn, a read, or a check that the job does not need — skip that rule. Ceremony you skip costs nothing; ceremony you apply costs a turn plus its re-billing for the rest of the session. Read this document once; never re-read it.

## The Turn Rule: Maximum Actions, Zero Ceremony

A turn may contain many tool calls. A turn ends only when the next action **depends on this turn's results** — never because one item of a list is finished.

**Speculative chains.** Predict the whole action sequence up front, then fire it as batched calls. Keep chaining while results confirm the assumptions; on the first surprise — unexpected output, missing file, failed assumption — drop to single-action mode and re-plan. Never keep firing on stale assumptions.

- **Creating or editing files?** All of them in one message — 2, 5, 10, 20 Writes together; one patch per file. Never one file per turn.
- **Searching?** Every query you will need at once. For a question set, put *all* the symbols into one alternation: `rg -n "TAX_RATE|MAX_ATTEMPTS|SESSION_TTL|calcVat" -m 3` beats twelve searches by an order of magnitude. Definitions, callers, imports, and tests belong in the same turn.
- **Reading several files?** All reads in one turn, each capped.
- **Reproducing a bug?** The failing command and the reads of the files it names go in the same turn — the file list is predictable from the error.

Forbidden between actions: status updates ("now I'll…"), restating the plan, thinking out loud, and any call whose only purpose is to announce or confirm.

**Loop guard.** Repeating an equivalent action (same search, same read, same fix attempt) means the plan is wrong, not that repetition is needed. Re-derive.

## Hard Output Caps

Binding: a result larger than its cap is a defect even when the command "worked".

- **A search hit is already the answer.** `rg -n` returns path, line, and content — for a fact, a constant, or a location, that *is* the finding. Opening the file afterwards to "confirm" is a defect; cite `path:line` from the hit.
- **File reads:** ~150 lines per read. Take the region the planned edit needs plus minimal glue (signature, imports, immediate call sites). Over ~300 lines is a defect unless the whole file is genuinely being rewritten.
- **Search output:** always bounded — `rg -n -m 20`, or `| head -n 40`. Bare recursive grep across a repo is a defect.
- **Diffs:** `git diff --stat` first; unified diff (`--unified=2`) only for files you are about to edit.
- **Command output:** `2>&1 | tail -n 30` by default. Ask for failing lines, not the transcript.
- **Never into context:** full logs, build output, lockfiles, generated or minified files, listings over ~50 entries, binary/base64, JSON over ~100 lines (`jq` the fields instead). No single result over ~10k tokens, ever.

Token math: ~10 tokens per source line, so a 150-line read is ~1.5k tokens — re-billed on every later turn.

## Delegate Only Above the Threshold

A subagent re-bills its own system prompt, tool definitions, and instructions on every one of its requests — tens of thousands of tokens to hand back a few lines. It pays off only when it absorbs a genuinely large scan.

- **Delegate when** honest main-thread discovery would pull in more than ~2,000 lines or ~15 files, or the target is unknown (unfamiliar vocabulary, no idea which directory). Below that, batch bounded searches yourself — strictly cheaper.
- **Never** run two subagents on the same question, and never delegate what one `rg` answers.
- Delegation prompt states the question, the format, and the budget: "return paths with line numbers, a one-line role for each, and the direct answer — no code, no file dumps, under 300 words."
- Delegate **facts and locations**; do the work yourself when the result is **content you must edit** — a summary cannot replace the lines you have to patch.

## Finish The Thinking, Then Emit

Planning is free; turns are not. Before the first write, settle everything: final content of every file, imports, signatures, call sites, config, tests.

- **One pass per file.** Complete and compiling: imports, error handling, types, exports, and its tests in the same turn. Stubs, `TODO`s, and placeholders are scheduled future turns — defects.
- **Never re-read what you just wrote.** The tool result already confirmed it; re-reading re-adds the whole file and costs a turn.
- **When results change the plan** (real failure, missing dependency), fold *everything* the new information implies into the next turn — all fixes, all files, one turn. Fixing one symptom per turn is the most expensive habit there is.

## Verify Once

- **Edits:** one verification turn after all writes — narrowest relevant test, capped output. Never per file, never per hunk, never re-run something that already passed, never add unrequested smoke checks "for confidence".
- **Lookups and reviews:** the answer is the verification. Cross-check only a specific claim you genuinely doubt; re-confirming facts already in the transcript buys nothing.
- Broaden a check only after a real failure or a concrete cross-module dependency.

## Session Hygiene

- **Reuse, never re-fetch.** Everything already in the transcript is free; re-reading an unchanged file or repeating a successful search is a defect.
- **Hold references, not content.** Keep `path:line` pointers; load lines only when editing them.
- **Your text is context.** No narration between calls, no restated plans or pasted logs. Final answer once. Prefer `path:line` over quoting.
- **Two failed attempts = stop patching.** Re-derive the cause from a fresh angle (smallest repro) instead of attempt three on a polluted transcript.
- **Long sessions:** keep one notes/plan file updated in place (objective, decisions, changed files, pending steps). It survives compaction — afterwards re-read only it, never the old transcript.
- **Stay cache-friendly:** treat the conversation as append-only; avoid repeatedly re-editing the same file.
- **No ceremony.** Todo lists only on genuinely long jobs, outcome-level, updated in place. Ask a question only when the answer changes what you build.

## Non-Negotiables

Density never overrides correctness: do not guess file contents instead of reading them; do not skip required tests, type checks, or builds; do not suppress errors; do not shrink a read until you must guess — an under-read that forces a second read costs more than the right-sized read. When correctness truly needs another turn or a bigger read, take it, and pack everything else outstanding into that same turn.

## Response Format

Once, compact: outcome in 1–2 sentences, changed files as `path:line`, verification (command + pass/fail), blockers if any. Lookups: the answers, one line each, nothing else. No restated task, no narration of steps already taken, no code blocks unless the user must copy or inspect them.
