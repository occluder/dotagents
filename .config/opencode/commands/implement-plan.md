---
description: Implement a plan file (parallelize independent tasks)
agent: build
---

Implement the plan in $ARGUMENTS.

## Before you start
- If $ARGUMENTS is empty or the file does not exist, stop and say so. Do not invent a plan or start coding.
- Read the whole plan before editing anything. If it contradicts the codebase (missing files, renamed symbols, dead assumptions), stop and report the conflict instead of improvising a new design.
- Reply with the task list, ids and dependency order, so I can confirm your reading before you touch code.

## Execution
- Go task by task in the plan's dependency order. Stay inside the files and scopes a task claims: no refactoring of adjacent code, no renaming public APIs, no drive-by fixes.
- Delegate independent tasks to parallel subagents, one per task, each given the plan path, its own task section verbatim (including "done when"), and its file ownership. Run dependent tasks only after their prerequisites pass verification.
- Shared or hot files belong to a single task or run in a fixed order; never let two agents edit the same file.
- Honor each task's stop condition. When "done when" is satisfied, stop, do not continue into polish.

## Verification
- After each task, run the verification the plan specifies (build, tests, lint). A task is not done until that passes.
- After the last task, run the repo's full build and test suite and check the integrated result against the plan's acceptance criteria.
- Fix failures your changes caused. Never disable, skip, or delete a test to get green. If a failure predates this work or needs a decision, report it.

## Scope rules
- Do not commit, push, or open a PR unless the plan or I ask for it.
- Do not add dependencies, config, or migrations the plan does not mention.
- If the plan is silent on something you must decide to proceed, stop and ask. Do not pick an approach silently.

## Report
Finish with: task id -> done or blocked, files changed per task, the exact verification command and its result, deviations from the plan, and anything left undone.
