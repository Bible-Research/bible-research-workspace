---
name: fix-good-first-issue-backend
description: Pick an open GitHub issue labeled "good first issue" in
  Bible-Research/bible_research, fix it on a new branch cut from
  main, open a PR, wait for checks, fix any
  failures, and run a reviewer loop until clean. Invoke when the user
  asks to fix a good-first-issue / pick up an issue from the
  bible_research backend repo.
---

# Fix a "Good First Issue" in bible_research

The target repo is `bible_research/` (the Django REST API repo of
this workspace). Run all `git`/`gh` commands from `bible_research/`.
The GitHub repo is `Bible-Research/bible_research` (remote
`origin`).

## 1. Pick an issue

List candidates:

    gh issue list --repo Bible-Research/bible_research \
      --label "good first issue" --state open

- If the user gave an issue number, use it; otherwise pick the top
  candidate.
- Always verify no PR already covers the issue — even when the
  user gave the number explicitly:

      gh pr list --repo Bible-Research/bible_research \
        --state all --search "#<N> in:title,body"

  Skip the issue if a PR references it (`Closes #<N>`,
  `Fixes #<N>`) or a `fix/issue-<N>-*` PR already exists. If the
  user explicitly asked for an already-claimed issue, tell them
  which PR covers it and ask whether to proceed anyway.
- Read the chosen issue fully:

      gh issue view <N> --repo Bible-Research/bible_research \
        --comments

If no unclaimed issues remain, stop and tell the user.

## 2. Decide which repo the fix belongs in

The issue was filed in `bible_research`, but its root cause may
actually live in `reactive-bible` (the React frontend) — or the fix
may need coordinated changes in both. Before branching, think this
through:

- Shared contract areas that usually touch BOTH repos:
  `fileset_id` provider routing (`ENGKJV` / `ENGESV_API` /
  `LVSGLU8` / DBT ids), the `error`/`error_code` failure contract,
  and note/tag/comment payload shapes.
- Symptoms that look like API bugs but are client bugs (wrong
  request params, stale cache, not handling an existing response
  field) → fix in `reactive-bible`.
- Genuinely backend symptoms (wrong status code, missing field,
  provider routing, queryset/permission bugs) → stay here.
- Search for a companion issue in the other repo:

      gh issue list --repo Bible-Research/reactive-bible \
        --state open --search "<keywords>"

Outcomes:

- Fix only here (most common) — continue to step 3.
- Fix only in `reactive-bible` — work there instead, following
  `reactive-bible/AGENTS.md` conventions and branching from ITS
  `main` branch; this skill's branch/PR steps still apply, just in
  the other repo (push to `Bible-Research/reactive-bible`).
- Fix in both — do each repo's part on its own branch and open two
  cross-linked PRs (each body references the other PR and issue).

If unsure, state your reasoning and ask the user before proceeding.

## 3. Branch from `main` — in a worktree

    git -C bible_research fetch origin
    git -C bible_research worktree add worktrees/issue-<N> \
      -b fix/issue-<N>-<short-slug> origin/main

Do all work inside `bible_research/worktrees/issue-<N>` — never
branch in the subrepo's main checkout (AGENTS.md). It needs its
own venv plus a `config.yaml` symlink to the main checkout's.

## 4. Fix the issue

- Read `bible_research/AGENTS.md` — it is the source of truth for
  backend conventions (provider routing, error contract, auth
  quirks, testing). `DEVELOPER_GUIDE.md` is partly stale.
- Implement the fix; add or update tests.
- Keep every line <= 79 characters, in every file type.
- NEVER run `makemigrations` / `migrate` / `sqlmigrate`. If the fix
  needs a migration, write the model change and tell the user which
  command to run — do not run it yourself.
- Leave the commented-out monthly TTS Cloud Scheduler in
  `terraform/scheduler.tf` as is — do NOT enable it.
- Verify before committing (from `bible_research/`):

      source venv/bin/activate
      python -m pytest <test-file-or-class> -q

  Run the tests for the app you changed (`annotations/tests.py`,
  `bible/tests/test_*.py`, `users`) — not the whole suite if it is
  slow. Tests run on SQLite in-memory; no `config.yaml` needed.
  Known pre-existing failures on `main` (not your regressions):
  `test_note_serializer`, `test_single_query_assertion`,
  `test_position_already_occupied`. Do NOT run
  `bible/services/dbt/dbt_integration_test.py` — it hits the live
  DBT API and needs a real `DBT_KEY`.
- If the change alters API behavior, update
  `bible_research/DEVELOPER_GUIDE.md` (workspace rule 10).

## 5. Commit, push, and open a PR

- Stage only files you edited — never `git add -A` / `git add .`.
  Never commit plan/scratch markdown files.
- Commit message format: `Type: Capitalized message`
  (e.g. `Fix: Return 404 for unknown book names`).
- Push:

      git push -u origin fix/issue-<N>-<short-slug>

- Create the PR ready for review (never a draft) with
  `--base main`. Follow `.devin/skills/create-pr/` (workspace
  root): write the body to a temp file and use `--body-file`
  (never inline `--body`):

      gh pr create \
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
- Repeat until every check passes — the worktree stays up until
  they do. (If the repo reports no checks at all,
  note that and continue.)

## 7. Reviewer loop, then report

Spawn a FRESH reviewer subagent on the PR diff
(`gh pr diff <PR-number>`). Fix everything useful it reports,
push, then spawn another fresh reviewer — repeat until a
round finds nothing meaningful. Keep the worktree until the
loop is clean AND checks are green, then remove it per
AGENTS.md.

Then report to the user: the issue number/title fixed, the PR
URL, and the final check status.
