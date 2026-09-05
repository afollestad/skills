---
name: session-handoff
description: Create a handoff prompt for a future AI session when resetting a thread or transferring work. Not for human-facing summaries or documentation.
argument-hint: "[Next focus area]"
effort: xhigh
model: fable
---

Produce a paste-ready prompt for the next session's goal. Infer it from the conversation; ask only if unclear.

Include context that affects the next steps: progress, paths/identifiers, decisions, constraints, remaining work, verification, and blockers. Include failed attempts only if their cause still matters. Omit chronology, resolved dead ends, generic advice, and unrelated work.

Return only the prompt unless asked for commentary. Start with the task; omit irrelevant sections:

```markdown
Primary goal:

- ...

Current state:

- ...

Relevant files and areas:

- `path/to/file`: why it matters

Decisions and constraints:

- ...

Open questions or risks:

- ...

Next steps:

1. ...
```

For an immediate session transition, use a native handoff tool if available, passing this prompt and relevant file references.
