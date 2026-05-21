---
name: address-pr-feedback
description: |
  Address, fix, respond to, or resolve GitHub pull request feedback. Use when
  the user asks to handle PR review comments, or address feedback on PRs.
compatibility: Requires git, gh, jq, and internet access. Optionally uses gt for Graphite-managed stacks.
argument-hint: "[PR URL|PR number|branch]"
---

This is a mutating workflow. You may edit files, rewrite commits, push branches, reply to comments, resolve threads, and hide eligible bot comments. Act immediately unless the user explicitly asks for a checkpoint.

All direct GitHub API, comment, review, and PR metadata operations must use the `gh` CLI. Do not use `curl`, direct API URLs outside `gh api`, or web fetching. When Graphite applies, `gt` may be used for stack-aware restacking and submission.

## Prerequisites

- Run `gh auth status`. If authentication fails, stop and tell the user to run `gh auth login`.
- Record the authenticated GitHub login with `gh api user --jq .login`; use it to avoid duplicate replies from prior runs.
- Inspect the worktree with `git status --short`. Do not overwrite unrelated dirty changes. If unrelated dirty changes would block the work, stop and explain the conflict.
- Determine the current branch with `git branch --show-current`.

## Find Target PRs

Use the PR URL, PR number, or branch the user supplied. If none was supplied, use the PR for the checked-out branch. Treat this as the anchor PR until stack discovery decides the final target set:

```bash
gh pr view --json number,title,url,baseRefName,headRefName,headRefOid,state,isCrossRepository,maintainerCanModify,headRepository,headRepositoryOwner
```

If no PR is found and the user did not supply one, stop and say there is no PR for the checked-out branch.

Load metadata for the anchor PR before stack discovery. After stack discovery, repeat this for every PR in the final target set:

```bash
gh pr view <PR> --json number,title,url,baseRefName,headRefName,headRefOid,body,comments,reviews,author,state,isCrossRepository,maintainerCanModify,headRepository,headRepositoryOwner
gh pr diff <PR>
```

Before editing, verify every PR head branch that may receive a fix is writable. For cross-repository PRs, require `maintainerCanModify == true` and a successful `gh pr checkout <PR>`; otherwise stop before edits and report that the branch cannot be updated. Make sure the local checkout is on the branch that should own the fix, or on a stack branch where the owning commit is reachable and can be amended safely. If a needed PR branch is not checked out locally, create it from the remote PR head with `gh pr checkout <PR>`. Do not apply fixes to whatever branch happened to be checked out.

## Stacks

Default to stack-aware behavior. If the PR belongs to a stack, gather feedback for every open PR in the connected stack, ordered from the lowest ancestor through the highest descendant. This applies whether the user supplies a PR URL, PR number, branch, or relies on the current branch; treat the target PR as the anchor used to discover the full stack, not as a boundary.

### Graphite

Treat Graphite as available only when both checks pass in the target checkout:

```bash
command -v gt
gt --no-interactive log short --stack
```

Use Graphite only when the target PR's `headRefName` is present as an exact branch name in the Graphite-tracked stack. Treat that Graphite output as the complete tracked stack containing the target branch, then include every open PR whose `headRefName` exactly matches a branch in that stack. Do not use `gt` just because it is installed. If `gt` exists but the target branch is not tracked by Graphite, Graphite commands fail, Graphite auth/init is missing, or the target PR branch is absent from `gt --no-interactive log short --stack`, fall back to `gh`-based stack inference.

When Graphite applies:

- Use `gt --no-interactive log short --stack` to identify the complete tracked stack containing the target branch, including ancestors and descendants.
- Map the stack branches to open PRs by `headRefName`, gather feedback for every mapped PR, and keep the PR order from lowest ancestor through highest descendant.
- Use `gt --no-interactive restack --upstack` after amending a non-tip branch.
- Use `gt --no-interactive submit --stack --update-only` to push the current branch and subsequent stack branches after fixes. If the installed Graphite version supports `--no-edit`, it may be added, but do not rely on it.
- Prefer Graphite-native commit operations when they fit the change, such as `gt --no-interactive absorb` for staged fixes that should be amended into owning commits or `gt --no-interactive modify` for the current branch.

### Fallback Stack Inference

Without applicable Graphite, infer stack links from open PR branch relationships:

```bash
gh pr list --state open --limit 100 --json number,title,url,baseRefName,headRefName,headRefOid,isCrossRepository,maintainerCanModify,headRepository,headRepositoryOwner
gh pr list --state open --head <branch> --json number,title,url,baseRefName,headRefName,headRefOid,isCrossRepository,maintainerCanModify,headRepository,headRepositoryOwner
gh pr list --state open --base <branch> --json number,title,url,baseRefName,headRefName,headRefOid,isCrossRepository,maintainerCanModify,headRepository,headRepositoryOwner
```

