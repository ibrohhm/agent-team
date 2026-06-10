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

### 3. Run implementer (initial)

Dispatch subagent `agent-team:implementer` with:
```
PLAN_PATH: <PLAN_PATH>
BRANCH: <BRANCH>
```

Check return:
- Contains `DONE:` → store commit info as `initial_commit_info`, proceed to step 4
- Contains `FAIL:` → **stop**. Report:
  ```
  [planner]     ✓
  [implementer] ✗ FAILED
  Reason: <everything after "FAIL: ">

  Branch <BRANCH> preserved with partial commits. Pick up from implementer.
  ```

### 4. Review-retry loop

Set `review_attempt = 1`, `max_attempts = 2`.

Track implementer results: start with list `["initial"]`.

**Loop:**

Dispatch subagent `agent-team:reviewer` with:
```
BRANCH: <BRANCH>
```

**If `REVIEW_RESULT: PASS`** → continue to step 5.

**If `REVIEW_RESULT: BLOCKED`:**

- If `review_attempt >= max_attempts` → **stop**. Report:
  ```
  [planner]     ✓
  [implementer] <implementer attempts summary>
  [reviewer]    ✗ BLOCKED after <review_attempt> attempt(s)

  <full reviewer output>

  Branch <BRANCH> preserved. Manual review required.
  ```
  Where `<implementer attempts summary>` is:
  - No retries yet: `✓ (initial)`
  - After one retry: `✓ (initial) + ✓ (retry 1)`

- Otherwise:
  - Extract the BLOCKERS lines from reviewer output
  - Dispatch subagent `agent-team:implementer` with:
    ```
    PLAN_PATH: <PLAN_PATH>
    BRANCH: <BRANCH>
    REVIEW_BLOCKERS:
    <paste BLOCKERS lines from reviewer output, one per line>
    ```
  - Check return:
    - Contains `DONE:` → append `"retry <review_attempt>"` to attempts list, increment `review_attempt`, **continue loop**
    - Contains `FAIL:` → **stop**. Report:
      ```
      [planner]     ✓
      [implementer] ✓ (initial) + ✗ FAILED (retry <review_attempt>)
      Reason: <everything after "FAIL: ">

      Branch <BRANCH> preserved with partial commits. Fix manually.
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

Build the implementer label from the attempts list:
- No retries (`["initial"]`): `✓ <initial_commit_info>`
- One retry (`["initial", "retry 1"]`): `✓ (initial) + ✓ (retry 1)`

Build the reviewer label:
- Passed on attempt 1: `✓ No blockers. <nit count> nit(s)`
- Passed after retry: `✓ (passed on attempt 2). <nit count> nit(s)`

```
Branch: <BRANCH>

[planner]     ✓ Plan written → <PLAN_PATH>
[implementer] <implementer label>
[reviewer]    <reviewer label>
[tester]      ✓ PASS

<nits section from reviewer output, if any>

Branch ready for human review.
```
