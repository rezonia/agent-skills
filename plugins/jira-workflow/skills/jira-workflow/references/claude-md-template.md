# CLAUDE.md `## Jira` Section Template

Insert this block into `./CLAUDE.md` so the skill can resolve project config without prompting on every invocation.

```markdown
## Jira

- **cloudId:** `rezlabs.atlassian.net`         # site host or UUID
- **projectKey:** `SKY`                         # primary project key
- **boardId:** `42`                             # numeric board ID for sprint queries
- **defaultIssueType:** `Task`                  # Task | Story | Bug
- **storyPointsField:** `customfield_10016`     # resolve once via getJiraIssueTypeMetaWithFields
- **epicLinkField:** `customfield_10014`        # parent for next-gen, epic-link for classic
- **sprintField:** `customfield_10020`
- **doneTransitionAuto:** `true`                # PR-merge automation handles Done
```

## How to populate

**TWG CLI (primary):**

1. `twg whoami -o json` — confirms the logged-in site. For TWG `--site`, use the site prefix (`rezlabs`) or the cloud ID UUID.
2. `twg jira board query --project <KEY> -o json` — find `boardId`.
3. `twg jira space issue-types --id-or-key <KEY> -o json` — confirm available issue types.
4. `twg jira workitem field create-metadata --space <KEY> --type Task -o json` — discover the **exact** customfield IDs for story points / epic link / sprint.
5. Persist findings to CLAUDE.md after the first session.

TWG `update` has native `--story-points`, `--sprint`, and `--parent` flags, but `create` needs `--field <storyPointsField>=N`, so keep `storyPointsField` populated.

**Atlassian MCP (fallback):**

1. Run `getAccessibleAtlassianResources` once to confirm cloudId.
2. Run `getJiraProjectIssueTypesMetadata` for projectKey to confirm available issue types.
3. Run `getJiraIssueTypeMetaWithFields` for the `Task` issuetype to discover the **exact** customfield IDs for story points / epic link / sprint (they vary by Jira instance).
4. Persist findings to CLAUDE.md after the first session.

## Field-ID gotchas

- **Next-gen / team-managed projects:** EPIC link is the standard `parent` field, not a customfield.
- **Classic projects:** EPIC link is typically `customfield_10014`.
- **Story points:** Most commonly `customfield_10016`, but instances vary — never hardcode without verifying.
- **Sprint:** Almost always `customfield_10020` for company-managed; team-managed expose it as `sprint`.
