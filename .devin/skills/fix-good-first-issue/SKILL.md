---
name: fix-good-first-issue
description: Pick an open GitHub issue labeled "good first issue" in
  Bible-Research/reactive-bible, fix it on a new branch stacked on the
  CURRENT branch, open a draft PR, wait for checks, fix any failures,
  and mark the PR ready for review. Invoke when the user asks to fix a
  good-first-issue / pick up an issue from the reactive-bible repo.
triggers:
  - user
---

# Fix a "Good First Issue" in reactive-bible

All work happens in the `reactive-bible/` repo (the frontend repo of
this monorepo workspace). Run all `git`/`gh` commands from
`reactive-bible/`. The GitHub repo is `Bible-Research/reactive-bible`
(remote `origin`).

## 1. Pick an issue

List candidates:

    gh issue list --repo Bible-Research/reactive-bible \
      --label "good first issue" --state open

- If the user gave an issue number, use it.
- Otherwise check open PRs first
  (`gh pr list --repo Bible-Research/reactive-bible --state open`)
  and skip issues already referenced by an open PR or branch.
- Pick the top remaining issue and read it fully:

      gh issue view <N> --repo Bible-Research/reactive-bible \
        --comments

If no unclaimed issues remain, stop and tell the user.

## 2. Branch from the CURRENT branch (critical)

Do NOT switch to main first. This skill is meant to be run
repeatedly; previous fixes may not be merged into main yet, so the
new branch MUST be created on top of whatever branch is currently
checked out:

    git -C reactive-bible rev-parse --abbrev-ref HEAD
    git -C reactive-bible switch -c fix/issue-<N>-<short-slug>

Record the current branch name — it becomes the PR base (stacked
PR). If the working tree has uncommitted changes, stop and ask the
user before proceeding.

## 3. Fix the issue

- Read `reactive-bible/CLAUDE.md` for frontend conventions.
- Implement the fix; add or update tests where appropriate.
- Keep every line <= 79 characters, in every file type.
- Verify before committing (from `reactive-bible/`):

      npm run lint
      npm run build
      npm test

## 4. Commit, push, and open a draft PR

- Stage only files you edited — never `git add -A` / `git add .`.
  Never commit plan/scratch markdown files.
- Commit message format: `Type: Capitalized message`
  (e.g. `Fix: Truncate long tag names in sidebar`).
- Push:

      git push -u origin fix/issue-<N>-<short-slug>

- Create the PR as a DRAFT with `--base <parent-branch>` — the
  branch recorded in step 2, NOT main — so the diff shows only this
  fix. Follow `.devin/workflows/create-pr.md`: write the body to a
  temp file and use `--body-file` (never inline `--body`):

      gh pr create --draft \
        --title "Type: Capitalized message" \
        --base <parent-branch> --head fix/issue-<N>-<short-slug> \
        --body-file pr-body.md
      rm pr-body.md

- The PR body must link the issue with `Closes #<N>`.

## 5. Wait for checks; fix failures

    gh pr checks <PR-number> --watch

- When the watch ends, inspect results: `gh pr checks <PR-number>`.
- If any check fails, get details
  (`gh pr checks <PR-number>` for links, `gh run view --log-failed`
  for logs), fix the code, commit, push, and watch again.
- Repeat until every check passes. Do not mark ready while checks
  are pending or failing.

## 6. Mark ready for review and report

    gh pr ready <PR-number>

Then report to the user: the issue number/title fixed, the PR URL,
the parent (base) branch, and the final check status.
