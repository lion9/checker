# SAPUI5 Course Task Checker — Design

**Date:** 2026-06-10
**Status:** Draft — awaiting user review
**Scope:** Local Claude Code skills only. GitHub/GitLab pipeline migration noted but out of scope.

## Problem

Mentors reviewing a SAPUI5 learning course need to verify that student PRs (one per curriculum task, e.g. `Task 2-3`, `Task 3-2`) meet the per-task requirements. Today this is manual: read the task description, check the diff, click through the app, leave review notes. The work repeats verbatim across cohorts and across tasks (branch-naming and base-branch checks alone are in every task).

The checker should:

- Identify which curriculum task a PR is for.
- Verify each task's rubric deterministically — both static (file structure, manifest entries, controller wiring) and runtime (the app actually starts and behaves correctly when clicked through).
- Produce a consistent report a mentor can paste into PR review.
- Be local-first today, portable to GitHub/GitLab Actions later without rewriting rubrics.

## Decisions

| Decision | Choice | Reason |
|---|---|---|
| Task routing | Branch name regex (`feature/task-M-N`), with mentor override | Students already follow this convention per the curriculum. Zero added friction. |
| Check depth | Static analysis + Playwright runtime via MCP | Static catches structural correctness; runtime catches behavioral correctness. Together they cover the rubric without LLM judgment in the verdict. |
| Code access | `gh pr checkout` into temp dir | Clean, isolated, reproducible per check. |
| Output format | Structured rubric (✓/✗/—) + 2–4 sentence narrative summary | Skimmable for mentor; consistent across all PRs; easy to paste into review. |
| Rubric storage | Inside each spoke's `SKILL.md` | One file to read and edit per task. Version-controlled. |
| Shared utilities | Reusable shared skills called by spokes | Branch validation, manifest parsing, runtime launch, Playwright assertions, i18n scanning, report formatting all overlap across tasks. |
| Hub vs direct invocation | Both — hub default, spokes also callable | `/check-task <PR>` for normal use; `/check-task-3-2 <PR>` for re-checks or when branch naming is wrong. |
| Runtime tool | Playwright MCP | Mature, scriptable, screenshots, ergonomic for clicking SAPUI5 controls. Network inspection covers the mock-server tasks. |
| Runtime failure handling | Fail-fast — "app didn't start" is itself a failed criterion | Honest signal. Static checks still run. Mentor sees clearly that the app is broken. |
| Architecture | Pure skills, no Node library backbone | Everything lives in markdown skill files. Migration to pipelines later replaces the runner, not the rubrics. |

## Architecture

Three layers, all under `.claude/skills/`:

1. **Hub** (`check-task`) — routes a PR to the right spoke. Parses branch, validates PR via `gh`, dispatches. No rubric logic.
2. **Spokes** (`check-task-<module>-<task>`) — one per curriculum task. Owns the rubric and the runbook. Calls shared skills.
3. **Shared skills** (`shared/*`) — reusable, single-purpose primitives. No rubric knowledge.

```
.claude/skills/
├── check-task/                    HUB
├── check-task-2-2/                SPOKES
├── check-task-2-3/
├── check-task-2-5/
├── check-task-3-1/
├── check-task-3-2/
├── check-task-3-3/
├── check-task-3-4/
├── check-task-3-5/
├── check-task-3-6/
└── shared/
    ├── pr-checkout/
    ├── branch-validate/
    ├── ui5-manifest-parse/
    ├── ui5-app-launch/
    ├── playwright-assertions/
    ├── i18n-coverage/
    └── report-format/
```

## Hub skill (`check-task`)

**Invocation:** `/check-task <PR-url>` or `/check-task <PR-url> --task 3-2`.

**Responsibilities:**

1. `gh pr view <url> --json headRefName,baseRefName,state,url,headRepository` to confirm the PR exists and capture metadata.
2. Parse `headRefName` against `^feature/task-(\d+)-(\d+)$`.
   - Match → dispatch to `check-task-<m>-<t>`.
   - No match → ask mentor for task ID, with the parsed branch name shown.
3. Confirm `baseRefName` is `main`. If not, fail early with a clear message before dispatching.
4. Confirm PR state is `open`. Closed/merged is allowed but emits a warning.
5. Hand the validated metadata + temp checkout path to the spoke.

**Failure modes the hub handles itself:**

- Invalid PR URL / 404 → fail with `gh` error.
- Branch doesn't match convention → ask mentor for task ID.
- Unknown task ID (no matching spoke) → list available spokes, ask mentor to pick.

**What the hub does NOT do:** rubric logic, file inspection, runtime. It is a router.

## Spoke skill template

