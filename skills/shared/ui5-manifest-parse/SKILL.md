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
