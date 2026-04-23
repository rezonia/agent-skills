# PR Title Patterns

## Required format

```
[<JIRA-KEY>] <type>(<scope>)?: <short description>
```

- `<JIRA-KEY>` — uppercase project key + dash + number, e.g. `SKY-123`
- `<type>` — one of: `feat | fix | chore | docs | refactor | test | perf | build | ci | style`
- `<scope>` — optional, e.g. `(auth)`, `(payslip)`
- `<short description>` — imperative, lowercase first letter

## Validation regex

```
^\[[A-Z]+-\d+\]\s+(feat|fix|chore|docs|refactor|test|perf|build|ci|style)(\([\w\-]+\))?:\s+.+
```

## Branch-name parsing

Common patterns this skill recognises to auto-extract the Jira key:

| Branch                                    | Extracted key |
|-------------------------------------------|---------------|
| `feature/SKY-123-add-oauth`               | `SKY-123`     |
| `bugfix/SKY-456_scheduler_segfault`       | `SKY-456`     |
| `SKY-789/refactor-payslip`                | `SKY-789`     |
| `chore/sky-101-update-deps`               | `SKY-101` (uppercased) |
| `main` / `dev` / `release/*`              | none — ask user |

Regex used: `[A-Za-z]+-\d+` then uppercase.

## Examples

### Good
- `[SKY-123] feat: add OAuth login flow`
- `[SKY-456] fix(scheduler): resolve segfault on call-center sync timeout`
- `[SKY-789] refactor: extract payslip calculation into service`
- `[SKY-241] feat(reporting): daily revenue export`

### Bad → fix
| Bad                                       | Fix                                          |
|-------------------------------------------|----------------------------------------------|
| `add oauth login`                         | `[SKY-123] feat: add oauth login`           |
| `SKY-123 add oauth login`                 | `[SKY-123] feat: add oauth login`           |
| `[SKY-123]: add oauth login`              | `[SKY-123] feat: add oauth login`           |
| `[sky-123] feat: add oauth login`         | `[SKY-123] feat: add oauth login`           |

## Why this format matters

The repo-level Jira automation (Jira → GitHub integration / Smart Commits) scans **PR titles and merge commit messages** for the bracketed key pattern. When the merge commit lands on `main`, the matching Jira issue is auto-transitioned to **Done**. Without the key, the skill must perform that transition manually — adding friction and risking missed updates.

## Multi-issue PRs

If a PR addresses multiple tickets:
- Pick the **primary** ticket for the title.
- List additional tickets in the PR body: `Also closes: SKY-124, SKY-125`.
- Manually transition non-primary tickets after merge if automation does not pick them up.
