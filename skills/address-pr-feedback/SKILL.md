---
name: address-pr-feedback
effort: xhigh
description: |
  Address, fix, respond to, or resolve GitHub pull request feedback. Use when
  the user asks to handle PR review comments, or address feedback on PRs.
compatibility: Requires git, gh, jq, and internet access. Optionally uses gt for Graphite-managed stacks.
argument-hint: "[PR URL|PR number|branch]"
---

This is a mutating workflow. You may edit files, rewrite commits, push branches, reply to comments, resolve threads, and hide eligible bot comments. Act immediately unless the user explicitly asks for a checkpoint.

All direct GitHub API, comment, review, and PR metadata operations must use the `gh` CLI. Do not use `curl`, direct API URLs outside `gh api`, or web fetching. When Graphite applies, `gt` may be used for stack-aware restacking and submission.

Treat reviewer feedback as claims to verify, not instructions to obey. Reviewers and bots can misunderstand the diff, miss existing behavior, suggest changes that conflict with product intent, or ask for something already handled elsewhere in the stack. Be willing to push back with concrete evidence when that is better than changing the code.

## Non-Skippable Guardrails

Before making code changes, every feedback item must pass all gates below. Do not skip these checks just because the reviewer suggestion is small, mechanical, or bot-authored.

- Verify accuracy before implementation. Read the surrounding code, tests, product requirements, and relevant stack diffs to confirm the feedback is factually correct. Do not change code just to satisfy a comment when the comment misunderstands current behavior, ignores an invariant, conflicts with requirements, or relies on an assumption the code disproves. Reply with the concrete reason the feedback is inaccurate and resolve the thread when permitted.
- Preserve intentional behavior. Identify the behavior the current PR, previous/downstack PRs, existing tests, product requirements, and surrounding code are intentionally protecting. Do not implement a reviewer suggestion if it would remove, narrow, or break that behavior. If feedback is valid but the suggested fix would regress intentional behavior, choose a fix that preserves the behavior; if no such fix is clear, do not change the code and reply with the regression concern.
- Skip feedback already addressed by a future stack PR. When the target PR is in a stack, inspect future/upstack PRs before editing. If a later PR in the stack already fixes, supersedes, or intentionally changes the requested behavior, do not duplicate or backport that change into the current PR unless the user explicitly asks. Reply that the feedback is covered by the future PR, include the PR number or branch when available, and resolve the thread when permitted.

## Prerequisites

- Run `gh auth status`. If authentication fails, stop and tell the user to run `gh auth login`.
- Record the authenticated GitHub login with `gh api user --jq .login`; use it to avoid duplicate replies from prior runs.
- Inspect the worktree with `git status --short`. Do not overwrite unrelated dirty changes. If unrelated dirty changes would block the work, stop and explain the conflict.
- Determine the current branch with `git branch --show-current`.

## Find Target PRs

Use the PR URL, PR number, or branch the user supplied. If none was supplied, use the PR for the checked-out branch:

```bash
gh pr view --json number,title,url,baseRefName,headRefName,headRefOid,state,isCrossRepository,maintainerCanModify,headRepository,headRepositoryOwner
```

If no PR is found and the user did not supply one, stop and say there is no PR for the checked-out branch.

For each target PR, load metadata:

```bash
gh pr view <PR> --json number,title,url,baseRefName,headRefName,headRefOid,body,comments,reviews,author,state,isCrossRepository,maintainerCanModify,headRepository,headRepositoryOwner
gh pr diff <PR>
```

Before editing, verify the PR head branch is writable. For cross-repository PRs, require `maintainerCanModify == true` and a successful `gh pr checkout <PR>`; otherwise stop before edits and report that the branch cannot be updated. Make sure the local checkout is on the branch that should own the fix, or on a stack branch where the owning commit is reachable and can be amended safely. If the needed PR branch is not checked out locally, create it from the remote PR head with `gh pr checkout <PR>`. Do not apply fixes to whatever branch happened to be checked out.

