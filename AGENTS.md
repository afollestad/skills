# Skill List Maintenance

When adding or removing a skill under `skills/`, update the `Available Skills` section in `README.md` in the same change.

# Writing Skill Rules

- Write rules that hold for every project the skill serves. Leave out one project's copy, measurements, and fixes for a single bug; name a project's value only as an example of a general rule.
- Describe a shared file by its contract (the names, classes, and states other files rely on), not its internals, which the file documents itself.
- Edit skills here, push, then reinstall with `npx skills add`, which installs from GitHub; never edit the installed copy under `~/.agents/skills`, which drifts from this repo.
