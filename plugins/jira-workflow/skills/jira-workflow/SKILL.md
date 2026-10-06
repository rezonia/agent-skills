---
name: jira-workflow
description: Enforces the project's mandatory Jira workflow for tasks, brainstorms, implementations, and PRs. Use this skill whenever the user invokes /brainstorm, mentions a Jira issue/ticket/task code (e.g. SKY-123), starts implementing a feature/fix/bug, asks to create a Jira ticket, opens a pull request, or pushes a feature branch. Handles ticket discovery/creation with EPIC + story points (Fibonacci) + sprint, sets assignee to the current Jira user, transitions to In Progress on work start, In Review on PR open, and enforces PR title format that triggers automatic Done transition on merge. Runs on Atlassian TWG CLI (`twg`) first with Atlassian MCP fallback, and includes a TWG CLI + TWG skills setup workflow (also triggers on "set up twg" or when the twg command is not found).
---

# Jira Workflow

## Overview

Mandatory Jira gating for every task in this repo. Every brainstorm, implementation, or fix MUST be backed by a Jira issue with story points and an EPIC. This skill standardizes ticket lookup, creation, assignee, status transitions, and PR titling.

## Scope

This skill handles: Jira issue discovery, creation, status transitions, assignee, story points, EPIC linking, sprint assignment, PR title formatting that drives automatic transitions, and TWG CLI setup for this workflow.

This skill does NOT handle: actual code implementation, code review, deployment, non-Jira TWG features (Confluence, Bitbucket, Loom, etc.), or non-Jira project management tools.

## Prerequisites

Two backends. TWG CLI has priority; Atlassian MCP is the fallback.

- **TWG CLI (primary):** `twg` on `PATH` (`@atlassian/twg-cli`, verified with 1.2.5), logged in via `twg login`, with TWG agent skills installed (`twg skills install`). Missing pieces are handled by the setup workflow in `references/twg-setup.md`.
- **Atlassian MCP (fallback):** Atlassian MCP server installed and authenticated in Claude Code, exposing `mcp__plugin_atlassian_atlassian__*` tools. Used only through the `jira-manager` subagent, which inherits Claude Code's tool surface.

## Backend Selection

No upfront check. Select the backend lazily on the **first Jira command of the session**, then cache it:

1. Run `twg whoami -o json`.
2. **Success** → backend = **TWG**. Cache the account ID. Log once: `Jira backend: TWG CLI`.
3. **Failure** (`command not found`, `AUTH_REQUIRED`, or any error):
   - Atlassian MCP tools available → one `AskUserQuestion`: **Set up TWG now (Recommended)** / **Use Atlassian MCP this session**. On MCP, log `Jira backend: Atlassian MCP (fallback) — <reason>`.
   - MCP not available → run the setup workflow (`references/twg-setup.md`).
   - Setup declined and no MCP → stop and tell the user to set up TWG or the Atlassian MCP. Do not improvise via `curl`.
4. Never fall back silently. Always log which backend is active and why.
5. If a TWG command fails with `AUTH_REQUIRED` mid-session, ask the user to run `! twg login`, retry once, then offer the MCP fallback.

## Execution Model

**Orchestrator (main session, both backends):**
- Run `AskUserQuestion` to collect summary / epic / points / sprint / confirmation.
- Read `./CLAUDE.md` `## Jira` block; prompt user once if missing, persist after confirmation.
- Surface the final key + URL + status to the user.

**TWG backend:** run `twg ... -o json` commands directly via Bash in the main session. Output is compact JSON, so no subagent is needed. Command reference: `references/twg-cli-cheatsheet.md`.

**MCP backend:** all MCP calls MUST run in the **`jira-manager` subagent** (`agents/jira-manager.md`) to keep the large MCP tool surface out of the main context. Dispatch with a task name (`query-epics-sprints` | `resolve-ticket` | `sync-ticket` | `transition`) + pre-resolved inputs. The subagent never prompts the user — it accepts inputs or returns `NEEDS_CONTEXT`.

