# SAPUI5 Course Task Checker — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a set of Claude Code skills that let a mentor run `/check-task <PR-url>` to automatically verify a SAPUI5 student PR against its curriculum rubric and receive a paste-ready markdown report.

**Architecture:** Three-layer pure-skills design: a hub (`check-task`) that routes to a spoke per curriculum task, spokes that own the rubric and runbook, and shared single-purpose utility skills called by spokes. All runtime behavior is delegated to `gh`, `npm`, and Playwright MCP — no Node library backbone.

**Tech Stack:** Claude Code skills (markdown SKILL.md files), GitHub CLI (`gh`), npm/SAPUI5 tooling, Playwright MCP for browser assertions.

---

## File Structure

All files created under `.claude/skills/` in the repo root.

```
.claude/skills/
├── check-task/SKILL.md                    Hub — routing only
├── check-task-2-2/SKILL.md               Spoke — generate empty app
├── check-task-2-3/SKILL.md               Spoke — JSON table (pilot spoke)
├── check-task-2-5/SKILL.md               Spoke — title edit
├── check-task-3-1/SKILL.md               Spoke — confirm dialog
├── check-task-3-2/SKILL.md               Spoke — record creation
├── check-task-3-3/SKILL.md               Spoke — formatter & i18n
├── check-task-3-4/SKILL.md               Spoke — mock server
├── check-task-3-5/SKILL.md               Spoke — OData binding
├── check-task-3-6/SKILL.md               Spoke — custom data (optional)
└── shared/
    ├── pr-checkout/SKILL.md               Shared — checkout PR to temp dir
    ├── branch-validate/SKILL.md           Shared — validate branch + base
    ├── ui5-manifest-parse/SKILL.md        Shared — parse webapp/manifest.json
    ├── ui5-app-launch/SKILL.md            Shared — npm install + npm run, poll until ready
    ├── playwright-assertions/SKILL.md     Shared — named assertion recipe library
    ├── playwright-run-script/SKILL.md     Shared — ad-hoc Playwright instructions
    ├── i18n-coverage/SKILL.md             Shared — scan for hardcoded strings
    └── report-format/SKILL.md            Shared — format final markdown report
```

Build order follows spec recommendation: shared skills first, then pilot spoke (2-3), then remaining spokes in curriculum order, hub last.

---

## Task 1: `shared/pr-checkout`

**Files:**
- Create: `.claude/skills/shared/pr-checkout/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: pr-checkout
description: Use to check out a GitHub PR into a temp directory and return its metadata. Input: PR URL. Output: absolute checkout path, branch name, base branch, head SHA.
---

# PR Checkout

## Input
- `pr_url` — full GitHub PR URL (e.g. `https://github.com/org/repo/pull/42`)

## Output
Returns a structured result:
```
{
  "path": "/tmp/checker/42-1717000000/",
  "branch": "feature/task-3-2",
  "base": "main",
  "sha": "abc1234",
  "pr_number": "42",
  "repo": "org/repo"
}
```

## Steps

1. Run `gh pr view <pr_url> --json number,headRefName,baseRefName,headRefOid,headRepository` and capture the JSON output.
   - If `gh` returns a non-zero exit code, surface the full stderr verbatim and stop.

2. Extract fields:
   - `pr_number` = `.number`
   - `branch` = `.headRefName`
   - `base` = `.baseRefName`
   - `sha` = `.headRefOid` (first 7 chars for display)
   - `repo` = `.headRepository.nameWithOwner`

3. Determine checkout path:
   - Primary: `/tmp/checker/<pr_number>-<unix_timestamp>/`
   - Fallback (if `/tmp` not writable): `~/.cache/checker/<pr_number>-<unix_timestamp>/`
   - Create the directory with `mkdir -p <path>`.

4. Run: `cd <path> && gh pr checkout <pr_number> --repo <repo> .`
   - If this fails, surface the full stderr verbatim and stop.

5. Return the structured result above. Print the path so the caller can pass it to downstream skills.

## Failure contract
Return structured failure, never throw:
```
{ "error": "<gh stderr verbatim>", "step": "view|checkout" }
```
```

- [ ] **Step 2: Verify the file was written correctly**

Run: `cat .claude/skills/shared/pr-checkout/SKILL.md`
Expected: full file content, no truncation.

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/shared/pr-checkout/SKILL.md
git commit -m "feat: add shared/pr-checkout skill"
```

