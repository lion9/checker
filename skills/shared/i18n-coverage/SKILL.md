---
name: i18n-coverage
description: Use to scan a SAPUI5 project for hardcoded user-facing strings that should be i18n references. Input: project path, optional ignore list. Output: list of offending file:line + string.
---

# i18n Coverage

## Input
- `project_path` — absolute path to the checked-out project root
- `ignore` — optional list of string literals to ignore (e.g. technical values like `"sap.m.Table"`)

## Output
```
[
  { "file": "webapp/view/Main.view.xml", "line": 12, "text": "Delete Record" }
]
```
Empty list = no hardcoded strings found (pass).

## Steps

1. Glob all `**/*.view.xml`, `**/*.fragment.xml`, and `**/*.controller.js` / `**/*.controller.ts` files under `project_path/webapp/`.

2. For each XML file:
   - Scan all attribute values that are user-visible: `text`, `title`, `placeholder`, `tooltip`, `description`, `label`, `headerText`, `noDataText`, `value` (if on a Label/Text element).
   - An attribute is an i18n reference if its value matches `\{i18n>[^}]+\}`.
   - Flag any non-empty attribute value that is NOT an i18n reference and NOT a binding expression (`{...}`) and NOT purely numeric and NOT in the `ignore` list.
   - Record `{ file: <relative path>, line: <line number>, text: <attribute value> }`.

3. For each JS/TS controller file:
   - Scan for string literals passed to SAPUI5 message-related APIs: `MessageToast.show(`, `MessageBox.show(`, `MessageBox.error(`, `MessageBox.confirm(`, and similar.
   - Flag any string literal argument that is not an i18n bundle lookup (not `this.getResourceBundle().getText(` or equivalent).
   - Record `{ file: <relative path>, line: <line number>, text: <string literal> }`.

4. Filter out any entries whose `text` appears in the `ignore` list.

5. Return the list. Return `[]` if nothing found.
