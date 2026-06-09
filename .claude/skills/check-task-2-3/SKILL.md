---
name: check-task-2-3
description: Use to check Task 2-3 PR (JSON table displayed in a SAPUI5 app). Invoked by check-task hub or directly with /check-task-2-3 <PR-url>.
---

# Check Task 2-3 — JSON Table

## Task Summary (verbatim from curriculum)

> **Task 2-3:** Display a list of books from a local JSON file in an `sap.m.Table`. The table must have at least 5 rows. The data must be loaded via a `sap.ui.model.json.JSONModel` initialized in the controller's `onInit` method. The model must be set on the view (not the component). Column headers must use i18n keys.

## Rubric

1. `[static]` Branch name matches `feature/task-2-3`
2. `[static]` Base branch is `main`
3. `[static]` `webapp/manifest.json` exists and is valid JSON
4. `[static]` `webapp/model/` directory contains a JSON data file (any `.json` file with book-like entries)
5. `[static]` Controller `onInit` initializes a `JSONModel` from the data file
6. `[static]` Model is set on the view (`.setModel(` called on `this.getView()`)
7. `[static]` Table column headers use `{i18n>...}` references, not hardcoded strings
8. `[runtime]` App starts without errors
9. `[runtime]` Table renders with at least 5 rows

## Execution Steps

### 1. Checkout

Invoke `shared/pr-checkout` with the PR URL.
- On failure: output the error, stop. No further steps run.
- On success: store `path`, `branch`, `base`, `sha`.

### 2. Branch validation

Invoke `shared/branch-validate` with:
- `branch` = branch from checkout
- `expected_pattern` = `^feature/task-2-3$`
- `base` = base from checkout
- `expected_base` = `main`

Record results for criteria 1 and 2.

### 3. Manifest check

Invoke `shared/ui5-manifest-parse` with `path`.
- If error: mark criterion 3 `fail`, detail = parse error. Continue.
- If success: mark criterion 3 `pass`.

### 4. Static file checks

Use Glob and Read on `path` directly:

**Criterion 4 — JSON data file:**
- Glob `path/webapp/model/*.json`.
- Pass if at least one file found and it contains an array with 5+ entries (peek first 20 lines).
- Fail detail: `"No JSON data file found under webapp/model/"` or `"Data file has fewer than 5 entries"`.

**Criterion 5 — `JSONModel` in `onInit`:**
- Grep `path/webapp/controller/` for `JSONModel` (case-sensitive).
- If found: also check the same file for `onInit` function containing the `JSONModel` initialization.
- Pass if both found. Fail detail: `"JSONModel not found in controller"` or `"JSONModel initialized outside onInit"`.

**Criterion 6 — model set on view:**
- Grep `path/webapp/controller/` for `.setModel(` preceded by `this.getView()` (within 3 lines of each other is acceptable).
- Pass if found. Fail detail: `"setModel not called on this.getView()"`.

**Criterion 7 — i18n column headers:**
- Read `path/webapp/view/` XML files.
- Find `<Column>` elements. Check that their `<Label>` or `text` attributes contain `{i18n>...}` rather than hardcoded strings.
- Pass if all column headers use i18n references. Fail detail: list hardcoded header values found.

### 5. Runtime — launch app

Invoke `shared/ui5-app-launch` with `path` and `script = "start"`.
- On failure: mark criteria 8 and 9 as `fail`, attach `log` as detail. Skip to step 7.
- On success: mark criterion 8 `pass`, store `url` and `pid`.

### 6. Runtime — Playwright assertions

Invoke `shared/playwright-assertions` with `url` and:
```
[
  {
    "recipe": "tableRendersRows",
    "selector": ".sapMList, .sapMTable, [role='grid']",
    "minRows": 5
  }
]
```

Record result as criterion 9.

Stop the dev server: `kill <pid>` (or `npm stop` in `path`).

### 7. Report

Invoke `shared/report-format` with:
- `task_id` = `"2-3"`
- `pr_url` = PR URL
- `branch`, `sha`, `checkout_path` = from checkout result
- `criteria` = all 9 criteria results in order

Output the formatted report.
