# Proposal

## Why

Grouped search (`group_by=book`, the only mode the frontend
calls) on SWORD filesets sequentially scans all ~1,189 chapters
of the module via pysword, which re-decompresses the same zText
chapter block once per verse (~35k decompressions for a whole-
Bible scan). Measured: ~8s on a developer machine for a common
word; on the production F1 instance this plausibly exceeds
gunicorn's 30s worker timeout and App Engine's 60s deadline, so
the worker is killed and the browser shows "Failed to fetch" —
and because no Python exception is raised and the frontend
swallows the error into UI state, Sentry never records it.
Additionally there is no kill switch for SWORD search: if it
misbehaves again the only remedy is a redeploy.

## What Changes

- **Resumable grouped search for SWORD filesets**: the search
  endpoint accepts an opaque `cursor` parameter; each request
  scans chapters only within a ~6s wall-clock budget and returns
  the matches found so far (grouped per book) plus
  `meta.next_cursor` when the scan is incomplete. The frontend
  iterates from Genesis to Revelation, merging per-book groups
  across responses, until `next_cursor` is absent. The scan
  never times out from the client's perspective — each response
  is a fast 200, not an error.
- **Decompression-block cache**: the pysword zText block cache
  makes each ~6s window cover far more ground (measured ~12x
  faster full scan locally: 8.3s -> 0.7s), so most searches
  complete in one or two windows.
- **Database-backed feature flag** `sword_search`: a
  `FeatureFlag` model (key, enabled, description) editable via
  Django admin, read-through-cached so checks cost ~nothing.
  When disabled, SWORD-fileset search requests return a coded
  error (`search_disabled`, HTTP 403) before provider dispatch;
  the search bar stays visible and the frontend shows a
  translated "search is not supported for this translation"
  message. Missing flag row defaults to enabled (kill-switch
  semantics, not dark launch).
- **Frontend**: `searchBibleGrouped` loops on `next_cursor`,
  merges verse lists per `book_id`, reports scan progress to the
  UI, and maps the new `search_disabled` code to a typed error.

## Capabilities

### New Capabilities

- `bible-search`: the `/api/v1/bible/search/` contract —
  grouped-by-book mode, SWORD resumable scanning with cursor +
  time budget, feature-flag gating of SWORD search, and the
  frontend's iterative consumption of `next_cursor`.

### Modified Capabilities

- `error-contract`: adds the `search_disabled` error code and
  its HTTP 403 mapping for feature-flagged-off SWORD search.

## Impact

- **Repos**: both — `bible_research` (scan helper, provider,
  view, new model + migration, admin, throttle-visible request
  pattern) and `reactive-bible` (search client loop, group
  merging, progress UI, error mapping, i18n strings).
- **APIs**: `GET /api/v1/bible/search/` gains an optional
  `cursor` query param and `meta.next_cursor` /
  `meta.progress` response fields; `search_disabled` added to
  the `error_code` taxonomy. No existing field changes.
- **Database**: new `FeatureFlag` table (migration required —
  a human runs `makemigrations`/`migrate`, agents may
  hand-write the migration file but never run it).
- **Providers**: SWORD-only. ESV/API.Bible/DBT search paths
  are untouched; the flag check runs only for SWORD filesets.
