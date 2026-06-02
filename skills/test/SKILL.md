---
name: test
description: Dispatches the tester agent. Runs go vet, go test -race -short, and golangci-lint. Triggers on "/test".
---

# /test

Dispatch the tester agent against the current working directory.

## Steps

### 1. Get working directory

```bash
pwd
```

Set:
- `WORKDIR` = output of `pwd`

### 2. Dispatch tester

Dispatch subagent `agent-team:tester` with:
```
WORKDIR: <WORKDIR>
```

### 3. Report result

- Contains `TEST_RESULT: PASS` → print:
  ```
  [tester] ✓ PASS
  ```
- Contains `TEST_RESULT: FAIL` → print:
  ```
  [tester] ✗ FAILED

  <full tester output>
  ```