**Prompt template for `jira-manager` (MCP only)**

```
Task: query-epics-sprints | resolve-ticket | sync-ticket | transition
Jira config (from ./CLAUDE.md ## Jira section):
  cloudId: <...>
  projectKey: <...>
  boardId: <...>
  storyPointsField: <...>
  epicLinkField: <...>
  sprintField: <...>
Current MCP accountId (cached): <...>
Inputs: <issue key / summary / epic / points / sprint / transition name>
Expected output: terse structured summary per jira-manager contract.
Work context: <repo path>
```

**Do NOT** run any Jira command for the bypass path ("skip jira") — just log and continue.

## Security Policy

- Never expose API tokens, OAuth credentials, or account IDs in chat or commits. Never read or print `~/.config/twg/auth.conf`.
- Never pass secrets as CLI flags or ask the user to paste tokens in chat; `twg login` is interactive and the user runs it via `!`.
- Install TWG CLI or TWG skills only after user confirmation, using the pinned package `@atlassian/twg-cli@1.2.5`.
- Quote all user-provided values (summary, description, JQL) as single shell arguments; never build commands by unquoted interpolation.
- Never auto-create Jira issues without user confirmation of summary + EPIC + story points + sprint.
- Refuse to bypass Jira gating unless user explicitly says "skip Jira" / "no ticket needed".
- Never modify another user's assignee without explicit user instruction; assign only to the current user (`--assignee me` on TWG).
- Do not leak internal ticket descriptions outside this conversation.

## Configuration Discovery

Before any Jira action, resolve project configuration:

1. **Read `./CLAUDE.md`** for a `## Jira` section containing:
   - `cloudId` (site host like `rezlabs.atlassian.net`, or UUID)
   - `projectKey` (e.g. `SKY`)
   - `boardId` (numeric, for sprint queries)
   - Optional: default `issueType`, `epicLinkField`, `storyPointsField`, `sprintField` customfield IDs
2. **If absent**: ask the user for missing values via `AskUserQuestion`, then offer to persist them to `./CLAUDE.md` under a `## Jira` section.
3. **Discover missing IDs** with the active backend (TWG: `jira board query`, `jira workitem field create-metadata`; MCP: `references/atlassian-mcp-cheatsheet.md`), then persist them.
4. **Current user**: cached from `twg whoami` (TWG) or `atlassianUserInfo` (MCP, inside `jira-manager`).

See `references/claude-md-template.md` for the exact `## Jira` block to insert.

## Trigger Decision Tree

```
User input
├── /brainstorm invoked            → run "Ticket Resolution"
├── Mentions issue code (SKY-NNN)  → run "Existing Ticket Sync"
├── "implement|build|fix|add ..."  → run "Ticket Resolution" (unless code already known)
├── Creating PR / pushing branch   → run "PR Title Enforcement"
├── "set up twg" / twg missing     → run "TWG Setup" (references/twg-setup.md)
└── Says "skip jira" / "no ticket" → bypass, log reason
```

## Workflow 1: Ticket Resolution (no code provided)

Use when user starts work without a Jira code.

**Orchestrator (main session):**
1. Ask: "Do you have a Jira issue code for this task?" (`AskUserQuestion`).
2. **If yes** → jump to "Existing Ticket Sync".
3. **If no** → fetch top 10 open Epics + active/future sprints for `boardId`:
   - TWG: `twg jira workitem query --jql "..." -n 10` and `twg jira board sprints query --board-id <id> --state active|future`.
   - MCP: dispatch `jira-manager` (task: `query-epics-sprints`).
4. Single `AskUserQuestion` batch: **Summary**, **EPIC** (options + "Create new EPIC"), **Story points** (`1, 2, 3, 5, 8, 13` with suggested rationale), **Sprint** (options + "Backlog"), and **Issue type** when ambiguous (`Bug` if fix/error/broken/regression keywords; else `Task`/`Story`).
5. Confirm the resolved set with user if non-trivial.

