# TWG Setup Workflow

Run when the backend probe (`twg whoami -o json`) fails, or when TWG skills are missing. The orchestrator runs it in the main session. Confirm with the user before any install step.

## Step 0 — Ask first

One `AskUserQuestion`:
- **Set up TWG CLI (Recommended)** → continue below.
- **Use Atlassian MCP this session** → only offer when `mcp__plugin_atlassian_atlassian__*` tools are available.
- **Skip Jira** → bypass, log the reason.

## Step 1 — CLI

```bash
command -v twg
```

Missing:
1. Check `command -v npm`. No npm → tell the user TWG needs Node.js + npm (or offer MCP) and stop setup.
2. Install the version verified with this skill (global npm install; mention it to the user):
   ```bash
   npm install -g @atlassian/twg-cli@1.2.5
   ```
3. Later upgrades: `twg update`.

## Step 2 — TWG skills

```bash
ls -d ~/.claude/skills/twg-* ~/.agents/skills/twg-* 2>/dev/null
```

None found:
```bash
twg skills install --agent claude -y
```
This installs the canonical `~/.agents/skills` set plus the Claude Code copy, without prompts.

## Step 3 — Login (user runs it)

OAuth device login needs a browser, so the agent cannot finish it. Ask the user to run:

```
! twg login
```

If the `## Jira` block `cloudId` is a host (e.g. `mycompany.atlassian.net`), suggest `! twg login --site mycompany` to skip the site picker. Never ask the user to paste tokens into chat or pass secrets as flags.

## Step 4 — Verify

```bash
twg whoami -o json
```

- OK → set backend to TWG, cache the account ID, resume the original workflow.
- Error with `error.remediation.command` → show the command, have the user run it via `!`, verify again.
- Still failing after one retry → `twg doctor -o json`, report the finding, offer MCP fallback.

## Notes

- Multiple Atlassian sites: `twg setup default-site --site <prefix>` saves the default without re-login.
- `twg setup` runs skills + login + upkeep + health in one interactive flow. It is a valid alternative for Steps 2–4 when the user prefers it (`! twg setup`).
- Never read or print `~/.config/twg/auth.conf`.
