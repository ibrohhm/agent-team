---
name: ui-spec
description: Standalone UI layout generator. Auto-detects *-ux.md in CWD, asks user to select screens and style, generates a Pencil .pen design file. Run after /product-design. Triggers on "/ui-spec".
---

# /ui-spec

Reads a UX design doc, lets user select screens and visual style, generates a Pencil `.pen` file.

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

Derive `APP_NAME` by stripping `-ux.md` from filename. Set `OUTPUT_PATH` = `<CWD>/<APP_NAME>.pen`.

---

### 2. Get screen list

Dispatch subagent `agent-team:ui-spec`:
```
UX_PATH: <UX_PATH>
```

On `FAIL:` → print `[ui-spec] FAILED: <reason>` and stop.

Extract screen list from `SCREENS:` output. Print to user:
```
Screens found in UX doc:
1. <Screen Name>
2. <Screen Name>
...

Which screens to generate? Enter numbers (e.g. 1,3,5) or "all":
```

Wait for user input. Parse into `SELECTED_SCREENS` list.

---

### 3. Select visual style

Call `mcp__pencil__get_guidelines` (no params) to list available styles.

On Pencil not connected → print and stop:
```
[ui-spec] ERROR: Pencil app not connected. Open Pencil and try again.
```

Print style list to user:
```
Available styles:
1. <style name> — <description>
2. <style name> — <description>
...

Which style?
```

Wait for user input. Set `CHOSEN_STYLE` = selected style name.

---

### 4. Pick generation mode

Print:
```
Generate all at once or one by one? (all / one)
```

Wait for user input. Set `MODE`:
- `all` or `a` → `batch`
- `one` or `o` or `1` → `single`

---

### 5a. Batch generation

If `MODE = batch`:

Dispatch subagent `agent-team:ui-spec`:
```
UX_PATH: <UX_PATH>
OUTPUT_PATH: <OUTPUT_PATH>
CONTEXT:
  screens: <SELECTED_SCREENS comma-separated>
  style: <CHOSEN_STYLE>
  mode: batch
```

On `FAIL:` → print `[ui-spec] FAILED: <reason>` and stop.
On `DONE:` → proceed to step 6.

---

### 5b. One-by-one generation

If `MODE = single`:

For each screen in `SELECTED_SCREENS`:

  Dispatch subagent `agent-team:ui-spec`:
  ```
  UX_PATH: <UX_PATH>
  OUTPUT_PATH: <OUTPUT_PATH>
  CONTEXT:
    screens: <current screen name only>
    style: <CHOSEN_STYLE>
    mode: single
  ```

  On `FAIL:` → print `[ui-spec] FAILED: <reason>` and stop.

  On `SCREEN_WRITTEN:` → call `mcp__pencil__get_screenshot` and display screenshot to user.

  Print:
  ```
  [<Screen Name>] Generated. next / redo / stop
  ```

  Wait for user input:
  - `next` or `n` → continue to next screen
  - `redo` or `r` → re-dispatch same screen (repeat this step)
  - `stop` or `s` → print summary and exit

After all screens complete, proceed to step 6.

---

### 6. Done

Print:
```
[ui-spec] ✓ Design written to <OUTPUT_PATH>
```
