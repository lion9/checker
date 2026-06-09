---
name: check-task-3-2
description: Use to check Task 3-2 PR (Add Record dialog via XML Fragment with form validation). Invoked by check-task hub or directly.
---

# Check Task 3-2 — Record Creation

## Task Summary (verbatim from curriculum)

> **Task 3-2:** Add an "Add" button above the books table. Clicking it opens a dialog implemented as a separate XML Fragment (`sap.ui.core.Fragment.load` or `this.loadFragment`). The dialog form must have fields for all required book properties. Submitting with empty required fields must show validation errors and block submission. Submitting with valid data adds a new row to the table.

## Rubric

1. `[static]` Branch name matches `feature/task-3-2`
2. `[static]` Base branch is `main`
3. `[static]` A `.fragment.xml` file exists under `webapp/`
4. `[static]` Controller uses `Fragment.load` or `this.loadFragment` to open the dialog
5. `[static]` Add and Cancel buttons with press handlers present in the fragment
6. `[runtime]` App starts without errors
7. `[runtime]` Clicking Add opens the fragment dialog
8. `[runtime]` Submitting empty form shows validation errors (dialog stays open)
9. `[runtime]` Submitting valid data closes the dialog and adds a row

## Execution Steps

### 1. Checkout

Invoke `shared/pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `shared/branch-validate` with pattern `^feature/task-3-2$`, base `main`. Record criteria 1–2.

### 3. Static checks

**Criterion 3:** `Glob(path + "/webapp/**/*.fragment.xml")` → pass if at least one file.

**Criterion 4:** Grep `path/webapp/controller/` for `Fragment.load` or `loadFragment`. Pass if found.

**Criterion 5:** Read the fragment XML. Check for both an Add/Submit button (`press` handler pointing to add/submit) and a Cancel button. Pass if both found.

### 4. App launch

Invoke `shared/ui5-app-launch` with `script = "start"`.
- On failure: mark criteria 6–9 fail, attach log, skip to report.
- On success: mark criterion 6 pass.

### 5. Playwright assertions

Invoke `shared/playwright-assertions` with:
```
[
  {
    "recipe": "clickOpensDialog",
    "buttonSelector": "text=Add",
    "dialogTextContains": ""
  },
  {
    "recipe": "formValidationBlocksEmptySubmit",
    "dialogSelector": ".sapMDialog",
    "submitButton": "text=Add, text=Save, text=Submit"
  }
]
```
Record results as criteria 7 and 8.

For criterion 9, invoke `shared/playwright-run-script`:
```
instructions: |
  1. Page is at <url>.
  2. Count the current number of rows in the table.
  3. Click the Add button above the table to open the dialog.
  4. Fill in all visible input fields in the dialog with valid data (e.g. Title: "Test Book", Author: "Test Author", Year: "2024").
  5. Click the Add/Save/Submit button in the dialog.
  6. Wait for the dialog to close (timeout 3s).
  7. Count the rows in the table again.
  8. Pass if the count increased by 1 and the dialog is closed.
```
Record result as criterion 9.

Stop the dev server.

### 6. Report

Invoke `shared/report-format` with all 9 criteria.
