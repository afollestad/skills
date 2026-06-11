---
name: self-review-diff
effort: xhigh
description: |
  Self-review, self review, self-reviewing, audit, check, or fix the current
  uncommitted Git diff or other uncommitted local Git changes before a commit or
  pull request. Use when the user asks for a repo-aware quality audit,
  pre-commit review, review of their diff, another pass,
  or another pass over local changes, or when asked to inspect local diffs,
  staged changes, unstaged changes, or untracked files for bugs, regressions,
  missing tests, lint, accessibility, stale code, or similar issues. The workflow
  loops after automatic or confirmed fixes until two consecutive clean passes
  verify that nothing remains to fix or report.
compatibility: Requires git. Optionally uses gt for Graphite-managed stacks. Uses repo-specific lint and test tools discovered from local guidance.
argument-hint: "[optional focus area]"
---

This is a mutating workflow by default. You may edit files to address clearly low-risk findings. Ask before higher-risk fixes, behavior changes, broad refactors, or creating a new commit. Do not push unless the user explicitly asks.

If the user asks for an audit-only review or says not to edit, treat this as read-only and report findings without modifying files.

## Prerequisites

- Inspect the worktree:
  ```bash
  git status --short
  git diff --stat
  git diff --staged --stat
  git diff --check
  git diff --staged --check
  git diff
  git diff --staged
  git ls-files --others --exclude-standard
  ```
- Run `git rev-parse --verify HEAD` before relying on `git diff HEAD` commands. If it succeeds, also inspect the combined tracked change:
  ```bash
  git diff HEAD --stat
  git diff HEAD --check
  git diff HEAD
  ```
- If the repository has no commits yet, skip the `HEAD` commands and review staged, unstaged, and untracked files directly.
- Include staged, unstaged, and relevant untracked files in the review. For untracked files, read the file contents directly because they do not appear in `git diff`.
- If the user supplies a focus area, prioritize that area but still account for the full local change set unless the user explicitly asks for a scoped-only audit. State the actual audit scope in the final report.
- When `HEAD` exists and a tracked file has both staged and unstaged changes, inspect the combined tracked change with `git diff HEAD -- "<file>"` and read the final file contents so interactions between staged and unstaged hunks are reviewed. Do not use the combined diff to decide what to stage.
- Record the current branch with `git branch --show-current`. If it is empty, record the detached `HEAD` SHA with `git rev-parse --short HEAD` and do not amend or create commits unless the user explicitly wants work done in detached HEAD or asks you to check out a branch first.
- Read applicable `AGENTS.md` files from the repo root and affected subdirectories before deciding on lint, tests, styling, commits, or review behavior.
- Do not overwrite unrelated dirty changes. If unrelated local changes affect the same files, work around them carefully or stop and explain the conflict.
- Preserve the user's staging intent. If committing or amending, stage only the intended fix hunks and keep pre-existing staged and unstaged changes separate whenever possible.
- Quote file paths in shell commands, especially paths from `git ls-files`, so spaces or shell metacharacters do not change which files are inspected.

## Understand the Changes

Build a repo-aware view before judging the changes:

- Identify the changed feature area, public interfaces, schemas, generated files, tests, snapshots, docs, and configuration touched by the local Git changes.
- Read nearby production code and tests for existing patterns.
- Search for call sites, references, feature flags, experiment gates, serializers, navigation paths, UI previews, and consumers of changed APIs.
- For regressions, search broadly enough to find unintended consequences outside the directly edited files.
- For UI changes, inspect the rendering path, accessibility labels/traits/roles, dynamic type or text scaling support, focus order, touch targets, loading/error/empty states, and snapshot or preview coverage.

Prefer `rg` for searches. Use structured tools when the repo provides them, such as language servers, code search helpers, generated type lookups, or project-specific CLIs.

## Audit Checklist

Review the changes for all of these categories:

