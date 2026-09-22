---
description: "Commit only agent-edited files; never push plan files"
trigger: always_on
---

# Commits: Only Agent-Edited Files

When the user asks to commit and/or push changes:

- **Only stage and commit files that you (the agent) created or
  modified during this session.** Never use `git add -A`,
  `git add .`, or `git commit -a` — stage each edited file
  explicitly by path.
- **Never commit or push plan files** — e.g. `PLAN.md`,
  `plan.md`, `*-plan.md`, `*-PLAN.md`, or any other
  planning/scratch markdown generated during the session. Leave
  them untracked or unstaged.
- **Do not sweep in unrelated changes.** Files the user edited
  themselves, or other pre-existing dirty files in the working
  tree, must be left out unless the user explicitly asks to
  include them.
- Before staging, run `git status` and confirm the staged set
  matches exactly the files you edited — nothing more.