---

## Task 2: `shared/branch-validate`

**Files:**
- Create: `.claude/skills/shared/branch-validate/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: branch-validate
description: Use to validate that a PR branch name matches the expected task pattern and that the base branch is correct. Input: actual branch, expected pattern, actual base, expected base. Output: list of pass/fail criteria.
---

# Branch Validate

## Input
- `branch` — actual head branch name (e.g. `feature/task-3-2`)
- `expected_pattern` — regex or literal (e.g. `feature/task-3-2`)
- `base` — actual base branch name (e.g. `main`)
- `expected_base` — expected base branch name (almost always `main`)

## Output
Returns a list of criteria results:
```
[
  { "id": "branch-name", "label": "Branch follows naming convention", "status": "pass|fail", "detail": "" },
  { "id": "base-branch", "label": "PR targets correct base branch", "status": "pass|fail", "detail": "" }
]
```

## Steps

1. Check branch name:
   - Compare `branch` against `expected_pattern` (treat as a regex if it contains `^` or `$`, otherwise as a literal string match).
   - Pass if match, fail with `detail`: `"Branch is '${branch}', expected '${expected_pattern}'"`.

2. Check base branch:
   - Compare `base` against `expected_base` (case-insensitive).
   - Pass if match, fail with `detail`: `"Base is '${base}', expected '${expected_base}'"`.

3. Return the list. Do not throw or abort — callers handle failure states.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/shared/branch-validate/SKILL.md
git commit -m "feat: add shared/branch-validate skill"
```

---

## Task 3: `shared/ui5-manifest-parse`

**Files:**
- Create: `.claude/skills/shared/ui5-manifest-parse/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: ui5-manifest-parse
description: Use to parse a SAPUI5 webapp/manifest.json and return a normalized object with dataSources, models, i18n bundle path, routing, and dependencies. Input: project root path. Output: normalized manifest object plus any warnings.
---

# UI5 Manifest Parse

## Input
- `project_path` — absolute path to the checked-out project root

## Output
```
{
  "appId": "com.example.app",
  "dataSources": [
    { "name": "mainService", "uri": "/sap/opu/odata/...", "type": "OData", "version": "2.0|4.0" }
  ],
  "models": [
    { "name": "", "dataSource": "mainService", "type": "sap.ui.model.odata.v2.ODataModel|v4" }
  ],
  "i18nBundle": "i18n/i18n",
  "routing": { "routes": [...], "targets": {...} },
  "dependencies": { "libs": {...} },
  "warnings": []
}
```

## Steps

1. Locate `webapp/manifest.json` under `project_path`.
   - If not found: return `{ "error": "webapp/manifest.json not found at <project_path>" }`.

2. Read and JSON-parse the file. If parse fails, return `{ "error": "manifest.json is not valid JSON: <parse error>" }`.

3. Extract fields (use safe navigation — missing optional fields should appear as `null` or `[]`, never crash):
   - `appId` = `["sap.app"]["id"]`
   - `dataSources` = entries from `["sap.app"]["dataSources"]`, each normalized to `{ name, uri, type, version }`. Version is `"4.0"` if `settings.odataVersion` is `"4.0"`, else `"2.0"`. Type defaults to `"OData"` if not set.
   - `models` = entries from `["sap.ui"]["technology"]` (skip) and `["sap.app"]["models"]` (use this). Each: `{ name, dataSource, type }`.
   - `i18nBundle` = `["sap.app"]["i18n"]` (may be a string or object with `bundleUrl`; normalize to string path).
   - `routing` = `["sap.ui5"]["routing"]` (return as-is).
   - `dependencies` = `["sap.ui5"]["dependencies"]`.

4. Append any warnings (non-fatal):
   - No `dataSources` defined: warn `"No dataSources found in manifest"`.
   - No `i18nBundle`: warn `"No i18n bundle configured"`.

5. Return the normalized object.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/shared/ui5-manifest-parse/SKILL.md
git commit -m "feat: add shared/ui5-manifest-parse skill"
```

---

## Task 4: `shared/ui5-app-launch`

