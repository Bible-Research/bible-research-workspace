# GET /api/v1/bible/translations/ (TranslationListView) — code review

## Summary

The endpoint works and correctly merges DBT results with the
SWORD/ESV/API.Bible registries, including the new
`text_options`/`audio_options` groups — the fileset-groups feature
is fully merged on this branch (`bible/services/fileset_groups.py`,
wired at `bible/services/translation_service.py:8,88-91`; commit
`bd39f28` is on `main`; the workspace AGENTS.md note calling it
unmerged is stale). However, every request makes a live,
uncached, un-throttled, un-timed-out call to b4.dbt.io, and DBT
failures are silently swallowed into a partial 200 response that
contradicts the documented error contract.

Verdict: **fair**

## Endpoint contract

- Route: `bible/urls.py:6-10` →
  `GET /api/v1/bible/translations/`
- Params: `language_iso` (optional, e.g. `eng`, `lvs`)
- Success: `200 {"results": [{abbr, name, language,
  language_iso, filesets[], text_options[], audio_options[]}]}`
  (`bible/views.py:933-948`)
- Errors: `{"error", "error_code"}` — `rate_limited`→429,
  `not_found`/`book_not_in_fileset`→404, else
  `provider_error`→502 (`bible/views.py:49-53,139-153`,
  `bible/utils/provider_errors.py`)
- Auth: `AllowAny` (`bible/views.py:915`); read-only, so the
  `guest` fallback is harmless.

## Findings

### Bugs / correctness

1. **DBT failures are silently swallowed** —
   `translation_service.py:52-55` catches all exceptions from
   `client.get_bibles`/`_process_translations`, logs, and
   continues with `processed = []`. A DBT outage returns HTTP 200
   with only the three registry entries; clients cannot
   distinguish "language has no translations" from "DBT is down".
   This contradicts `DEVELOPER_GUIDE.md:369-374`, which states
   the endpoint surfaces upstream failures via
   `error`/`error_code`, and the view's own 502 path
   (`views.py:950-955`) is unreachable for DBT errors — the main
   provider. If graceful degradation is intended, it is
   undocumented and untested; the response should carry a flag
   (e.g. `partial: true`) or the failure should propagate.
2. **`limit=500` silently truncates** — `translation_service.py:36`
   sends `limit=500` and ignores DBT's `meta.pagination`; an
   unfiltered call returns at most the first 500 of ~2,500 DBT
   bibles with no indication of truncation.
3. **Duplicate `abbr` rows possible** — registry entries are
   appended unconditionally (`translation_service.py:80-83`):
   for `language_iso=eng` the response can contain DBT's own ESV
   record plus the `ENGESV_API` registry entry (both `abbr:ESV`),
   and likewise NIV. No dedupe; a frontend keying on `abbr` sees
   two rows.
4. **Unconditional key access masks data bugs as provider
   errors** — `t['abbr']`, `t['name']`, `t['language']`,
   `t['iso']`, `t['filesets']` (`views.py:933-944`). Any DBT
   record missing a key raises `KeyError` → 502
   `provider_error`, a server-side shape bug reported to clients
   as an upstream failure.
5. **Client construction outside the failure-swallowing `try`** —
   `get_default_dbt_client()` at `translation_service.py:35`
   raises (missing `DBT_KEY`) before the `try`, so a config
   problem 502s the whole endpoint instead of degrading to
   registry entries the way a downstream DBT failure does —
   inconsistent.

### Security & auth

6. **`AllowAny` + no throttling + live upstream call** —
   `views.py:915`, no `DEFAULT_THROTTLE_CLASSES` in
   `settings.py:368-382`. Every anonymous request costs one DBT
   API call against the shared `DBT_KEY` quota — a trivial
   quota-exhaustion/DoS vector, amplified by finding 8 (no
   cache).
7. **`language_iso` is unvalidated** — any string is uppercased
   and forwarded as DBT `language_code`
   (`translation_service.py:37-38`) and lowercased for registry
   filtering. Low risk (query-param passthrough, not injection),
   but inconsistent with the strict param validation in the
   neighbouring views (`views.py:371-379,696-705`).

### Performance

8. **No caching at all** — this is the most cache-friendly
   endpoint in the API (deterministic, anonymous,
   rarely-changing data), yet there is no `CACHES` config, no
   `cache_page`, and no `Cache-Control`/`ETag` headers. Each
   request blocks on a live HTTPS call to b4.dbt.io (~300-900 ms
   per the client docstring, `dbt/client.py:62-64,384-391`).
