---
name: jira-workflow
description: Enforces the project's mandatory Jira workflow for tasks, brainstorms, implementations, and PRs. Use this skill whenever the user invokes /brainstorm, mentions a Jira issue/ticket/task code (e.g. SKY-123), starts implementing a feature/fix/bug, asks to create a Jira ticket, opens a pull request, or pushes a feature branch. Handles ticket discovery/creation with EPIC + story points (Fibonacci) + sprint, sets assignee to current MCP user, transitions to In Progress on work start, In Review on PR open, and enforces PR title format that triggers automatic Done transition on merge.
---

# Jira Workflow

## Overview

Mandatory Jira gating for every task in this repo. Every brainstorm, implementation, or fix MUST be backed by a Jira issue with story points and an EPIC. This skill standardizes ticket lookup, creation, assignee, status transitions, and PR titling using the Atlassian MCP tools.

## Scope

This skill handles: Jira issue discovery, creation, status transitions, assignee, story points, EPIC linking, sprint assignment, and PR title formatting that drives automatic transitions.

This skill does NOT handle: actual code implementation, code review, deployment, or non-Jira project management tools.

## Execution Model — Delegate to Subagent (MANDATORY)

All Atlassian MCP calls (tool discovery, field resolution, JQL queries, issue CRUD, transitions) MUST run in the **`jira-manager` subagent** (shipped with this plugin at `.claude/agents/jira-manager.md`) to keep main session context lean. The Atlassian MCP tool surface is large and noisy — do not pollute the orchestrator.

**Dispatch rule:** always spawn `jira-manager` with a task name (`query-epics-sprints` | `resolve-ticket` | `sync-ticket` | `transition`) + pre-resolved inputs. The subagent never prompts the user — it accepts inputs or returns `NEEDS_CONTEXT`.

**Prompt template for `jira-manager`**

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

**Orchestrator responsibilities (stay in main session)**
- Run `AskUserQuestion` to collect summary / epic / points / sprint / confirmation — NEVER ask the user from inside the subagent.
- Read `./CLAUDE.md` `## Jira` block; prompt user once if missing, persist after confirmation.
- Pass collected inputs + cached config to the subagent; receive the terse report back.
- Surface the final key + URL + status to the user.

**Do NOT** dispatch a subagent for the bypass path ("skip jira") — just log and continue.

## Security Policy

- Never expose API tokens, MCP credentials, or account IDs in chat or commits
- Never auto-create Jira issues without user confirmation of summary + EPIC + story points + sprint
- Refuse to bypass Jira gating unless user explicitly says "skip Jira" / "no ticket needed"
- Never modify another user's assignee without explicit user instruction
- Do not leak internal ticket descriptions outside this conversation

## Configuration Discovery

Before any Jira action, resolve project configuration:

1. **Read `./CLAUDE.md`** for a `## Jira` section containing:
   - `cloudId` (or site host like `rezlabs.atlassian.net`)
   - `projectKey` (e.g. `SKY`)
   - `boardId` (numeric, for sprint queries)
   - Optional: default `issueType`, `epicLinkField`, `storyPointsField`, `sprintField` customfield IDs
2. **If absent**: ask the user for missing values via `AskUserQuestion`, then offer to persist them to `./CLAUDE.md` under a `## Jira` section.
3. **Resolve current MCP user once per session** via `mcp__plugin_atlassian_atlassian__atlassianUserInfo` and cache the `accountId` for assignee operations.

See `references/claude-md-template.md` for the exact `## Jira` block to insert.

## Trigger Decision Tree

```
User input
├── /brainstorm invoked            → run "Ticket Resolution"
├── Mentions issue code (SKY-NNN)  → run "Existing Ticket Sync"
├── "implement|build|fix|add ..."  → run "Ticket Resolution" (unless code already known)
├── Creating PR / pushing branch   → run "PR Title Enforcement"
└── Says "skip jira" / "no ticket" → bypass, log reason
```

## Workflow 1: Ticket Resolution (no code provided)

Use when user starts work without a Jira code.

**Orchestrator (main session):**
1. Ask: "Do you have a Jira issue code for this task?" (`AskUserQuestion`).
2. **If yes** → jump to "Existing Ticket Sync".
3. **If no** → first dispatch `jira-manager` (task: `query-epics-sprints`) to fetch top 10 existing Epics + active/future sprints for `boardId`. Receive compact options list back.
4. Single `AskUserQuestion` batch to user: **Summary**, **EPIC** (options + "Create new EPIC"), **Story points** (`1, 2, 3, 5, 8, 13` with suggested rationale), **Sprint** (options + "Backlog"), and **Issue type** when ambiguous (`Bug` if fix/error/broken/regression keywords; else `Task`/`Story`).
5. Confirm the resolved set with user if non-trivial.