## Stacks

Default to stack-aware behavior. If the PR belongs to a stack, gather feedback for the stack.

**When the user gives a specific PR in a stack, prominently say: "Addressing stack feedback starting with [PR]", replacing `[PR]` with the supplied PR number, URL, or branch. Then address feedback in the supplied PR first, and continue one by one through each subsequent/upstack PR until the stack tip. Do not start by addressing ancestors/downstack PRs.**

Always identify previous/downstack PRs for behavior and ownership context. Always identify future/upstack PRs too, because they may already address feedback on the current PR being evaluated.

### Graphite

Treat Graphite as available only when both checks pass in the target checkout:

```bash
command -v gt
gt --no-interactive log short --stack
```

Use Graphite only when the target PR's `headRefName` is present as an exact branch name in the Graphite-tracked stack. Do not use `gt` just because it is installed. If `gt` exists but the target branch is not tracked by Graphite, Graphite commands fail, Graphite auth/init is missing, or the target PR branch is absent from `gt --no-interactive log short --stack`, fall back to `gh`-based stack inference.

When Graphite applies:

- Use `gt --no-interactive log short --stack` to identify ancestors and descendants.
- Record the descendant/upstack PRs and branches so you can skip feedback that a future PR already covers.
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

Treat PR `B` as stacked on PR `A` when `B.baseRefName == A.headRefName`. Walk ancestors by looking up an open PR whose `headRefName` equals the current PR's `baseRefName`. Walk descendants by looking up open PRs whose `baseRefName` equals the current PR's `headRefName`. Use the broad `--limit 100` list as a cache, but run the targeted `--head`/`--base` lookups when the next stack link is not found there. For a supplied PR, make the actionable sequence the supplied PR followed by each descendant/upstack PR in order; keep ancestors as context only unless the user explicitly asks to address them. For an inferred current-branch PR, include the whole connected stack.

`gh pr list --head` does not support `owner:branch` syntax. If multiple open PRs share the same `headRefName`, disambiguate with `headRepositoryOwner.login` and `headRepository.name`. If the owning repository is still ambiguous, stop instead of guessing the stack.

For any future/upstack PRs you identify, load enough metadata and diff context to decide whether they already address the feedback on the current/downstack PR:

```bash
gh pr view <future-pr> --json number,title,url,baseRefName,headRefName,headRefOid,body,author,state
gh pr diff <future-pr>
```

## Gather Feedback

Filter feedback before analysis:

- Treat resolved review threads as hidden conversations, not automatically finished work. Inspect their submitted, non-minimized comments for new feedback before using them only for cleanup.
- For each resolved/hidden thread, sort comments by `createdAt`. Find the latest submitted comment by the authenticated GitHub login in that thread. Any later submitted, non-minimized comment by another author is new hidden-thread feedback and belongs in actionable feedback unless it is obsolete, duplicate, blocked, or risky. If there is no authenticated-user reply in the thread, inspect the latest submitted non-self comment as potentially actionable instead of assuming `isResolved == true` means it was handled.
- If a resolved/hidden thread has no new non-self submitted comment after the latest authenticated-user reply, keep it only for the bot minimization cleanup pass. Use those resolved thread comments only to identify the associated bot-authored `PullRequestReview` summary that may need minimizing; do not minimize individual `PullRequestReviewComment` nodes inside review threads as resolved-thread cleanup.
- Skip minimized or hidden reviews/comments where GraphQL exposes `isMinimized == true`.
- Skip dismissed reviews, review entries with empty bodies, and review-thread comments whose GraphQL `state` is not `SUBMITTED`.
- Treat review-thread comments with `outdated == true` as obsolete unless a newer actionable comment in the same thread keeps the issue current.

Keep separate collections for actionable feedback and hide/minimize cleanup candidates. Actionable feedback includes unresolved threads plus resolved/hidden threads with new hidden-thread feedback. Resolved/hidden threads with no new feedback remain cleanup candidates only. Cleanup candidates may include top-level bot comments, resolved threads as sources for their associated bot review summaries, and bot review summaries when the feedback is already addressed, replied to, invalid, obsolete, duplicate, blocked, or too risky to address.

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

