---
name: check-task-2-2
description: Use to check Task 2-2 PR (generate an empty SAPUI5 Fiori app scaffold). Invoked by check-task hub or directly with /check-task-2-2 <PR-url>.
---

# Check Task 2-2 — Generate Empty App

## Task Summary (verbatim from curriculum)

> **Task 2-2:** Use the Yeoman SAP Fiori generator (`yo @sap/fiori`) to generate an empty SAPUI5 Fiori application. The generated app must have the standard Fiori scaffold file structure. Commit the generated files without modification.

## Rubric

1. `[static]` Branch name matches `feature/task-2-2`
2. `[static]` Base branch is `main`
3. `[static]` `webapp/manifest.json` exists and is valid JSON
4. `[static]` `webapp/manifest.json` contains `sap.app`, `sap.ui`, and `sap.ui5` sections
5. `[static]` `webapp/Component.js` or `webapp/Component.ts` exists
6. `[static]` `webapp/index.html` exists
7. `[static]` `webapp/view/` directory exists with at least one `.view.xml` file
8. `[static]` `webapp/controller/` directory exists with at least one `.controller.js` or `.controller.ts` file
9. `[static]` `ui5.yaml` or `ui5-deploy.yaml` exists at project root
10. `[static]` `package.json` exists at project root and contains a `start` script

## Execution Steps

### 1. Checkout

Invoke `shared/pr-checkout` with the PR URL.
- On failure: output error, stop.

### 2. Branch validation

Invoke `shared/branch-validate` with:
- `branch` = branch from checkout
- `expected_pattern` = `^feature/task-2-2$`
- `base` = base from checkout
- `expected_base` = `main`

Record results for criteria 1 and 2.

### 3. Manifest check

Invoke `shared/ui5-manifest-parse` with `path`.
- Pass criterion 3 if no error.
- Criterion 4: pass if manifest result contains non-empty `sap.app`, `sap.ui`, and `sap.ui5` sections.

### 4. Static file checks

**Criteria 5–10:** Use Glob to check for each file/directory:

- Criterion 5: `Glob(path + "/webapp/Component.*")` → pass if result non-empty.
- Criterion 6: `Glob(path + "/webapp/index.html")` → pass if found.
- Criterion 7: `Glob(path + "/webapp/view/*.view.xml")` → pass if at least one file.
- Criterion 8: `Glob(path + "/webapp/controller/*.controller.*")` → pass if at least one file.
- Criterion 9: `Glob(path + "/ui5*.yaml")` → pass if at least one file.
- Criterion 10: Read `path/package.json`, parse JSON, check `scripts.start` is defined.

No runtime checks for this task (app scaffold, no meaningful behavior to drive with Playwright).

### 5. Report

Invoke `shared/report-format` with all 10 criteria results.
