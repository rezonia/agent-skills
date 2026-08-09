# gh / GraphQL Cookbook

Exact commands the skill runs. Stored verbatim so they are copied, not improvised
(GraphQL is the most error-prone surface). Substitute `OWNER` / `REPO` / `PR` /
`SHA` / IDs. POSIX shell only.

**Prerequisite:** `gh` authenticated with **write** access (reply + resolve need
it). On `403`, instruct the user to re-auth: `gh auth login`.

## 0. Detect PR on current branch

```bash
gh pr view --json number,title,headRefName --jq '.number'
# empty / error → ask the user for a PR number or URL
```

Portable OWNER / REPO:

```bash
gh repo view --json owner,name --jq '.owner.login + " " + .name'
```

## 1. Fetch unresolved review threads (idempotent core)

```bash
gh api graphql -f query='
query($owner:String!,$repo:String!,$pr:Int!){
  repository(owner:$owner,name:$repo){
    pullRequest(number:$pr){
      reviewThreads(first:100){
        nodes{
          id
          isResolved
          comments(first:1){
            nodes{ databaseId author{login} path line body }
          }
        }
      }
    }
  }
}' -f owner=OWNER -f repo=REPO -F pr=PR \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[]
        | select(.isResolved==false)
        | {threadId:.id,
           commentId:.comments.nodes[0].databaseId,
           author:.comments.nodes[0].author.login,
           path:.comments.nodes[0].path,
           line:.comments.nodes[0].line,
           body:.comments.nodes[0].body}'
```

Key fields:
- `threadId` (e.g. `PRRT_kwD...`, an opaque node ID) → used by the **resolve** mutation.
- `commentId` (`databaseId`, an integer) → used by the **reply** REST call.

## 2. Reply to a specific comment (REST)

```bash
gh api repos/OWNER/REPO/pulls/PR/comments \
  -f body="Fixed in \`SHA\`. EXPLANATION" \
  -F in_reply_to=COMMENT_DB_ID \
  --jq '.html_url'
```

## 3. Resolve a thread (GraphQL mutation) — ONLY after reply

```bash
gh api graphql -f query='
mutation($threadId:ID!){
  resolveReviewThread(input:{threadId:$threadId}){
    thread{ id isResolved }
  }
}' -f threadId=THREAD_ID --jq '.data.resolveReviewThread.thread.isResolved'
```

## 4. Trigger a fresh Codex review (only if a Codex thread exists)

```bash
gh pr comment PR --body "@codex review"
```

## Bot vs Human Reviewer

Identify bots by known logins or a `[bot]` suffix:
- `chatgpt-codex-connector` (Codex)
- `coderabbitai`
- `github-actions`
- `copilot-pull-request-reviewer`

All unresolved threads are fixed uniformly. Bot detection is used only for (a) the
`@codex review` close step, and (b) noting severity badges (Codex emits `P1/P2/P3`).

## Pagination Note

`reviewThreads(first:100)` covers virtually every PR. If `pageInfo.hasNextPage`
were true you would follow the `endCursor` — not implemented (YAGNI); add cursor
follow-up only if a real PR exceeds 100 threads.