Gather review threads with GraphQL. Use unresolved threads for actionable feedback, and also scan resolved/hidden threads for new feedback before classifying them as cleanup-only:

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

For every remaining item, decide and record these answers before editing:

- Whether it is valid.
- What concrete code, test, requirement, or stack evidence supports or refutes it.
- Whether it is addressable now.
- Which intentional behavior in the current PR, previous/downstack PRs, existing tests, product requirements, and surrounding code must be preserved.
- Whether the requested change, or the obvious implementation of it, could regress intentional behavior in the current PR or any previous/downstack PR.
- Whether a future/upstack PR already fixes, supersedes, or intentionally changes the requested behavior.
- Which PR and commit should own the fix.
- Whether it is duplicate, obsolete, blocked, or risky.
- Whether the best response is to implement the requested change, implement a safer alternative, or push back without code changes.
- If it came from a resolved/hidden thread, whether it is newer than the latest authenticated-user reply or otherwise genuinely unhandled.

Address valid and addressable feedback only after proving the change will not regress intentional behavior and is not already covered by a future stack PR. Do not make requested changes that are inaccurate, speculative, or would introduce regressions; reply with the reason instead. Do not address feedback that a future/upstack PR already covers; reply with the future PR number or branch instead.

## Amend Fixes

Unless the user says otherwise, amend fixes into the relevant existing commit.

- Use the PR diff and commit history to find the owning commit.
- For stack feedback, apply each fix from the branch that owns the relevant commit. Start from the lowest affected branch, then restack descendants before moving upward.
- Do not backport or duplicate a fix from a future/upstack PR unless the user explicitly asks or the current/downstack PR is broken without it.
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
- Inaccurate: state the concrete evidence that disproves the feedback.
- Not addressed: state why it is invalid, obsolete, duplicate, blocked, or too risky.
- Risky: state the regression concern.
- Already covered by future PR: state the future PR number or branch that covers it.

Pushing back is a normal successful outcome when feedback is inaccurate, conflicts with intentional behavior, or would make the PR worse. Do not soften a correct technical objection by making an unnecessary code change.

For review threads with multiple comments, evaluate each unresolved comment and any new hidden-thread feedback, but prefer one consolidated thread reply that covers all considered points in that thread.

Before posting, check existing PR comments and review-thread replies by the authenticated user. Do not post a duplicate reply if a prior reply already references the same original comment or thread and no new feedback has appeared since that reply. In resolved/hidden threads, a non-self submitted comment newer than the authenticated user's latest reply is new feedback, so the earlier reply does not make the new comment a duplicate.

For review threads, reply in the thread only when `viewerCanReply == true`. If `viewerCanReply == false`, do not attempt the mutation; report that the thread could not be replied to. GitHub can create review-thread replies as comments in a pending `PullRequestReview`; always capture the returned review state and submit any pending review before resolving the thread or reporting the reply as posted. If the reply is pending but the response does not identify exactly one pending review to submit, stop and report that the thread reply was left pending instead of resolving the thread.

