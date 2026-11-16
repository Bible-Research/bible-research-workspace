# Tasks

## 1. Backend: scan deadline + cursor + block cache

- [x] 1.1 backend: In `SwordClient._load`, wrap the loaded
  bible's `_decompressed_text` in `functools.lru_cache(
  maxsize=8)`; verify via a scan benchmark or unit test that
  `scan_sword_matches` over `LVSGLU8` completes markedly
  faster (locally ~8s -> ~1s) and chapter reads still pass
  their tests.
- [x] 1.2 backend: Add `deadline_s` and `start_index` params
  to `scan_sword_matches`; return `(matches, truncated,
  next_index)` where `next_index` is the unscanned chapter
  position or `None`; add
  `SEARCH_SCAN_BUDGET_SECONDS = 6` constant and check the
  budget between chapters. Verify with a unit test that a
  `deadline_s=0` scan returns `next_index` and matches only
  from the first chapters.
- [x] 1.3 backend: Extend `SwordProvider.search_grouped(
  query, books, sort_by, cursor=None)` to decode the cursor,
  pass `start_index`, and emit `meta.next_cursor` (opaque
  encoded index) plus `meta.progress` (`chapters_scanned`,
  `chapters_total`); verify a two-window call sequence
  returns disjoint results that union to the full scan.
- [x] 1.4 backend: Accept+validate `cursor` in
  `BibleSearchView` (charset/shape check → 400
  `invalid_parameter` on malformed values; pass through to
  `search_grouped`); verify tests cover valid, invalid, and
  absent cursors. Non-SWORD providers ignore the param.
- [x] 1.5 backend: Update the `BibleSearchView` docstring/
  drf-spectacular schema and `DEVELOPER_GUIDE.md` search
  section for `cursor`/`next_cursor`/`progress`; verify the
  docs render and `awk 'length > 79 {...}'` is clean on
  touched files.

## 2. Backend: FeatureFlag model + search gate

- [x] 2.1 backend: Add `FeatureFlag` model to `bible/
  models.py` (`key` unique, `enabled`, `description`,
  `updated_at`) and hand-write the migration file (agents do
  NOT run `makemigrations`); verify the model/migration load
  under pytest on SQLite (test DB builds the table).
- [x] 2.2 backend: Add `is_sword_search_enabled()` helper —
  Django cache read-through (~30s TTL) defaulting to enabled
  on a missing row — and register `FeatureFlag` in
  `bible/admin.py` with save-time cache invalidation; verify
  unit tests cover enabled, disabled, missing-row, and
  cache-invalidation paths.
- [x] 2.3 backend: In `BibleSearchView.get`, after param
  validation and before provider dispatch, return 403
  `{error, error_code: "search_disabled", message}` when the
  flag is off and `is_sword_fileset(fileset_id)`; verify
  tests assert the coded 403 for SWORD ids and normal
  behavior for a non-SWORD id with the flag off.
- [ ] 2.4 backend: Note in the PR body that a human must run
  `python manage.py migrate` on prod before/alongside deploy
  (and `makemigrations --check` to confirm the hand-written
  migration matches the model). Verification = PR checklist
  item, not code.

## 3. Frontend: iterative search client + disabled state

- [x] 3.1 frontend: Loop in `searchBibleGrouped`: forward
  `cursor` on `meta.next_cursor`, merge groups by `book_id`
  (append + re-sort verses, first-seen canonical group
  order), thread `AbortSignal` across windows; verify vitest
  mocks returning two windows produce one merged result.
- [x] 3.2 frontend: Add `onProgress` callback plumbing
  (`meta.progress`) and render scan progress in
  `SearchRoute` (determinate indicator or scanned-books
  count); verify a component test shows progress across
  windows and final groups.
- [x] 3.3 frontend: Map `search_disabled` to a typed error
  in `src/api/errors.ts` and render a translated
  "search is not supported for this translation" message in
  `SearchRoute` (add i18n keys for all locales); verify a
  test asserts the message, not a generic error.
- [x] 3.4 frontend: Update `DEVELOPER_GUIDE.md`/docs for the
  resumable search contract and flag-gated behavior; verify
  `npm run lint` is clean and line length ≤79 on touched
  files.

## 4. Integration verification

- [x] 4.1 both: Manual smoke — search `Jēzus` on `LVSGLU8`
  locally/prod: completes without "Failed to fetch", groups
  appear in canonical order, progress indicator advances.
- [x] 4.2 backend: Force multi-window (tiny
  `SEARCH_SCAN_BUDGET_SECONDS` in a test or local settings):
  verify repeated windows converge to the same merged result
  as a single full scan.
- [x] 4.3 backend: Toggle `sword_search` off in Django admin
  → SWORD search 403s with `search_disabled`; toggle on →
  search works; ESV/DBT searches unaffected throughout.