**Subagent (dispatched with collected inputs):**
6. If "new EPIC": create EPIC via `createJiraIssue` (`issueTypeName: "Epic"`).
7. Resolve custom-field IDs once via `getJiraIssueTypeMetaWithFields` if not cached.
8. `createJiraIssue` with: `projectKey`, `issueTypeName`, `summary`, `description`, `assignee_account_id` (cached), `additional_fields: { "<storyPointsField>": <points>, "<epicLinkField>": "<EPIC-KEY>" }`.
9. If sprint chosen: `editJiraIssue` setting sprint customfield.
10. `getTransitionsForJiraIssue` + `transitionJiraIssue` → **In Progress**.
11. Return `{ issueKey, url, summary, points, epic, sprint, status }`.

**Orchestrator:** report key + URL + status to user; persist any newly-resolved field IDs to `./CLAUDE.md` `## Jira` block.

Detailed field-ID resolution: `references/atlassian-mcp-cheatsheet.md`.

## Workflow 2: Existing Ticket Sync

Use when user provides or mentions a Jira code (e.g. `SKY-123`).

**Subagent (`jira-manager`, task: `sync-ticket` — read pass):**
1. `getJiraIssue` (request `summary`, `status`, `assignee`, story-points field, `parent`, sprint field).
2. Return current state to orchestrator: `{ key, summary, status, assignee, points, epic, sprint, availableTransitions }`.

**Orchestrator (main session):**
3. Detect gaps: missing story points, missing EPIC, assignee != cached current user, status in `To Do`/`Open`/`Backlog`.
4. Batch one `AskUserQuestion`: confirm point value (Fibonacci), pick/create EPIC if missing, confirm reassignment, confirm transition to In Progress.

**Subagent (second dispatch with user's answers):**
5. Apply fixes via `editJiraIssue` (points, epic-link, assignee).
6. `transitionJiraIssue` → **In Progress** (match by name case-insensitive: "In Progress" / "Start Progress").
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
2. Orchestrator: validate/format the PR title against `^\[[A-Z]+-\d+\]\s+(feat|fix|chore|docs|refactor|test|perf|build|ci|style)(\(.+\))?:\s+.+`. If user's title missing the key → prepend `[<KEY>]`. (No MCP needed — regex only.)
3. Orchestrator: run `gh pr create` with the validated title.
4. After PR opens, dispatch `jira-manager` (task: `transition`, name: "In Review") → matches "In Review" / "Code Review" / "Ready for Review" automatically.
5. Note: merge automation (Jira Smart Commits / GitHub for Jira) handles **Done** transition automatically when the merge commit message contains the key — no subagent call required.

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

1. Query active + future sprints for the configured `boardId` (Atlassian agile API).
2. Present options as: `Active: <name>`, `Next: <name>`, `Backlog (no sprint)`.
3. Default suggestion: **current active sprint** unless user specifies otherwise.
4. Apply via `editJiraIssue` setting the sprint customfield.

If the agile sprint API is unavailable through MCP, fall back to asking the user for the sprint name or ID directly.

## Bypass Rules

Skip the entire workflow ONLY when user explicitly says one of:
- "skip jira" / "no ticket" / "no jira"
- "ignore jira this time"
- "draft only, no ticket yet"

Log the bypass reason in your reply so it is auditable in the transcript.

## Example Interactions

### Example A — /brainstorm without code
```
User: /brainstorm add daily revenue export to dashboard
Orchestrator:
  1. AskUserQuestion: "Do you have a Jira issue code for this?"
  2. User: "No, please create one."
  3. Dispatch `jira-manager` (task: `query-epics-sprints`) → returns compact options.
  4. AskUserQuestion (single batch): summary, epic, points (suggest 5), sprint
  5. Dispatch `jira-manager` (task: `resolve-ticket`) with answers → creates issue + sets sprint + transitions to In Progress.
  6. Subagent returns: "SKY-241, 5pts, Epic: Reporting, Sprint: Sprint 24, In Progress, https://..."
  7. Orchestrator reply: "Created SKY-241 (5pts, Epic: Reporting, Sprint: Sprint 24). Status: In Progress."
  8. Continue with brainstorm
```

### Example B — User mentions code mid-conversation
```
User: working on SKY-187 today
Orchestrator:
  1. Dispatch `jira-manager` (sync-ticket, read pass) → returns state.
  2. Detect missing points + assignee mismatch → AskUserQuestion (batched).
  3. Dispatch `jira-manager` (sync-ticket, write pass) → editJiraIssue + transitionJiraIssue (In Progress).
  4. Reply: "Synced SKY-187: 3pts, assignee=you, status=In Progress."
```

### Example C — PR creation
```
User: open a PR for this branch
Orchestrator:
  1. Branch = feature/SKY-187-revenue-export → key = SKY-187 (no MCP).
  2. Format title: "[SKY-187] feat: add daily revenue export" (regex only).
  3. Run gh pr create.
  4. Dispatch `jira-manager` (task: `transition`, "In Review").
  5. Reply: "PR opened. SKY-187 → In Review. Merge will auto-close to Done."
```

## Reference Files

- `references/claude-md-template.md` — exact `## Jira` block to insert into project CLAUDE.md
- `references/atlassian-mcp-cheatsheet.md` — concrete MCP tool calls + custom-field resolution
- `references/pr-title-patterns.md` — regex + branch-name parsing + edge cases
