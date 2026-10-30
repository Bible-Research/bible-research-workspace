---
name: fix-sentry-issue-backend
description: Pull unresolved production errors from the Sentry
  bible-research-api project, reproduce and fix them on a
  branch cut from main, open a draft PR, and mark the Sentry
  issue resolved only after the PR merges. Invoke when the
  user asks to fix Sentry issues / triage the Sentry queue in
  bible_research.
---

# Fix an unresolved Sentry issue (bible-research-api)

The target repo is `bible_research/`; the GitHub repo is
`Bible-Research/bible_research` (remote `origin`). The Sentry
project is `bible-research-api` in org `bible-research`, hosted
in the **EU region — `https://de.sentry.io`**, NOT sentry.io.

## 0. Auth — required before anything else

    source ~/.config/devin/sentry.env   # provides SENTRY_API_TOKEN

If `SENTRY_API_TOKEN` is unset, STOP and tell the user to set it
up once:

1. Sentry (de.sentry.io) → org Settings → Developer Settings →
   New Internal Integration.
2. Permissions: `Issue & Event: Read & Write`, `Project: Read`,
   `Release: Read`, `Organization: Read`. Save, copy the token.
3. `echo 'SENTRY_API_TOKEN=<token>' >> ~/.config/devin/sentry.env`

API helpers used throughout:

    BASE="https://de.sentry.io/api/0"
    AUTH="Authorization: Bearer $SENTRY_API_TOKEN"
    ORG="bible-research"
    PROJECT="bible-research-api"

## 1. Fetch the work queue

    curl -s -H "$AUTH" \
      "$BASE/projects/$ORG/$PROJECT/issues/\
?query=is:unresolved+level:error+environment:production\
&statsPeriod=14d&sort=freq"

Only `level:error` events are auto-fixable. The backend's
warning-level streams — `error_code:rate_limited` /
`error_code:provider_error` from `capture_provider_exception`
— are trend-only upstream outages, NOT bugs to fix. Excluded by
the level filter; surface them as a digest if the user asks.

There is no user-report stream on the backend — "Report a
problem" events live in the `reactive-bible-web` project; a
report whose root cause is server-side arrives here via the
cross-repo check below.

## 2. Pick and analyze one item

- Default: ONE item per run — the highest-count/recent issue.
  If the user said "fix all", enumerate first and post the
  list.
- Fetch `GET $BASE/issues/<id>/events/latest/` — exception
  type, stack frames, `error_code` tag (`internal_error` =
  caught server bug vs untagged unhandled 500), request URL/
  view, release (`GAE_VERSION`), breadcrumbs.

## 3. Reproduce and fix

    git -C bible_research switch main
    git -C bible_research pull --ff-only
    git -C bible_research switch -c fix/sentry-<issue_short_id>

- Read `bible_research/AGENTS.md` FIRST; 79-char lines
  everywhere.
- Write a failing test reproducing the event, then fix.
- Verify: `source venv/bin/activate &&
  python -m pytest <touched-app tests> -q`.
- NEVER run `makemigrations`/`migrate`/`sqlmigrate` — if the
  fix needs a migration, write the model change and report the
  command for the user to run.
- Local `config.yaml` points at the PRODUCTION Aiven Postgres
  via read-only user `ai_agent` — read-only investigation of
  live data is allowed and useful for reproducing; writes and
  DDL will fail by design.
- If the root cause is actually frontend (a bad request shape,
  a missing parameter the UI must send), switch to
  `reactive-bible` per fix-sentry-issue-frontend. If unsure,
  state reasoning and ask the user.
- Stage only files you edited (never `git add -A`); commit
  format `Type: Capitalized message`; update
  `DEVELOPER_GUIDE.md` if behavior changed.

## 4. Draft PR

- Push `-u origin`, draft PR `--base main` per
  `.devin/workflows/create-pr.md` (body via `--body-file`).
- Body must link the Sentry issue URL
  (`https://bible-research.sentry.io/issues/<id>/`), quote the
  top stack frame and request path, and include a verification
  section (curl command or test that reproduces the error).
- `gh pr checks <N> --watch`; fix until green; `gh pr ready`.
- `export PATH="/opt/homebrew/bin:$PATH"` on every exec call —
  gh/gcloud are not on the default PATH.

## 5. Resolve — only after merge

    curl -s -X PUT -H "$AUTH" \
      -H 'Content-Type: application/json' \
      -d '{"status": "resolvedInNextRelease"}' \
      "$BASE/issues/<id>/"

(Backend release = `GAE_VERSION`; resolution takes effect on
the next App Engine deploy.) NEVER resolve before merge; NEVER
`ignore`/`archive` an issue without asking the user.

## Guardrails

- If >3 actionable items exist, post the queue list and let
  the user pick before fixing.
- If a needed migration or prod-data fix is identified, stop
  and hand the command to the user — do not run it.
- Report at the end: issue handled, PR URL, Sentry resolution
  status.
