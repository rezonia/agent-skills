# TWG CLI Cheatsheet (jira-workflow skill)

Primary backend. Every command below was checked against `twg <command> --help` for `@atlassian/twg-cli` 1.2.5. When unsure about a flag, run `twg help describe "<command path>"` before guessing.

Rules:
- Always pass `-o json`. Responses are JSON; failures return `{ "ok": false, "error": { "code", "message", "remediation": { "command" } } }`.
- Pass `--site <site-prefix-or-cloudId>` only when the `## Jira` block `cloudId` differs from the TWG default site. Convert a host like `rezlabs.atlassian.net` to its prefix (`rezlabs`); a UUID is passed as-is.
- Quote every user-provided value (summary, description, JQL) as a single shell argument.
- Do not use `twg api` for this workflow; the native commands cover every operation.

## Session bootstrap

```bash
twg whoami -o json          # backend probe + current user (cache the account ID)
twg doctor -o json          # optional: auth + connectivity diagnosis after a failure
```

## Configuration discovery

```bash
twg jira board query --project <KEY> -o json                              # find boardId
twg jira workitem field create-metadata --space <KEY> --type Task -o json # customfield IDs for create
twg jira workitem field update-metadata --id <KEY-123> -o json            # editable custom fields on an issue
twg jira space issue-types --id-or-key <KEY> -o json                      # available issue types
```

Look up story points / sprint / epic link by field name in the metadata output and persist the `customfield_*` IDs to the `## Jira` block.

## Discovery

```bash
# Recent open EPICs
twg jira workitem query --jql "project = <KEY> AND issuetype = Epic AND statusCategory != Done ORDER BY created DESC" -n 10 -o json

# Active + future sprints (run both)
twg jira board sprints query --board-id <boardId> --state active -o json
twg jira board sprints query --board-id <boardId> --state future -o json

# One issue (summary, status, assignee, parent, custom fields)
twg jira workitem get <KEY-123> -o json
```

## Create

```bash
# New EPIC
twg jira workitem create --space <KEY> --type Epic --summary "<epic summary>" \
  --description "<context>" --description-format markdown --assignee me -o json

# New Task/Story/Bug linked to EPIC, with story points
twg jira workitem create --space <KEY> --type Task --summary "<summary>" \
  --description "<context>" --description-format markdown --assignee me \
  --parent <EPIC-KEY> --field <storyPointsField>=<points> -o json
```

`create` has no `--story-points` flag. If `storyPointsField` is unknown, create without it, then set points with `update --story-points` (below).

Classic projects that reject `--parent` for epics: use `--field <epicLinkField>='"<EPIC-KEY>"'` (quoted JSON string).

## Update

```bash
twg jira workitem update --id <KEY-123> --story-points <points> -o json
twg jira workitem update --id <KEY-123> --parent <EPIC-KEY> -o json
twg jira workitem update --id <KEY-123> --assignee me -o json
twg jira workitem update --id <KEY-123> --sprint <sprintId> -o json
```

Flags combine in one call, e.g. `--story-points 3 --parent SKY-100 --assignee me`. If a native flag is rejected on a non-standard instance, use `--field <customfield_ID>=<value>`.

## Transitions

```bash
twg jira workitem transitions query --id <KEY-123> -o json                  # list valid transitions
twg jira workitem transition --id <KEY-123> --transition-id <id> -o json    # apply by ID (or exact status name)
```

Match transition names case-insensitively:
- Start work → `In Progress`, `Start Progress`
- Open PR → `In Review`, `Code Review`, `Ready for Review`
- Merge → handled by the repo's Jira automation; do not transition.

## Error handling

| Signal | Recovery |
|---|---|
| `twg: command not found` | Run the TWG setup workflow (`twg-setup.md`) or fall back to MCP |
| `error.code = AUTH_REQUIRED` | Ask user to run `! twg login` (interactive OAuth), retry once |
| Any error with `error.remediation.command` | Show that command to the user; run it via `!` if it is interactive |
| Field rejected / not found | Re-run `field create-metadata` or `update-metadata`, retry with `--field <id>=<value>`, update `## Jira` block |
| Transition not valid | Re-run `transitions query`; status may have moved |
| Permission denied | Surface to user; do not retry |
| Issue not found | Confirm key and project with user |
