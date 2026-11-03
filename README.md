# Aidan's AI Skills

This repo contains AI skills that I frequently use with agents like Claude Code. 

They are not specific to any company or repository.

# Available Skills

- [`address-pr-feedback`](skills/address-pr-feedback/SKILL.md): Address, fix, respond to, or resolve GitHub pull request feedback.
- [`review-pr`](skills/review-pr/SKILL.md): Review a PR and submit suggestions as comments, along with an optional approval/request for changes.
- [`self-review-diff`](skills/self-review-diff/SKILL.md): Self-review the current uncommitted Git diff or other uncommitted local Git changes before a commit or pull request.
- [`session-handoff`](skills/session-handoff/SKILL.md): Produce a prompt, summary, or context package intended as input for a future AI session.
- [`static-site-rules`](skills/static-site-rules/SKILL.md): Build or change a hand-written HTML/CSS/JS static site with a consistent design system, theming, assets, and GitHub Pages publishing.

# Setup

_Installing all skills:_

```bash
npx skills add https://github.com/afollestad/skills -g
```

_Installing a specific skill:_

```bash
npx skills add https://github.com/afollestad/skills -g --skill review-pr
```

See [skills.sh](https://skills.sh/) for more documentation.