**Execute (TWG: Bash in main session; MCP: `jira-manager` task `resolve-ticket`):**
6. If "new EPIC": create the EPIC (type `Epic`), capture key.
7. Resolve custom-field IDs once if not cached (TWG: `field create-metadata`; MCP: `getJiraIssueTypeMetaWithFields`).
8. Create the issue: type, summary, description, assignee = current user, parent/epic link = EPIC key, story points.
   - TWG: `create ... --assignee me --parent <EPIC> --field <storyPointsField>=<points>` (`create` has no `--story-points`; if the field ID is unknown, set points afterwards with `update --story-points`).
9. If sprint chosen: set it (TWG: `update --sprint <id>`; MCP: `editJiraIssue` sprint field).
10. Transition → **In Progress** (TWG: `transitions query` + `transition --transition-id`).
11. Collect `{ issueKey, url, summary, points, epic, sprint, status }`.

**Orchestrator:** report key + URL + status to user; persist any newly resolved field IDs to `./CLAUDE.md` `## Jira` block.

Command details: `references/twg-cli-cheatsheet.md` (TWG) or `references/atlassian-mcp-cheatsheet.md` (MCP).

## Workflow 2: Existing Ticket Sync

Use when user provides or mentions a Jira code (e.g. `SKY-123`).

**Read pass (TWG: `twg jira workitem get <KEY> -o json`; MCP: `jira-manager` task `sync-ticket`):**
1. Fetch `summary`, `status`, `assignee`, story-points field, `parent`, sprint field.
2. Build current state: `{ key, summary, status, assignee, points, epic, sprint }`.

**Orchestrator (main session):**
3. Detect gaps: missing story points, missing EPIC, assignee != current user, status in `To Do`/`Open`/`Backlog`.
4. Batch one `AskUserQuestion`: confirm point value (Fibonacci), pick/create EPIC if missing, confirm reassignment, confirm transition to In Progress.

**Write pass (TWG: one `update` with combined flags; MCP: second `jira-manager` dispatch):**
5. Apply fixes (points, epic, assignee). TWG: `update --id <KEY> --story-points N --parent <EPIC> --assignee me`.
6. Transition → **In Progress** (match by name case-insensitive: "In Progress" / "Start Progress").
7. Collect final synced state.

**Orchestrator:** confirm sync summary to user (key, summary, points, epic, sprint, status).

## Workflow 3: PR Title Enforcement

Use when user runs `gh pr create`, mentions opening a PR, or pushes a feature branch.

**Required title format:**
```
[<JIRA-KEY>] <type>: <short description>
```
Examples:
- `[SKY-123] feat: add OAuth login flow`
- `[SKY-456] fix: resolve scheduler segfault`
- `[SKY-789] refactor: extract payslip service`

Steps:
1. Orchestrator: determine active Jira key (from session context, branch name `feature/SKY-123-...`, or ask user).
2. Orchestrator: validate/format the PR title against `^\[[A-Z]+-\d+\]\s+(feat|fix|chore|docs|refactor|test|perf|build|ci|style)(\(.+\))?:\s+.+`. If user's title missing the key → prepend `[<KEY>]`. (No Jira call — regex only.)
3. Orchestrator: run `gh pr create` with the validated title.
4. After PR opens, transition → **In Review** (matches "In Review" / "Code Review" / "Ready for Review"):
   - TWG: `twg jira workitem transitions query --id <KEY>` + `twg jira workitem transition --id <KEY> --transition-id <id>`.
   - MCP: dispatch `jira-manager` (task: `transition`, name: "In Review").
5. Note: merge automation (Jira Smart Commits / GitHub for Jira) handles **Done** transition automatically when the merge commit message contains the key — no Jira call required.

See `references/pr-title-patterns.md` for full regex + edge cases.

## Status State Machine

```
To Do ──start work──▶ In Progress ──open PR──▶ In Review ──merge PR──▶ Done
                          ▲                                              │
                          └──────────reopen / revert────────────────────┘
```

