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
