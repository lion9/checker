---
name: check-task-3-3
description: Use to check Task 3-3 PR (date formatter and full i18n coverage). Invoked by check-task hub or directly.
---

# Check Task 3-3 — Formatter & i18n

## Task Summary (verbatim from curriculum)

> **Task 3-3:** Add a formatter function that renders a book's publication year as `"Published: YYYY"`. All user-facing strings in the app (column headers, button labels, dialog titles, messages) must use the i18n bundle — no hardcoded strings allowed.

## Rubric

1. `[static]` Branch name matches `feature/task-3-3`
2. `[static]` Base branch is `main`
3. `[static]` A formatter file exists (e.g. `webapp/model/formatter.js` or `webapp/model/formatter.ts`)
4. `[static]` The formatter returns a string matching `Published: <year>` pattern
5. `[static]` `webapp/manifest.json` has an i18n bundle configured
6. `[static]` No hardcoded user-facing strings in XML views or controllers (i18n-coverage clean)
7. `[runtime]` App starts without errors
8. `[runtime]` The Published column shows text matching `Published: \d{4}` in at least one row

## Execution Steps

### 1. Checkout

Invoke `pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `branch-validate` with pattern `^feature/task-3-3$`, base `main`. Record criteria 1–2.

### 3. Manifest check

Invoke `ui5-manifest-parse` with `path`.
- Criterion 5: pass if `i18nBundle` is non-null and non-empty in the result.

### 4. Static checks

**Criterion 3:** `Glob(path + "/webapp/model/formatter.*")` → pass if at least one file.

**Criterion 4:** Read the formatter file. Grep for a string literal or template containing `Published:` followed by a year placeholder. Pass if found. Fail detail: `"Formatter does not produce 'Published: YYYY' format"`.

**Criterion 6 — i18n coverage:**
Invoke `i18n-coverage` with `path`.
- Pass if result is empty list.
- Fail if any offending strings found; detail = first 5 offending entries (file:line: text).

### 5. App launch

Invoke `ui5-app-launch` with `script = "start"`.
- On failure: mark criteria 7–8 fail, attach log, skip to report.
- On success: mark criterion 7 pass.

### 6. Playwright assertions

Invoke `playwright-assertions` with:
```
[
  {
    "recipe": "columnFormatterMatches",
    "columnLabel": "Published",
    "regex": "^Published: \\d{4}$"
  }
]
```
Record result as criterion 8.

Stop the dev server.

### 7. Report

Invoke `report-format` with all 8 criteria.
