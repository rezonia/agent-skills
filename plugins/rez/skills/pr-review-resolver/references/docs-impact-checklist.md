# Docs Impact Checklist

Standalone (no-claudekit) docs evaluation after a fix. Project-agnostic — no
project-specific rule names. Mirrors how a docs-manager agent would reason without
requiring one.

Run through each item for the change just committed:

- [ ] Did the change alter a **public contract** (API, wire format, payload fields,
  CLI flags, config keys)? → update the contract doc.
- [ ] Did it change **observable behavior** described in docs (dispatch, routing,
  fallbacks, defaults)? → update the architecture / behavior doc.
- [ ] Did it add / remove / rename a **component or file** referenced in docs?
- [ ] Are there **paired docs** that must stay in sync (e.g. a backend contract + an
  admin guide)? Look for a sync note in the repo's doc index or CLAUDE.md.
- [ ] Do cross-references / indexes still resolve after the change?

If every item is "no" → state `Docs impact: none` and skip. Otherwise make a
**separate** docs commit and post a docs follow-up reply on the related comment
(see `reply-templates.md`).
