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
