---
name: product-design
description: Two-agent pipeline for product design. Runs product-manager agent (conversational PRD) then ux-designer agent (conversational UX doc). Generates <app-name>-prd.md and <app-name>-ux.md in the current working directory. Triggers on "/product-design <app idea>".
---

# /product-design

Two-stage pipeline: PM first → UX second. Both conversational. Both interruptible.

## Steps

### 1. Derive variables

From ARGUMENTS (the free-text app idea):

Derive `APP_NAME`:
- Take the first 2 meaningful words, lowercase, join with hyphens
- Strip stop words: "for", "a", "an", "the", "of", "to"
- Examples:
  - "cashier app for small warung" → `cashier-app`
  - "inventory management system" → `inventory-management`
  - "employee attendance tracker" → `employee-attendance`
  - "calculator" → `calculator`

Run:
```bash
pwd
```

Set:
- `CWD` = output of `pwd`
- `PRD_PATH` = `<CWD>/<APP_NAME>-prd.md`
- `UX_PATH` = `<CWD>/<APP_NAME>-ux.md`

---

### 2. PM — Generate questions

Dispatch subagent `agent-team:product-manager` with:
```
APP_IDEA: <ARGUMENTS>
OUTPUT_PATH: <PRD_PATH>
```

If output contains `FAIL:` — print reason and stop.

Extract numbered questions from `QUESTIONS:` output.

---

### 3. PM — Ask questions

Ask each question to the user one at a time. Wait for each answer before asking the next.

If user says `stop` or `done` at any point — print:
```
[product-design] PM phase stopped. No PRD written.
```
Stop.

Build `PM_CONTEXT`:
```
Q: <question 1>
A: <user answer 1>

Q: <question 2>
A: <user answer 2>
...
```

---

### 4. PM — Write PRD

Dispatch subagent `agent-team:product-manager` with:
```
APP_IDEA: <ARGUMENTS>
OUTPUT_PATH: <PRD_PATH>
CONTEXT: <PM_CONTEXT>
```

If output contains `FAIL:` — print reason and stop.

Extract path from `PRD_WRITTEN:`. Print:
```
[product-design] ✓ PRD written to <path>
```

---

### 5. Offer UX phase

Print:
```
PRD complete. Continue to UX design phase? (yes / stop)
```

Wait for user response. If user says `stop`, `no`, or `done` — print:
```
[product-design] Done.
  PRD: <PRD_PATH>
```
Stop.

---

### 6. UX — Generate questions

Dispatch subagent `agent-team:ux-designer` with:
```
PRD_PATH: <PRD_PATH>
OUTPUT_PATH: <UX_PATH>
```

If output contains `FAIL:` — print reason and stop.

Extract numbered questions from `QUESTIONS:` output.

---

### 7. UX — Ask questions

Ask each question to the user one at a time. Wait for each answer before asking the next.

If user says `stop` or `done` at any point — print:
```
[product-design] UX phase stopped. No UX doc written.
  PRD: <PRD_PATH>
```
Stop.

Build `UX_CONTEXT`:
```
Q: <question 1>
A: <user answer 1>

Q: <question 2>
A: <user answer 2>
...
```

---

### 8. UX — Write UX doc

Dispatch subagent `agent-team:ux-designer` with:
```
PRD_PATH: <PRD_PATH>
OUTPUT_PATH: <UX_PATH>
CONTEXT: <UX_CONTEXT>
```

If output contains `FAIL:` — print reason and stop.

Extract path from `UX_WRITTEN:`. Print:
```
[product-design] ✓ UX doc written to <path>
```

---

### 9. Offer backlog

Print:
```
Generate backlog? (yes / stop)
```

Wait for user response. If user says `stop`, `no`, or `done` — print:
```
[product-design] Done.
  PRD: <PRD_PATH>
  UX:  <UX_PATH>
```
Stop.

---

### 10. Dispatch backlog-generator

Set `BACKLOG_PATH` = `<CWD>/<APP_NAME>-backlog.md`

Dispatch subagent `agent-team:backlog-generator` with:
```
PRD_PATH: <PRD_PATH>
UX_PATH: <UX_PATH>
OUTPUT_PATH: <BACKLOG_PATH>
```

If output contains `FAIL:` — print reason, then print:
```
[product-design] Done.
  PRD: <PRD_PATH>
  UX:  <UX_PATH>
```
Stop.

Extract path from `BACKLOG_WRITTEN:`. Print:
```
[product-design] ✓ Backlog written to <path>
```

---

### 11. Report completion

```
[product-design] Done.
  PRD:     <PRD_PATH>
  UX:      <UX_PATH>
  Backlog: <BACKLOG_PATH>
```
