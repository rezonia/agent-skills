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

1. Ask: "Do you have a Jira issue code for this task?" (`AskUserQuestion`).
2. **If yes** → jump to "Existing Ticket Sync".
3. **If no** → propose a new ticket. Gather in one `AskUserQuestion` batch:
   - **Summary** — concise imperative title.
   - **EPIC** — query existing epics via `searchJiraIssuesUsingJql` with `project = <KEY> AND issuetype = Epic ORDER BY created DESC` (top 10), present as options + "Create new EPIC".
   - **Story points (Fibonacci)** — suggest a value with rationale, options: `1, 2, 3, 5, 8, 13`.
   - **Sprint** — query active + future sprints, present as options + "Backlog".
4. Confirm summary + EPIC + points + sprint with user.
5. Create the ticket via `createJiraIssue`:
   - `projectKey`.
   - `issueTypeName`: infer from intent — `"Bug"` if user mentions fix/bug/error/broken/regression/issue, otherwise `"Task"` (or user-chosen `Story`). Confirm in the AskUserQuestion batch when ambiguous.
   - `summary`, `description` (include brainstorm context).
   - `assignee_account_id`: cached current-user accountId.
   - `additional_fields`: `{ "<storyPointsField>": <points>, "<epicLinkField>": "<EPIC-KEY>" }` — resolve field IDs once via `getJiraIssueTypeMetaWithFields` and cache in CLAUDE.md.
6. If "new EPIC" chosen: create EPIC first (`issueTypeName: "Epic"`), then link the task.
7. If a sprint chosen: edit the issue with `editJiraIssue` setting the sprint customfield.
8. Transition the new issue to **In Progress** via `getTransitionsForJiraIssue` + `transitionJiraIssue`.
9. Report the issue key + URL back to the user.

Detailed field-ID resolution: `references/atlassian-mcp-cheatsheet.md`.

## Workflow 2: Existing Ticket Sync

Use when user provides or mentions a Jira code (e.g. `SKY-123`).

1. Fetch via `getJiraIssue` (request `summary`, `status`, `assignee`, story-points field, `parent`, sprint field).
2. **Verify story points present.** If missing → ask user (Fibonacci options) and `editJiraIssue` to set it.
3. **Verify EPIC link present** (parent for next-gen, or epic-link customfield). If missing → run EPIC selection sub-flow from Workflow 1, then link.
4. **Verify assignee = current MCP user.** If not → ask "Reassign SKY-123 to you?" → if yes, `editJiraIssue` with `{ "assignee": { "accountId": "<cached>" } }`.
5. **Transition status to "In Progress"** if currently `To Do` / `Open` / `Backlog`:
   - Get available transitions via `getTransitionsForJiraIssue`.
   - Match by name (case-insensitive) for "In Progress" / "Start Progress".
   - Apply via `transitionJiraIssue`.
6. Confirm sync summary to user (key, summary, points, epic, sprint, status).

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
1. Determine the active Jira key (from session context, branch name `feature/SKY-123-...`, or ask).
2. Validate proposed title matches `^\[[A-Z]+-\d+\]\s+(feat|fix|chore|docs|refactor|test|perf|build|ci|style)(\(.+\))?:\s+.+`.
3. If user provided a title without the key → prepend `[<KEY>]`.
4. After PR is opened, transition Jira issue to **In Review** (look for transitions named "In Review", "Code Review", or "Ready for Review").
5. Note: merge automation (Jira Smart Commits / GitHub for Jira) handles **Done** transition automatically when the merge commit message contains the key.

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
Skill:
  1. AskUserQuestion: "Do you have a Jira issue code for this?"
  2. User: "No, please create one."
  3. Query existing Epics → present 5 options + "Create new"
  4. AskUserQuestion: summary, epic, points (suggest 5), sprint
  5. createJiraIssue → SKY-241
  6. transitionJiraIssue → In Progress
  7. Reply: "Created SKY-241 (5pts, Epic: Reporting, Sprint: Sprint 24). Status: In Progress."
  8. Continue with brainstorm
```

### Example B — User mentions code mid-conversation
```
User: working on SKY-187 today
Skill:
  1. getJiraIssue SKY-187
  2. Detect missing story points → ask, set to 3
  3. Detect assignee != current user → ask, reassign
  4. transitionJiraIssue → In Progress
  5. Reply: "Synced SKY-187: 3pts, assignee=you, status=In Progress."
```

### Example C — PR creation
```
User: open a PR for this branch
Skill:
  1. Branch = feature/SKY-187-revenue-export → key = SKY-187
  2. Format title: "[SKY-187] feat: add daily revenue export"
  3. After gh pr create succeeds → transitionJiraIssue → In Review
  4. Reply: "PR opened. SKY-187 → In Review. Merge will auto-close to Done."
```

## Reference Files

- `references/claude-md-template.md` — exact `## Jira` block to insert into project CLAUDE.md
- `references/atlassian-mcp-cheatsheet.md` — concrete MCP tool calls + custom-field resolution
- `references/pr-title-patterns.md` — regex + branch-name parsing + edge cases
