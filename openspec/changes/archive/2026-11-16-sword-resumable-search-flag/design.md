# Design

## Context

See proposal.md "Why". Relevant current state:

- `scan_sword_matches` (`bible/services/search_grouping.py`)
  walks `client.list_chapters(fileset_id)` sequentially, calling
  `SwordClient.get_chapter_verses` per chapter, which calls
  `bible.get()` **per verse**. Each pysword `_text_for_index`
  calls `_decompressed_text(testament, buf_num)` with no
  caching — the same zText chapter block is re-seeked and
  re-decompressed once per verse (~35k decompressions per
  whole-Bible scan vs ~1,189 blocks actually needed).
- Measured locally vs `LvGluck8.zip`: full-module scan ~8.3s;
  with an `lru_cache` on `_decompressed_text` ~0.7s (~12x).
- Prod: GAE standard F1, `min_instances: 1`, gunicorn default
  `--timeout 30` — a slow scan gets its worker SIGKILLed, so no
  exception reaches Sentry; the browser sees "Failed to fetch".
- `SearchRateThrottle` scope is `search` at `30/min` — an
  iterative client making several window requests per search
  fits comfortably.
- Frontend `searchBibleGrouped` (`src/api/search.ts`) does a
  single `publicFetch` and `SearchRoute` merges nothing — it
  expects one complete `groups` array.
- Feature flags: none exist anywhere. No GCP Secret Manager
  changes allowed (cost rule); env-var flags would need secret
  or app.yaml changes plus a redeploy to toggle.

## Goals / Non-Goals

**Goals:**

- A SWORD grouped search always returns a fast 200 — partial
  results + resume cursor instead of a killed worker.
- The scan itself gets ~10x faster via zText block caching, so
  most searches finish inside one window.
- A runtime-toggleable, database-backed kill switch for SWORD
  search, manageable in Django admin.
- Cursor contract is stateless — no server-side session.

**Non-Goals:**

- No streaming (SSE/NDJSON) endpoint — the cursor loop
  achieves progressive results with plain request/response.
- No inverted-index artifact (word→verse postings) — revisit
  only if more SWORD modules are added.
- No change to ESV/API.Bible/DBT search behavior, to
  non-grouped `search()` pagination semantics, or to the
  passage/audio endpoints.
- No new GCP secrets, env vars, or infrastructure.

## Decisions

### D1: Cursor + time budget over streaming or async jobs

Each request scans at most `SEARCH_SCAN_BUDGET_SECONDS`
(default 6) wall-clock seconds, checked between chapters in
`scan_sword_matches`. On expiry it returns matches so far plus
`next_cursor`. Chosen over:

- *Streaming (SSE/NDJSON)*: progressive UX too, but holds a
  gunicorn worker for the whole scan, can't survive the 30s
  worker timeout anyway, and complicates the Vercel rewrite
  path. The cursor loop gets the same UX with plain 200s.
- *Async job + polling*: correct shape for long work but needs
  a task queue/storage we don't have; overkill for a ~1-7s
  scan.

Cursor encoding: the chapter worklist index (position in
`list_chapters` output after `books` filtering), serialized
e.g. `base64("v2:<ctx-hash>:<index>")` (implemented: bound
to a hash of `fileset_id|books|query` so a replayed cursor
under different params is rejected) — opaque, stateless,
deterministic
since the worklist is stable per fileset+books. An
undecodable/out-of-range cursor → 400 `invalid_parameter`.
The query+fileset must be resent each window (they're what
the scan needs anyway); we deliberately do not bind the
cursor to the query — a cursor from a different query just
resumes at a position, harmless.

`search_grouped` signature grows `cursor=None`; the provider
returns `{groups, meta: {total, truncated, next_cursor?,
progress}}` where `progress = {chapters_scanned,
chapters_total}` lets the UI show "Searching Judges (34/66)"
or a percent. `meta.total` = matches returned so far (lower
bound while scanning, consistent with `truncated`).

Non-grouped `search()` keeps its existing page/limit contract;
the shared scan helper gains the deadline so paginated mode
may return `truncated: true` sooner, but gains no cursor.

### D2: zText block cache in `SwordClient`

