# Rezonia Agent Skills

Private Claude Code skills marketplace for Rezonia engineering workflows.

## Skills

| Skill | Description |
|---|---|
| [jira-workflow](./plugins/jira-workflow) | Mandatory Jira gating for brainstorms, implementations, and PRs. Ticket creation/sync with EPIC + Fibonacci story points + sprint, assignee = current Jira user, status transitions, PR title enforcement. Atlassian TWG CLI first, Atlassian MCP fallback, built-in TWG setup workflow. |
| [laravel-php-guidelines](./plugins/laravel-php-guidelines) | PHP + Laravel coding standards: PER style, strict typing, early-return control flow, Laravel helpers, Carbon, PHP 8 attributes, translation, routes/config/enums, full file & class naming conventions. |
| [google-admob-kmp](./plugins/google-admob-kmp) | Google Mobile Ads (AdMob) for Kotlin Multiplatform: banner, interstitial, native, app open, rewarded ads on Android and iOS. Covers iOS Swift bridge pattern, UMP/GDPR/ATT consent, SDK v12+ type renames, and Compose Multiplatform rendering. |
| [filament-guidelines](./plugins/filament-guidelines) | Filament 5.x conventions: simple resources, model policy per resource, `$action` notification pattern, `fas-*` icons, SPA/wire:navigate compatibility, dark mode, Filament Blade components, schema/table class extraction. |
| [rez](./plugins/rez) | Shared Rezonia org-wide skills: `rez:create-gh-release` (backfill GitHub Release pages for existing tags) and `rez:pr-review-resolver` (resolve PR review threads end-to-end — fetch unresolved threads, verify each comment, fix/push-back, commit, push, reply, resolve), and `rez:plan-decision-brief` (Vietnamese decision brief pinned at the top of every plan and printed before fixes: key decisions, scope, breaking changes, risks, scalability, design patterns, plus coding/backend/frontend/fullstack items). Standalone via gh + git; auto-enhances with claudekit when present. |

## Install as a Claude Code plugin marketplace

In Claude Code:

```
/plugin marketplace add rezonia/agent-skills
/plugin install jira-workflow@rezonia-agent-skills
```

Or via CLI:

```bash
claude plugin marketplace add rezonia/agent-skills
claude plugin install jira-workflow@rezonia-agent-skills
```

Private repo requires your GitHub auth to have read access to `rezonia/agent-skills`.

> **`jira-workflow` prerequisite:** the skill uses the **Atlassian TWG CLI** first and falls back to the **Atlassian MCP**. Recommended setup:
>
> ```bash
> npm install -g @atlassian/twg-cli@1.2.5
> twg skills install --agent claude -y
> twg login
> ```
>
> If `twg` is missing or not logged in, the skill offers to run this setup for you (you run the interactive `twg login`), or to use the Atlassian MCP for the session when it is installed and authenticated.

## Install with `npx skills`

Works for Claude Code and any other agent the [`skills` CLI](https://github.com/vercel-labs/skills) supports (Codex, Cursor, OpenCode, ...). It discovers every skill in this repo through `.claude-plugin/marketplace.json`.

```bash
# See available skills
npx skills add rezonia/agent-skills --list

# Install one or more skills (project scope)
npx skills add rezonia/agent-skills --skill jira-workflow --skill laravel-php-guidelines

# Install globally for Claude Code only
npx skills add rezonia/agent-skills --skill pr-review-resolver -g -a claude-code

# Update installed skills later
npx skills update
```

Private repo: the CLI uses your existing Git credentials, GitHub CLI login, or SSH. You can also pass `git@github.com:rezonia/agent-skills.git` as the source.

Differences from the plugin install:

- Skills are installed by their folder name: `rez:create-gh-release` becomes `create-gh-release`, `rez:pr-review-resolver` becomes `pr-review-resolver`, `rez:plan-decision-brief` becomes `plan-decision-brief`.
- Only skill folders are copied, not plugin agents. `jira-workflow` handles this itself: without the `jira-manager` agent it runs `references/jira-manager-contract.md` in a general-purpose subagent.

## Manual install (fallback)

```bash
git clone git@github.com:rezonia/agent-skills.git
ln -s "$(pwd)/agent-skills/plugins/jira-workflow" ~/.claude/skills/jira-workflow
# or per-project:
ln -s "$(pwd)/agent-skills/plugins/jira-workflow" <project>/.claude/skills/jira-workflow
```

## Repo layout

```
agent-skills/
├── .claude-plugin/
│   └── marketplace.json         # plugin marketplace manifest
├── plugins/
│   ├── jira-workflow/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── skills/jira-workflow/
│   │   │   ├── SKILL.md
│   │   │   └── references/   # incl. jira-manager-contract.md
│   │   └── agents/jira-manager.md   # thin wrapper over the contract
│   ├── laravel-php-guidelines/
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills/laravel-php-guidelines/
│   │       ├── SKILL.md
│   │       └── references/
│   ├── google-admob-kmp/
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills/google-admob-kmp/
│   │       ├── SKILL.md
│   │       └── references/
│   └── filament-guidelines/
│       ├── .claude-plugin/plugin.json
│       └── skills/filament-guidelines/
│           ├── SKILL.md
│           └── references/
│   └── rez/
│       ├── .claude-plugin/plugin.json
│       └── skills/
│           ├── create-gh-release/SKILL.md
│           ├── pr-review-resolver/
│           │   ├── SKILL.md
│           │   └── references/
│           └── plan-decision-brief/
│               ├── SKILL.md
│               └── references/   # incl. claude-md-rule-snippet.md
└── README.md
```

## Contributing a new skill

1. Scaffold: `~/.claude/skills/.venv/bin/python3 ~/.claude/skills/skill-creator/scripts/init_skill.py <name> --path ./plugins`
2. Validate: `~/.claude/skills/.venv/bin/python3 ~/.claude/skills/skill-creator/scripts/quick_validate.py ./plugins/<name>/skills/<name>`
3. Add an entry to `.claude-plugin/marketplace.json`.
4. Keep the skill self-contained in `plugins/<plugin>/skills/<name>/`. `npx skills` installs only that folder, so plugin `agents/` must not be required (see `jira-workflow` for the contract + fallback pattern).
5. Check discovery: `npx skills add . --list`.
6. PR to `main`.