Every spoke follows the same shape. Spoke `SKILL.md` sections, in order:

1. **Frontmatter** — `name`, `description` starting with the trigger phrase, `allowed-tools` if needed.
2. **Task summary** — verbatim task text from the curriculum, as a quoted appendix block. Source of truth for what the student was asked to do.
3. **Rubric** — numbered list of criteria, each marked `[static]` or `[runtime]`.
4. **Execution steps** — runbook: which shared skills to invoke and in what order.
5. **Report contract** — call `shared/report-format` with the rubric results.

Adding a new task = copy a spoke, replace the task summary and rubric, adjust the runbook. Spokes do not reimplement parsing or browser control; they consume shared skills.

## Shared skills

Each is small, single-purpose, called by spokes via the `Skill` tool. Listed with input → output contract.

### `shared/pr-checkout`
- **In:** PR URL.
- **Does:** `gh pr view` to capture metadata, then `gh pr checkout` into `/tmp/checker/<pr-number>-<timestamp>/`.
- **Out:** absolute checkout path, branch, base, head SHA.
- **Fail:** PR not found, auth missing, checkout conflict — surface `gh` stderr verbatim.

### `shared/branch-validate`
- **In:** actual branch, expected pattern (e.g. `feature/task-3-2`), base, expected base (`main`).
- **Out:** list of pass/fail criteria with messages.
- **Used by:** every spoke at the top of its run.

### `shared/ui5-manifest-parse`
- **In:** project path.
- **Does:** locates `webapp/manifest.json`, parses it, returns a normalized object.
- **Out:** `appId`, `dataSources[]`, `models[]`, `i18nBundle`, `routing`, `dependencies`, plus warnings.
- **Used by:** Tasks 3-3 (i18n bundle), 3-4 (V2/V4 dataSources), 3-5 (models bound to dataSources), 2-2/2-3 (basic structure), 3-6 (mockdata wiring).

### `shared/ui5-app-launch`
- **In:** project path, start script name (`start` default, `start-mock` for 3-4+).
- **Does:** `npm install` (with timeout), `npm run <script>` in background, polls localhost until app responds or timeout.
- **Out:** localhost URL + background process handle; OR `failed` with captured stdout/stderr.
- **Fail-fast contract:** if install or start fails, return `failed` immediately. Spoke marks all `[runtime]` criteria failed and attaches the captured error.

### `shared/playwright-assertions`
- **In:** localhost URL + list of named assertion recipes with parameters.
- **Recipes (initial set, grown as spokes need them):**
  - `tableRendersRows(selector, minRows)` — table exists with at least N rows.
  - `clickReplacesCellWithInput(rowSelector, columnLabel)` — for Task 2-5.
  - `clickOpensDialog(buttonSelector, dialogTextContains)` — for 3-1, 3-2.
  - `dialogCancelClosesWithoutChange(dialogSelector, tableSelector)` — for 3-1, 3-2.
  - `formValidationBlocksEmptySubmit(dialogSelector, submitButton)` — for 3-2.
  - `columnFormatterMatches(columnLabel, regex)` — for 3-3 (`Published: \d{4}`).
  - `i18nKeyResolves(text)` — text on screen came from the bundle, not hardcoded.
  - `networkRequestSucceeds(urlSubstring)` — for 3-4 metadata loads.
  - `iconTabBarHasTabs(labels[])` — for 3-5.
- **Out:** per-assertion pass/fail + screenshot path on failure.
- **Lifecycle:** tears down the browser; spoke is responsible for stopping the dev server.

### `shared/playwright-run-script`
- **In:** localhost URL + an ad-hoc instruction block describing the browser interaction and what to assert.
- **Does:** runs a one-off Playwright script for novel checks that don't fit a named recipe.
- **Out:** pass/fail + screenshot path on failure.
- **When to use:** spoke needs a check no recipe in `playwright-assertions` covers and the check isn't reusable enough to promote to a recipe yet. If a pattern recurs, promote it to `playwright-assertions`.

### `shared/i18n-coverage`
- **In:** project path, optional ignore list.
- **Does:** scans `**/*.view.xml`, `**/*.fragment.xml`, controllers for hardcoded user-facing strings that aren't `{i18n>...}` references.
- **Out:** list of offending file:line + string.
- **Used by:** 3-3 primarily; later spokes can call as a regression guard.

### `shared/report-format`
- **In:** rubric criteria results (id, label, status, optional detail/screenshot), task ID, PR URL.
- **Out:** markdown block:
  - Top: 2–4 sentence narrative — overall verdict, biggest issue, what to fix first.
  - Body: rubric table with `[✓]` / `[✗]` / `[—]` (skipped), each row linkable to its detail.
  - Footer: PR URL, branch, head SHA, timestamp, temp checkout path.

