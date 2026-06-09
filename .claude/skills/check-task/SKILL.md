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
