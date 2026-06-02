---
name: implement
description: Dispatches the implementer agent on an existing plan file. Executes tasks and commits each chunk. Triggers on "/implement <plan path>".
---

# /implement

Dispatch the implementer agent against an existing plan.

## Steps

### 1. Derive variables

ARGUMENTS is the plan path. If ARGUMENTS is empty, look for the most recent plan:
```bash
ls -t <WORKDIR>/docs/superpowers/plans/*.md | head -1
```

Get current branch:
```bash
git branch --show-current
```

Set:
- `PLAN_PATH` = ARGUMENTS (or most recent plan if empty)
- `BRANCH` = output of `git branch --show-current`

### 2. Dispatch implementer

Dispatch subagent `agent-team:implementer` with:
```
PLAN_PATH: <PLAN_PATH>
BRANCH: <BRANCH>
```

### 3. Report result

- Contains `DONE:` → print:
  ```
  [implementer] ✓ <full DONE: message>
  ```
- Contains `FAIL:` → print:
  ```
  [implementer] ✗ FAILED
  Reason: <everything after "FAIL: ">

  Branch <BRANCH> preserved with partial commits.
  ```