9. **No upstream timeout** — `self.session.get(...)` in
   `get_bibles` (`dbt/client.py:114-116`; same in
   `get_timestamps:278`, `get_copyright:311`, `search:368`)
   passes no `timeout`, unlike the ESV/API.Bible clients which
   use `timeout=10` (`esv/client.py:56,96,180`,
   `apibible/client.py:73,136,186,195`). A stalled upstream
   connection parks a GAE worker indefinitely.
10. **Debug log dumps the full payload** —
    `translation_service.py:50` logs every raw translation (up to
    500 records); noisy and expensive at DEBUG level.

### Code quality & consistency

11. **Triplicated iso-filter blocks** — `translation_service.py:
    57-79` repeats the same list-comprehension filter three
    times for SWORD/ESV/API.Bible; a single loop over provider
    listings removes ~20 lines.
12. **Response envelope inconsistent** — returns `{"results":[]}`
    (`views.py:948`) while every sibling bible endpoint uses
    `{"data":...}` (`views.py:422,467,528,746`). Documented, so
    changing it is a contract break — noted for consistency only.
13. **Docstring example URLs wrong** — `views.py:911-912` shows
    `/api/v1/translations/`; the real path is
    `/api/v1/bible/translations/`.
14. **No `@extend_schema`** — unlike `BibleSearchView`
    (`views.py:536-606`), `language_iso` never reaches the Swagger
    docs for this view.
15. **Service bypasses the client's `language` param** —
    `get_bibles(language=None, **kwargs)` exists
    (`dbt/client.py:95`), but the service stuffs `language_code`
    into `**kwargs` instead; works because kwargs are copied
    straight into query params (`dbt/client.py:106-111`), but is
    fragile and unclear.

### Test coverage

Present and good for happy paths: response option fields
(`test_book_not_in_fileset.py:281-313`), service grouping of DBT
filesets (`:316-381`), codec/bitrate passthrough (`:383-424`),
anonymous access (`test_public_read_permissions.py:54-62`),
SWORD/ESV/API.Bible inclusion (`test_sword.py:117-125`,
`test_esv_client.py:279-300`, `test_apibible_client.py:704-724`),
plus 269 lines of `group_filesets` unit tests
(`test_fileset_groups.py`).

Missing:

- No test pins the DBT-failure degradation behaviour (finding 1)
  — the most important contract decision here is untested.
- No test for the view's 502 path when the service raises.
- No test for malformed DBT entries / missing keys (finding 4).
- No test for invalid `language_iso` values or the 500-cap.

## Refactoring recommendations

1. **[P0] Cache the response** — key on `language_iso` (default
   locmem or `cache_page`, TTL ~1-24h) and send a
   `Cache-Control: public, max-age=…` header. Removes the
   per-request upstream call entirely; biggest perf + abuse win.
   Effort: S-M.
2. **[P0] Add `timeout=10` to all `session.get` calls in
   `dbt/client.py`** (lines 114, 278, 311, 368) — match the
   ESV/API.Bible clients; prevents worker-pool exhaustion on a
   hung upstream. Effort: S.
3. **[P0] Decide and pin the DBT-failure contract** — either
   propagate the error so the view returns 502 per the documented
   contract, or keep degradation and add a `partial`/`degraded`
   flag plus a regression test. Today the behaviour is accidental
   and invisible to clients. Effort: S-M.
4. **[P1] Validate `language_iso`** — `^[a-zA-Z]{2,3}$`, else
   400, consistent with sibling views' input validation. Effort:
   S.
5. **[P1] Dedupe merged entries** — when a registry entry's abbr
   matches a DBT record, merge their filesets (or drop the
   registry row) so `abbr` is unique per language. Effort: M.
6. **[P1] Defensive response mapping in the view** — use
   `t.get(...)` or a small serializer so a malformed DBT record
   skips/degrades instead of producing a misleading 502. Effort:
   S.
7. **[P1] Add an `AnonRateThrottle`** (or cache, which makes it
   mostly moot) for this `AllowAny` endpoint to protect the DBT
   quota. Effort: S.
8. **[P2] Collapse the three iso-filter blocks** in
   `translation_service.py:57-79` into a loop over
   `(listing_fn,)` pairs. Effort: S.
9. **[P2] Fix the docstring URLs** (`views.py:911-912`) and add
   `@extend_schema` documenting `language_iso`. Effort: S.
10. **[P2] Move `get_default_dbt_client()` inside the `try`** in
    `translation_service.py` so a missing `DBT_KEY` degrades the
    same way as a downstream DBT failure. Effort: S.
11. **[P2] Document or handle the `limit=500` cap** — either
    page through DBT `meta.pagination` or note the truncation
    bound in the service docstring. Effort: S.