Treat PR `B` as stacked on PR `A` when `B.baseRefName == A.headRefName`. Starting from the target PR, walk ancestors by looking up an open PR whose `headRefName` equals the current PR's `baseRefName`, and walk descendants by looking up open PRs whose `baseRefName` equals the current PR's `headRefName`. Continue until no open PR exists in either direction, then combine ancestors, the target PR, and descendants into one ordered stack from base-most PR to tip-most PR. Use the broad `--limit 100` list as a cache, but run the targeted `--head`/`--base` lookups when the next stack link is not found there.

`gh pr list --head` does not support `owner:branch` syntax. If multiple open PRs share the same `headRefName`, disambiguate with `headRepositoryOwner.login` and `headRepository.name`. If the owning repository is still ambiguous, stop instead of guessing the stack.

## Gather Feedback

Ignore already-resolved feedback before analysis:

- Skip review threads where `isResolved == true` for feedback analysis, but keep their node data for the bot minimization cleanup pass. A resolved review thread can still keep a bot review block visible in GitHub until the bot's review comments and review summary are minimized.
- Skip minimized or hidden reviews/comments where GraphQL exposes `isMinimized == true`.
- Skip dismissed reviews, review entries with empty bodies, and review-thread comments whose GraphQL `state` is not `SUBMITTED`.
- Treat review-thread comments with `outdated == true` as obsolete unless a newer unresolved comment in the same thread keeps the issue current.

Keep separate collections for actionable feedback and hide/minimize cleanup candidates. Actionable feedback excludes already-resolved threads. Cleanup candidates may include resolved threads, outdated bot comments, and bot review summaries when the feedback is already addressed, replied to, invalid, obsolete, duplicate, blocked, or too risky to address.

When GraphQL returns an `author`, request `author { __typename login }`. Treat the author as confidently automated for hiding only when `author.__typename == "Bot"`. For REST issue comments, continue using `user.type == "Bot"`.

Gather top-level PR comments:

```bash
gh api repos/{owner}/{repo}/issues/<number>/comments --paginate
```

The REST response includes `node_id` and `user.type`. Use `user.type` for bot detection. REST issue comments do not expose minimization state, so check those comment nodes with GraphQL before analyzing or hiding top-level comments. Repeat `-F ids[]=...` for each comment node ID:

```bash
gh api graphql -F ids[]=<comment-node-id> -f query='
query($ids: [ID!]!) {
  nodes(ids: $ids) {
    ... on IssueComment {
      id
      body
      url
      isMinimized
      minimizedReason
      viewerCanMinimize
      author { __typename login }
    }
  }
}'
```

Gather a quick view of review bodies:

```bash
gh pr view <PR> --json reviews
```

For complete review-body feedback, fetch reviews through GraphQL and use only non-dismissed reviews with non-empty bodies:

```bash
gh api graphql --paginate -F owner='{owner}' -F name='{repo}' -F number=<number> -f query='
query($owner: String!, $name: String!, $number: Int!, $endCursor: String) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $number) {
      reviews(first: 100, after: $endCursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          body
          state
          url
          isMinimized
          minimizedReason
          viewerCanMinimize
          author { __typename login }
        }
      }
    }
  }
}'
```

Gather review threads with GraphQL. Use unresolved threads for actionable feedback, and keep resolved threads available for the bot minimization cleanup pass:

```bash
gh api graphql --paginate -F owner='{owner}' -F name='{repo}' -F number=<number> -f query='
query($owner: String!, $name: String!, $number: Int!, $endCursor: String) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $number) {
      id
      number
      reviewThreads(first: 100, after: $endCursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          viewerCanReply
          viewerCanResolve
          path
          line
          originalLine
          startLine
          comments(first: 100) {
            totalCount
            nodes {
              id
              body
              url
              createdAt
              state
              outdated
              author { __typename login }
              isMinimized
              minimizedReason
              viewerCanMinimize
              pullRequestReview {
                id
                url
                isMinimized
                minimizedReason
                viewerCanMinimize
                author { __typename login }
              }
            }
          }
        }
      }
    }
  }
}'
```

`gh api graphql --paginate` paginates the outer `reviewThreads` connection here. If a thread's `comments.totalCount` is greater than the number of fetched comment nodes, fetch the remaining comments for that thread with a separate thread-node query before evaluating it:

```bash
gh api graphql --paginate -F threadId=<thread-id> -f query='
query($threadId: ID!, $endCursor: String) {
  node(id: $threadId) {
    ... on PullRequestReviewThread {
      comments(first: 100, after: $endCursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          body
          url
          createdAt
          state
          outdated
          author { __typename login }
          isMinimized
          minimizedReason
          viewerCanMinimize
          pullRequestReview {
            id
            url
            isMinimized
            minimizedReason
            viewerCanMinimize
            author { __typename login }
          }
        }
      }
    }
  }
}'
```

## Evaluate Feedback

For every remaining item, decide and record:

- Whether it is valid.
- Whether it is addressable now.
- Whether addressing it could regress behavior in the current PR or any previous/downstack PR.
- Which PR and commit should own the fix.
- Whether it is duplicate, obsolete, blocked, or risky.

Address valid and addressable feedback. Do not make requested changes that would introduce regressions; reply with the reason instead.

## Amend Fixes