**Files:**
- Create: `.claude/skills/shared/ui5-app-launch/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: ui5-app-launch
description: Use to install dependencies and start a SAPUI5 dev server, polling until the app is reachable. Input: project path, npm script name. Output: localhost URL and process handle, or failure with captured logs.
---

# UI5 App Launch

## Input
- `project_path` — absolute path to the checked-out project root
- `script` — npm script to run (default: `start`; use `start-mock` for tasks that need mock data)

## Output (success)
```
{ "url": "http://localhost:8080", "pid": 12345 }
```

## Output (failure)
```
{ "error": "install|start|timeout", "log": "<captured stdout+stderr>" }
```

## Steps

1. Run `npm install` in `project_path` with a 5-minute timeout.
   - Capture combined stdout+stderr.
   - If exit code is non-zero: return `{ "error": "install", "log": "<captured output>" }` immediately.

2. Run `npm run <script>` in `project_path` in the background. Capture its PID and combined stdout+stderr (stream to a temp log file at `<project_path>/.checker-server.log`).

3. Poll `http://localhost:8080` (HEAD request) every 3 seconds for up to 60 seconds.
   - First 200 or 301/302 response: proceed.
   - 60 seconds elapsed with no response: kill the background process, read the log file, return `{ "error": "timeout", "log": "<log file content>" }`.

4. Return `{ "url": "http://localhost:8080", "pid": <pid> }`.

## Caller responsibility
The calling spoke MUST stop the dev server after Playwright assertions complete. Use `kill <pid>` or `npm stop` in `project_path`.

## Fail-fast contract
If this skill returns an `error` field, the spoke MUST mark every `[runtime]` rubric criterion as `[✗]` and attach the `log` as detail. Static checks still run and report normally.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/shared/ui5-app-launch/SKILL.md
git commit -m "feat: add shared/ui5-app-launch skill"
```

---

## Task 5: `shared/playwright-assertions`

**Files:**
- Create: `.claude/skills/shared/playwright-assertions/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/shared/playwright-assertions/SKILL.md
git commit -m "feat: add shared/playwright-assertions skill"
```

---

## Task 6: `shared/playwright-run-script`

**Files:**
- Create: `.claude/skills/shared/playwright-run-script/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: playwright-run-script
description: Use to run a one-off Playwright interaction that doesn't fit any named recipe in playwright-assertions. Input: localhost URL and a free-form instruction block describing the interaction and assertion. Output: pass/fail and screenshot on failure.
---

# Playwright Run Script

## When to use
Use this skill when a spoke needs a browser check that no named recipe in `shared/playwright-assertions` covers AND the check is unlikely to recur across multiple tasks. If the same pattern appears in two or more tasks, add it as a named recipe to `playwright-assertions` instead.

## Input
- `url` — base URL of the running app
- `instructions` — a free-form block (prose or pseudocode) describing:
  1. What to navigate to / click / interact with
  2. What to assert (the pass condition)
  3. What a failure looks like

## Output
```
{ "status": "pass|fail", "detail": "", "screenshot": null | "<path>" }
```

## Steps

1. Read `instructions` carefully and plan the interaction sequence.
2. Use Playwright MCP to navigate to `url`.
3. Execute the interaction sequence described in `instructions`.
4. Evaluate the assertion.
5. If pass: return `{ "status": "pass", "detail": "", "screenshot": null }`.
6. If fail: capture screenshot to `<project_path>/screen-run-script.png`, return `{ "status": "fail", "detail": "<what failed>", "screenshot": "<path>" }`.

## Lifecycle
Tears down the browser session at the end. Dev server left running — caller stops it.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/shared/playwright-run-script/SKILL.md
git commit -m "feat: add shared/playwright-run-script skill"
```

---

## Task 7: `shared/i18n-coverage`

**Files:**
- Create: `.claude/skills/shared/i18n-coverage/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: i18n-coverage
description: Use to scan a SAPUI5 project for hardcoded user-facing strings that should be i18n references. Input: project path, optional ignore list. Output: list of offending file:line + string.
---

# i18n Coverage

## Input
- `project_path` — absolute path to the checked-out project root
- `ignore` — optional list of string literals to ignore (e.g. technical values like `"sap.m.Table"`)

## Output
```
[
  { "file": "webapp/view/Main.view.xml", "line": 12, "text": "Delete Record" }
]
```
Empty list = no hardcoded strings found (pass).

## Steps

1. Glob all `**/*.view.xml`, `**/*.fragment.xml`, and `**/*.controller.js` / `**/*.controller.ts` files under `project_path/webapp/`.

