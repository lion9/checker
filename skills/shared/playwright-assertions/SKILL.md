---
name: playwright-assertions
description: Use to run named Playwright assertion recipes against a running SAPUI5 app. Input: localhost URL and a list of named recipes with parameters. Output: per-assertion pass/fail results with screenshot paths on failure.
---

# Playwright Assertions

## Prerequisites
Playwright MCP must be available in the session.

## Input
- `url` — base URL of the running app (e.g. `http://localhost:8080`)
- `assertions` — list of assertion objects, each with `recipe` and recipe-specific params (see recipes below)

## Output
```
[
  { "id": "tableRendersRows", "status": "pass|fail", "detail": "", "screenshot": null | "/tmp/checker/42-ts/screen-1.png" }
]
```

## Lifecycle
This skill navigates to `url` at the start and tears down the browser session at the end. The dev server is left running — the calling spoke is responsible for stopping it.

## Recipes

### `tableRendersRows`
Parameters: `selector` (CSS or SAPUI5 control selector), `minRows` (integer)

Steps:
1. Navigate to `url`.
2. Wait for the element matching `selector` to be visible (timeout 10s).
3. Count visible row elements within `selector` (look for `tr[role="row"]` or equivalent SAPUI5 table row).
4. Pass if count >= `minRows`, fail with `detail`: `"Found <n> rows, expected at least <minRows>"`.
5. On fail: capture screenshot to `<project_path>/screen-tableRendersRows.png`.

### `clickReplacesCellWithInput`
Parameters: `rowSelector` (selector for the target row), `columnLabel` (column header text)

Steps:
1. Navigate to `url`.
2. Find the cell in the row matching `rowSelector` under the column with header text `columnLabel`.
3. Click the cell.
4. Assert that an `<input>` or `sap.m.Input` control appears within that cell (timeout 5s).
5. Pass if input found. Fail with `detail`: `"No input appeared in cell after click"`.
6. On fail: screenshot.

### `clickOpensDialog`
Parameters: `buttonSelector`, `dialogTextContains`

Steps:
1. Navigate to `url`.
2. Find and click the element matching `buttonSelector`.
3. Wait for a dialog (role=dialog or `.sapMDialog`) to appear (timeout 5s).
4. Assert the dialog contains text matching `dialogTextContains`.
5. Pass if dialog visible and text matches. Fail with detail.
6. On fail: screenshot.

### `dialogCancelClosesWithoutChange`
Parameters: `dialogSelector`, `tableSelector`

Steps:
1. (Assume dialog is already open from a prior step, or re-open it.)
2. Capture a snapshot of the rows in `tableSelector` (row count + first-row text).
3. Click the Cancel button within `dialogSelector` (look for button with text "Cancel" or icon `sap-icon://decline`).
4. Assert the dialog is no longer visible (timeout 3s).
5. Assert row count and first-row text in `tableSelector` are unchanged.
6. Pass if both pass. Fail with detail.
7. On fail: screenshot.

### `formValidationBlocksEmptySubmit`
Parameters: `dialogSelector`, `submitButton`

Steps:
1. (Assume dialog is open.)
2. Do NOT fill in any fields.
3. Click the element matching `submitButton`.
4. Assert the dialog is still open (not closed).
5. Assert at least one validation error message (`.sapMInputBaseMessage`, `aria-invalid`, or visible ValueState error) is present.
6. Pass if dialog stays open and error present. Fail with detail.
7. On fail: screenshot.

### `columnFormatterMatches`
Parameters: `columnLabel`, `regex`

Steps:
1. Navigate to `url`.
2. Find the column with header text `columnLabel`.
3. Read the text content of the first visible cell in that column.
4. Assert the text matches `regex`.
5. Pass if match. Fail with `detail`: `"Cell text '<text>' did not match /<regex>/"`.
6. On fail: screenshot.

### `i18nKeyResolves`
Parameters: `text` (expected rendered text from i18n bundle)

Steps:
1. Navigate to `url`.
2. Assert that `text` appears somewhere in the rendered page (exact string, case-sensitive).
3. Assert that the raw i18n key (e.g. `{i18n>someKey}`) does NOT appear anywhere in the page source.
4. Pass if both hold. Fail if text not found or raw key visible.
5. On fail: screenshot.

### `networkRequestSucceeds`
Parameters: `urlSubstring`

Steps:
1. Navigate to `url` with network interception enabled.
2. Collect all outgoing requests.
3. Assert that at least one request URL contains `urlSubstring` and received a 2xx response.
4. Pass if found. Fail with `detail`: `"No 2xx response for requests containing '<urlSubstring>'"`.
5. On fail: screenshot.

### `iconTabBarHasTabs`
Parameters: `labels` (array of expected tab label strings)

Steps:
1. Navigate to `url`.
2. Find the `sap.m.IconTabBar` (look for `.sapMITB` container).
3. Collect all visible tab labels.
4. Assert every label in `labels` appears in the collected tab labels (case-insensitive).
5. Pass if all found. Fail with `detail`: `"Missing tabs: <list>"`.
6. On fail: screenshot.
