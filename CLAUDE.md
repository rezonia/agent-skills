# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Rezonia's private **Claude Code plugin marketplace**. It does not ship runnable application code — it ships **agent skills** (Markdown + reference docs) consumed by Claude Code via the `/plugin marketplace` system. There is no build, no test runner, no compile step.

## Repo layout (load-bearing)

```
.claude-plugin/marketplace.json                        # marketplace manifest — every plugin MUST be registered here
plugins/<plugin-name>/.claude-plugin/plugin.json        # plugin manifest (name, description, version)
plugins/<plugin-name>/plugins/<plugin-name>/SKILL.md     # skill entry point with YAML frontmatter
plugins/<plugin-name>/plugins/<plugin-name>/references/  # progressive-disclosure docs loaded on demand
plugins/<plugin-name>/agents/<agent-name>.md            # optional: bundled sub-agents
```

Each plugin is one directory under `plugins/`. The `SKILL.md` frontmatter `description` is the routing signal Claude Code uses to decide when to activate the skill — it must enumerate concrete triggers (file types, commands, ticket patterns, etc.), not just describe the topic.

## Authoring conventions

- **Progressive disclosure**: keep `SKILL.md` short and link out to `references/*.md`. Do not inline full reference content into `SKILL.md`.
- **Trigger-rich descriptions**: the frontmatter `description` should list every situation that should activate the skill (see existing skills for the pattern — file extensions, slash commands, framework names, ticket key patterns).
- **Security Policy section** is required in every `SKILL.md` (refusal rules, secret handling, scope guardrails).
- **Scope section** must explicitly state what the skill does *not* handle, to prevent over-activation.
- Two registration sites must stay in sync when adding/renaming a skill: `.claude-plugin/marketplace.json` (`plugins[]` entry) and `README.md` (skills table).

## Common commands

Scaffold and validate a new skill (uses the global skill-creator venv, not a repo-local one):

```bash
~/.claude/plugins/.venv/bin/python3 ~/.claude/plugins/skill-creator/scripts/init_skill.py <name> --path ./skills
~/.claude/plugins/.venv/bin/python3 ~/.claude/plugins/skill-creator/scripts/quick_validate.py ./plugins/<name>/plugins/<name>
```

Validate the full plugin (including plugin.json):

```bash
claude plugin validate ./plugins/<name>
```

Install from this marketplace (for testing changes end-to-end):

```
/plugin marketplace add rezonia/agent-skills
/plugin install <skill-name>@rezonia-agent-skills
```

## When editing a skill

1. Edit `plugins/<name>/SKILL.md` and/or its `references/*.md` directly — never create `*-v2.md` or "enhanced" copies.
2. If the skill's purpose, name, or trigger surface changed, update both `.claude-plugin/marketplace.json` and the `README.md` skills table.
3. Run `quick_validate.py` against the skill directory before committing.
