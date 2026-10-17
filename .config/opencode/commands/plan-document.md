---
description: Write a parallel-agent implementation plan to docs/
agent: build
---

Write an implementation plan for the work in this conversation to `docs/<slug>.md`, where slug is $1 (if $1 is empty, derive a short kebab-case slug from the work). Create `docs/` if it is missing.

Do NOT implement the plan. Do NOT make code changes. The only artifact is that markdown file; exploration is read-only.

Several agents will execute this plan in parallel and none of them saw this conversation. They must never have to guess what to do, where to stop, or what success looks like. Write dense and unambiguous: no fluff, no motivation, no restating the obvious.

## Required structure

`# <title>` then, on the next lines, repository, branch, and the objective in 1-3 lines.

`## Shared context` - what every task needs and would otherwise re-discover: relevant paths, existing patterns to follow (cite the file the pattern comes from), build/test/lint commands, and known traps.

`## Tasks` - one `### Task <id>: <name>` per task, each with:
- **Goal**: one sentence.
- **Depends on**: task ids, or "none".
- **Files**: exact paths this task owns. No other task may touch them.
- **Steps**: ordered, concrete edits or commands. Name real files, symbols, and types; never "implement X".
- **Done when**: observable criteria plus the exact command that proves it.
- **Out of scope**: what a correct implementation must not do.

`## Integration` - verification after all tasks land: the full build/test command and the checks that prove the objective is met.

`## Parallelism` - which tasks can run simultaneously, which are serialized, and which files are shared or hot.

## Rules
- Every step must be actionable without re-reading this conversation.
- Prefer many small tasks with disjoint file ownership over a few large ones.
- A task that needs a decision only I can make gets a `## Open questions` entry and a **Blocked** marker. Do not invent the answer.
- Keep it as short as it can be while staying unambiguous.
