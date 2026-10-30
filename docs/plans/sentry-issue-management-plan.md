# Sentry reporting standardization & agent-driven issue fixing

**Status:** partial — Part 1 code merged via
bible_research#111 + reactive-bible#198 (+ commit on #195,
branch-resident); Part 2 skills written at
`.devin/skills/fix-sentry-issue-{frontend,backend}/`.
Pending: Sentry UI config (1d), SENTRY_API_TOKEN (2a).
**Date:** 2026-10-27
**Repos:** `reactive-bible`, `bible_research` + workspace `.devin/skills/`
**Builds on:** `sentry-monitoring-plan.md` (implemented),
`media-debug-sentry-apk-plan.md` (PR #195 closed unmerged),
`bug-report-feature-plan.md` (menu flow shipped in fdb9a41)

## Context — what streams into Sentry today

`reactive-bible-web` project:

- Unhandled exceptions + rejections — `reportError` via
  `initGlobalErrorHandlers`. Level error; real bugs.
- React render crashes — `ErrorBoundary.componentDidCatch`
  (+ `reportError`, client-deduped). Level error; real bugs.
- Resource-load failures — `reportError(resource)` →
  `captureMessage`, deduped per URL/session, query string
  stripped. Level warning; usually real but low priority.
- "Report a problem" menu flow — `sendUserReport` →
  `captureMessage('user report')` + `captureFeedback`. Level
  info; deliberate feedback, not a bug.
- Media-debug snapshots (PR #195, pending) —
  `sendMediaDebugReport` → `captureMessage('media-debug
  snapshot')` + attachment. Level info; diagnostics, not a bug.

`bible-research-api` project:

- Unhandled 500s — `DjangoIntegration`. Level error; real bugs.
- Caught server bugs (GCS signing, catch-alls) —
  `capture_internal_exception`. Level error, tagged
  `error_code:internal_error`; real bugs.
- Upstream provider outages — `capture_provider_exception`
  (`rate_limited`, `provider_error` only). Level warning,
  tagged `error_code`; trend-only.
- Logs — `LoggingIntegration(event_level=None)`. Breadcrumbs
  only, never events.

Existing filters: `sentryBeforeSend` drops `RateLimitError`,
`BookNotInFilesetError`, `AbortError`, "Session expired"
(`reactive-bible/src/utils/sentry.ts`); backend `before_send`
strips `request.data` on auth paths
(`bible_research/bible_research/utils.py`). Both sides DSN-gated
— zero traffic without a DSN.

## Problem

The event taxonomy is sound, but the **issue stream is not
machine-triaged**:

1. `user report` and `media-debug snapshot` message events each
   create a permanent unresolved issue group. Any naive
   `is:unresolved` query (or "new issue" alert) mixes deliberate
   info streams with real bugs.
2. No `ignoreErrors` on the frontend for classic noise:
   network flakes (`TypeError: Failed to fetch` / `Load failed`
   / `NetworkError`), `ResizeObserver loop` messages,
   browser-extension errors, chunk 404 storms right after a
   Vercel deploy (hashed-asset URLs go stale in open tabs).
3. Tag taxonomy is ad hoc: frontend uses free-form `context`
   strings + `reportType`; backend uses `error_code`. Nothing
   documents which tag/level combinations mean "actionable bug".
4. Frontend `release` is only set when the sentry vite plugin
   runs (`SENTRY_AUTH_TOKEN` present at build); Android builds
   tag `production` until PR #195's `VITE_APP_ENV` lands.
5. Alert rules from `sentry-monitoring-plan.md` are still
   pending — if created without exclusions they will email on
   `user report` submissions.

Goal: a documented contract where **`is:unresolved
level:error environment:production` = bugs an agent should
fix**, and everything else is deliberately routed elsewhere.

## Part 1 — Standardize reporting

### 1a. Event-stream contract (document; small code changes)

Actionable streams must all be `level:error` exception events.
Non-bug streams must be `captureMessage` at `info`/`warning`
with a `reportType` tag:

- (no `reportType` tag) — exceptions, level error →
  agent-fixable issue queue.
- `reportType:resource-load` — warning → issue queue,
  low-priority filter.
- `reportType:user-report` — info → excluded from the *error*
  queue (it is not an exception) but consumed by the agent
  skill as investigation tasks (2c). Today only developers
  file reports, so every one is high-signal — revisit triage
  rules if real users ever report at volume.
- `reportType:media-debug` — info → excluded; environment
  `android-debug`.

Code changes:

- `errorReporter.tsx`: add `reportType: 'resource-load'` to the
  existing resource `captureMessage` tags (keeps `context`).
- `mediaDebug.ts` (PR #195): tag `reportType: 'media-debug'`
  on snapshot messages (currently only `reason` is tagged).
- `userReport.ts`: already tags `reportType: 'user-report'` — no
  change.

### 1b. Noise filters (`sentryBeforeSend` additions)

Extend `src/utils/sentry.ts` drop list with:

- `err.name === 'ChunkLoadError'` or messages matching
  `Loading chunk \d+ failed|Loading CSS chunk` — stale tabs
  after deploy; the resource-load warning already records the
  missing asset.
- `ResizeObserver loop` messages — benign browser noise.
- Optionally tag (not drop) network failures: `TypeError`
  with message in {`Failed to fetch`, `Load failed`,
  `NetworkError when attempting`} gets
  `tags: { errorKind: 'network' }` so Sentry-side rules can
  route them without losing availability signal. Decide drop
  vs tag after one week of volume data.
- Keep everything currently dropped. `ClientInputError` stays
  reportable (it's a bug by design).

Backend needs no equivalent — its capture list is already an
explicit allowlist (`_CAPTURED_ERROR_CODES`).

### 1c. Release/environment correctness

- Always set `release` in `main.tsx` when a value exists:
  `release: import.meta.env.VITE_APP_RELEASE`, and emit
  `VITE_APP_RELEASE=<sha>` in the Vercel build env too (today
  release only exists via the vite plugin when sourcemap upload
  is enabled — events from builds without the token are
  releaseless).
- PR #195 already fixes APK env tagging
  (`VITE_APP_ENV=android|android-debug`); merge it before
  treating `environment` as trustworthy.
- No `environment`/`release` changes needed on the backend
  (`GAE_VERSION` covers both).

### 1d. Sentry-side project config (manual, UI — user)

Per project (`bible-research` org at **de.sentry.io**):

- Inbound filters ON: browser extensions, localhost, legacy
  browsers, web crawlers, health checks.
- Data scrubbing: add `password`, `token` to sensitive fields
  (complements both `before_send`s).
- Alert rules (email), matching the pending table in
  `sentry-monitoring-plan.md` but with exclusions:
  - New issue: `environment:production AND level:error` —
    excludes `user report`/`media-debug`/resource warnings.
  - Regression, spike: same level filter.
  - Provider fails (api): `error_code:rate_limited OR
    error_code:provider_error`, low frequency, no page.
  - Never alert on `reportType:*` or `environment:development`/
    `preview`/`android-debug`.
- Saved queries (share with org):
  - "Agent queue": `is:unresolved level:error
    environment:production`
  - "User feedback": `reportType:user-report`
  - "Provider health" (api): `level:warning error_code:*`

### 1e. Doc updates

- `reactive-bible/AGENTS.md` + `bible_research/AGENTS.md`: the
  event-stream contract table (what each level/tag means, what
  agents may auto-fix vs must leave alone).
- `DEVELOPER_GUIDE.md` (frontend): the `reportType` taxonomy.

## Part 2 — Agent skill: fix unresolved Sentry issues

Mirror the existing `fix-good-first-issue-{frontend,backend}`
skills: `.devin/skills/fix-sentry-issue-{frontend,backend}/SKILL.md`
(plain `name`/`description` frontmatter only — extra fields
break discovery).

### 2a. Auth & access (user does once)

- `sentry-cli` is already installed (`/opt/homebrew/bin`), but
  it has no issue-listing commands — agents use the REST API.
- Region: **EU** — base URL is `https://de.sentry.io/api/0/`,
  NOT sentry.io.
- Create a Sentry **internal integration** (org Settings →
  Developer Settings → New Internal Integration):
  permissions `Issue & Event: Read & Write`, `Project: Read`,
  `Release: Read`, `Organization: Read`. Copy its token.
- Store in `~/.config/devin/sentry.env` as
  `SENTRY_API_TOKEN=...` (home-dir config, never in a repo;
  keep separate from the Vercel build-time
  `SENTRY_AUTH_TOKEN`, which is release-upload scoped).
- Skill sources it:
  `source ~/.config/devin/sentry.env` before `curl` calls.
- Optional alternative: the official Sentry MCP server (hosted
  at `mcp.sentry.io`) gives agents `find_issues`/
  `get_issue_details` tools with OAuth — nicer UX, but adds an
  MCP dependency and EU-region support must be verified. REST
  API is the baseline.

### 2b. API surface the skill uses

```bash
BASE="https://de.sentry.io/api/0"
AUTH="Authorization: Bearer $SENTRY_API_TOKEN"

# Actionable queue per project:
GET $BASE/projects/bible-research/reactive-bible-web/issues/\
?query=is:unresolved+level:error+environment:production\
&statsPeriod=14d&sort=freq
GET $BASE/projects/bible-research/bible-research-api/issues/?...

# Issue detail + latest event (stacktrace, tags, breadcrumbs,
# contexts, release):
GET $BASE/issues/<issue_id>/
GET $BASE/issues/<issue_id>/events/latest/

# User reports — two enumeration paths:
#  a) feedback entries with linked event ids:
GET $BASE/projects/bible-research/reactive-bible-web/\
user-feedback/
#  b) every 'user report' message groups under ONE issue —
#     list its events (each carries the userReport context):
GET $BASE/issues/<user_report_issue_id>/events/

# Mark after merge — never before:
PUT $BASE/issues/<issue_id>/ {"status": "resolvedInNextRelease"}
```

### 2c. Skill workflow (mirrors fix-good-first-issue)

1. Fetch the project's work queue — two sources:
   a. Bugs: `is:unresolved level:error
      environment:production` (info/warning streams excluded).
   b. User reports: all events on the `user report` message
      issue (`reportType:user-report`, or the `user-feedback/`
      list), filtered against the processed ledger (2d).
      Reports are dev-filed today, so each is a high-signal
      bug report, not noise.
2. Analyze the work item:
   - Error issues: map event → code via exception type,
     sourcemapped stack frames, `context`/`error_code` tags,
     release sha, breadcrumbs.
   - User reports: read the `userReport` context (`note`,
     `reportedBy`, `bookId`, `chapter`, fileset ids) and try
     to reproduce — navigate to the passage, re-issue the
     same API calls (read-only against prod where safe), run
     the affected flow. Reproduced → treat as a bug and
     continue. Can't reproduce → log event_id + verdict in
     the ledger, report back to the user; no speculative fix.
3. Check whether the fix belongs in the *other* repo (existing
   skill convention — e.g. a frontend `ProviderError` spike
   caused by a backend contract change).
4. Branch `fix/sentry-<issue_short_id>` (or
   `fix/report-<event_id_prefix>` for user reports) off
   `origin/main`, write failing test reproducing the event,
   fix, run `npm run lint && npm test -- --run <touched>` /
   `pytest <touched>`.
5. Draft PR per `.devin/workflows/create-pr.md`; body links the
   Sentry issue URL, quotes the top frame (or the report text
   for user reports), and includes the UI-verification section
   required by global rules.
6. After the PR merges:
   - Bug issues: `PUT` `resolvedInNextRelease` (frontend:
     release = commit sha; backend: `GAE_VERSION` — next deploy
     auto-resolves).
   - User reports: record event_id + PR URL in the ledger.
     The catch-all `user report` issue group stays unresolved
     by design — it's a stream, not a bug.
   Never resolve pre-merge; never `ignore`/`archive` an issue
   without asking the user.

### 2d. Guardrails

- User reports are in scope (dev-filed → high signal), but a
  report that can't be reproduced is logged to the ledger and
  reported back — never a speculative fix, never resolved
  silently.
- `media-debug` stays read-only (diagnostics stream) — digest
  output only, no fixes.
- Processed-report ledger:
  `~/.config/devin/sentry-seen-reports.json` — maps
  event_id → `{verdict, pr_url, date}`. Home-dir state, never
  committed; it is the dedupe key since Sentry feedback has
  no per-report status field.
- Cap one issue per run unless the user says "fix all"; post
  the queue list first when >3 actionable items exist.
- If `SENTRY_API_TOKEN` is missing, the skill prints setup
  steps (2a) instead of probing.
- 79-char rule, commit format, per-repo staging rules all
  apply — identical to the good-first-issue skills.

## Task order

1. Frontend PR: `reportType` tags, `sentryBeforeSend` noise
   additions, `release` from `VITE_APP_RELEASE`, tests +
   `DEVELOPER_GUIDE.md`.
2. Merge PR #195 (adds `media-debug` tag requirement — fold
   the tag change into it or a follow-up).
3. User: Sentry UI config (1d), internal-integration token →
   `~/.config/devin/sentry.env`.
4. Workspace: write `fix-sentry-issue-frontend` +
   `fix-sentry-issue-backend` skills; update both AGENTS.md
   files with the event-stream contract.
5. Smoke: skill lists the queue read-only; then fix one real
   issue end-to-end.

## Open questions

1. Drop network-failure events entirely, or keep tagged
   `errorKind:network` (recommend tag; revisit after volume
   data)?
2. Should handled user reports also open a GitHub issue for
   durable tracking? The ledger covers dedupe; GitHub issues
   would make report→fix linkage visible to the reporter
   (`bug-report-feature-plan.md`'s DB+GitHub pipeline remains
   the bigger unbuilt alternative). Decide once non-dev
   reporters exist.
3. Resource-load warnings: keep in the agent queue at
   `level:error`? (Recommend: keep at warning, agent checks them
   weekly rather than per-run.)
