---
name: fix-good-first-issue-backend
description: Pick an open GitHub issue labeled "good first issue" in
  Bible-Research/bible_research, fix it on a new branch cut from
  main, open a draft PR, wait for checks, fix any
  failures, and mark the PR ready for review. Invoke when the user
  asks to fix a good-first-issue / pick up an issue from the
  bible_research backend repo.
triggers:
  - user
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
  `reactive-bible/CLAUDE.md` conventions and branching from ITS
  `main` branch; this skill's branch/PR steps still apply, just in
  the other repo (push to `Bible-Research/reactive-bible`).
- Fix in both — do each repo's part on its own branch and open two
  cross-linked PRs (each body references the other PR and issue).

If unsure, state your reasoning and ask the user before proceeding.

## 3. Branch from `main`

Switch to `main`, update it, and create the fix branch from it:

    git -C bible_research switch main
    git -C bible_research pull --ff-only
    git -C bible_research switch -c fix/issue-<N>-<short-slug>

If the working tree has uncommitted changes that block the switch,
stop and ask the user before proceeding.

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
  `bible_research/DEVELOPER_GUIDE.md` (workspace rule 6).

## 5. Commit, push, and open a draft PR

- Stage only files you edited — never `git add -A` / `git add .`.
  Never commit plan/scratch markdown files.
- Commit message format: `Type: Capitalized message`
  (e.g. `Fix: Return 404 for unknown book names`).
- Push:

      git push -u origin fix/issue-<N>-<short-slug>

- Create the PR as a DRAFT with `--base main`. Follow
  `.devin/workflows/create-pr.md` (workspace root): write the body
  to a temp file and use `--body-file` (never inline `--body`):

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
  are pending or failing. (If the repo reports no checks at all,
  note that and continue.)

## 7. Mark ready for review and report

    gh pr ready <PR-number>

Then report to the user: the issue number/title fixed, the PR URL,
and the final check status.