**Discipline rules for shared skills:**

- Pure — no rubric knowledge, don't know which task they're serving.
- Composable — spokes call any subset; the only ordering constraints are `pr-checkout` before any project-path consumer, and `ui5-app-launch` before `playwright-assertions`.
- Failure surfaces are explicit — return structured failure data, never throw the run.

## Data flow (one full check)

```
Mentor: /check-task https://github.com/student/repo/pull/42

[hub: check-task]
  ├─ gh pr view → {branch: feature/task-3-2, base: main, state: open}
  ├─ regex match → task = 3-2
  ├─ Skill(pr-checkout, url) → /tmp/checker/42-<ts>/
  └─ Skill(check-task-3-2, {pr, path, branch, base, sha})

[spoke: check-task-3-2]
  ├─ Skill(branch-validate, ...) → [✓ branch], [✓ base]
  ├─ static checks (Glob/Grep/Read on path):
  │     [✓] *.fragment.xml exists
  │     [✓] Fragment.load / loadFragment in controller
  │     [✓] Add + Cancel buttons with press handlers
  │     [✗] Validation: only Name checked, others missing
  ├─ Skill(ui5-app-launch, {path, script: start})
  │     → http://localhost:8080  (OR failed → mark all [runtime] ✗)
  ├─ Skill(playwright-assertions, url, [recipes...])
  │     → per-assertion results + screenshots
  ├─ stop dev server
  └─ Skill(report-format, ...)
        → final markdown block to mentor
```

**Failure paths:**

- `pr-checkout` fails → hub aborts, no spoke runs. Output is the `gh` error.
- `ui5-app-launch` fails → spoke continues, static results stand, every `[runtime]` row marked `[✗]` with the captured install/start log attached. Narrative leads with "App did not start — fix this first."

**Cleanup:** temp checkout left on disk for mentor inspection (path printed in the report footer). Mentor removes manually. A `--cleanup` flag can be added later.

## Per-task rubric coverage matrix

| Task | branch-validate | manifest-parse | app-launch | playwright | i18n-coverage | static (Glob/Grep/Read) |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| 2-2 generate empty app | ✓ | ✓ (basic) | — | — | — | ✓ (Fiori scaffold files exist) |
| 2-3 JSON table | ✓ | ✓ | ✓ | ✓ (table renders ≥5 rows) | — | ✓ (model init in onInit) |
| 2-5 title edit | ✓ | — | ✓ | ✓ (click → input swap, save → text) | — | ✓ (Actions column markup) |
| 3-1 confirm dialog | ✓ | — | ✓ | ✓ (delete → dialog → Yes/No) | — | ✓ |
| 3-2 record creation | ✓ | — | ✓ | ✓ (fragment dialog + validation) | — | ✓ (Fragment.load) |
| 3-3 formatter & i18n | ✓ | ✓ (i18n bundle) | ✓ | ✓ (`Published: YYYY` regex) | ✓ | ✓ (formatter file) |
| 3-4 mock server | ✓ | ✓ (V2 + V4 dataSources) | ✓ (start-mock) | ✓ (network: metadata 200s) | — | ✓ (ui5-mock.yaml, package.json dep, metadata*.xml) |
| 3-5 OData binding | ✓ | ✓ (models bound) | ✓ (start-mock) | ✓ (IconTabBar 3 tabs, both tables populate) | — | ✓ (view structure) |
| 3-6 custom data (opt) | ✓ | ✓ | ✓ (start-mock) | ✓ (table shows custom values) | — | ✓ (mockdata files) |

This confirms `branch-validate`, `app-launch`, `playwright`, and `manifest-parse` are reused enough to belong in `shared/`. `i18n-coverage` covers one task today plus regression guarding for later spokes — also `shared/`. No skill ended up in `shared/` with only a single caller.

The curriculum tasks form a strict dependency chain (each PR builds on the merged previous task). The checker does not model the chain — each check evaluates one PR against one task's rubric. Skipped prerequisites surface naturally as failed criteria.

## Directory layout

```
checker/
├── .claude/
│   └── skills/
│       ├── check-task/SKILL.md
│       ├── check-task-2-2/SKILL.md
│       ├── check-task-2-3/SKILL.md
│       ├── check-task-2-5/SKILL.md
│       ├── check-task-3-1/SKILL.md
│       ├── check-task-3-2/SKILL.md
│       ├── check-task-3-3/SKILL.md
│       ├── check-task-3-4/SKILL.md
│       ├── check-task-3-5/SKILL.md
│       ├── check-task-3-6/SKILL.md
│       └── shared/
│           ├── pr-checkout/SKILL.md
│           ├── branch-validate/SKILL.md
│           ├── ui5-manifest-parse/SKILL.md
│           ├── ui5-app-launch/SKILL.md
│           ├── playwright-assertions/SKILL.md
│           ├── playwright-run-script/SKILL.md
│           ├── i18n-coverage/SKILL.md
│           └── report-format/SKILL.md
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-06-10-checker-design.md
└── README.md
```

