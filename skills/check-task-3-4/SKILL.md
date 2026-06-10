---
name: check-task-3-4
description: Use to check Task 3-4 PR (OData mock server setup with ui5-mock). Invoked by check-task hub or directly.
---

# Check Task 3-4 — Mock Server

## Task Summary (verbatim from curriculum)

> **Task 3-4:** Replace the JSON data source with an OData V2 or V4 service. Configure a `ui5-mock` mock server (`ui5.yaml` and `ui5-mock.yaml`) that serves the OData metadata and mock data locally. The app must start with `npm run start-mock` and load data from the mock server. `@sap-ux/ui5-middleware-fe-mockserver` must be listed as a dev dependency.

## Rubric

1. `[static]` Branch name matches `feature/task-3-4`
2. `[static]` Base branch is `main`
3. `[static]` `ui5-mock.yaml` exists at project root
4. `[static]` `package.json` has `start-mock` script
5. `[static]` `package.json` dev dependencies include `@sap-ux/ui5-middleware-fe-mockserver`
6. `[static]` `webapp/localService/` contains at least one `metadata*.xml` file
7. `[static]` `webapp/manifest.json` has at least one OData dataSource (V2 or V4)
8. `[runtime]` App starts with `start-mock` script without errors
9. `[runtime]` OData metadata request returns 2xx

## Execution Steps

### 1. Checkout

Invoke `pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `branch-validate` with pattern `^feature/task-3-4$`, base `main`. Record criteria 1–2.

### 3. Manifest check

Invoke `ui5-manifest-parse` with `path`.
- Criterion 7: pass if `dataSources` array is non-empty and at least one entry has type `OData`.

### 4. Static checks

**Criterion 3:** `Glob(path + "/ui5-mock.yaml")` → pass if found.

**Criterion 4:** Read `path/package.json`, parse JSON, check `scripts["start-mock"]` is defined.

**Criterion 5:** Read `path/package.json`, parse JSON, check `devDependencies["@sap-ux/ui5-middleware-fe-mockserver"]` is defined.

**Criterion 6:** `Glob(path + "/webapp/localService/metadata*.xml")` → pass if at least one file.

### 5. App launch

Invoke `ui5-app-launch` with `script = "start-mock"`.
- On failure: mark criteria 8–9 fail, attach log, skip to report.
- On success: mark criterion 8 pass.

### 6. Playwright assertions

Invoke `playwright-assertions` with:
```
[
  {
    "recipe": "networkRequestSucceeds",
    "urlSubstring": "$metadata"
  }
]
```
Record result as criterion 9.

Stop the dev server.

### 7. Report

Invoke `report-format` with all 9 criteria.
