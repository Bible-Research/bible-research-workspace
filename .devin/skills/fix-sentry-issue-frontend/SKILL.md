---
name: fix-sentry-issue-frontend
description: Pull unresolved production errors and developer
  "Report a problem" reports from the Sentry reactive-bible-web
  project, reproduce and fix them on a branch cut from main,
  open a draft PR, and mark the Sentry issue resolved only
  after the PR merges. Invoke when the user asks to fix
  Sentry issues / triage the Sentry queue in reactive-bible.
---

# Fix an unresolved Sentry issue (reactive-bible-web)

The target repo is `reactive-bible/`; the GitHub repo is
`Bible-Research/reactive-bible` (remote `origin`). The Sentry
project is `reactive-bible-web` in org `bible-research`, hosted
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

This is a different token from the Vercel build-time
`SENTRY_AUTH_TOKEN` (which is release-upload scoped).

API helpers used throughout:

    BASE="https://de.sentry.io/api/0"
    AUTH="Authorization: Bearer $SENTRY_API_TOKEN"
    ORG="bible-research"
    PROJECT="reactive-bible-web"

## 1. Fetch the work queue — two sources

### 1a. Bug issues (exceptions)

    curl -s -H "$AUTH" \
      "$BASE/projects/$ORG/$PROJECT/issues/\
?query=is:unresolved+level:error+environment:production\
&statsPeriod=14d&sort=freq"

Only `level:error` exception events are auto-fixable.
`reportType:*` streams and warning-level events are excluded
from this query by the level filter.

### 1b. User reports ("Report a problem" menu flow)

Every `sendUserReport` call emits a `user report` message event
that groups under ONE issue. Find that issue, then list its
events:

    curl -s -H "$AUTH" \
      "$BASE/projects/$ORG/$PROJECT/issues/\
?query=reportType:user-report"
    curl -s -H "$AUTH" "$BASE/issues/<issue_id>/events/"

(Equivalent: `GET $BASE/projects/$ORG/$PROJECT/user-feedback/`
lists feedback entries with linked event ids.)

Each event carries a `userReport` context: `note` (the
description), `reportedBy`, `bookId`, `chapter`,
`textFilesetId`, `audioFilesetId`. Filter out event ids already
in the processed ledger (step 7). Today only developers file
reports, so treat every unprocessed report as a high-signal bug
report.

## 2. Pick and analyze one item

- Default: ONE item per run — the highest-count/recent bug
  issue, or the oldest unprocessed user report. If the user
  said "fix all", enumerate first and post the list.
- Error issue: fetch
  `GET $BASE/issues/<id>/events/latest/` — exception type,
  sourcemapped stack frames (file/line/function), `context` /
  `errorKind` tags, release sha, breadcrumbs.
- User report: read the `userReport` context and try to
  reproduce — navigate to that passage/fileset, re-issue the
  same API calls (read-only against prod is fine), run the
  flow. Reproduced → proceed as a bug. Can't reproduce → write
  `{event_id: {verdict: "cannot-reproduce", date}}` to the
  ledger, report back to the user, and move on — NEVER a
  speculative fix.

## 3. Decide which repo the fix belongs in

Same convention as fix-good-first-issue-frontend: a frontend
`ProviderError` spike can be a backend contract change
(`error`/`error_code`, `fileset_id` routing). If the root cause
is in `bible_research`, work there per `bible_research/AGENTS.md`
(pytest, NEVER run migrations) and use the
fix-sentry-issue-backend branch/PR flow against
`Bible-Research/bible_research`. If unsure, state reasoning and
ask the user.

## 4. Branch and fix

    git -C reactive-bible switch main
    git -C reactive-bible pull --ff-only
    git -C reactive-bible switch -c fix/sentry-<issue_short_id>
    # or: fix/report-<event_id_prefix> for user reports

- Read `reactive-bible/AGENTS.md`; 79-char lines everywhere.
- Write a failing test reproducing the event first, then fix.
- Verify: `npm run lint` (0/0), `npm run build`,
  `npm test -- --run` (scoped tests at minimum).
- Update `DEVELOPER_GUIDE.md` if behavior changed.
- If the working tree has unrelated dirty files, stage only
  your own files by path; never disturb the user's edits.
  `export PATH="/opt/homebrew/bin:$PATH"` on every exec call —
  node/npm/gh are not on the default PATH.

## 5. Draft PR

- Stage only files you edited (never `git add -A`). Commit
  format `Type: Capitalized message`.
- Push `-u origin`, then draft PR `--base main` per
  `.devin/workflows/create-pr.md` (body via `--body-file`).
- Body must: link the Sentry issue URL
  (`https://bible-research.sentry.io/issues/<id>/` — EU orgs
  also accept de.sentry.io URLs), quote the top stack frame
  (or the report text for user reports), and include the
  UI-verification section required by global rules.
- `gh pr checks <N> --watch`; fix until green; `gh pr ready`.

## 6. Resolve — only after merge

Bug issues:

    curl -s -X PUT -H "$AUTH" \
      -H 'Content-Type: application/json' \
      -d '{"status": "resolvedInNextRelease"}' \
      "$BASE/issues/<id>/"

(Frontend release = commit sha; resolution takes effect on the
next deploy.) NEVER resolve before merge; NEVER `ignore` /
`archive` an issue without asking the user.

## 7. Processed-report ledger

`~/.config/devin/sentry-seen-reports.json` maps each handled
user-report event_id to `{verdict, pr_url, date}` — Sentry
feedback has no per-report status field, and the catch-all
`user report` issue must stay unresolved by design (it's a
stream, not a bug). Create the file if absent; add an entry
after every handled report (fixed → `pr_url`; unreproducible →
`cannot-reproduce` + note).

## Guardrails

- `media-debug` events (`environment:android-debug`) are
  read-only diagnostics — digest output only, never auto-fix.
- If >3 actionable items exist, post the queue list and let
  the user pick before fixing.
- Report at the end: issue/report handled, PR URL, Sentry
  resolution status, and any reports logged as
  cannot-reproduce.