1. Potential bugs in logic, state handling, concurrency, async flows, nullability, parsing, serialization, navigation, persistence, or error handling.
2. Missing edge cases, including empty inputs, absent optional fields, partial failures, retries, pagination, localization, time zones, permissions, and unsupported platforms.
3. Potential regressions. Trace consumers and nearby workflows broadly enough to catch behavior changes outside the edited files.
4. Performance issues, including unnecessary work in hot paths, repeated allocation, blocking I/O, slow rendering, expensive recomposition/re-rendering, or inefficient queries.
5. Dead or stale code, unused symbols, obsolete branches, duplicated helpers, stale TODOs, or code made unreachable by the changes.
6. File-size pressure. Check whether changed files are becoming hard to maintain and whether new logic belongs in an existing local abstraction. Treat file splitting as higher-risk unless it is mechanical and clearly isolated.
7. Missing test coverage, including unit, integration, snapshot, preview, fixture, migration, and regression coverage appropriate to the changed behavior.
8. Missing useful comments or class/function/property documentation. Add inline comments only when they explain non-obvious intent, constraints, invariants, or tradeoffs. Tests, scoped docs, and `AGENTS.md` guidance do not replace a short local comment when changed code has a surprising invariant that future readers would otherwise need to infer.
9. Missing or stale `AGENTS.md` guidance when the change reveals a durable workflow rule, repo convention, validation command, or pitfall future agents need.
10. Lint warnings or errors. Use the smallest relevant lint or format check from repo guidance. Do not run broad lint tasks when local instructions forbid them.
11. Accessibility issues in UI changes, including labels, roles, states, contrast, text scaling, focus order, keyboard/screen-reader navigation, hit targets, and motion sensitivity.
12. Other general problems: naming, readability, maintainability, security, privacy, logging, metrics, observability, ownership, docs, rollout safety, and compatibility.

## Fix Policy

Automatically fix findings only when the fix is low risk and clearly correct:

- Obvious lint, formatting, import, warning, or typo fixes limited to changed files or intended fix hunks.
- Small missing tests that directly cover changed behavior using existing test helpers, fixtures, and patterns without changing production semantics.
- Straightforward accessibility labels, hints, or state metadata omissions in newly changed UI when the intended semantics are obvious from visible text or existing nearby patterns.
- Trivial private stale code removal when static references prove it is unused and removal cannot affect public API, generated contracts, resources, serialization, dependency injection, reflection, or entrypoints.
- Clarifying comments for non-obvious changed logic when the invariant or constraint is directly supported by nearby code, tests, or local documentation. Treat these as low-risk fixes even when tests or scoped docs already cover the behavior, because the comment belongs at the point where the invariant can be misread.

Ask for confirmation before fixing higher-risk findings:

- Behavior changes, API or schema changes, persistence changes, data migrations, networking contract changes, generated-code changes, feature flag behavior, or rollout logic.
- Whole-repo formatters, broad auto-fix commands, or lint fixes that may rewrite unrelated files.
- Accessibility changes that alter focus order, grouping, roles, hit targets, keyboard navigation, gestures, layout, or visible UI behavior.
- Dead-code removal involving public symbols, serialized fields, resources, dependency injection, reflection, build configuration, command entrypoints, or anything that may be discovered dynamically.
- Snapshot baseline changes, broad refactors, file splitting, moving code across ownership boundaries, or changing public documentation.
- Comments or documentation that introduce new product policy, architectural policy, ownership guidance, or speculative rationale not already supported by the codebase.
- New test infrastructure, broad fixture rewrites, snapshot harness changes, or test helper changes that affect unrelated tests.
- Existing test expectation rewrites, unless the change is purely mechanical and cannot mask a behavior change.
- Any fix that may regress existing behavior or alter a previous/downstack commit's intent.
- Creating a new commit when no relevant existing commit should own the fix.
- Updating `AGENTS.md`, unless the user specifically asked for guidance changes.

If a finding is valid but not safe to fix automatically, report it with the concrete risk, affected files, and the recommended fix.

## Validation

After automatic low-risk fixes or user-confirmed higher-risk fixes, run the smallest relevant checks that cover the changed area:

