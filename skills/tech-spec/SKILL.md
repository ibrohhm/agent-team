---
name: tech-spec
description: Standalone tech spec generator. Auto-detects *-prd.md and *-ux.md in CWD, asks technical clarification questions, writes a technical specification markdown, then reviews and refines it. Run after /product-design. Triggers on "/tech-spec".
---

# /tech-spec

Reads PRD and UX doc, runs Q&A to clarify tech context, writes a technical spec, then reviews and refines it.

## Steps

### 1. Resolve paths

Check ARGUMENTS for `PRD_PATH=<path>` and `UX_PATH=<path>` overrides. For any not provided, run:

```bash
pwd
ls <CWD>/*-prd.md 2>/dev/null
ls <CWD>/*-ux.md 2>/dev/null
```

Zero or multiple matches → print specific error and stop:
```
[tech-spec] ERROR: No PRD file found. Run /product-design first or specify PRD_PATH=<path>
[tech-spec] ERROR: Multiple PRD files found. Specify: PRD_PATH=<path>
[tech-spec] ERROR: No UX file found. Specify UX_PATH=<path>
[tech-spec] ERROR: Multiple UX files found. Specify: UX_PATH=<path>
```

Derive `APP_NAME` by stripping `-prd.md` from PRD filename. Set `OUTPUT_PATH` = `<CWD>/<APP_NAME>-tech-spec.md`.

---

### 2. Generate questions

Dispatch subagent `agent-team:tech-spec`:
```
PRD_PATH: <PRD_PATH>
UX_PATH: <UX_PATH>
```

On `FAIL:` — print reason and stop. Extract numbered questions from `QUESTIONS:` output.

---

### 3. Ask questions

Ask each question one at a time. Wait for answer before asking next.

On `stop` or `done` → print `[tech-spec] Stopped. No tech spec written.` and stop.

Build `TECH_CONTEXT` as Q/A pairs.

---

### 4. Write tech spec

Dispatch subagent `agent-team:tech-spec`:
```
PRD_PATH: <PRD_PATH>
UX_PATH: <UX_PATH>
OUTPUT_PATH: <OUTPUT_PATH>
CONTEXT: <TECH_CONTEXT>
```

On `FAIL:` → print `[tech-spec] FAILED: <reason>` and stop.

On `TECH_SPEC_WRITTEN:` → print `[tech-spec] ✓ Tech spec written to <path>`.

---

### 5. Review tech spec

Read the written spec. Act as a senior software engineer reviewing for any app type. Check:

**Security**
- Resource-scoped IDs accepted as client params instead of derived from session?
- DB access model specified (service role vs anon key, RLS, API gateway)?
- Brute-force/rate-limit protection on auth endpoints?
- Session expiry edge cases handled (expired session leaving orphaned records)?

**Data model correctness**
- Value types consistent with domain (integers for sub-unit-less currencies, enums for finite states)?
- Snapshots used for mutable references that must be point-in-time?
- Human-readable identifiers present where operators reference records?
- Timezone policy defined when date-range queries span user-local time?
- Soft-delete patterns consistent?
- Column referenced in prose but missing from table definition?

**Concurrency & correctness**
- Race conditions on shared state transitions (approval flows, status machines)?
- Idempotency on non-idempotent mutations where retries are realistic?
- Orphaned state when sessions or processes die mid-flight?

**Performance**
- Indexes for high-frequency/report queries?
- Unbounded list queries paginated?

**Consistency**
- Comments contradicting resolved decisions?
- DB constraints inconsistent with stated business rules?

Classify each finding:
- **TRIVIAL** — unambiguous, non-behavioral: missing index, stale comment, missing FK column referenced in prose, obvious type fix, missing constraint implied by a stated rule. Apply directly.
- **CONFIRM** — behavioral or architectural: new schema fields, auth flow changes, new actions, session behavior, security model, pagination. Ask user.

No findings → print `[tech-spec] ✓ Review complete. No issues found.` and stop.

---

### 6. Apply trivial fixes

For each TRIVIAL finding: apply to `OUTPUT_PATH`, print `[tech-spec] auto-fixed: <description>`.

---

### 7. Ask about CONFIRM findings

For each CONFIRM finding, one at a time:
```
[Review #N] <title>

<what the problem is and why it matters>

Proposed fix: <concrete change>

Accept, reject, or different approach?
```

On `stop` or `done` — skip remaining and proceed to step 8.

Build `REVIEW_CONTEXT` as: Finding / Decision / Change entries.

---

### 8. Apply confirmed changes

If any CONFIRM items accepted or given alternatives, dispatch subagent `agent-team:tech-spec`:
```
PRD_PATH: <PRD_PATH>
UX_PATH: <UX_PATH>
OUTPUT_PATH: <OUTPUT_PATH>
CONTEXT: <TECH_CONTEXT>
REVIEW_CONTEXT: <REVIEW_CONTEXT>
```

On `FAIL:` → print reason and stop.

On `TECH_SPEC_WRITTEN:` → print `[tech-spec] ✓ Tech spec updated to <path>`.
