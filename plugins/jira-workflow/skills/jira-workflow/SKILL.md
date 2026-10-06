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
5. If `jira-manager` reports TWG `AUTH_REQUIRED` mid-session, ask the user to run `! twg login`, re-dispatch once, then offer the MCP fallback.

## Execution Model — Delegate to Subagent (MANDATORY)

All Jira operations (field resolution, JQL queries, issue CRUD, transitions) MUST run in the **`jira-manager` subagent** (`agents/jira-manager.md`) on **both backends**. This keeps raw `twg` JSON output and the large MCP tool surface out of the main context.

**Orchestrator (main session) keeps only:**
- Backend selection (`twg whoami` probe) and TWG setup (`references/twg-setup.md`) — both need user interaction.
- `AskUserQuestion` to collect summary / epic / points / sprint / confirmation. NEVER ask the user from inside the subagent.
- Reading `./CLAUDE.md` `## Jira` block; prompt user once if missing, persist after confirmation.
- Surfacing the final key + URL + status to the user.

**Dispatch rule:** always spawn `jira-manager` with the cached `backend` (`twg` | `mcp`), a task name (`query-epics-sprints` | `resolve-ticket` | `sync-ticket` | `transition`), and pre-resolved inputs. The subagent never prompts the user and never switches backends — it accepts inputs or returns `NEEDS_CONTEXT` / `BLOCKED`.

**Handling `BLOCKED` from `jira-manager`:**
- TWG `AUTH_REQUIRED` / `twg` missing → ask the user to run `! twg login` (or run setup), then re-dispatch once; still blocked → offer the MCP fallback.
- MCP tools missing → offer TWG setup.

**Prompt template for `jira-manager`**

```
Backend: twg | mcp
Task: query-epics-sprints | resolve-ticket | sync-ticket | transition
Jira config (from ./CLAUDE.md ## Jira section):
  cloudId: <...>
  projectKey: <...>
  boardId: <...>
  storyPointsField: <...>
  epicLinkField: <...>
  sprintField: <...>
Site override (TWG only, optional): <prefix-or-cloudId>
Current accountId (cached): <...>
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
3. **Discover missing IDs** via `jira-manager` on the active backend (it returns `resolvedFieldIds`), then persist them.
4. **Current user**: cached from the orchestrator's `twg whoami` probe (TWG) or from `atlassianUserInfo` reported by `jira-manager` (MCP).

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
3. **If no** → dispatch `jira-manager` (task: `query-epics-sprints`) to fetch top 10 open Epics + active/future sprints for `boardId`. Receive compact options list back.
4. Single `AskUserQuestion` batch: **Summary**, **EPIC** (options + "Create new EPIC"), **Story points** (`1, 2, 3, 5, 8, 13` with suggested rationale), **Sprint** (options + "Backlog"), and **Issue type** when ambiguous (`Bug` if fix/error/broken/regression keywords; else `Task`/`Story`).
5. Confirm the resolved set with user if non-trivial.

**Subagent (`jira-manager`, task: `resolve-ticket`, dispatched with collected inputs):**
6. If "new EPIC": create the EPIC (type `Epic`), capture key.
7. Resolve custom-field IDs once if not cached.
8. Create the issue: type, summary, description, assignee = current user, parent/epic link = EPIC key, story points.
9. If sprint chosen: set the sprint.
10. Transition → **In Progress**.
11. Return `{ issueKey, url, summary, points, epic, sprint, status, resolvedFieldIds? }`.

**Orchestrator:** report key + URL + status to user; persist any newly resolved field IDs to `./CLAUDE.md` `## Jira` block.

Per-backend commands used by `jira-manager`: `references/twg-cli-cheatsheet.md` (TWG) or `references/atlassian-mcp-cheatsheet.md` (MCP).

## Workflow 2: Existing Ticket Sync

Use when user provides or mentions a Jira code (e.g. `SKY-123`).

**Subagent (`jira-manager`, task: `sync-ticket` — read pass):**
1. Fetch `summary`, `status`, `assignee`, story-points field, `parent`, sprint field.
2. Return current state: `{ key, summary, status, assignee, points, epic, sprint }`.

**Orchestrator (main session):**
3. Detect gaps: missing story points, missing EPIC, assignee != current user, status in `To Do`/`Open`/`Backlog`.
4. Batch one `AskUserQuestion`: confirm point value (Fibonacci), pick/create EPIC if missing, confirm reassignment, confirm transition to In Progress.

**Subagent (`jira-manager`, second dispatch with user's answers — write pass):**
5. Apply fixes (points, epic, assignee, sprint).
6. Transition → **In Progress** (match by name case-insensitive: "In Progress" / "Start Progress").
7. Return final synced state.

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
4. After PR opens, dispatch `jira-manager` (task: `transition`, name: "In Review") → matches "In Review" / "Code Review" / "Ready for Review" automatically.
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

1. `jira-manager` queries active + future sprints for the configured `boardId` (TWG: `jira board sprints query --state active|future`; MCP: agile API).
2. Present options as: `Active: <name>`, `Next: <name>`, `Backlog (no sprint)`.
3. Default suggestion: **current active sprint** unless user specifies otherwise.
4. `jira-manager` applies it via `update --sprint <id>` (TWG) or `editJiraIssue` setting the sprint customfield (MCP).

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
  3. Dispatch `jira-manager` (backend: twg, task: `query-epics-sprints`) → compact epic + sprint options.
  4. AskUserQuestion (single batch): summary, epic, points (suggest 5), sprint.
  5. Dispatch `jira-manager` (backend: twg, task: `resolve-ticket`) → creates SKY-241, sets sprint, transitions to In Progress.
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
  4. Resume Existing Ticket Sync on TWG: `jira-manager` read pass → AskUserQuestion for gaps → `jira-manager` write pass (update + transition).
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
  5. Dispatch `jira-manager` (backend: mcp, task: `transition`, "In Review").
  6. Reply: "PR opened. SKY-187 → In Review. Merge will auto-close to Done."
```

## Reference Files

- `references/twg-cli-cheatsheet.md` — verified `twg` commands for every operation (primary backend)
- `references/twg-setup.md` — TWG CLI + TWG skills install, login, verify
- `references/atlassian-mcp-cheatsheet.md` — concrete MCP tool calls + custom-field resolution (fallback backend)
- `references/claude-md-template.md` — exact `## Jira` block to insert into project CLAUDE.md
- `references/pr-title-patterns.md` — regex + branch-name parsing + edge cases
