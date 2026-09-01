---
name: context-budget
description: Minimize the number of API requests and input-token cost on any task — building, fixing, reviewing, or answering questions about code — by executing the whole job in two mega-batched requests with hard caps on what any single call may return.
---

# Context Budget

<!-- version: 1.04 -->

## The Unit of Cost Is the Request

Every API request re-sends the entire transcript and re-bills it. Cost ≈ transcript size × number of requests. Ten tool calls inside one request cost one billing round; the same ten calls spread over ten requests cost ten rounds — plus nine re-billings of everything each round accumulated. Request count is the primary metric. Seconds and individual call bytes are secondary.

So the default shape of a task is **two requests**:

- **Request 1 — reconnaissance mega-batch.** Everything whose inputs you already know: the failing test run, every search, every read — including speculative reads of files you will probably edit, since a speculative capped read is far cheaper than a second request.
- **Request 2 — execution mega-batch.** The complete fix/design planned during request 1: all Writes/Edits, then the verification command in the same request — the harness executes batched calls in order, so the test sees the just-written files. Done.

A third request exists only as contingency: request 1's results genuinely surprised you, or verification failed. Re-planning on surprise is the design, not a failure; drifting into request 4+ means the plan, not the task, is wrong.

**Request budgets** (one request = one assistant message with any number of tool calls):

| Job | Budget |
|---|---|
| Fact lookup / quiz / "where is X" | 1–2 |
| Bugfix (failing test, few files) | 2–3 |
| Review / audit (no edits) | 2 |
| Feature / refactor / build from spec | 2–4 |
| Unknown huge codebase | 3–4 (one delegated exploration counts once) |

Never issue a request that contains a single call unless it is the final one and the job is otherwise complete.

**The no-regression rule.** This protocol may never make a job cost more than it would without it, and never cost correctness. If a rule here would add a request, a read, or a check the job does not need — skip that rule. Read this document once; never re-read it.

## One Call, One Whole Job

A call you did not issue costs nothing, and a call merged into an existing request costs almost nothing. Merge ruthlessly:

- **Chained shell beats separate calls.** `node test/a && node test/b && node test/c` is one call, not three. `mkdir -p` before writes, `rg` feeding `sort`/`head` — one pipeline, one call.
- **One alternation for all facts.** For any question set, put every symbol in a single `rg -n "TAX_RATE|MAX_ATTEMPTS|SESSION_TTL|calcVat" -m 3` instead of one search per question. The hit — path, line, content — is already the answer; re-opening the file to "confirm" is a defect. Cite `path:line` from the hit.
- **Whole-scope read for small scopes.** If everything in scope totals under ~1,000 lines, skip per-file reads entirely: one bounded call (`cat lib/*.js bin/*.js`) covers the entire surface — full coverage by construction, one request. For bigger scopes, read the hit regions of all relevant files in one parallel batch.
- **All files at once.** 2, 5, 10, 20 Writes in one message; one patch per file; never one file per request.

## Hard Output Caps

Binding: a result larger than its cap is a defect even when the command "worked".

- **File reads:** ~150 lines per read; over ~300 is a defect unless the whole file is genuinely being rewritten. Speculative reads in request 1 follow the same cap — they are insurance against request 3, not a license to dump.
- **Search output:** `rg -n -m 20`, or `| head -n 40`. Bare recursive grep across a repo is a defect.
- **Command output:** `2>&1 | tail -n 30` by default — failing lines, not transcripts.
- **Never into context:** full logs, build output, lockfiles, generated/minified files, listings over ~50 entries, binary/base64, JSON over ~100 lines (`jq` the fields). No single result over ~10k tokens, ever.

**Coverage outranks caps.** When the task is exhaustive by definition — "find all defects", "list every caller" — the file set in scope is part of the requirement. Enumerate it once, look at every file in it at least once, and prefer the whole-scope read so coverage costs one call. Caps limit *how much* you read from a file, never *whether* you read it.

## Delegate Only Above the Threshold

A subagent re-bills its own system prompt and instructions on every one of its requests — tens of thousands of tokens to hand back a few lines. Delegate only when honest main-thread discovery would pull in more than ~2,000 lines or ~15 files, or the target is unknown (unfamiliar vocabulary, no candidate directory). Never two subagents on one question; never delegate what one `rg` answers. The delegation prompt states the question, the format, and the budget: "paths with line numbers, one-line role of each, the direct answer — no code, under 300 words." Delegate facts and locations; do the work yourself when the result is content you must edit.

## Finish The Thinking, Then Emit

Planning is free; requests are not. While request 1's results stream in, decide everything: the final content of every file, imports, signatures, call sites, config, tests — so request 2 executes the finished design in one pass.

- **One pass per file**, complete and compiling: imports, error handling, types, exports, tests in the same Write. Stubs and `TODO`s are scheduled future requests — defects.
- **Never re-read what you just wrote**; the tool result already confirmed it.
- **On surprise or failed verification**, fold everything the new information implies into one next request — all fixes, all files, plus its own verification. Fixing one symptom per request is the most expensive habit there is.

## Session Hygiene

- **Reuse, never re-fetch.** Everything already in the transcript is free; re-reading an unchanged file or repeating a successful search is a defect.
- **Hold references, not content** (`path:line`); load lines only when editing them.
- **Your text is context.** No narration between calls, no restated plans or pasted logs. Final answer once. Prefer `path:line` over quoting.
- **Two failed attempts = stop patching.** Re-derive from a fresh angle instead of attempt three on a polluted transcript.
- **Long sessions:** one notes/plan file updated in place survives compaction — afterwards re-read only it.
- **No ceremony.** Todo lists only on genuinely long jobs, updated in place. Ask a question only when the answer changes what you build.

## Non-Negotiables

Density never overrides correctness: do not guess file contents instead of reading them; do not skip required tests, type checks, or builds; do not suppress errors; do not shrink a read until you must guess — an under-read that forces an extra request costs more than the right-sized read. When correctness truly needs another request or a bigger read, take it, and pack everything else outstanding into that same request.

## Response Format

Once, compact: outcome in 1–2 sentences, changed files as `path:line`, verification (command + pass/fail), blockers if any. Lookups: the answers, one line each, nothing else. No restated task, no narration of steps already taken, no code blocks unless the user must copy or inspect them.
