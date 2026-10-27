---
title: Add plan-decision-brief skill
date: 2026-10-27
summary: New rez skill pins a Vietnamese decision brief on plans and prints it before fixes
---

# Add plan-decision-brief skill

## What happened
Added `plugins/rez/skills/plan-decision-brief/`: SKILL.md + references (core-sections, architecture-and-patterns, type-sections, brief-template, claude-md-rule-snippet). Registered in marketplace.json and README; rez plugin bumped 0.1.0 -> 0.2.0.

## Decision
- Plan mode / /ak-plan / plan .md: brief pinned at top of the plan file and printed in chat, ~50 lines.
- /ak-fix or cook without plan: chat only, ~20 lines, after root cause + fix choice, before first edit.
- Architecture scalability + design pattern (which, where, why, or why none) is always present.
- Enforcement = skill triggers + optional CLAUDE.md rule snippet (no hook).

## Verification
quick_validate, `claude plugin validate` (plugin + marketplace), `npx skills add . --list` all pass. Three dry-run subagents (backend plan 44 lines, frontend plan 53 lines, fix 18 lines) produced complete briefs and surfaced ambiguities, now fixed: line counting includes blanks/tables, undecided items go to "Cần user xác nhận", undecided mitigations marked "(đề xuất)", consecutive bold lines replaced by list items (Markdown merged them), fix timing clarified.

## Next steps
Commit on a dedicated branch; optionally add the rule snippet to CLAUDE.md. Pre-existing validator warnings (author, LICENSE, icon) remain.

> Historical work record — not durable authority. Prefer docs/specs/ADRs for current decisions.