2. For each XML file:
   - Scan all attribute values that are user-visible: `text`, `title`, `placeholder`, `tooltip`, `description`, `label`, `headerText`, `noDataText`, `value` (if on a Label/Text element).
   - An attribute is an i18n reference if its value matches `\{i18n>[^}]+\}`.
   - Flag any non-empty attribute value that is NOT an i18n reference and NOT a binding expression (`{...}`) and NOT purely numeric and NOT in the `ignore` list.
   - Record `{ file: <relative path>, line: <line number>, text: <attribute value> }`.

3. For each JS/TS controller file:
   - Scan for string literals passed to SAPUI5 message-related APIs: `MessageToast.show(`, `MessageBox.show(`, `MessageBox.error(`, `MessageBox.confirm(`, and similar.
   - Flag any string literal argument that is not an i18n bundle lookup (not `this.getResourceBundle().getText(` or equivalent).
   - Record `{ file: <relative path>, line: <line number>, text: <string literal> }`.

4. Filter out any entries whose `text` appears in the `ignore` list.

5. Return the list. Return `[]` if nothing found.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/shared/i18n-coverage/SKILL.md
git commit -m "feat: add shared/i18n-coverage skill"
```

---

## Task 8: `shared/report-format`

**Files:**
- Create: `.claude/skills/shared/report-format/SKILL.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: report-format
description: Use to format a structured rubric result set into a paste-ready markdown PR review block. Input: rubric criteria results, task ID, PR URL, branch, SHA, checkout path. Output: formatted markdown.
---

# Report Format

## Input
```
{
  "task_id": "3-2",
  "pr_url": "https://github.com/org/repo/pull/42",
  "branch": "feature/task-3-2",
  "sha": "abc1234",
  "checkout_path": "/tmp/checker/42-1717000000/",
  "criteria": [
    {
      "id": "branch-name",
      "label": "Branch follows naming convention",
      "type": "static",
      "status": "pass|fail|skip",
      "detail": ""
    }
  ]
}
```

`status` values:
- `pass` → render as `[✓]`
- `fail` → render as `[✗]`
- `skip` → render as `[—]`

## Output

Produce the following markdown block (output verbatim, ready to paste into a GitHub PR review):

```
## Task <task_id> Checker Report

<2–4 sentence narrative. Rules:
- If all criteria pass: "All rubric criteria passed. The implementation looks complete."
- If any criterion fails: Lead with the overall verdict ("X of Y criteria failed."), name the biggest blocker, state what to fix first.
- Never write more than 4 sentences.
- Do not repeat every failed criterion — the table below covers that.
>

### Rubric

| Status | Criterion | Detail |
|--------|-----------|--------|
| [✓] | Branch follows naming convention | |
| [✗] | Validation: all required fields checked | Only Name field has validation |
| [—] | App starts and renders table | Skipped (app did not start) |

### Metadata

- **PR:** <pr_url>
- **Branch:** <branch>
- **Head SHA:** <sha>
- **Checked out to:** <checkout_path>
- **Checked at:** <ISO 8601 timestamp>
```

## Steps

1. Count pass/fail/skip totals from `criteria`.
2. Write the narrative:
   - All pass: "All <N> rubric criteria passed. The implementation looks complete."
   - Some fail: "X of N criteria failed. [Name the top blocker — the first `fail` criterion in the list]. Fix this first before addressing the remaining issues."
   - App-didn't-start case (all `[runtime]` criteria are `fail` and any detail contains "App did not start"): lead with "App did not start — fix this first. Static checks are reported below."
3. Render the rubric table in input order.
4. Append the metadata footer.
5. Output the complete markdown block. This is the final output of the entire check run.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/shared/report-format/SKILL.md
git commit -m "feat: add shared/report-format skill"
```

---

## Task 9: Pilot spoke — `check-task-2-3` (JSON table)

This is the pilot spoke. It exercises every shared skill. If contracts are wrong, they surface here before being replicated.

**Files:**
- Create: `.claude/skills/check-task-2-3/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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
```

- [ ] **Step 2: Verify the file looks right**

Run: `wc -l .claude/skills/check-task-2-3/SKILL.md`
Expected: under 150 lines.

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/check-task-2-3/SKILL.md
git commit -m "feat: add check-task-2-3 pilot spoke"
```

---

## Task 10: Spoke — `check-task-2-2` (Generate empty app)

