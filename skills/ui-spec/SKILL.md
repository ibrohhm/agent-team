---
name: ui-spec
description: Standalone UI page content lister. Auto-detects *-ux.md in CWD, outputs a structured markdown file with full page content for all screens. Run after /product-design. Triggers on "/ui-spec".
---

# /ui-spec

Reads a UX design doc, outputs a structured markdown file with full page content for all screens ready for design execution.

## Steps

### 1. Resolve UX path

Check ARGUMENTS for `UX_PATH=<path>` override. If not provided, auto-detect:

```bash
pwd
ls <CWD>/*-ux.md 2>/dev/null
```

Zero matches → print and stop:
```
[ui-spec] ERROR: No UX file found. Run /product-design first or specify UX_PATH=<path>
```

Multiple matches → print and stop:
```
[ui-spec] ERROR: Multiple UX files found. Specify: UX_PATH=<path>
```

Derive `APP_NAME` by stripping `-ux.md` from filename. Set `OUTPUT_PATH` = `<CWD>/<APP_NAME>-screens.md`.

---

### 2. Get screen list

Dispatch subagent `agent-team:ui-spec`:
```
UX_PATH: <UX_PATH>
```

On `FAIL:` → print `[ui-spec] FAILED: <reason>` and stop.

Extract all screen names from `SCREENS:` output into `SELECTED_SCREENS`.

---

### 3. Generate page content

Dispatch subagent `agent-team:ui-spec`:
```
UX_PATH: <UX_PATH>
OUTPUT_PATH: <OUTPUT_PATH>
CONTEXT:
  screens: <SELECTED_SCREENS comma-separated>
```

On `FAIL:` → print `[ui-spec] FAILED: <reason>` and stop.

---

### 4. Done

Print:
```
[ui-spec] ✓ Screen content written to <OUTPUT_PATH>
```