## Naming conventions

- **Hub:** `check-task` (matches `/check-task` slash command).
- **Spokes:** `check-task-<module>-<task>` — direct map to the curriculum's `Task M-N` notation, no mental translation.
- **Shared:** `shared/<verb>-<noun>` — verbs first reads naturally in spoke runbooks.
- Each skill's `description` frontmatter starts with the trigger phrase a mentor would actually say (`"Use to check Task 3-2 PR..."`).

## SKILL.md size discipline

- **Hub:** short — routing only.
- **Spokes:** medium — verbatim task text + rubric + runbook. Target under ~250 lines.
- **Shared:** short to medium — single-purpose, explicit input/output contract at the top.

## Mentor entry points

- `/check-task <PR-url>` — default, auto-routes.
- `/check-task <PR-url> --task 3-2` — override task ID.
- `/check-task-3-2 <PR-url>` — direct spoke invocation (re-check, force-route).

## Future migration to GitHub/GitLab Actions (out of scope, but supported)

The shared skills are the right seams. When the time comes:

1. Each shared skill's behavior gets a thin Node script equivalent (`scripts/pr-checkout.mjs`, `scripts/manifest-parse.mjs`, etc.) — same input/output contract.
2. Each spoke's rubric becomes a YAML/JSON file (`rubrics/task-3-2.yaml`) mirroring today's rubric block.
3. A single `checker.yml` GitHub Action runs on PR open/sync: parse branch → load matching rubric → run scripts in sequence → post the formatted report as a PR comment.
4. The hub's branch-parsing logic moves into the workflow's `if:` conditions.

The key property that makes this clean: **no spoke owns runtime code today**. Spokes are rubric + runbook. Migration replaces the runner, keeps the rubrics.

## Build order recommendation

1. All `shared/*` skills.
2. **Task 2-3** spoke as the pilot — exercises every shared skill at minimum complexity. Any contract gaps surface here before being replicated.
3. The remaining spokes in curriculum order (2-2, 2-5, 3-1, 3-2, 3-3, 3-4, 3-5, 3-6).
4. Hub last — straightforward once the spokes exist.

## Extensibility — adding new task spokes

The architecture supports adding new tasks without structural changes:

- **New spoke = copy + rename + edit.** Copy any existing spoke directory, rename to `check-task-<m>-<n>`, replace task summary + rubric + runbook. No hub edit needed.
- **Hub auto-discovery.** The hub matches branches against `^feature/task-(\d+)-(\d+)$` and dispatches by name. New spokes are picked up via the session's available-skills list — no registry to update.
- **Shared skills stay pure.** Spokes consume them as-is. Novel browser checks use `shared/playwright-run-script` so adding a task never forces an edit to `shared/playwright-assertions`. Patterns that recur across two or more tasks should be promoted to a named recipe in `playwright-assertions`.
- **Branch-name format is the contract.** Every new task must use `feature/task-<m>-<n>`. Anything else requires `--task` override at invocation.
- **Cross-task dependencies surface as failed criteria.** Spokes don't model the curriculum's dependency chain. If a student skipped Task 3-2 and submits Task 3-3, the 3-3 rubric will fail naturally on missing prerequisites. Mentor reads "what's missing," not "you skipped a task."
- **Rubric versioning** is git history. If the curriculum updates a task next cohort, edit the spoke's rubric block; the diff is the changelog.

**Scale target:** the pure-skills design comfortably handles ~10–20 total tasks (the curriculum's likely growth over 1–2 cohorts of expansion). At that scale every spoke remains its own readable prompt and the shared set stays small. If spoke count grows much beyond ~25, revisit the runner choice (Option B in brainstorming — Node library backbone — becomes more attractive once token cost per check matters).

## Open questions for implementation phase

- Exact selectors used in `playwright-assertions` recipes — depend on what students actually emit (mostly `sap.m.*` defaults, but worth confirming on a sample repo).
- `npm install` timeout default — start at 5 minutes, adjust based on observation.
- Where the temp checkout lives if `/tmp` is not writable on a mentor's machine (fall back to `~/.cache/checker/`).
- Whether `report-format` should also produce a JSON variant from day one for future pipeline reuse, or wait until migration. Default: wait.
