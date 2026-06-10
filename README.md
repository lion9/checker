# checker

Claude Code plugin for checking SAPUI5 student PRs. Provides hub and spoke skills for tasks 2-2 through 3-6.

## Installation

```bash
claude plugin marketplace add https://github.com/lion9/checker
claude plugin install checker@checker
```

## Usage

```
/check-task <PR-url>
```

Auto-routes to the correct spoke based on the PR's branch name.
