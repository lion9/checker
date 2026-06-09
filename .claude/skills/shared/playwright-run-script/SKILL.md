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
