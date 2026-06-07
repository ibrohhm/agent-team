---
name: tech-spec
description: Standalone tech spec generator. Auto-detects *-prd.md and *-ux.md in CWD, asks technical clarification questions, writes a technical specification markdown. Run after /product-design. Triggers on "/tech-spec".
---

# /tech-spec

Standalone stage. Reads PRD and UX doc, runs a Q&A phase to clarify technical context, then writes a technical specification.

## Steps

### 1. Derive paths

Check ARGUMENTS for explicit overrides first:
- If ARGUMENTS contains `PRD_PATH=<path>` — use that value
- If ARGUMENTS contains `UX_PATH=<path>` — use that value

For any path not explicitly provided, auto-detect from CWD:

```bash
pwd
```

Set `CWD` = output of `pwd`.

```bash
ls <CWD>/*-prd.md 2>/dev/null
ls <CWD>/*-ux.md 2>/dev/null
```

**Auto-detection rules:**
- Zero matches → print error and stop (see step 2)
- Exactly one match → use that path
- Multiple matches → print error and stop (see step 2)

---

### 2. Validate files exist

If any file is missing or unresolvable, print the specific error and stop:

```
[tech-spec] ERROR: No PRD file found in CWD. Run /product-design first or specify PRD_PATH=<path>
[tech-spec] ERROR: Multiple PRD files found. Specify: PRD_PATH=<path>
[tech-spec] ERROR: No UX file found in CWD. Specify UX_PATH=<path>
[tech-spec] ERROR: Multiple UX files found. Specify: UX_PATH=<path>
```

---

### 3. Derive APP_NAME and OUTPUT_PATH

Strip `-prd.md` suffix from PRD filename to get `APP_NAME`.

Example: `kasir-in-prd.md` → `APP_NAME` = `kasir-in`

Set `OUTPUT_PATH` = `<CWD>/<APP_NAME>-tech-spec.md`

---

### 4. Generate questions

Dispatch subagent `agent-team:tech-spec` with:
```
PRD_PATH: <PRD_PATH>
UX_PATH: <UX_PATH>
```

If output contains `FAIL:` — print reason and stop.

Extract numbered questions from `QUESTIONS:` output.

---

### 5. Ask questions

Ask each question to the user one at a time. Wait for each answer before asking the next.

If user says `stop` or `done` at any point — print:
```
[tech-spec] Stopped. No tech spec written.
```
Stop.

Build `TECH_CONTEXT`:
```
Q: <question 1>
A: <user answer 1>

Q: <question 2>
A: <user answer 2>
...
```

---

### 6. Write tech spec

Dispatch subagent `agent-team:tech-spec` with:
```
PRD_PATH: <PRD_PATH>
UX_PATH: <UX_PATH>
OUTPUT_PATH: <OUTPUT_PATH>
CONTEXT: <TECH_CONTEXT>
```

If output contains `FAIL:` — print reason and stop:
```
[tech-spec] FAILED: <reason>
```

If output contains `TECH_SPEC_WRITTEN:` — extract the path and print:
```
[tech-spec] ✓ Tech spec written to <path>
```
