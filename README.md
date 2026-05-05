# Rezonia Agent Skills

Private Claude Code skills marketplace for Rezonia engineering workflows.

## Skills

| Skill | Description |
|---|---|
| [jira-workflow](./skills/jira-workflow) | Mandatory Jira gating for brainstorms, implementations, and PRs. Ticket creation/sync with EPIC + Fibonacci story points + sprint, assignee = current MCP user, status transitions, PR title enforcement. |
| [laravel-php-guidelines](./skills/laravel-php-guidelines) | PHP + Laravel coding standards: PER style, strict typing, early-return control flow, Laravel helpers, Carbon, PHP 8 attributes, translation, routes/config/enums, full file & class naming conventions. |
| [google-admob-kmp](./skills/google-admob-kmp) | Google Mobile Ads (AdMob) for Kotlin Multiplatform: banner, interstitial, native, app open, rewarded ads on Android and iOS. Covers iOS Swift bridge pattern, UMP/GDPR/ATT consent, SDK v12+ type renames, and Compose Multiplatform rendering. |
| [filament-guidelines](./skills/filament-guidelines) | Filament 5.x conventions: simple resources, model policy per resource, `$action` notification pattern, `fas-*` icons, SPA/wire:navigate compatibility, dark mode, Filament Blade components, schema/table class extraction. |

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

## Manual install (fallback)

```bash
git clone git@github.com:rezonia/agent-skills.git
ln -s "$(pwd)/agent-skills/skills/jira-workflow" ~/.claude/skills/jira-workflow
# or per-project:
ln -s "$(pwd)/agent-skills/skills/jira-workflow" <project>/.claude/skills/jira-workflow
```

## Repo layout

```
agent-skills/
├── .claude-plugin/
│   └── marketplace.json         # plugin marketplace manifest
├── skills/
│   ├── jira-workflow/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── laravel-php-guidelines/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── google-admob-kmp/
│   │   ├── SKILL.md
│   │   └── references/
│   └── filament-guidelines/
│       ├── SKILL.md
│       └── references/
└── README.md
```

## Contributing a new skill

1. Scaffold: `~/.claude/skills/.venv/bin/python3 ~/.claude/skills/skill-creator/scripts/init_skill.py <name> --path ./skills`
2. Validate: `~/.claude/skills/.venv/bin/python3 ~/.claude/skills/skill-creator/scripts/quick_validate.py ./skills/<name>`
3. Add an entry to `.claude-plugin/marketplace.json`.
4. PR to `main`.
