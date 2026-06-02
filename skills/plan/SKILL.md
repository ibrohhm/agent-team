---
name: plan
description: Dispatches the planner agent on a task description. Explores the codebase and writes an implementation plan. Triggers on "/plan <task description>".
---

# /plan

Dispatch the planner agent for a single task description.

## Steps

### 1. Derive variables

Get working directory and date:
```bash
pwd
date +%Y-%m-%d
```

From ARGUMENTS (free-text after `/plan`):
- Lowercase, replace spaces/special chars with hyphens
- Strip non-alphanumeric/hyphen chars
- Truncate to 30 chars, remove trailing hyphens

Set:
- `WORKDIR` = output of `pwd`
- `DATE` = output of `date +%Y-%m-%d`
- `SLUG` = derived from ARGUMENTS
- `PLAN_PATH` = `<WORKDIR>/docs/superpowers/plans/<DATE>-<SLUG>.md`

Create plan directory if needed:
```bash
mkdir -p <WORKDIR>/docs/superpowers/plans
```

### 2. Dispatch planner

Dispatch subagent `agent-team:planner` with:
```
TASK: <original ARGUMENTS>
PLAN_PATH: <PLAN_PATH>
WORKDIR: <WORKDIR>
```

### 3. Report result

- Contains `PLAN_WRITTEN:` → print:
  ```
  [planner] ✓ Plan written → <PLAN_PATH>
  ```
- Contains `FAIL:` → print:
  ```
  [planner] ✗ FAILED
  Reason: <everything after "FAIL: ">
  ```
