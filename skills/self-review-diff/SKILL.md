---
name: self-review-diff
description: Audit staged, unstaged, and untracked changes before a commit or PR, and fix clear low-risk issues. Use for local diff reviews, quality audits, or another pass over changes.
argument-hint: "[optional focus area]"
effort: xhigh
---

Fix clear low-risk findings unless the user requests an audit without edits. Ask before higher-risk changes or new commits.

## Inspect

Read applicable `AGENTS.md` guidance and inspect:

```bash
git status --short
git branch --show-current
git diff
git diff --staged
git ls-files --others --exclude-standard
```

Read untracked files directly. If `HEAD` exists, inspect `git diff HEAD` and final file contents for staged/unstaged interactions. Preserve unrelated edits and staging intent.

Review all local changes unless the user limits scope; prioritize any requested focus. Read surrounding code/tests and trace affected consumers.

## Audit and Fix

Check:

- Correctness, edge cases, concurrency, error handling, and regressions.
- Security, privacy, compatibility, and rollout safety.
- Performance, stale code, duplication, readability, and oversized files.
- Relevant test coverage, lint, and documentation of non-obvious invariants.
- UI accessibility: labels, roles, states, contrast, text scaling, focus, navigation, hit targets, and motion.
- Durable workflow rules or pitfalls missing from `AGENTS.md`.

Automatically fix scoped formatting or typos, small tests using existing helpers, obvious accessibility metadata, proven-unused private code, and comments explaining supported invariants.

Ask before changing behavior, contracts, persistence, generated code, rollout logic, UI interactions/layout, snapshots, test expectations/infrastructure, public documentation, or `AGENTS.md`. Also ask before broad refactors/formatters or removing dynamically referenced code. Report deferred findings with files, risks, and recommended fixes.

## Validate and Repeat

Run the smallest relevant lint, tests, type checks, or snapshots. Reuse passing checks whose scope is unchanged; rerun failed, incomplete, or uncertain checks. Explain skipped checks. Leave fixes uncommitted after failed validation unless requested otherwise.

Each pass, refresh status/diffs and run `git diff --check`, `git diff --staged --check`, and `git diff HEAD --check` if `HEAD` exists. Check untracked text with `git diff --no-index --check -- /dev/null "<file>"`; empty output is clean even with exit `1`. Validate binaries through normal artifact checks.

Reread changed files and repeat relevant searches until two consecutive passes find no fixes or unresolved findings. Reset after a fix; stop and report deferred or recurring issues.

## Commits

Follow the user's commit preference. Otherwise, after validation, amend the owning commit. Leave fixes uncommitted if none owns them; ask if ownership is ambiguous.

Require explicit direction to commit in detached `HEAD`, rewrite non-tip commits outside a stack, or perform complex rebases. Use `GIT_EDITOR=true` for rebase continuation. Do not push or run `gh stack push`, `submit`, or `sync` unless requested.

For `gh stack`:

- Use it only if `gh stack view --json` succeeds and includes the current branch in `branches[].name`; exit `2` means no tracked stack. Avoid interactive bare `view` and `modify`. Recover from exit `10` (unfinished modify session) with `gh stack modify --abort`.
- Check out the owning branch, amend with `git commit --amend`, then run `gh stack rebase --upstack` from that branch to update descendants. With multiple remotes and no `remote.pushDefault`, pass the intended `--remote <name>`.
- On rebase exit `3`, stage understood resolutions and run `gh stack rebase --continue`; `--abort` restores all branches. Report conflicts you cannot resolve confidently.

Report scope, fixes, remaining findings, clean-pass count, validation run/reused/skipped, and commit/stack status. State when no issues remain.