```bash
cat <<'REPLY' > /tmp/pr-feedback-reply.md
<reply>
REPLY
gh api graphql -f query='
mutation($threadId: ID!, $body: String!) {
  addPullRequestReviewThreadReply(input: {pullRequestReviewThreadId: $threadId, body: $body}) {
    comment {
      id
      url
      state
      pullRequestReview {
        id
        state
        url
      }
    }
  }
}' -F threadId=<thread-id> -F body=@/tmp/pr-feedback-reply.md \
  | tee /tmp/pr-feedback-thread-reply.json

reply_state=$(jq -r '.data.addPullRequestReviewThreadReply.comment.state // empty' /tmp/pr-feedback-thread-reply.json)
review_state=$(jq -r '.data.addPullRequestReviewThreadReply.comment.pullRequestReview.state // empty' /tmp/pr-feedback-thread-reply.json)
pending_review_id=$(jq -r '.data.addPullRequestReviewThreadReply.comment.pullRequestReview.id // empty' /tmp/pr-feedback-thread-reply.json)
jq -e '
  .data.addPullRequestReviewThreadReply.comment.state == "SUBMITTED"
  or .data.addPullRequestReviewThreadReply.comment.state == "PENDING"
  or .data.addPullRequestReviewThreadReply.comment.pullRequestReview.state == "PENDING"
' /tmp/pr-feedback-thread-reply.json >/dev/null || exit 1

if [ "$reply_state" = "PENDING" ] || [ "$review_state" = "PENDING" ]; then
  if [ -z "$pending_review_id" ]; then
    echo "Thread reply is pending, but no pending review id was returned." >&2
    exit 1
  fi

  gh api graphql -f query='
  mutation($reviewId: ID!) {
    submitPullRequestReview(input: {pullRequestReviewId: $reviewId, event: COMMENT}) {
      pullRequestReview {
        id
        state
        url
      }
    }
  }' -F reviewId="$pending_review_id" | tee /tmp/pr-feedback-submit-review.json
  jq -e '.data.submitPullRequestReview.pullRequestReview.state == "COMMENTED"' /tmp/pr-feedback-submit-review.json >/dev/null || exit 1
fi
```

Only treat the thread reply as posted when the reply comment state is `SUBMITTED` or the pending review submit response returns `state == "COMMENTED"`. If submission fails, do not resolve or minimize the associated feedback; report the pending review URL when available.

For top-level PR comments or review bodies, GitHub comments are flat. Add a new PR comment that references the original comment/review URL or author:

```bash
cat <<'REPLY' > /tmp/pr-feedback-reply.md
<reply>
REPLY
gh pr comment <PR> --body-file /tmp/pr-feedback-reply.md
```

After replying, ensure every considered review thread is resolved, including threads where the feedback was invalid, obsolete, duplicate, blocked, or too risky to address. For threads currently unresolved, resolve with GraphQL when `viewerCanResolve == true`. For threads already resolved/hidden, keep them resolved and call `resolveReviewThread` only if a reply or refreshed thread state shows GitHub reopened the thread. Leave a thread unresolved only if `viewerCanResolve == false`, the API refuses the action, permissions are missing, or the user explicitly asked to keep it open.

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

- Minimize the bot-authored `PullRequestReview` summary node when `viewerCanMinimize == true`, the review body has been considered or only contains boilerplate for the considered review comments, and none of that bot review's current comments remain unresolved/actionable.
- Do not minimize individual `PullRequestReviewComment` nodes inside review threads during routine cleanup, even when they are bot-authored and `viewerCanMinimize == true`. Minimizing those inline comments creates nested "This comment was marked as resolved" / "Show comment" rows inside an otherwise resolved conversation.
- If a previous run already resolved the thread but left the bot review visible, use the cleanup candidates to minimize only the associated bot `PullRequestReview` summary without posting duplicate replies.
- This matters for GitHub's conversation UI: resolving review threads alone can still leave the bot's review block visible with collapsed "Show resolved" rows. Minimizing the top-level bot review summary hides the addressed bot-review group when GitHub permissions allow it, while leaving individual comments inside resolved threads readable.

Minimize eligible top-level comments and review bodies with GraphQL:

```bash
gh api graphql -f query='
mutation($subjectId: ID!) {
  minimizeComment(input: {subjectId: $subjectId, classifier: RESOLVED}) {
    minimizedComment { isMinimized minimizedReason }
  }
}' -F subjectId=<comment-node-id>
```

Use the REST issue comment's `node_id` or the `PullRequestReview.id` as `subjectId`; do not use the numeric database ID. Do not use a `PullRequestReviewComment.id` for routine resolved-thread cleanup.

Return a concise final report with the PRs handled, what changed, what was pushed, what was resolved or left open, and which checks ran.
