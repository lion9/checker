---
name: check-task-3-5
description: Use to check Task 3-5 PR (OData model binding with IconTabBar showing multiple entity sets). Invoked by check-task hub or directly.
---

# Check Task 3-5 — OData Binding

## Task Summary (verbatim from curriculum)

> **Task 3-5:** Use the OData model from Task 3-4 to display data in the app. Add an `sap.m.IconTabBar` with at least 3 tabs, each tab bound to a different OData entity set. Both tables must populate from the mock server when the app starts with `start-mock`.

## Rubric

1. `[static]` Branch name matches `feature/task-3-5`
2. `[static]` Base branch is `main`
3. `[static]` `webapp/manifest.json` has a model bound to an OData dataSource
4. `[static]` View contains an `sap.m.IconTabBar` with at least 3 `IconTabFilter` elements
5. `[static]` Each tab's content is bound to a different OData path
6. `[runtime]` App starts with `start-mock` without errors
7. `[runtime]` IconTabBar renders with 3 tabs
8. `[runtime]` At least one table in a tab populates with rows from mock data

## Execution Steps

### 1. Checkout

Invoke `pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `branch-validate` with pattern `^feature/task-3-5$`, base `main`. Record criteria 1–2.

### 3. Manifest check

Invoke `ui5-manifest-parse` with `path`.
- Criterion 3: pass if `models` contains at least one entry with a non-null `dataSource` pointing to an OData source.

### 4. Static checks

**Criteria 4–5:** Read `path/webapp/view/*.view.xml`.
- Criterion 4: count `<IconTabFilter>` elements. Pass if >= 3.
- Criterion 5: for each `IconTabFilter`, check that its content container has a binding path (`items="{/...}"` or `binding="{/...}"`). Pass if at least 2 have distinct OData entity paths.

### 5. App launch

Invoke `ui5-app-launch` with `script = "start-mock"`.
- On failure: mark criteria 6–8 fail, attach log, skip to report.
- On success: mark criterion 6 pass.

### 6. Playwright assertions

Invoke `playwright-assertions` with:
```
[
  {
    "recipe": "iconTabBarHasTabs",
    "labels": []
  },
  {
    "recipe": "tableRendersRows",
    "selector": ".sapMITBContent .sapMList, .sapMITBContent .sapMTable",
    "minRows": 1
  }
]
```
Note: `iconTabBarHasTabs` with empty `labels` just checks that 3+ tabs are rendered without checking specific labels. Record results as criteria 7 and 8.

Stop the dev server.

### 7. Report

Invoke `report-format` with all 8 criteria.
