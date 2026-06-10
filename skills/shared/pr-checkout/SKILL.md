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
