---
name: check-task-2-5
description: Use to check Task 2-5 PR (inline title editing — clicking a cell replaces it with an input). Invoked by check-task hub or directly.
---

# Check Task 2-5 — Title Edit

## Task Summary (verbatim from curriculum)

> **Task 2-5:** Add an "Actions" column to the books table. Clicking the Edit button in a row should replace the Title cell with an `sap.m.Input` control so the user can edit the title inline. Clicking Save should revert to text and update the model.

## Rubric

1. `[static]` Branch name matches `feature/task-2-5`
2. `[static]` Base branch is `main`
3. `[static]` Table view contains an "Actions" column markup
4. `[static]` Controller contains an edit handler that sets a cell to editable state
5. `[static]` Controller contains a save handler that reverts editable state
6. `[runtime]` App starts without errors
7. `[runtime]` Clicking Edit on a row makes the Title cell show an input field
8. `[runtime]` Clicking Save reverts the input back to text

## Execution Steps

### 1. Checkout

Invoke `pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `branch-validate` with pattern `^feature/task-2-5$`, base `main`.
Record criteria 1–2.

### 3. Static checks

**Criterion 3 — Actions column:**
- Read `path/webapp/view/*.view.xml`.
- Grep for `Actions` or a `<Column>` that has a button labeled Edit/Save (i18n key or text).
- Pass if found. Fail detail: `"No Actions column found in view XML"`.

**Criterion 4 — edit handler:**
- Grep `path/webapp/controller/` for a function containing `editable`, `setProperty`, or switching between `sap.m.Text` and `sap.m.Input`.
- Pass if found. Fail detail: `"No edit handler found in controller"`.

**Criterion 5 — save handler:**
- Grep `path/webapp/controller/` for a save/confirm function that reverts editable state.
- Pass if found. Fail detail: `"No save handler found in controller"`.

### 4. App launch

Invoke `ui5-app-launch` with `script = "start"`.
- On failure: mark criteria 6–8 fail, attach log, skip to step 6.
- On success: mark criterion 6 pass.

### 5. Playwright assertions

Invoke `playwright-assertions` with:
```
[
  {
    "recipe": "clickReplacesCellWithInput",
    "rowSelector": ".sapMListItem:first-child",
    "columnLabel": "Title"
  }
]
```
Record result as criterion 7.

For criterion 8 (Save reverts to text), invoke `playwright-run-script` with:
```
instructions: |
  1. The page is at <url>.
  2. Click the Edit button in the first row.
  3. Wait for an input to appear in the Title cell.
  4. Type "Test Title" into the input.
  5. Click the Save button in the same row.
  6. Assert that the Title cell now shows plain text (not an input control) and that the text is "Test Title" or non-empty.
  Pass if the input is replaced by text after Save.
```
Record result as criterion 8.

Stop the dev server.

### 6. Report

Invoke `report-format` with all 8 criteria.
