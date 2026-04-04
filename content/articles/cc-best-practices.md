---
title: Claude Code Best Practices
draft: false
tags:
---

Best practices for getting the most out of Claude Code, based on a deep analysis of every HTTP call it makes during a session, captured via mitmproxy. See [[cc-api-analysis|Claude Code API Call Analysis]] for the full breakdown.


## High-Level Workflow

### Implementing a feature with minimal back-and-forth

For any medium-complexity feature, ask Claude to plan before it codes:

> "Before writing any code, give me a step-by-step implementation plan."

Review the plan, correct misunderstandings, then say "go ahead." This catches wrong assumptions at the cheapest point — before any code exists. Fixing a plan takes seconds; fixing a half-built implementation costs turns.

Also ask for tests alongside the implementation:

> "Implement X and write tests that verify the acceptance criteria."

Tests give Claude a self-check mechanism and give you a way to verify correctness without reading every line.

When done, review `git diff` rather than the conversation. Point at specific code if something's off, rather than re-explaining the feature.

The failure mode to avoid: vague prompt → partial implementation → mid-task correction → Claude loses original intent → inconsistent result. The plan step eliminates most of this.

## Best Practices

### Give precise, targeted prompts

The difference between:

> "edit the title of the HTML page"

and:

> "Edit `<title>test 2</title>` to `<title>test 3</title>` in `src/index.html`"

is one fewer API call — Glob gets eliminated. Providing the file path skips the search turn. Providing the exact old and new strings may skip the Read turn too. Every skipped turn is ~16k fewer tokens against your limit.

### Stay within the 1-hour prompt cache window

The system prompt — all ~4,000 words of behavior rules, tool definitions, memory instructions, and environment context — is cached globally with a **1-hour TTL**. When the cache is warm, each turn costs ~1 uncached input token. When cold, you pay to re-ingest the full prompt.

Back-to-back sessions within an hour are dramatically cheaper than sessions spaced hours apart. Consolidating work into one continuous session beats spreading it across multiple short ones.

### Batch tasks into a single session

The 6 startup requests, the Haiku quota probe, and the Haiku title generation run **once per session**, not once per task. Ten tasks in one session costs far less than ten one-task sessions. Keep a session open while working on related tasks rather than closing and reopening between each one.

### Use `/compact` before context grows large

Auto-compaction is disabled by default (`tengu_sm_compact: false`). Past roughly 150k tokens of context, each turn's cost climbs fast. Running `/compact` manually resets the rolling context and keeps per-turn costs flat. The token counter in the status bar (populated from the startup quota probe headers) shows where you stand.

### Plan work around the session reset window

When you hit the usage limit, the status bar shows the exact "resets at" time. Use that window intentionally: queue up the next batch of tasks, review completed work, or switch to tasks that don't need Claude. Going in with a prepared list means you start the next window immediately productive rather than figuring out what to do next.

### Use `claude -p` for scripted work

Non-interactive mode (`claude -p`) skips the 1-token Haiku quota probe on startup — the first real API call provides the same rate-limit header info anyway. For batch scripts or automation, this is a small but free saving.

### Avoid Fast Mode unless speed is critical

Fast Mode ("Penguin Mode" internally) uses the **same Opus 4.6 model** — not a different one. It's faster, but billed at $30/$150 per million tokens (input/output) as extra usage, outside your plan's included allocation, from the first token. If you're hitting the session limit, Fast Mode burns through your budget faster without changing the underlying token consumption.


## Summary

| Practice | Impact |
|----------|--------|
| Stay within 1-hour prompt cache window | High — ~4k uncached tokens saved per turn |
| Give precise file paths and edit strings | High — eliminates 1-2 tool-use turns per task |
| Batch multiple tasks in one session | High — amortizes 6 fixed startup requests |
| Plan tasks for the next reset window | Medium — eliminates dead time between sessions |
| Use `/compact` before hitting 150k context | Medium — prevents exponential per-turn growth |
| Use `claude -p` for automation | Low — skips ~9-token quota probe |
| Avoid Fast Mode unless needed | Situational — increases cost rate significantly |

The compounding case: a 3-turn task within the cache window costs roughly **3-4x less** than a 4-turn task outside it. Precise prompts and staying in the cache window together are the most effective combination.