- Re-run `git status --short` and the relevant diff/stat commands to verify the final local changes contain only the intended fixes.
- Always rerun `git diff --check`, `git diff --staged --check`, and, when `HEAD` exists, `git diff HEAD --check`.
- For each relevant untracked text file from `git ls-files --others --exclude-standard`, also run `git diff --no-index --check -- /dev/null "<file>"` or an equivalent whitespace check because plain `git diff --check` does not inspect untracked additions. `git diff --no-index` exits `1` for ordinary differences, so treat empty output as a clean whitespace check. For untracked binary files or snapshots, skip the whitespace check and validate them with the repo's normal artifact or snapshot workflow instead.
- Run focused unit tests, snapshot tests, type checks, or lint commands when local guidance or repo conventions identify them.
- For Kotlin or Android, prefer focused Spotless or ktfmt checks and targeted tests. Do not run broad `:lint` Gradle tasks unless the local repo explicitly requires them.
- For iOS repos with Cash CLI guidance, prefer focused `cash lint` and targeted tests or snapshots.
- Before the final report, run the applicable validation commands for the final pass even if that pass made no fixes, so the report reflects the final local state.
- If a check cannot run because a tool is missing, auth is unavailable, dependencies are not installed, or the command is too broad for the requested change, say so in the final report.
- Do not amend or create a commit after failed validation unless the user explicitly asks you to preserve the failing state. Report the failure and leave the fixes uncommitted.

## Iteration Loop

After validating any automatic low-risk fix or user-confirmed higher-risk fix, restart this workflow from [Prerequisites](#prerequisites) against the updated local change set. Do not require the user to ask for another pass.

A clean pass means the prerequisites, changed-file inspection, relevant searches, full audit checklist, and validation all ran against the current worktree, with no low-risk fixes applied and no higher-risk findings needing confirmation.

Repeat the audit, fix, and validation cycle until two consecutive clean passes complete. After the first clean pass, immediately run a fresh confirmation pass from [Prerequisites](#prerequisites) against the current worktree. For the confirmation pass, rerun the prerequisite commands, re-read the final contents of changed tracked files and relevant untracked files, redo the relevant searches, and do not rely on earlier command output or remembered conclusions. Only move on to commit or amend decisions after the second consecutive clean pass, unless the user explicitly requested a different commit timing.

If the confirmation pass finds a low-risk fix or a higher-risk finding, handle it under the normal fix policy, reset the consecutive clean-pass count, and restart from [Prerequisites](#prerequisites) after any validation. Do not report the loop as clean until two consecutive clean passes have completed after the most recent fix or finding.

If a pass finds higher-risk findings, pause for confirmation before fixing them. After applying and validating confirmed fixes, restart from [Prerequisites](#prerequisites). If the user declines or defers a higher-risk fix, stop the loop and include the remaining finding in the final report instead of repeatedly asking about it.

Guard against infinite loops. Track repeated finding/fix pairs within the session; if the same issue returns after a fix or validation step, stop and report it as a blocker with the files and commands involved.

## Commit and Amend Policy

Follow any user-specified commit or amend preference first.

If the user gives no preference, prefer amending fixes into the relevant existing commit after focused lint/tests. Use the changed files, branch, and commit history to identify the owning commit. If no relevant commit exists, leave fixes uncommitted or ask before creating a new commit. If multiple commits plausibly own the fix, do not guess; report the ambiguity and ask before amending.

When Graphite is applicable:

- Treat Graphite as active only when `gt` is installed, `gt --no-interactive log short --stack` succeeds, and the current branch is present in the tracked stack.
- Prefer Graphite-native operations such as `gt --no-interactive modify` for amending the current branch when applicable.
- Do not use raw `git commit --amend` or manual rebases for Graphite-tracked commits unless Graphite cannot perform the requested operation and the user confirms the fallback.
- Restack affected descendants with `gt --no-interactive restack --upstack` after changing a non-tip branch.
- Do not use `gt submit` or push unless the user explicitly asks.

When Graphite is not applicable, use normal Git amend/rebase operations that match the user's requested workflow. Confirm before rewriting commits other than the current branch tip or before complex rebases. Use `GIT_EDITOR=true` for rebase continuation commands that would otherwise open an editor.

## Report Structure

End with a concise report:

```markdown
Audited:
- <local change scope>

Auto-fixed:
- <low-risk fixes, or "None">

Needs confirmation:
- <finding, risk, and recommended fix, or "None">

Passes:
- <number of audit passes completed; say whether two consecutive clean passes completed or why the loop stopped>

Validation:
- <checks run and result>
- <checks skipped and reason>

Commit status:
- <amended/committed/left uncommitted; mention Graphite if used, and whether any auto-fixes remain staged, unstaged, or untracked>
```

If no issues are found, say that clearly and still include any residual test or validation gaps.
