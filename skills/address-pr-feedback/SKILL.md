---
name: address-pr-feedback
description: Address GitHub PR feedback by verifying comments, fixing valid issues, replying, and resolving threads.
argument-hint: "[PR URL|PR number|branch]"
effort: xhigh
model: fable
---

Use `gh` for GitHub operations. Edit, amend, push, reply, resolve threads, and minimize eligible bot feedback unless the user limits scope or requests a checkpoint.

## Locate the Work

Check `gh auth status` and record `gh api user --jq .login` for reply deduplication. Inspect the worktree and branch; preserve unrelated changes.

Use the supplied target or the checked-out branch's PR. Stop if none exists.

```bash
gh pr view <PR> --json number,title,url,body,baseRefName,headRefName,headRefOid,state,isCrossRepository,maintainerCanModify,headRepository,headRepositoryOwner
gh pr diff <PR>
```

Before editing, verify the head is writable and check out the owning branch with `gh pr checkout <PR>`. Cross-repository PRs require `maintainerCanModify` and successful checkout.

## Stack Scope

For a supplied PR in a stack, announce "Addressing stack feedback starting with [PR]" and work from that PR through its descendants to the tip. Inspect ancestors for context only. Without a supplied target, include the whole connected stack.

Inspect ancestor and descendant diffs for behavior, fix ownership, and changes already made later in the stack.

### Tracked Stacks

Use `gh stack` only when `gh stack view --json` succeeds and includes the target's `headRefName` in `branches[].name`. Exit `2` means no tracked stack. `branches[]` gives stack order; `pr.number` identifies each PR.

If absent, try `gh stack checkout <PR-URL>` and recheck. A bare number can select an unrelated stack. Fall back to branch relationships if verification fails.

Switch layers with `gh stack checkout <branch>`. Avoid interactive commands: bare `view`, `switch`, and `modify`. Exit `10` means an unfinished modify session; recover with `gh stack modify --abort`.

With multiple remotes and no `remote.pushDefault`, pass `--remote <name>` to rebase/push. Resolve ambiguity before `checkout`, which lacks this flag.

### Inferred Stacks

Map PRs with `gh pr list --state open --limit 100 --json number,url,baseRefName,headRefName,headRepository,headRepositoryOwner`. B follows A when `B.baseRefName == A.headRefName`. Find missing links with `--head <baseRefName>` for ancestors and `--base <headRefName>` for descendants.

Disambiguate identical branch names by head repository and owner. `--head` does not support `owner:branch`. Stop if ownership remains ambiguous.

## Gather Feedback

Fetch top-level comments with `gh api repos/{owner}/{repo}/issues/<number>/comments --paginate`. Use GraphQL through `gh api graphql --paginate` for reviews and threads, requesting:

- Reviews: `id`, `body`, `state`, `url`, minimization fields, and author.
- Threads: `id`, `isResolved`, `viewerCanReply`, `viewerCanResolve`, path/line context, and comments.
- Thread comments: `id`, `body`, `url`, `createdAt`, `state`, `outdated`, minimization fields, author, and associated `pullRequestReview` ID/URL/author/minimization fields.
- Minimization fields: `isMinimized`, `viewerCanMinimize`; author: `{ __typename login }`.

Paginate reviews, threads, and comments using `$endCursor` and `pageInfo { hasNextPage endCursor }`. Nested comments need separate pagination when fetched nodes fall short of `comments.totalCount`. Query REST issue-comment `node_id`s through GraphQL for minimization state.

Ignore minimized items, dismissed or empty reviews, unsubmitted thread comments, and outdated comments unless newer feedback keeps the issue current.

In resolved threads, inspect eligible comments from others after the authenticated user's latest submitted reply, ordered by `createdAt`. Without a self-reply, inspect the latest eligible non-self comment. Threads without new feedback are cleanup candidates only.

Keep actionable feedback separate from bot cleanup candidates, including summaries linked to resolved threads.

## Evaluate and Fix

Verify feedback against code, tests, requirements, and stack diffs. Identify the owning PR/commit. Fix valid issues while preserving intentional behavior; choose a safer alternative if the suggestion would regress it.

Explain inaccurate, duplicate, obsolete, blocked, or risky feedback. If a later PR covers it, cite that PR or branch and skip the change unless the user requests a backport.

Amend the owning commit; create one only if none owns the fix. Follow repository commit rules, including `Co-authored-by: Codex <noreply@openai.com>` for Codex-authored commits. Run focused lint/tests before committing and pushing; report unavailable checks.

Work upward from the lowest affected branch within scope. Use `git commit --amend` on the owning branch, then update descendants:

- Tracked stack: run `gh stack rebase --upstack` from the amended branch, then `gh stack push`. Rebasing from the tip leaves descendants stale. Push updates active branches with `--force-with-lease`; partial success is possible, so reconcile rejected branches before retrying.
- Inferred stack: rebase descendants and push each changed branch with `git push --force-with-lease`.

On rebase exit `3`, stage resolutions and run `gh stack rebase --continue`; `--abort` restores all branches. Use `GIT_EDITOR=true` for continuation commands that open an editor. Report conflicts you cannot resolve confidently.

## Reply and Resolve

Reply once per thread with the outcome and supporting evidence. Skip replies already covered by the authenticated user unless new feedback has appeared.

For threads, require `viewerCanReply` and use GraphQL `addPullRequestReviewThreadReply` with `pullRequestReviewThreadId` and `body`. Capture the returned comment's `state`, `url`, and `pullRequestReview { id state url }`.

If a reply or its review is `PENDING`, submit that exact review with `submitPullRequestReview` (`pullRequestReviewId`, `event: COMMENT`). A reply is posted only when its state is `SUBMITTED` or submission returns `COMMENTED`. If the review ID is missing/ambiguous or submission fails, report the pending reply without resolving or minimizing its feedback.

For top-level feedback, use `gh pr comment <PR> --body-file <file>` and reference the original URL, unless repository guidance excludes these replies. Use temporary body files to preserve quoting.

After replying, refresh thread state. Resolve considered threads, including declined feedback, with `resolveReviewThread(input: {threadId: ...})` when `viewerCanResolve`. Skip already-resolved threads or those the user wants open. Report permission/API failures.

## Bot Cleanup

After considering their feedback, minimize only top-level comments and review summaries with `viewerCanMinimize`. Automation must be confirmed by REST `user.type == "Bot"`, GraphQL `author.__typename == "Bot"`, or an exact known Codex login for this repository. Do not infer automation from username substrings.

A bot summary is eligible when its body is considered or boilerplate and no current comments remain actionable or unresolved. Include eligible summaries from earlier runs. Never minimize inline `PullRequestReviewComment` nodes during cleanup.

Use GraphQL `minimizeComment(input: {subjectId: ..., classifier: RESOLVED})` with the issue comment's `node_id` or `PullRequestReview.id`, not a numeric database ID.

Report PRs handled, changes, pushes, resolved/open threads, and validation.
