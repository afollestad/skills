---
name: session-handoff
description: |
  Invoke this skill whenever the user asks you to produce a prompt, summary, or context package intended as input for a future AI session. This covers: handoff prompts, context dumps, new-thread prompts, paste-ready prompts for fresh sessions, or any request to distill this conversation so another Claude instance or background agent can continue the work. Also invoke when the user mentions context window limits, thread length concerns, or wanting to reset/wrap up while preserving progress for a next session. This skill owns the output format and relevance filtering — do not attempt these handoffs without it. Do NOT invoke for human-audience artifacts: docs, READMEs, commit messages, tickets, release notes, standup updates, code explanations, or handoffs to human coworkers.
argument-hint: "[Next focus area]"
effort: xhigh
model: fable
---

Turn the current session into a prompt that another agent can use immediately.

The goal is not to summarize the entire conversation. The goal is to preserve only the accurate and valuable context that will help the next session make progress faster (while minimizing the initial context window).

## When To Use This Skill

Use this skill when the user wants any version of these outcomes:

- Start a fresh session without losing the important context.
- Hand the work to another agent or another thread.
- Convert the current conversation into a reusable prompt.
- Focus the next session on one specific follow-up task and carry forward only the relevant details.

## Core Behavior

1. Identify the next-session goal.
2. Use that goal as the relevance filter for everything you include.
3. Review the current session and preserve only the facts that change how the next agent should act.
4. Produce a prompt that is ready to paste into a new session, or feed the same content into the host agent's native handoff feature when one exists.

If the user already said what the next session should do, do not ask again. If the goal is still ambiguous after reading the conversation, ask one short clarifying question before drafting the handoff.

## Relevance Rules

Pull forward details like these when they will save the next session from rediscovering them:

- The user's current objective.
- Files, symbols, commands, URLs, PRs, errors, or branches that matter to the next step.
- Decisions that were already made.
- Constraints, preferences, or non-goals the user stated.
- Work that is already done.
- Work that was attempted but failed for a specific reason that still matters.
- Known risks, blockers, or unanswered questions.

Leave out details like these unless the user explicitly asks for a full history:

- A play-by-play of every step taken.
- Resolved dead ends that no longer affect the next session.
- Generic advice the next agent could infer on its own.
- Background context that is unrelated to the stated next goal.

If the user narrows the next-session goal, narrow the handoff aggressively too.

## Writing Guidance

Write for the next agent, not about the next agent.

- Prefer direct, concrete statements over narrative.
- Favor facts, decisions, and next steps over chronology.
- Mention exact file paths and identifiers when they matter.
- If code was changed, say what changed and what remains.
- If verification is still pending, mention what should be checked next.
- Keep the prompt dense and useful. Short is good, but completeness matters more than being terse.

When possible, present the handoff as if it were the opening prompt of the next session rather than a retrospective summary.

## Output Format

Return only the handoff prompt unless the user asks for commentary.

Do not start the prompt with meta-commentary like "Continue this work in a fresh session" — the prompt is already going into a new session, so that framing is redundant. Jump straight into the substance.

Use this structure, omitting sections that truly do not apply:

```markdown
Primary goal:
- ...

Current state:
- ...

Relevant files and areas:
- `path/to/file`: why it matters
- `path/to/other_file`: why it matters

Decisions and constraints to preserve:
- ...

Open questions or risks:
- ...

Recommended first steps:
1. ...
2. ...
3. ...
```

## Native Handoff Tools

If the host environment has a built-in handoff, new-thread, or background-thread tool, prefer using it when the user wants the session transition to happen immediately.

In that case:

- Use the same relevance rules from this skill.
- Put the same high-signal content into the tool's prompt or goal field.
- Include specific file references if the host agent supports them.
- Do not pad the tool input with narration about how you created the handoff.

If there is no native handoff mechanism, return the paste-ready prompt directly.

## Examples

Example 1:

User intent: "Create a handoff for a new session that should finish the auth refactor."

Good behavior: Focus on the auth refactor, the files already touched, the remaining verification, and the exact blocker. Do not include unrelated UI cleanup work from earlier in the conversation.

Example 2:

User intent: "Summarize this thread so another agent can debug the failing CI job."

Good behavior: Center the prompt on the failing job, the relevant logs or test names, suspected root cause, and the next checks to run. Leave out unrelated feature design discussion.