**Files:**
- Create: `.claude/skills/check-task-2-2/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/check-task-2-2/SKILL.md
git commit -m "feat: add check-task-2-2 spoke"
```

---

## Task 11: Spoke — `check-task-2-5` (Title edit)

**Files:**
- Create: `.claude/skills/check-task-2-5/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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

Invoke `shared/pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `shared/branch-validate` with pattern `^feature/task-2-5$`, base `main`.
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

Invoke `shared/ui5-app-launch` with `script = "start"`.
- On failure: mark criteria 6–8 fail, attach log, skip to step 6.
- On success: mark criterion 6 pass.

### 5. Playwright assertions

Invoke `shared/playwright-assertions` with:
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

For criterion 8 (Save reverts to text), invoke `shared/playwright-run-script` with:
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

Invoke `shared/report-format` with all 8 criteria.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/check-task-2-5/SKILL.md
git commit -m "feat: add check-task-2-5 spoke"
```

---

## Task 12: Spoke — `check-task-3-1` (Confirm dialog)

**Files:**
- Create: `.claude/skills/check-task-3-1/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/check-task-3-1/SKILL.md
git commit -m "feat: add check-task-3-1 spoke"
```

---

## Task 13: Spoke — `check-task-3-2` (Record creation)

**Files:**
- Create: `.claude/skills/check-task-3-2/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/check-task-3-2/SKILL.md
git commit -m "feat: add check-task-3-2 spoke"
```

---

## Task 14: Spoke — `check-task-3-3` (Formatter & i18n)

**Files:**
- Create: `.claude/skills/check-task-3-3/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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

Invoke `shared/pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `shared/branch-validate` with pattern `^feature/task-3-3$`, base `main`. Record criteria 1–2.

### 3. Manifest check

Invoke `shared/ui5-manifest-parse` with `path`.
- Criterion 5: pass if `i18nBundle` is non-null and non-empty in the result.

### 4. Static checks

**Criterion 3:** `Glob(path + "/webapp/model/formatter.*")` → pass if at least one file.

**Criterion 4:** Read the formatter file. Grep for a string literal or template containing `Published:` followed by a year placeholder. Pass if found. Fail detail: `"Formatter does not produce 'Published: YYYY' format"`.

**Criterion 6 — i18n coverage:**
Invoke `shared/i18n-coverage` with `path`.
- Pass if result is empty list.
- Fail if any offending strings found; detail = first 5 offending entries (file:line: text).

### 5. App launch

Invoke `shared/ui5-app-launch` with `script = "start"`.
- On failure: mark criteria 7–8 fail, attach log, skip to report.
- On success: mark criterion 7 pass.

### 6. Playwright assertions

Invoke `shared/playwright-assertions` with:
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

Invoke `shared/report-format` with all 8 criteria.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/check-task-3-3/SKILL.md
git commit -m "feat: add check-task-3-3 spoke"
```

---

## Task 15: Spoke — `check-task-3-4` (Mock server)

**Files:**
- Create: `.claude/skills/check-task-3-4/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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

Invoke `shared/pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `shared/branch-validate` with pattern `^feature/task-3-4$`, base `main`. Record criteria 1–2.

### 3. Manifest check

Invoke `shared/ui5-manifest-parse` with `path`.
- Criterion 7: pass if `dataSources` array is non-empty and at least one entry has type `OData`.

### 4. Static checks

**Criterion 3:** `Glob(path + "/ui5-mock.yaml")` → pass if found.

**Criterion 4:** Read `path/package.json`, parse JSON, check `scripts["start-mock"]` is defined.

**Criterion 5:** Read `path/package.json`, parse JSON, check `devDependencies["@sap-ux/ui5-middleware-fe-mockserver"]` is defined.

**Criterion 6:** `Glob(path + "/webapp/localService/metadata*.xml")` → pass if at least one file.

### 5. App launch

Invoke `shared/ui5-app-launch` with `script = "start-mock"`.
- On failure: mark criteria 8–9 fail, attach log, skip to report.
- On success: mark criterion 8 pass.

### 6. Playwright assertions

Invoke `shared/playwright-assertions` with:
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

Invoke `shared/report-format` with all 9 criteria.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/check-task-3-4/SKILL.md
git commit -m "feat: add check-task-3-4 spoke"
```

---

## Task 16: Spoke — `check-task-3-5` (OData binding)

**Files:**
- Create: `.claude/skills/check-task-3-5/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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

