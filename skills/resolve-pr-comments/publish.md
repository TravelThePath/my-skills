# Publish — commit, reply, resolve

Reached when every item has a verdict (no open `high`/`ask`). One flow, with a commit step only when at least one item is `Fix`.

## Flow

1. **Preview.** Show:
   - `git diff --stat` and a focused summary per changed file (if any Fix)
   - verification commands and their results
   - the reply that will be posted to each thread
   - threads to resolve, outdated threads to resolve without a reply (count + one-line each), and any Defer follow-up drafts

2. **Authorization.**
   - Explicit `resolve` / `fix` / `publish` request → authorized, continue.
   - Review-only request → ask once: `Commit and push fixes, then post replies and resolve processed threads?` Continue only if confirmed.

3. **Commit + push** (only if any `Fix`). Fixed replies must reference a commit visible on the PR, so this happens before reply/resolve.
   - Stage only the intended files (never `git add -A`). Group fixes by file/behavior, apply, then run the applicable verification (infer commands from `CLAUDE.md`/`AGENTS.md`, `Makefile`, package scripts, `go.mod`, or CI). Print commands before running.
   - Do not run a fresh CodeRabbit/uncommitted review — this skill is already responding to existing review.
   - Commit with a descriptive message and push.

4. **Reply + resolve.** Re-fetch PR head and unresolved thread IDs (state may have moved since Step 1). For each `Defer`, create the follow-up issue/note first so its reply cites a real reference. Post each reply, then resolve each processed thread. Resolve any outdated threads in the same pass — **without a reply**. Print only `All replies posted. Now resolving threads.` then `All N threads resolved.`

## Stop conditions

Stop before any GitHub write (commit, push, reply, resolve) and ask, naming the reason, if any hold:

- The local diff includes work outside the recorded verdicts.
- An item is missing a verdict, or an `Ask` was never converted.
- A `Fix` commit is not yet visible on PR head.
- Verification failed and the user did not accept the limitation.
- A planned reply went stale after re-fetch (the thread was edited/resolved).
- A Defer reply would cite a tracking issue that was never created.

## Replies

Every thread gets a reply before it is resolved. The one exception is **outdated threads** (`is_outdated`), resolved without a reply.

| Verdict | Reply |
| --- | --- |
| Fix | `Fixed in <commit>. <one-line what changed>` |
| Reply | The technical reason no code change is made — name the convention/peer/line. One line for noise/style. |
| Defer | `Valid concern; tracked in <issue/draft>, not in this PR.` Do not claim an issue exists unless created. |

Deduplicated groups: post the same reply body to each thread, resolve each independently. Skip none.

## gh mutations

Use `--silent` for all writes.

```bash
# Inline review thread reply (thread resolution dedups re-runs; no marker needed)
gh api graphql --silent \
  -f query='mutation($threadId: ID!, $body: String!) { addPullRequestReviewThreadReply(input: { pullRequestReviewThreadId: $threadId, body: $body }) { comment { id } } }' \
  -F threadId="$thread_id" -f body="$reply_body"

# Resolve a thread
gh api graphql --silent \
  -f query='mutation($threadId: ID!) { resolveReviewThread(input: {threadId: $threadId}) { thread { isResolved } } }' \
  -F threadId="$thread_id"
```

PR-level (issue) comments have no resolved state. Their reply **must** end with the marker `<!-- resolve-pr-comments:reply -->` so the fetch script drops it on the next run instead of re-surfacing it as fresh feedback:

```bash
gh api --silent repos/{owner}/{repo}/issues/{number}/comments \
  -f body="<reply>

<!-- resolve-pr-comments:reply -->"
```

To find threads to resolve, re-fetch unresolved IDs and match against the ones you processed (resolve by thread ID, never by comment ID):

```bash
gh api graphql --paginate -F owner='{owner}' -F repo='{repo}' -F number={number} -f query='
  query($owner: String!, $repo: String!, $number: Int!, $endCursor: String) {
    repository(owner: $owner, name: $repo) { pullRequest(number: $number) {
      reviewThreads(first: 100, after: $endCursor) {
        pageInfo { hasNextPage endCursor }
        nodes { id isResolved }
      } } } }' \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false) | .id'
```
