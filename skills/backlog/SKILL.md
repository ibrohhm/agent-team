---
name: backlog
description: Standalone backlog generator. Auto-detects *-prd.md, *-ux.md, and *-tech-spec.md in CWD, or accepts explicit paths. Dispatches backlog-generator agent. Triggers on "/backlog".
---

# /backlog

Standalone stage. Reads PRD, UX doc, and tech spec from CWD. Writes a flat product backlog in markdown.

## Steps

### 1. Derive paths

Check ARGUMENTS for explicit overrides first:
- If ARGUMENTS contains `PRD_PATH=<path>` — use that value
- If ARGUMENTS contains `UX_PATH=<path>` — use that value
- If ARGUMENTS contains `TECH_SPEC_PATH=<path>` — use that value

For any path not explicitly provided, auto-detect from CWD:

```bash
pwd
```

Set `CWD` = output of `pwd`.

```bash
ls <CWD>/*-prd.md 2>/dev/null
ls <CWD>/*-ux.md 2>/dev/null
ls <CWD>/*-tech-spec.md 2>/dev/null
```

**Auto-detection rules:**
- Zero matches → print error and stop (see step 2)
- Exactly one match → use that path
- Multiple matches → print error and stop (see step 2)

---

### 2. Validate all three files exist

If any file is missing or unresolvable, print the specific error and stop:

```
[backlog] ERROR: No PRD file found in CWD. Run /product-design first or specify PRD_PATH=<path>
[backlog] ERROR: Multiple PRD files found. Specify: PRD_PATH=<path>
[backlog] ERROR: No UX file found in CWD. Specify UX_PATH=<path>
[backlog] ERROR: Multiple UX files found. Specify: UX_PATH=<path>
[backlog] ERROR: No tech spec found in CWD. Run /tech-spec first or specify TECH_SPEC_PATH=<path>
[backlog] ERROR: Multiple tech spec files found. Specify: TECH_SPEC_PATH=<path>
```

---

### 3. Derive APP_NAME and OUTPUT_PATH

Strip `-prd.md` suffix from PRD filename to get `APP_NAME`.

Example: `kasir-in-prd.md` → `APP_NAME` = `kasir-in`

Set `OUTPUT_PATH` = `<CWD>/<APP_NAME>-backlog.md`

---

### 4. Dispatch backlog-generator

Dispatch subagent `agent-team:backlog-generator` with:
```
PRD_PATH: <PRD_PATH>
UX_PATH: <UX_PATH>
TECH_SPEC_PATH: <TECH_SPEC_PATH>
OUTPUT_PATH: <OUTPUT_PATH>
```

---

### 5. Handle result

If output contains `FAIL:` — print the reason and stop:
```
[backlog] FAILED: <reason>
```

If output contains `BACKLOG_WRITTEN:` — extract the path and print:
```
[backlog] ✓ Backlog written to <path>
```
