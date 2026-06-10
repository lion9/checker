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
