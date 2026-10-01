---
name: fix-good-first-issue-frontend
description: Pick an open GitHub issue labeled "good first issue" in
  Bible-Research/reactive-bible, fix it on a new branch cut from
  main, open a draft PR, wait for checks, fix any failures,
  and mark the PR ready for review. Invoke when the user asks to fix a
  good-first-issue / pick up an issue from the reactive-bible repo.
triggers:
  - user
---

# Fix a "Good First Issue" in reactive-bible

The target repo is `reactive-bible/` (the frontend repo of this
workspace) — but see step 2: the root cause may live in the
`bible_research` backend, or need changes in both. Run all
`git`/`gh` commands from the repo you end up working in. The
GitHub repo is `Bible-Research/reactive-bible` (remote `origin`).

## 1. Pick an issue

List candidates:

    gh issue list --repo Bible-Research/reactive-bible \
      --label "good first issue" --state open

- If the user gave an issue number, use it; otherwise pick the top
  candidate.
- Always verify no PR already covers the issue — even when the
  user gave the number explicitly:

      gh pr list --repo Bible-Research/reactive-bible \
        --state all --search "#<N> in:title,body"

  Skip the issue if a PR references it (`Closes #<N>`,
  `Fixes #<N>`) or a `fix/issue-<N>-*` PR already exists. If the
  user explicitly asked for an already-claimed issue, tell them
  which PR covers it and ask whether to proceed anyway.
- Read the chosen issue fully:

      gh issue view <N> --repo Bible-Research/reactive-bible \
        --comments

If no unclaimed issues remain, stop and tell the user.

## 2. Decide which repo the fix belongs in

The issue was filed in `reactive-bible`, but its root cause may
actually live in `bible_research` (the Django REST API) — or the
fix may need coordinated changes in both. Before branching, think
this through:

- Shared contract areas that usually touch BOTH repos:
  `fileset_id` provider routing (`ENGKJV` / `ENGESV_API` /
  `LVSGLU8` / DBT ids), the `error`/`error_code` failure contract,
  and note/tag/comment payload shapes.
- Symptoms that look like client bugs but are API bugs (empty
  `verses`, wrong status code, missing response field) → fix in
  `bible_research`.
- Genuinely frontend symptoms (UI state, rendering, caching, not
  consuming an existing field) → stay here.
- Search for a companion issue in the other repo:

      gh issue list --repo Bible-Research/bible_research \
        --state open --search "<keywords>"

Outcomes:

- Fix only here (most common) — continue to step 3.
- Fix only in `bible_research` — work there instead, following
  `bible_research/AGENTS.md` conventions (pytest, no migrations)
  and branching from ITS `main` branch; this skill's branch/PR
  steps still apply, just in the other repo (push to
  `Bible-Research/bible_research`).
- Fix in both — do each repo's part on its own branch and open two
  cross-linked PRs (each body references the other PR and issue).

If unsure, state your reasoning and ask the user before proceeding.

## 3. Branch from `main`

Switch to `main`, update it, and create the fix branch from it:

    git -C reactive-bible switch main
    git -C reactive-bible pull --ff-only
    git -C reactive-bible switch -c fix/issue-<N>-<short-slug>

If the working tree has uncommitted changes that block the switch,
stop and ask the user before proceeding.

## 4. Fix the issue

- Read `reactive-bible/CLAUDE.md` for frontend conventions.
- Implement the fix; add or update tests where appropriate.
- Keep every line <= 79 characters, in every file type.
- Verify before committing (from `reactive-bible/`):

      npm run lint
      npm run build
      npm test

## 5. Commit, push, and open a draft PR

- Stage only files you edited — never `git add -A` / `git add .`.
  Never commit plan/scratch markdown files.
- Commit message format: `Type: Capitalized message`
  (e.g. `Fix: Truncate long tag names in sidebar`).
- Push:

      git push -u origin fix/issue-<N>-<short-slug>

- Create the PR as a DRAFT with `--base main`. Follow
  `.devin/workflows/create-pr.md`: write the body to a temp file
  and use `--body-file` (never inline `--body`):

      gh pr create --draft \
        --title "Type: Capitalized message" \
        --base main --head fix/issue-<N>-<short-slug> \
        --body-file pr-body.md
      rm pr-body.md

- The PR body must link the issue with `Closes #<N>`.

## 6. Wait for checks; fix failures

    gh pr checks <PR-number> --watch

- When the watch ends, inspect results: `gh pr checks <PR-number>`.
- If any check fails, get details
  (`gh pr checks <PR-number>` for links, `gh run view --log-failed`
  for logs), fix the code, commit, push, and watch again.
- Repeat until every check passes. Do not mark ready while checks
  are pending or failing.

## 7. Mark ready for review and report

    gh pr ready <PR-number>

Then report to the user: the issue number/title fixed, the PR URL,
and the final check status.