This skill drives the first two transitions explicitly. The merge → Done transition relies on the PR title containing the Jira key plus repository-level Jira automation.

## Story Point Suggestion Heuristic (Fibonacci)

| Points | Effort                                  | Examples                                       |
|-------:|-----------------------------------------|------------------------------------------------|
| 1      | < 1 hr, single file, trivial            | typo fix, copy update                          |
| 2      | 1–3 hr, isolated change                 | small Filament action, new validation rule     |
| 3      | half day, single module                 | new resource page, simple service              |
| 5      | 1 day, multi-file/module                | new feature with tests, schema migration       |
| 8      | 2–3 days, cross-module                  | new domain area, integration with vendor       |
| 13     | 1 week, architectural / risky           | refactor of core flow, new payment provider    |

Always present a suggested value with one-line rationale, then let user confirm or override.

## Sprint Selection

1. Query active + future sprints for the configured `boardId` (TWG: `jira board sprints query --state active|future`; MCP: agile API via `jira-manager`).
2. Present options as: `Active: <name>`, `Next: <name>`, `Backlog (no sprint)`.
3. Default suggestion: **current active sprint** unless user specifies otherwise.
4. Apply via `update --sprint <id>` (TWG) or `editJiraIssue` setting the sprint customfield (MCP).

If the sprint query fails on either backend, fall back to asking the user for the sprint name or ID directly.

## Bypass Rules

Skip the entire workflow ONLY when user explicitly says one of:
- "skip jira" / "no ticket" / "no jira"
- "ignore jira this time"
- "draft only, no ticket yet"

Log the bypass reason in your reply so it is auditable in the transcript.

## Example Interactions

### Example A — /brainstorm without code (TWG)
```
User: /brainstorm add daily revenue export to dashboard
Orchestrator:
  1. AskUserQuestion: "Do you have a Jira issue code for this?" → "No, please create one."
  2. twg whoami -o json → OK. Log "Jira backend: TWG CLI".
  3. twg jira workitem query (epics) + twg jira board sprints query (active, future).
  4. AskUserQuestion (single batch): summary, epic, points (suggest 5), sprint.
  5. twg jira workitem create ... → SKY-241; update --sprint; transition → In Progress.
  6. Reply: "Created SKY-241 (5pts, Epic: Reporting, Sprint: Sprint 24). Status: In Progress."
  7. Continue with brainstorm.
```

### Example B — TWG not set up
```
User: working on SKY-187 today
Orchestrator:
  1. twg whoami -o json → "twg: command not found".
  2. AskUserQuestion: Set up TWG now (Recommended) / Use Atlassian MCP this session.
  3. User picks setup → references/twg-setup.md (install CLI, install TWG skills, `! twg login`, verify).
  4. Resume Existing Ticket Sync on TWG: get → AskUserQuestion for gaps → update + transition.
  5. Reply: "Synced SKY-187: 3pts, assignee=you, status=In Progress."
```

### Example C — PR creation (MCP fallback)
```
User: open a PR for this branch
Orchestrator:
  1. Backend cached: Atlassian MCP (fallback — user chose MCP this session).
  2. Branch = feature/SKY-187-revenue-export → key = SKY-187 (no Jira call).
  3. Format title: "[SKY-187] feat: add daily revenue export" (regex only).
  4. Run gh pr create.
  5. Dispatch `jira-manager` (task: `transition`, "In Review").
  6. Reply: "PR opened. SKY-187 → In Review. Merge will auto-close to Done."
```

## Reference Files

- `references/twg-cli-cheatsheet.md` — verified `twg` commands for every operation (primary backend)
- `references/twg-setup.md` — TWG CLI + TWG skills install, login, verify
- `references/atlassian-mcp-cheatsheet.md` — concrete MCP tool calls + custom-field resolution (fallback backend)
- `references/claude-md-template.md` — exact `## Jira` block to insert into project CLAUDE.md
- `references/pr-title-patterns.md` — regex + branch-name parsing + edge cases
