# Sentry integration standard (reusable playbook)

**Status:** reference — distilled from the implemented
`bible-research` setup (reactive-bible#192/#194/#198 + fdb9a41,
bible_research#99/#111, `.devin/skills/fix-sentry-issue-*`).
**Purpose:** give an AI agent in ANY repository the steps and
code patterns to integrate Sentry the same way, including
API-driven org/project/alert setup and the operational
gotchas discovered in production.
**Sources:** `sentry-issue-management-plan.md`,
`media-debug-sentry-apk-plan.md`, both repos' `AGENTS.md`
"Sentry event streams" sections.

---

## 1. Architecture

- **One Sentry organization; one project per deployable**
  (never one per environment). Suffix by deployable type:
  `<product>-api` (Django), `<product>-web` (React). Environment
  is a tag sent by the SDK: `production`, `preview`,
  `development`, `android`, `android-debug`.
- **Thin chokepoint, not an abstraction layer.** All capture
  calls funnel through the app's existing reporting paths
  (frontend `reportError`/`ErrorBoundary`, backend provider-
  error helpers + `DjangoIntegration`). Provider swap later =
  ~2 files. GlitchTip accepts Sentry-protocol DSNs as the
  self-hosted escape hatch.
- **DSN-gated everything.** No DSN → the SDK never initializes
  → zero traffic, zero bundle impact (tree-shaken init).

## 2. Provisioning via REST API (no UI needed)

Region matters — pick the org's region host (`de.sentry.io`
for EU) and use it everywhere, including `SENTRY_URL` for
tooling.

```bash
BASE="https://de.sentry.io/api/0"
AUTH="Authorization: Bearer $TOKEN"
# Org token (sntrys_, org:ci scope) CANNOT create projects —
# needs a user token (sntryu_) or internal integration.
POST $BASE/teams/<org>/<team>/projects/ \
  {"name":"bible-research-api","platform":"python-django"}
GET  $BASE/projects/<org>/<proj>/keys/        # → DSN
```

- **Alerts: use `/workflows/`, not `/rules/`** — the legacy
  rules endpoint 404s on current Sentry. Detectors are
  auto-created per project; workflow `triggers` only accept
  state-change types (`first_seen_event`,
  `regression_event`, `reappeared_event`); count-based
  conditions (`issue_occurrences`) go in the filter block.
  Always scope alerts to `environment:production` AND
  `level:error` so deliberate info/warning streams
  (user reports, diagnostics) never page.
- **Project keys rate limit** (abuse protection for the
  public DSN): `PUT …/keys/<id>/` with a NESTED object —
  `{"rateLimit":{"window":3600,"count":100}}`. Flat or
  snake_case fields silently no-op.
- **Inbound filters**: `PUT …/filters/<filter>/`
  `{"active":true}`. Web project: `browser-extensions`,
  `web-crawlers`, `legacy-browsers`, `localhost`;
  `filtered-transaction` is on by default. Backend: leave
  web-noise filters off (never match).
- **Org scrubbing**: `PUT /organizations/<org>/`
  `{"dataScrubber":true,"sensitiveFields":["password","token",
  "authorization","api_key","secret","access_token",
  "refresh_token","session","csrftoken"]}` — belt-and-
  suspenders under the SDK-side `before_send` scrubbers.
- **Code mappings** (stack-trace→repo links): if frames are
  repo-relative already, BOTH prefixes stay empty — verify by
  reading a real event's `filename`/`abs_path` first
  (`GET /issues/<id>/events/latest/`).
- **Agent access token**: create a Sentry internal
  integration (Issue&Event RW, Project R, Release R, Org R);
  store as `SENTRY_API_TOKEN` in `~/.config/devin/sentry.env`
  — never in a repo, and separate from the build-time
  `SENTRY_AUTH_TOKEN` (release-upload scoped).

## 3. Backend pattern (Django/DRF)

- `sentry-sdk` in requirements; init in `settings.py` ONLY
  when `SENTRY_DSN` resolves (env → local config → cloud
  secret store; GCP: metadata secret in Terraform + version
  via `gcloud secrets versions add`; project-wide
  `secretAccessor` binding already covers new secrets).
- `DjangoIntegration` (unhandled 500s = level error),
  `LoggingIntegration(level=INFO, event_level=None)` — logs
  are breadcrumbs, NEVER events; DEBUG dropped.
- `release`: `GAE_VERSION`/deploy sha → `resolvedInNextRelease`
  auto-resolves at next deploy.
- `send_default_pii=False`; `before_send` strips
  `request.data` on auth/user paths (`/api/token` with OR
  without trailing slash + query strings; guard non-dict
  `request`).
- **Explicit capture allowlist for caught exceptions** —
  handled failures can't reach `DjangoIntegration`: capture
  upstream/provider outages at WARNING with an `error_code`
  tag (`rate_limited`, `provider_error`); skip expected states
  (`not_found`, `book_not_in_fileset`); capture internal
  server bugs at ERROR tagged `error_code:internal_error`.
- CORS: allow `sentry-trace` + `baggage` headers so frontend
  traces propagate.

## 4. Frontend pattern (React + Vite)

- `Sentry.init` before `createRoot`, guarded by
  `import.meta.env.VITE_SENTRY_DSN` — absent DSN means the
  whole block tree-shakes out of the bundle.
