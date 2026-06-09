---
name: check-task-3-1
description: Use to check Task 3-1 PR (delete confirmation dialog). Invoked by check-task hub or directly.
---

# Check Task 3-1 — Confirm Dialog

## Task Summary (verbatim from curriculum)

> **Task 3-1:** Add a Delete button to each row in the books table. Clicking Delete must open a confirmation dialog (`sap.m.Dialog` or `MessageBox.confirm`). Clicking Yes in the dialog removes the row from the model. Clicking No (or Cancel) closes the dialog without changes.

## Rubric

1. `[static]` Branch name matches `feature/task-3-1`
2. `[static]` Base branch is `main`
3. `[static]` Delete button present in table row markup
4. `[static]` Controller contains a delete handler that opens a dialog
5. `[static]` Controller handles both Yes and No/Cancel outcomes
6. `[runtime]` App starts without errors
7. `[runtime]` Clicking Delete opens a confirmation dialog
8. `[runtime]` Clicking Cancel closes the dialog and leaves the table unchanged
9. `[runtime]` Clicking Yes removes the row from the table

## Execution Steps

### 1. Checkout

Invoke `shared/pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `shared/branch-validate` with pattern `^feature/task-3-1$`, base `main`. Record criteria 1–2.

### 3. Static checks

**Criterion 3:** Grep `path/webapp/view/` for `Delete` button text or `sap-icon://delete` icon. Pass if found.

**Criterion 4:** Grep `path/webapp/controller/` for `MessageBox` or `sap.m.Dialog` usage in a delete handler. Pass if found.

**Criterion 5:** Grep for both a yes/confirm callback (deleting from model) and a cancel/no callback (doing nothing). Pass if both found.

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
    "buttonSelector": "text=Delete",
    "dialogTextContains": "confirm"
  },
  {
    "recipe": "dialogCancelClosesWithoutChange",
    "dialogSelector": ".sapMDialog",
    "tableSelector": ".sapMList, .sapMTable"
  }
]
```
Record results as criteria 7 and 8.

For criterion 9, invoke `shared/playwright-run-script`:
```
instructions: |
  1. Page is at <url>.
  2. Count the number of rows in the table.
  3. Click the Delete button in the first row.
  4. Wait for the confirmation dialog to appear.
  5. Click the Yes button in the dialog.
  6. Wait for the dialog to close.
  7. Count the rows in the table again.
  8. Pass if the count decreased by 1.
```
Record result as criterion 9.

Stop the dev server.

### 6. Report

Invoke `shared/report-format` with all 9 criteria.
