---
name: ship
description: Orchestrates a clarifier → planner → implementer → reviewer → tester pipeline from free-text. Clarifies scope, creates a branch, plans, implements, reviews, and tests — all before human touch. Triggers on "/ship <task description>".
---

# /ship

Run the full autonomous ship pipeline from a free-text task description.

## Steps

### 0. Clarify task scope

```bash
pwd
```

Dispatch subagent `agent-team:clarifier` with:
```
TASK: <original free-text from ARGUMENTS>
WORKDIR: <output of pwd>
```

Check return:
- Contains `CLEAR` → proceed to step 1
- Contains `CLARIFY:` → ask user with AskUserQuestion using the question after `CLARIFY:`, then append the answer to ARGUMENTS: `<original ARGUMENTS>. <answer>`. Proceed to step 1 with enriched ARGUMENTS.

### 1. Derive slug, infer type, and create branch

From ARGUMENTS (the free-text after `/ship`):
- Lowercase the text
- Replace spaces and special characters with hyphens
- Strip all characters that are not alphanumeric or hyphens
- Truncate to 30 characters
- Remove trailing hyphens

Example: `"Add payment retry logic for DANA"` → `add-payment-retry-logic-for-d`

Infer TYPE from ARGUMENTS (first match wins, case-insensitive):
- Contains "fix", "bug", "repair", "resolve", "broken", "crash" → `TYPE=fix`
- Contains "refactor", "cleanup", "clean up", "restructure" → `TYPE=refactor`
- Contains "test", "spec", "coverage" → `TYPE=test`
- Contains "chore", "bump", "upgrade", "update dep" → `TYPE=chore`
- Default → `TYPE=feat`

Get today's date:
```bash
date +%Y-%m-%d
```

Set these variables (used in subsequent steps):
- `BRANCH` = `<type>/<slug>`
- `PLAN_PATH` = `<repo-root>/docs/superpowers/plans/<date>-<slug>.md`
- `WORKDIR` = output of `pwd`

Check for uncommitted changes:
```bash
git status --porcelain
```
If output is non-empty → **stop**:
```
Cannot start: working tree has uncommitted changes. Commit or stash first.
```

Check if branch already exists:
```bash
git branch --list <type>/<slug>
```
If non-empty:
- Look for existing plan:
  ```bash
  ls <repo-root>/docs/superpowers/plans/*<slug>.md 2>/dev/null | tail -1
  ```
- If plan file found → set PLAN_PATH to that file, print `Resuming: branch <BRANCH>, plan <PLAN_PATH>`, **skip to step 3**
- If no plan file → **stop**:
  ```
  Branch <BRANCH> exists but no plan found. Delete the branch or add a plan file to resume.
  ```

Create branch:
```bash
git checkout -b <type>/<slug>
```

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
- If it contains `AMBIGUOUS:` → run `git checkout main && git branch -D <BRANCH>`, then **stop**:
  ```
  [planner] ✗ AMBIGUOUS
  Question: <everything after "AMBIGUOUS: ">

  Branch deleted. Clarify and re-run /ship.
  ```
- If it contains `FAIL:` → run `git checkout main && git branch -D <BRANCH>`, then **stop**:
  ```
  [planner] ✗ FAILED
  Reason: <everything after "FAIL: ">

  Branch deleted.
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
  - Extract blocker lines: take all lines starting with `- ` that appear after `BLOCKERS:` and before `NITS:` (or end of output) in the reviewer output
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
