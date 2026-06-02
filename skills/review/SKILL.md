---
name: review
description: Dispatches the reviewer agent on a branch. Diffs against main and emits Blocker/Nit findings. Triggers on "/review [branch]".
---

# /review

Dispatch the reviewer agent on a branch.

## Steps

### 1. Derive branch

If ARGUMENTS is provided, use it as the branch name.

If ARGUMENTS is empty, use current branch:
```bash
git branch --show-current
```

Set:
- `BRANCH` = ARGUMENTS or current branch

### 2. Dispatch reviewer

Dispatch subagent `agent-team:reviewer` with:
```
BRANCH: <BRANCH>
```

### 3. Report result

Print the full reviewer output as-is.
