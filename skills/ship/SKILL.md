---
name: ship
description: Orchestrates a planner → implementer → reviewer → tester pipeline from free-text. Creates a branch, plans, implements, reviews, and tests — all before human touch. Triggers on "/ship <task description>".
---

# /ship

Run the full autonomous ship pipeline from a free-text task description.

## Steps

### 1. Derive slug and create branch

From ARGUMENTS (the free-text after `/ship`):
- Lowercase the text
- Replace spaces and special characters with hyphens
- Strip all characters that are not alphanumeric or hyphens
- Truncate to 30 characters
- Remove trailing hyphens

Example: `"Add payment retry logic for DANA"` → `add-payment-retry-logic-for-d`

Run:
```bash
git checkout -b feat/<slug>
```

Get today's date:
```bash
date +%Y-%m-%d
```

Set these variables (used in subsequent steps):
- `BRANCH` = `feat/<slug>`
- `PLAN_PATH` = `<repo-root>/docs/superpowers/plans/<date>-<slug>.md`
- `WORKDIR` = output of `pwd`

Create the plan directory if it does not exist:
```bash
mkdir -p <repo-root>/docs/superpowers/plans
```

### 2. Run planner

Dispatch subagent `agent-team:planner` with this exact message body:
```
TASK: <original free-text from ARGUMENTS>
PLAN_PATH: <PLAN_PATH>
WORKDIR: <WORKDIR>
```

Check the agent's return message:
- If it contains `PLAN_WRITTEN:` → extract the path and continue to step 3
- If it contains `FAIL:` → **stop**. Report to user:
  ```
  [planner] ✗ FAILED
  Reason: <everything after "FAIL: ">

  Branch feat/<slug> preserved. Pick up from planner.
  ```

### 3. Run implementer

Dispatch subagent `agent-team:implementer` with:
```
PLAN_PATH: <PLAN_PATH>
BRANCH: <BRANCH>
```

Check return:
- Contains `DONE:` → extract commit count, continue to step 4
- Contains `FAIL:` → **stop**. Report:
  ```
  [planner]     ✓
  [implementer] ✗ FAILED
  Reason: <everything after "FAIL: ">

  Branch <BRANCH> preserved with partial commits. Pick up from implementer.
  ```

### 4. Run reviewer

Dispatch subagent `agent-team:reviewer` with:
```
BRANCH: <BRANCH>
```

Check return:
- Contains `REVIEW_RESULT: PASS` → continue to step 5
- Contains `REVIEW_RESULT: BLOCKED` → **stop**. Report:
  ```
  [planner]     ✓
  [implementer] ✓
  [reviewer]    ✗ BLOCKED

  <full reviewer output>

  Branch <BRANCH> preserved. Resolve blockers before testing.
  ```

### 5. Run tester

Dispatch subagent `agent-team:tester` with:
```
WORKDIR: <WORKDIR>
```

Check return:
- Contains `TEST_RESULT: PASS` → continue to step 6
- Contains `TEST_RESULT: FAIL` → **stop**. Report:
  ```
  [planner]     ✓
  [implementer] ✓
  [reviewer]    ✓
  [tester]      ✗ FAILED

  <full tester output>

  Branch <BRANCH> preserved. Fix failing tests.
  ```

### 6. Print success summary

```
Branch: <BRANCH>

[planner]     ✓ Plan written → <PLAN_PATH>
[implementer] ✓ <commit info from DONE: message>
[reviewer]    ✓ No blockers. <nit count> nit(s)
[tester]      ✓ PASS

<nits section from reviewer output, if any>

Branch ready for human review.
```
