---
name: product-design
description: Two-agent pipeline for product design. Runs product-manager agent (conversational PRD) then ux-designer agent (conversational UX doc), with a review phase after each. Generates <app-name>-prd.md and <app-name>-ux.md in the current working directory. Triggers on "/product-design <app idea>".
---

# /product-design

Two-stage pipeline: PM first → UX second. Both conversational. Both reviewed after writing.

## Steps

### 1. Derive variables

From ARGUMENTS (free-text app idea), derive `APP_NAME`:
- First 2 meaningful words, lowercase, hyphenated
- Strip stop words: for, a, an, the, of, to
- Examples: "cashier app for small warung" → `cashier-app`, "inventory management system" → `inventory-management`

Run `pwd`. Set `CWD`, `PRD_PATH` = `<CWD>/<APP_NAME>-prd.md`, `UX_PATH` = `<CWD>/<APP_NAME>-ux.md`.

---

### 2. PM — Generate questions

Dispatch subagent `agent-team:product-manager`:
```
APP_IDEA: <ARGUMENTS>
OUTPUT_PATH: <PRD_PATH>
```

On `FAIL:` — print reason and stop. Extract questions from `QUESTIONS:`.

---

### 3. PM — Ask questions

Ask each question one at a time. Wait for answer before asking next.

On `stop` or `done` → print `[product-design] PM phase stopped. No PRD written.` and stop.

Build `PM_CONTEXT` as Q/A pairs.

---

### 4. PM — Write PRD

Dispatch subagent `agent-team:product-manager`:
```
APP_IDEA: <ARGUMENTS>
OUTPUT_PATH: <PRD_PATH>
CONTEXT: <PM_CONTEXT>
```

On `FAIL:` — print reason and stop. On `PRD_WRITTEN:` → print `[product-design] ✓ PRD written to <path>`.

---

### 5. Review PRD

Read the written PRD. Act as a senior product manager reviewing for completeness and clarity. Check:

**Completeness**
- Every must-have feature has at least one user story?
- Every target user role has at least one user story?
- Success metrics are measurable, not vague ("users are happy" is not measurable)?
- Business rules cover edge cases for all must-have features?

**Clarity**
- Business rules use definite language (no "as needed", "when appropriate", "etc.")?
- Feature descriptions unambiguous — could two engineers build different things from the same description?
- Scope boundary clear — what is explicitly out of scope?

**Consistency**
- User stories match the feature list (no orphan stories, no uncovered features)?
- Problem statement matches the target users described?
- Nice-to-have features not creeping into must-have list?

Classify each finding:
- **TRIVIAL** — unambiguous fix: missing user story for a stated feature, vague metric with obvious rewrite, formatting inconsistency, feature listed twice. Apply directly.
- **CONFIRM** — requires product decision: missing target user, undefined business rule edge case, ambiguous feature scope, conflicting requirements. Ask user.

No findings → print `[product-design] ✓ PRD review complete. No issues found.`

---

### 6. Apply trivial PRD fixes

For each TRIVIAL finding: apply to `PRD_PATH`, print `[product-design] auto-fixed: <description>`.

---

### 7. Ask about CONFIRM PRD findings

For each CONFIRM finding, one at a time:
```
[PRD Review #N] <title>

<what the problem is and why it matters>

Proposed fix: <concrete change>

Accept, reject, or different approach?
```

On `stop` or `done` — skip remaining and proceed to step 8.

Build `PRD_REVIEW_CONTEXT` as: Finding / Decision / Change entries.

---

### 8. Apply confirmed PRD changes

If any CONFIRM items accepted or given alternatives, dispatch subagent `agent-team:product-manager`:
```
APP_IDEA: <ARGUMENTS>
OUTPUT_PATH: <PRD_PATH>
CONTEXT: <PM_CONTEXT>
REVIEW_CONTEXT: <PRD_REVIEW_CONTEXT>
```

On `FAIL:` → print reason and stop. On `PRD_WRITTEN:` → print `[product-design] ✓ PRD updated to <path>`.

---

### 9. Offer UX phase

Print:
```
PRD complete. Continue to UX design phase? (yes / stop)
```

On `stop`, `no`, or `done` → print:
```
[product-design] Done.
  PRD: <PRD_PATH>
```
Stop.

---

### 10. UX — Generate questions

Dispatch subagent `agent-team:ux-designer`:
```
PRD_PATH: <PRD_PATH>
OUTPUT_PATH: <UX_PATH>
```

On `FAIL:` — print reason and stop. Extract questions from `QUESTIONS:`.

---

### 11. UX — Ask questions

Ask each question one at a time. Wait for answer before asking next.

On `stop` or `done` → print:
```
[product-design] UX phase stopped. No UX doc written.
  PRD: <PRD_PATH>
```
Stop.

Build `UX_CONTEXT` as Q/A pairs.

---

### 12. UX — Write UX doc

Dispatch subagent `agent-team:ux-designer`:
```
PRD_PATH: <PRD_PATH>
OUTPUT_PATH: <UX_PATH>
CONTEXT: <UX_CONTEXT>
```

On `FAIL:` — print reason and stop. On `UX_WRITTEN:` → print `[product-design] ✓ UX doc written to <path>`.

---

### 13. Review UX doc

Read the written UX doc and the PRD. Act as a senior UX designer reviewing for coverage and completeness. Check:

**Coverage**
- Every must-have PRD feature has a corresponding user flow?
- Every user flow has corresponding screen descriptions?
- All screens referenced in flows exist in Screen Descriptions?

**Completeness**
- Empty states defined for every list or data-dependent screen?
- Error states defined for every flow that can fail (network, validation, permission)?
- Loading/in-progress states mentioned where operations take time?
- Destructive actions (delete, void, cancel) have confirmation steps?

**Consistency**
- Flow step count matches screen complexity (a 5-step flow needs 5 describable states)?
- User actions in Screen Descriptions match what flows say users can do?
- Edge Case Handling covers scenarios mentioned in PRD Business Rules?

Classify each finding:
- **TRIVIAL** — unambiguous fix: missing empty state for a list screen, screen mentioned in flow but not in Screen Descriptions, inconsistent screen name. Apply directly.
- **CONFIRM** — requires design decision: missing flow for a must-have feature, missing screen, missing error handling for a critical path, new interaction pattern. Ask user.

No findings → print `[product-design] ✓ UX review complete. No issues found.`

---

### 14. Apply trivial UX fixes

For each TRIVIAL finding: apply to `UX_PATH`, print `[product-design] auto-fixed: <description>`.

---

### 15. Ask about CONFIRM UX findings

For each CONFIRM finding, one at a time:
```
[UX Review #N] <title>

<what the problem is and why it matters>

Proposed fix: <concrete change>

Accept, reject, or different approach?
```

On `stop` or `done` — skip remaining and proceed to step 16.

Build `UX_REVIEW_CONTEXT` as: Finding / Decision / Change entries.

---

### 16. Apply confirmed UX changes

If any CONFIRM items accepted or given alternatives, dispatch subagent `agent-team:ux-designer`:
```
PRD_PATH: <PRD_PATH>
OUTPUT_PATH: <UX_PATH>
CONTEXT: <UX_CONTEXT>
REVIEW_CONTEXT: <UX_REVIEW_CONTEXT>
```

On `FAIL:` → print reason and stop. On `UX_WRITTEN:` → print `[product-design] ✓ UX doc updated to <path>`.

---

### 17. Report completion

```
[product-design] Done.
  PRD: <PRD_PATH>
  UX:  <UX_PATH>
```
