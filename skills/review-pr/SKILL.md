---
name: review-pr
description: Review a GitHub PR and prepare prioritized comments for submission, with optional approval or request for changes.
argument-hint: "[PR URL]"
effort: xhigh
model: fable
---

Review without changing repository files. Use `gh` for all GitHub operations.

## Inspect

Confirm the target PR and authentication with `gh auth status`.

```bash
gh pr view <PR> --json number,title,body,baseRefName,headRefName,headRefOid,comments,reviews
gh pr diff <PR>
gh api repos/{owner}/{repo}/issues/{number}/comments --paginate
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate
```

Read the complete diff and surrounding code. Check correctness, security, performance, maintainability, and tests. Exclude issues already raised in comments or reviews, regardless of wording.

## Prepare Feedback

Include actionable findings without praise or filler:

- P0 — Blocking: security vulnerabilities, data loss, or broken functionality.
- P1 — Important: logic errors, missing edge cases, error handling, or test gaps.
- P2 — Suggestion: naming, refactoring, style, or documentation improvements.
- P3 — Nit: minor, optional improvements.

Keep findings worth posting, including useful nits. Prefer questions where natural. Present them by priority with an index, file, line, and comment text.

Unless already specified, ask which indices to submit and whether to approve, request changes, or comment.

## Submit

Write payloads to PR-specific temporary JSON files to preserve quoting and avoid collisions. Submit selected inline comments as one review:

```json
{
  "event": "COMMENT",
  "body": "",
  "comments": [
    {
      "path": "relative/file.ext",
      "line": 42,
      "side": "RIGHT",
      "body": "**[P1]** Comment text"
    }
  ]
}
```

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input /tmp/pr_review_{number}.json
```

Use the chosen `APPROVE`, `REQUEST_CHANGES`, or `COMMENT` event, an empty review body, and a bold priority prefix on each comment.

Confirm lines against the file at the PR head. Use `line` and `side`, not `position`; use `RIGHT` for new/changed lines. Comment only on paths in the diff.

For a finding outside the diff hunks, post a separate file-level comment to `repos/{owner}/{repo}/pulls/{number}/comments` with `body`, `path`, `subject_type: "file"`, and `commit_id` set to `headRefOid`. Mention the line in the body. Keep each finding separate.

Return the review URL.
