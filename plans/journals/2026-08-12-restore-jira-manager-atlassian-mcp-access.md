---
title: Restore Jira manager Atlassian MCP access
date: 2026-08-12
summary: Removed restrictive Jira-manager tool allowlists so installed Atlassian MCP tools are inherited dynamically.
---

# Restore Jira manager Atlassian MCP access

## What happened
`jira-manager` could not use Atlassian MCP tools from a separately installed Claude Code plugin. Its explicit `tools:` lists were restrictive allowlists.

## Decision
Removed `tools:` from both `.claude/agents/jira-manager.md` and `plugins/jira-workflow/agents/jira-manager.md`. The agent now inherits host tools; its Jira-only task and security instructions remain the operational boundary. Updated skill guidance and synchronized plugin/marketplace version `0.2.1`.

## Validation
`claude plugin validate ./plugins/jira-workflow` passed with its existing missing-author warning. `git diff --check` passed. The documented global `quick_validate.py` path is unavailable.

## Next steps
Reinstall or update `jira-workflow@rezonia-agent-skills` in a Claude Code session with the Atlassian plugin installed, then invoke `jira-manager` for a read-only Jira operation.

> Historical work record — not durable authority. Prefer docs/specs/ADRs for current decisions.
