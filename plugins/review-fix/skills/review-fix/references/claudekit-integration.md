# Claudekit Integration

Maps each workflow step to its enhanced implementation when claudekit is detected
(Step 0). Load this ONLY when the corresponding capability flag is set. Every row
has a standalone fallback — delegation is optional acceleration, never a
prerequisite.

| Step | CK-enhanced | Standalone fallback |
|------|-------------|---------------------|
| Context (4a) | `Explore` / `ck:scout` subagent over the path | Read + Grep around the line |
| Complex fix (4b) | `ck:debug` root cause → `fullstack-developer` / `planner` | direct minimal edit |
| Post-fix review (pre-push) | `code-reviewer` agent | self-check + show diff |
| Docs (5) | `docs-manager` agent | `docs-impact-checklist.md` |

## Guardrails

- Outputs of a delegated agent must match the standalone path's shape, so the loop
  is identical regardless of mode.
- Pass **minimal** context to subagents: path, line, comment body, PR number. Never
  the full session history.
- If a delegated agent errors or is absent at call time, fall back inline — never
  abort the loop.
- `--no-ck` skips this file entirely.