- Integrations: `browserTracingIntegration` +
  `replayIntegration({maskAllText:true, blockAllMedia:true})`;
  `replaysSessionSampleRate:0` + `replaysOnErrorSampleRate:1.0`
  (only config that fits the ~50/mo free replay tier);
  `tracesSampleRate:0.1`; `tracePropagationTargets:[API_BASE_URL]`.
- **Always define release + env at build** — do NOT rely on
  the sourcemap plugin for release (it only runs when upload
  creds exist): vite `define` bakes
  `VITE_APP_ENV = VITE_APP_ENV ?? VERCEL_ENV ?? ''` and
  `VITE_APP_RELEASE = VITE_APP_RELEASE ?? VERCEL_GIT_COMMIT_SHA
  ?? ''`; pass `release: <val> || undefined` (never `''`).
- Sourcemap plugin gated on ALL of `SENTRY_AUTH_TOKEN` +
  `SENTRY_ORG` + `SENTRY_PROJECT` (process.env only, NEVER
  `VITE_`-prefixed); `sourcemap:'hidden'` + delete maps after
  upload, and emit maps only when upload is enabled.
- `beforeSend` drop list — expected/handled failures
  (app-specific error classes, `AbortError` including
  DOMException, 'Session expired') PLUS generic web noise:
  `ChunkLoadError`/`Loading chunk \d+ failed`/`Loading CSS
  chunk` (stale tabs post-deploy), `ResizeObserver loop`.
  Match patterns against `event.message` +
  `exception.values[0].value` too — GlobalHandlers' onerror
  path produces message-only events without an Error instance.
- Tag-don't-drop availability signals: `TypeError` network
  failures get `errorKind:'network'`.
- Re-apply `context` in `beforeSend` when the Sentry
  Dedupe integration keeps the GlobalHandlers copy of an
  unhandled rejection (it drops the tagged duplicate).
- Strip query strings from resource URLs before capture
  (signed URLs leak credentials).

## 5. Event-stream contract (the core standard)

**`is:unresolved level:error environment:production` is THE
agent-fixable bug queue** — every stream is designed to be
either in it or deliberately out:

- Unhandled exceptions/500s — auto or `reportError`,
  exception, level **error** → in queue.
- Internal bugs (caught 500s) — `captureException`, level
  **error**, tag `error_code:internal_error` → in queue.
- Resource-load failures — `captureMessage`, **warning**,
  `reportType:resource-load` → low-priority view.
- Provider outages — `captureException`, **warning**,
  `error_code:{rate_limited,provider_error}` → trend-only.
- User bug reports — `captureMessage('user report')` +
  `captureFeedback` linked by `associatedEventId`, **info**,
  `reportType:user-report` → agent task stream (see §7).
- Debug snapshots — `captureMessage` + attachment, **info**,
  `reportType:*` (e.g. `media-debug`) → excluded entirely.

Rules: actionable = level error exceptions; non-bug streams =
`captureMessage` at info/warning with a `reportType` tag;
routine logs = breadcrumbs only. Document this table in each
repo's `AGENTS.md`.

## 6. Operational gotchas (learned in production)

- **EU-region 401s**: `sntrys_` tokens are region-bound —
  the bundler plugin/cli 401s on release upload unless
  `SENTRY_URL=https://de.sentry.io` is set.
- **Vercel env vars bake in at build time** — setting them
  changes nothing until a redeploy; `VITE_*` is the only
  prefix that reaches the client; verify the Production scope
  checkbox. Diagnostic: a `[sentry-env-debug]` console.log in
  `vite.config.ts` printing var PRESENCE (never values) in the
  build log.
- **`*.vercel.app` subdomains are reserved to the project of
  the same name** — confirm which project serves the prod
  domain (forks leave the upstream project's deployment live
  at the repo-named domain, serving stale builds forever).
- **`gcloud app deploy` uploads the local working tree** —
  always deploy from a clean worktree on `origin/main`.
- **CI builds need secrets too**: APK/workflow builds need
  `VITE_SENTRY_DSN` as a repo secret (`gh secret set`) —
  missing → silent no-op builds.
- **Debug-APK channel pattern**: non-main CI builds with
  `VITE_MEDIA_DEBUG=1` + `applicationIdSuffix ".debug"`
  (side-by-side install), offline Sentry transport +
  localStorage ring buffer + unlock-triggered report upload
  for lock-screen diagnostics. Fresh debug keystore per CI
  run → uninstall before reinstall.
- **Preview envs**: `import.meta.env.MODE` is `production` on
  Vercel previews — bake `VERCEL_ENV` via define or previews
  report as production.
- **Verify prod by grepping the served bundle** for
  `ingest.<region>.sentry.io` — the only ground truth that
  the DSN reached the build.

## 7. Agent operations

- Issue queue per project:
  `GET /projects/<org>/<proj>/issues/?query=is:unresolved+
  level:error+environment:production&sort=freq`
- `GET /issues/<id>/events/latest/` — frames, tags,
  breadcrumbs, contexts, release.
- User reports: `GET /projects/<org>/<proj>/user-feedback/`
  or list events on the `user report` message issue (the
  `userReport` context carries the note/reporter/passage).
- Resolve ONLY after the fix merges:
  `PUT /issues/<id>/ {"status":"resolvedInNextRelease"}`.
  Never resolve pre-merge; never archive without asking.
- Reference skill: `.devin/skills/fix-sentry-issue-{front,
  back}end/` in this workspace.