Invoke `shared/pr-checkout`. On failure: stop.

### 2. Branch validation

Invoke `shared/branch-validate` with pattern `^feature/task-3-5$`, base `main`. Record criteria 1–2.

### 3. Manifest check

Invoke `shared/ui5-manifest-parse` with `path`.
- Criterion 3: pass if `models` contains at least one entry with a non-null `dataSource` pointing to an OData source.

### 4. Static checks

**Criteria 4–5:** Read `path/webapp/view/*.view.xml`.
- Criterion 4: count `<IconTabFilter>` elements. Pass if >= 3.
- Criterion 5: for each `IconTabFilter`, check that its content container has a binding path (`items="{/...}"` or `binding="{/...}"`). Pass if at least 2 have distinct OData entity paths.

### 5. App launch

Invoke `shared/ui5-app-launch` with `script = "start-mock"`.
- On failure: mark criteria 6–8 fail, attach log, skip to report.
- On success: mark criterion 6 pass.

### 6. Playwright assertions

Invoke `shared/playwright-assertions` with:
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

Invoke `shared/report-format` with all 8 criteria.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/check-task-3-5/SKILL.md
git commit -m "feat: add check-task-3-5 spoke"
```

---

## Task 17: Spoke — `check-task-3-6` (Custom data — optional)

**Files:**
- Create: `.claude/skills/check-task-3-6/SKILL.md`

- [ ] **Step 1: Create the spoke**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/check-task-3-6/SKILL.md
git commit -m "feat: add check-task-3-6 spoke"
```

---

## Task 18: Hub — `check-task`

Hub is written last, once all spokes exist and their names are confirmed.

**Files:**
- Create: `.claude/skills/check-task/SKILL.md`

- [ ] **Step 1: Create the hub**

```markdown
---
name: check-task
description: Use to check any SAPUI5 student PR. Auto-routes to the correct spoke based on branch name. Usage: /check-task <PR-url> or /check-task <PR-url> --task <M-N>.
---

# Check Task — Hub

## Invocation

```
/check-task <PR-url>
/check-task <PR-url> --task 3-2
```

## Responsibilities

1. Fetch PR metadata.
2. Parse branch name to determine task ID.
3. Dispatch to the matching spoke.

This skill does NO rubric checking. It is a pure router.

## Steps

### 1. Parse arguments

- Extract `pr_url` (required — the first argument).
- Extract `--task <M-N>` override if present (e.g. `--task 3-2` → task ID `3-2`).

### 2. Fetch PR metadata

Run:
```
gh pr view <pr_url> --json number,headRefName,baseRefName,state,url,headRepository
```

- If `gh` returns non-zero: output the full stderr and stop. Do not proceed.
- Parse the JSON response.

Store:
- `branch` = `headRefName`
- `base` = `baseRefName`
- `state` = `state` (`OPEN`, `CLOSED`, `MERGED`)
- `pr_number` = `number`

### 3. Validate PR state

- If `state` is `CLOSED` or `MERGED`: emit a warning `"⚠️ PR is ${state} — continuing check, but results may reflect merged code."`. Do not stop.
- If `state` is `OPEN`: proceed silently.

### 4. Validate base branch

- If `base` is not `main` (case-insensitive): output `"❌ PR targets '${base}', expected 'main'. Fix the PR base before checking."` and stop.

### 5. Determine task ID

If `--task` override was provided:
- Use that task ID directly. Skip regex parsing.

Otherwise:
- Match `branch` against `^feature/task-(\d+)-(\d+)$`.
- Match → task ID = `"${m}-${t}"` (e.g. `"3-2"`).
- No match → output:
  ```
  Branch '${branch}' does not match the expected pattern `feature/task-M-N`.
  Please provide the task ID manually: /check-task <PR-url> --task M-N
  ```
  Stop.

### 6. Dispatch to spoke

Known task IDs and their spokes:
| Task ID | Spoke skill name |
|---------|-----------------|
| 2-2 | check-task-2-2 |
| 2-3 | check-task-2-3 |
| 2-5 | check-task-2-5 |
| 3-1 | check-task-3-1 |
| 3-2 | check-task-3-2 |
| 3-3 | check-task-3-3 |
| 3-4 | check-task-3-4 |
| 3-5 | check-task-3-5 |
| 3-6 | check-task-3-6 |

- If task ID is in the table: invoke the matching spoke using the `Skill` tool, passing the PR URL.
- If task ID is NOT in the table: output:
  ```
  No checker exists for Task ${task_id}.
  Available tasks: 2-2, 2-3, 2-5, 3-1, 3-2, 3-3, 3-4, 3-5, 3-6
  ```
  Stop.

## What the hub does NOT do

- No file inspection
- No rubric checking
- No PR checkout (the spoke or shared/pr-checkout handles this)
- No report formatting
```

