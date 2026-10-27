# Always-On Rule Snippet

Skill activation depends on the model matching the description. To make every
plan and fix include the brief, the user can paste this block into their
project or global `CLAUDE.md` / `AGENTS.md`. Never add it without approval.

```markdown
## Decision brief (plan-decision-brief)

- Every implementation plan (Plan mode, /ak-plan, any plan `.md`) starts with a
  Vietnamese section "Tóm tắt quyết định" (~50 lines max), pinned right after the
  title. The rest of the plan stays in English. Print the same brief in chat.
- Before applying a fix or direct implementation without a plan file (/ak-fix,
  cook without a plan), print a Vietnamese brief (~20 lines max) in chat after the
  root cause is proven and before editing code. Do not write it to a file.
- The brief always includes architecture scalability and design patterns
  (which pattern, where, why — or why none).
- Use the `plan-decision-brief` skill for section lists and templates.
```

Notes:

- The skill name in the snippet works for both installs: `rez:plan-decision-brief`
  (plugin) and `plan-decision-brief` (`npx skills`).
- If the project already sets a response language rule, this snippet only
  overrides it for the brief section.