Wrap `_decompressed_text` per loaded `SwordBible` instance in
`_load` (after `get_bible_from_module`) with a small
`functools.lru_cache(maxsize=8)`. Only needs the current
chapter's block during a sequential scan; 8 entries ≈ a few
hundred KB worst case. `lru_cache` is thread-safe and the
underlying read is pure I/O. Scoped to the instance — ESV/
API.Bible/DBT unreachable, and regular chapter reads
(`get_chapter_verses` per-verse loop) speed up for free.

Alternative considered: whole-module in-memory plaintext
(~5 MB/fileset) — simpler still, but the block cache already
brings scans under the deadline and doesn't pin module-sized
memory on F1's tight RAM budget.

### D3: `FeatureFlag` model in `bible` app

```
FeatureFlag(key: str unique, enabled: bool,
            description: str, updated_at)
```

- Looked up via a helper `is_sword_search_enabled()` that
  reads Django `cache` (locmem in prod) with a short TTL
  (~30s) and falls back to **enabled** on a missing row —
  kill-switch semantics: the feature ships on; ops disables
  it without touching the flag-check code.
- Registered in `bible/admin.py` so ops toggles it in Django
  admin; admin saves must bust the cache entry.
- Check placement: `BibleSearchView.get`, after parameter
  validation, before provider dispatch, guarded by
  `is_sword_fileset(fileset_id)` — returns the standard coded
  error shape (`error`/`error_code`/`message`) at 403 via a
  `search_disabled` code, so `errors.ts` branches on it like
  every other coded error.

Alternatives considered:

- *Registry constant in `SWORD_TRANSLATIONS`*: zero infra but
  toggling needs a deploy — fails the "kill switch" goal.
- *Env var / setting*: toggling needs a Secret Manager or
  app.yaml change + redeploy; user explicitly excluded GCP
  secret changes.
- *django-flags/waffle*: real dependency for one flag; a
  4-field model + admin is smaller.

### D4: Frontend window loop in `searchBibleGrouped`

`searchBibleGrouped` loops: fetch → append groups → while
`meta.next_cursor`, re-fetch with `cursor` param. Merge by
`book_id`: if a chunk repeats a book, append its verses and
re-sort by (chapter, verse_start); group order re-sorted
canonically at the end via the existing book ordering
(`structure`/`bibleUtils` order or the order groups arrive —
groups arrive canonically within each window, so merge in
first-seen order). Accept an optional `onProgress` callback
receiving `meta.progress` for the UI to render a determinate
indicator; `AbortSignal` is honored between windows. The
`search_disabled` body flows through `throwFromErrorBody` →
map to a typed error (e.g. `SearchDisabledError`) in
`errors.ts`; `SearchRoute` renders a translated
"search not supported for this translation" empty-state.

## Risks / Trade-offs

- [Cursor resend cost] → each window re-runs
  `list_chapters` (cheap, cached module structure) and the
  flag check — bounded overhead.
- [Throttle exhaustion mid-search] → `search` scope is
  30/min; worst-case scans need ~1-3 windows after D2. If it
  proves noisy, exempt cursor-followups by making the budget
  bigger, not by loosening the throttle.
- [Flag check adds a DB read] → read-through cache with
  short TTL; SWORD searches are low-volume.
- [Missing-row-defaults-enabled could surprise] → documented
  kill-switch semantics; the migration can seed the row
  explicitly so admin shows it.
- [Hand-written migration drift] → the agent writes the
  migration file but a human runs `makemigrations --check` /
  `migrate`; CI `makemigrations --check` equivalent is noted
  as a manual step in tasks.

## Migration Plan

1. Backend PR: scan deadline + cursor + block cache +
   FeatureFlag model/migration + admin + view gate + tests.
   Migration is hand-written; **the user runs
   `python manage.py makemigrations --check` and
   `migrate`** (agent-forbidden commands) before deploy.
2. Frontend PR: iterative `searchBibleGrouped`, progress UI,
   `search_disabled` mapping + i18n strings. Safe to ship
   first — `next_cursor` absence keeps it a single fetch on
   the old backend.
3. Verify on prod: SWORD search for a common word completes;
   toggling the flag in admin returns `search_disabled`.
