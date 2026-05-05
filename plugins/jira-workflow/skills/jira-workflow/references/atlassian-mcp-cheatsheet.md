# Atlassian MCP Cheatsheet (jira-workflow skill)

All tools below are namespaced `mcp__plugin_atlassian_atlassian__*`.

## Bootstrap (per session)

```
atlassianUserInfo()                                       # cache accountId
getAccessibleAtlassianResources()                         # confirm cloudId
```

## Discover existing tickets

### List recent EPICs
```
searchJiraIssuesUsingJql(
  cloudId, jql="project = SKY AND issuetype = Epic ORDER BY created DESC",
  fields=["summary","status"], maxResults=10
)
```

### Fetch one issue
```
getJiraIssue(
  cloudId, issueIdOrKey="SKY-123",
  fields=["summary","status","assignee","parent",
          "customfield_10016",   # story points
          "customfield_10020"]   # sprint
)
```

### Discover field IDs (run once, persist to CLAUDE.md)
```
getJiraProjectIssueTypesMetadata(cloudId, projectIdOrKey="SKY")
getJiraIssueTypeMetaWithFields(cloudId, projectIdOrKey="SKY", issueTypeId="<id>")
```

## Create

### New EPIC
```
createJiraIssue(
  cloudId, projectKey="SKY", issueTypeName="Epic",
  summary="<epic summary>",
  description="<context>",
  assignee_account_id="<cached>"
)
```

### New Task with EPIC + points
```
createJiraIssue(
  cloudId, projectKey="SKY", issueTypeName="Task",
  summary="<task summary>",
  description="<context + brainstorm output>",
  assignee_account_id="<cached>",
  additional_fields={
    "customfield_10016": 5,           # story points
    "customfield_10014": "SKY-100"    # epic link (classic) -- or use parent for next-gen
    # for next-gen: "parent": { "key": "SKY-100" }
  }
)
```

## Update

### Set assignee
```
editJiraIssue(cloudId, issueIdOrKey="SKY-123",
              fields={ "assignee": { "accountId": "<cached>" } })
```

### Set sprint
```
editJiraIssue(cloudId, issueIdOrKey="SKY-123",
              fields={ "customfield_10020": <sprintId> })
```

### Set story points (when missing)
```
editJiraIssue(cloudId, issueIdOrKey="SKY-123",
              fields={ "customfield_10016": 3 })
```

## Transitions

### List available transitions
```
getTransitionsForJiraIssue(cloudId, issueIdOrKey="SKY-123")
```

### Apply
```
transitionJiraIssue(cloudId, issueIdOrKey="SKY-123",
                    transition={ "id": "<id>" })
```

Match by transition `name` (case-insensitive):
- Start work → `In Progress`, `Start Progress`
- Open PR    → `In Review`, `Code Review`, `Ready for Review`
- Merge      → handled automatically by repo's Jira automation

## Error handling

| Error                                  | Recovery                                           |
|----------------------------------------|----------------------------------------------------|
| `field 'customfield_XXXXX' not found`  | Re-run `getJiraIssueTypeMetaWithFields`, update CLAUDE.md |
| `Transition not valid for this issue`  | Re-fetch transitions; status may have advanced     |
| `User does not have permission`        | Surface to user; do NOT retry blindly              |
| `Issue does not exist`                 | Confirm key spelling and project with user        |
