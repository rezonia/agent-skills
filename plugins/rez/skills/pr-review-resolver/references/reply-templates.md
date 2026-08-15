# Reply Templates

Concise reply patterns posted to a review comment after a fix. Always reference the
commit SHA, be specific about the mechanism, keep to 1–3 sentences. Never
performative ("Great catch!").

## Core template

```
Fixed in `<sha>`. <one–two sentences: what changed + why it resolves the comment>.
```

## Variants

**Standard fix**
```
Fixed in `<sha>`. <change>, so <reviewer's concern> no longer holds.
```

**Out-of-scope / deferred** (no commit, no resolve)
```
Acknowledged — this is outside the scope of this PR (<reason>). Tracked separately / left as-is.
```

**Docs follow-up**
```
Docs updated in `<sha>` to capture this behavior: <files>.
```

**Disagree** (rare — state reasoning, ask the reviewer; do NOT resolve)
```
<reasoning>. Could you confirm whether you'd still like this changed?
```

## Rules

- Reference the commit SHA that addresses the comment.
- Be specific about the mechanism, not just "done".
- 1–3 sentences. No filler, no praise.
- Reply BEFORE resolving the thread (see Security Policy).
- For out-of-scope and disagree, do not resolve — leave the thread open.