- [ ] **Step 2: Verify file length is short**

Run: `wc -l .claude/skills/check-task/SKILL.md`
Expected: under 100 lines.

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/check-task/SKILL.md
git commit -m "feat: add check-task hub skill"
```

---

## Task 19: Smoke test the full chain

- [ ] **Step 1: Verify all skill files exist**

```bash
find .claude/skills -name "SKILL.md" | sort
```

Expected output (18 files):
```
.claude/skills/check-task-2-2/SKILL.md
.claude/skills/check-task-2-3/SKILL.md
.claude/skills/check-task-2-5/SKILL.md
.claude/skills/check-task-3-1/SKILL.md
.claude/skills/check-task-3-2/SKILL.md
.claude/skills/check-task-3-3/SKILL.md
.claude/skills/check-task-3-4/SKILL.md
.claude/skills/check-task-3-5/SKILL.md
.claude/skills/check-task-3-6/SKILL.md
.claude/skills/check-task/SKILL.md
.claude/skills/shared/branch-validate/SKILL.md
.claude/skills/shared/i18n-coverage/SKILL.md
.claude/skills/shared/playwright-assertions/SKILL.md
.claude/skills/shared/playwright-run-script/SKILL.md
.claude/skills/shared/pr-checkout/SKILL.md
.claude/skills/shared/report-format/SKILL.md
.claude/skills/shared/ui5-app-launch/SKILL.md
.claude/skills/shared/ui5-manifest-parse/SKILL.md
```

- [ ] **Step 2: Verify no skill references undefined types/functions from other skills**

Check that every shared skill name referenced in a spoke's runbook matches an actual file:
```bash
grep -r "shared/" .claude/skills/check-task-*/SKILL.md | grep -oP "shared/[a-z-]+" | sort -u
```
Expected: all values appear as directory names under `.claude/skills/shared/`.

- [ ] **Step 3: Verify hub dispatch table is complete**

```bash
grep -oP "check-task-\d+-\d+" .claude/skills/check-task/SKILL.md | sort
```
Expected: 9 entries matching the 9 spoke names.

- [ ] **Step 4: Final commit**

```bash
git add .
git commit -m "chore: verify all skill files complete"
```

---

## Self-Review Notes

**Spec coverage check:**

| Spec requirement | Covered by |
|---|---|
| Hub routes PR to spoke | Task 18 (`check-task`) |
| Shared `pr-checkout` | Task 1 |
| Shared `branch-validate` | Task 2 |
| Shared `ui5-manifest-parse` | Task 3 |
| Shared `ui5-app-launch` | Task 4 |
| Shared `playwright-assertions` with all 9 recipes | Task 5 |
| Shared `playwright-run-script` | Task 6 |
| Shared `i18n-coverage` | Task 7 |
| Shared `report-format` | Task 8 |
| Pilot spoke Task 2-3 | Task 9 |
| Spoke 2-2 | Task 10 |
| Spoke 2-5 | Task 11 |
| Spoke 3-1 | Task 12 |
| Spoke 3-2 | Task 13 |
| Spoke 3-3 | Task 14 |
| Spoke 3-4 | Task 15 |
| Spoke 3-5 | Task 16 |
| Spoke 3-6 | Task 17 |
| Hub last | Task 18 |
| Fail-fast on app launch | Task 4 + all spokes steps 4/5 |
| `/tmp` fallback to `~/.cache/checker/` | Task 1 |
| 5-minute `npm install` timeout | Task 4 |
| `--task` override | Task 18 |
| Unknown task ID message | Task 18 |
| Report narrative + rubric table + metadata footer | Task 8 |

All spec requirements are covered. No TBD placeholders. Type/name consistency verified — `shared/pr-checkout` returns `path`/`branch`/`base`/`sha` consistently referenced across all spokes.
