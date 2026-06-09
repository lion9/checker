---
name: check-task-3-6
description: Use to check Task 3-6 PR (custom mock data files extending the OData mock server). This is an optional curriculum task. Invoked by check-task hub or directly.
---

# Check Task 3-6 — Custom Data (Optional Task)

## Task Summary (verbatim from curriculum)

> **Task 3-6 (Optional):** Add custom mock data files to `webapp/localService/mockdata/` so that the mock server returns realistic data. Each entity set must have a corresponding JSON file. The tables in the app must display this custom data when started with `start-mock`.

## Rubric

1. `[static]` Branch name matches `feature/task-3-6`
2. `[static]` Base branch is `main`
3. `[static]` `webapp/localService/mockdata/` directory exists with at least 2 JSON files
4. `[static]` Each mockdata JSON file contains an array with at least 3 entries
5. `[static]` `webapp/manifest.json` has dataSources with `mockdataSettings` or `localUri` pointing to mockdata
6. `[runtime]` App starts with `start-mock` without errors
7. `[runtime]` Table renders rows sourced from the custom mockdata (row count >= 3)

## Execution Steps

### 1. Checkout

Invoke `shared/pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `shared/branch-validate` with pattern `^feature/task-3-6$`, base `main`. Record criteria 1–2.

### 3. Manifest check

Invoke `shared/ui5-manifest-parse` with `path`.
- Criterion 5: pass if at least one dataSource entry references `localService/mockdata` or has `settings.localUri`.

### 4. Static checks

**Criterion 3:** `Glob(path + "/webapp/localService/mockdata/*.json")` → pass if at least 2 files.

**Criterion 4:** For each mockdata JSON file found, parse and check that the root value is an array with >= 3 items. Pass if all files pass. Fail detail: list any files with fewer than 3 entries.

### 5. App launch

Invoke `shared/ui5-app-launch` with `script = "start-mock"`.
- On failure: mark criteria 6–7 fail, attach log, skip to report.
- On success: mark criterion 6 pass.

### 6. Playwright assertions

Invoke `shared/playwright-assertions` with:
```
[
  {
    "recipe": "tableRendersRows",
    "selector": ".sapMList, .sapMTable, [role='grid']",
    "minRows": 3
  }
]
```
Record result as criterion 7.

Stop the dev server.

### 7. Report

Invoke `shared/report-format` with all 7 criteria.