Unless the user says otherwise, amend fixes into the relevant existing commit.

- Use the PR diff and commit history to find the owning commit.
- For stack feedback, apply each fix from the branch that owns the relevant commit. Start from the lowest affected branch, then restack descendants before moving upward.
- If Graphite applies, prefer `gt --no-interactive absorb` or `gt --no-interactive modify` when appropriate, then restack.
- If Graphite does not apply, use normal Git amend/rebase operations and push affected current/subsequent branches with `--force-with-lease`.
- Create a new commit only when no existing commit owns the fix.
- Follow repository commit-message rules, including required trailers. For Codex-authored commits, include:

```text
Co-authored-by: Codex <noreply@openai.com>
```

Run focused tests or checks appropriate to the changed code before pushing. If checks cannot run, say why in the final report.

## Push Stack Updates

When feedback is addressed in a stack, push the current branch and every subsequent branch whose history changed.

- With applicable Graphite: `gt --no-interactive restack --upstack`, then `gt --no-interactive submit --stack --update-only`.
- Without Graphite: manually restack/rebase descendants as needed and push changed branches with `git push --force-with-lease`.

Stop and report conflicts instead of guessing through a conflicted restack or rebase.

## Reply and Resolve

Reply to every considered feedback item, whether addressed or not. Keep replies concise:

- Addressed: state what changed.
- Not addressed: state why it is invalid, obsolete, duplicate, blocked, or too risky.
- Risky: state the regression concern.

For review threads with multiple comments, evaluate each unresolved comment but prefer one consolidated thread reply that covers all considered points in that thread.

Before posting, check existing PR comments and review-thread replies by the authenticated user. Do not post a duplicate reply if a prior reply already references the same original comment or thread and no new feedback has appeared since that reply.

For review threads, reply in the thread only when `viewerCanReply == true`. If `viewerCanReply == false`, do not attempt the mutation; report that the thread could not be replied to.

```bash
cat <<'REPLY' > /tmp/pr-feedback-reply.md
<reply>
REPLY
gh api graphql -f query='
mutation($threadId: ID!, $body: String!) {
  addPullRequestReviewThreadReply(input: {pullRequestReviewThreadId: $threadId, body: $body}) {
    comment { url }
  }
}' -F threadId=<thread-id> -F body=@/tmp/pr-feedback-reply.md
```

For top-level PR comments or review bodies, GitHub comments are flat. Add a new PR comment that references the original comment/review URL or author:

```bash
cat <<'REPLY' > /tmp/pr-feedback-reply.md
<reply>
REPLY
gh pr comment <PR> --body-file /tmp/pr-feedback-reply.md
```

After replying, resolve every considered review thread with GraphQL when `viewerCanResolve == true`, including threads where the feedback was invalid, obsolete, duplicate, blocked, or too risky to address. Leave a thread unresolved only if `viewerCanResolve == false`, the API refuses the action, permissions are missing, or the user explicitly asked to keep it open.

```bash
gh api graphql -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: {threadId: $threadId}) {
    thread { id isResolved }
  }
}' -F threadId=<thread-id>
```

Hide/minimize addressed comments and reviews only when `viewerCanMinimize == true` and the author is confidently automated:

- GitHub reports `user.type == "Bot"`.
- Or GraphQL reports `author.__typename == "Bot"`.
- Or the username exactly matches the known Codex GitHub user for the repo/context. If no exact Codex username is known, do not use this exception.

If `user.type` or `author.__typename` is missing or ambiguous, do not hide the comment or review. Do not hide comments based on username substrings. Never hide comments from uncertain or human-looking accounts.

Also minimize eligible bot-authored review nodes after their feedback has been considered:

- Minimize the original `PullRequestReviewComment` nodes in every considered thread when `viewerCanMinimize == true` and the author is confidently automated.
- Minimize the bot-authored `PullRequestReview` summary node when `viewerCanMinimize == true`, the review body has been considered or only contains boilerplate for the considered review comments, and none of that bot review's current comments remain unresolved/actionable.
- If a previous run already resolved the thread but left the bot review visible, use the cleanup candidates to minimize eligible bot `PullRequestReviewComment` nodes and the associated bot `PullRequestReview` without posting duplicate replies.
- This matters for GitHub's conversation UI: resolving review threads alone can still leave the bot's review block visible with collapsed "Show resolved" rows. Minimizing the bot's review comments plus the review summary hides the whole addressed bot-review block when GitHub permissions allow it.

Minimize eligible top-level comments, review bodies, and review comments with GraphQL:

```bash
gh api graphql -f query='
mutation($subjectId: ID!) {
  minimizeComment(input: {subjectId: $subjectId, classifier: RESOLVED}) {
    minimizedComment { isMinimized minimizedReason }
  }
}' -F subjectId=<comment-node-id>
```

Use the REST issue comment's `node_id`, the `PullRequestReview.id`, or the `PullRequestReviewComment.id` as `subjectId`; do not use the numeric database ID.

Return a concise final report with the PRs handled, what changed, what was pushed, what was resolved or left open, and which checks ran.
