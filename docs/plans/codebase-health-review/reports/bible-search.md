# GET /api/v1/bible/search/ (BibleSearchView) — code review

## Summary

The endpoint correctly routes four providers (ESV, API.Bible, SWORD,
DBT), honors the `error`/`error_code` contract without leaking
exception text, and has strong test coverage — except the API.Bible
branch, which is entirely untested. The main problems are a
whitespace-only `query` that scans and returns the entire SWORD
Bible, a full-Bible local scan on every SWORD request, no upstream
timeout on DBT calls, and no throttling on an endpoint whose grouped
mode fans out to ~30 upstream calls per request. Verdict: **fair**.

## Endpoint contract

- Route: `bible/urls.py:21-25` → `BibleSearchView.get`
  (`bible/views.py:641-725`). `permission_classes = [AllowAny]`
  (views.py:639); `/api/v1/bible` is a `PUBLIC_READ_PREFIXES`
  member (middleware.py:16-20), so requests stay anonymous.
- Params:
  - `query` — required, non-empty (views.py:645-649). No length
    cap, no `.strip()`; forwarded verbatim to upstream `q`/`query`
    params or used as a lowercase substring needle for SWORD.
  - `fileset_id` — required (views.py:650-654). Routing order:
    `is_esv_fileset` → `is_apibible_fileset` → `is_sword_fileset`
    → else DBT (views.py:707-720). Not validated; unknown ids hit
    DBT and surface whatever upstream returns. Audio fileset ids
    (`LVSGLU8C*DA`, `ENGNIVC1DA`) canonicalize to their text
    module — a reasonable default, undocumented.
  - `limit` (default 15), `page` (default 1) — parsed with `int()`,
    must be >= 1, else 400 (views.py:675-705). No upper bound.
    Ignored entirely when `group_by=book` (views.py:656-673).
  - `sort_by` — forwarded to DBT only (views.py:831-835).
  - `books` — CSV USFM ids; filters SWORD scan and DBT `books`
    param; ignored by ESV/API.Bible branches.
  - `group_by` — only `'book'` accepted, else 400
    (views.py:659-664).
- Success (paged): `200 {"data": {"verses": [{book_id, chapter,
  verse_start, verse_text}], "meta": {"pagination": {total, count,
  per_page, current_page, total_pages}}}}`. DBT may emit
  `"meta": {}` when upstream pagination is absent
  (views.py:847-865).
- Success (grouped): `200 {"data": {"groups": [{book_id, count,
  verses}], "meta": {"total", "truncated"}}}` (views.py:788-825).
- Errors: 400 `{"error": msg}` (no `error_code`); provider failures
  via `provider_error_response` → `rate_limited` 429 / `not_found`
  404 / `provider_error` 502 (views.py:139-153, 721-725).

## Findings

### Bugs / correctness

1. **Whitespace-only `query` returns the whole SWORD Bible** —
   `if not query` (views.py:645) passes `' '`; `scan_sword_matches`
   lowercases it and `' ' in text` is true for nearly every verse
   (search_grouping.py:309-326). One request builds ~31k verse
   dicts and returns them all — a correctness bug and a resource
   issue in one. Same param reaches ESV/DBT/API.Bible upstream.
2. **`limit` has no upper bound** — `limit=999999` is forwarded to
   DBT/API.Bible as-is and, for ESV, produces inconsistent
   pagination: the client silently clamps `page-size` to 100
   (esv/client.py:169) while the view reports `per_page: 999999`
   and echoes upstream `total_pages` computed on size 100
   (views.py:738-744). Metadata no longer describes the page.
3. **All hand-rolled 400 bodies lack `error_code`**
   (views.py:646-705) — same contract gap flagged on the
   timestamps endpoint: the frontend branches on `error_code` and
   gets `undefined` for validation failures.
4. **SWORD grouped search has no result cap** — DBT/ESV/API.Bible
   grouped fetches honor `MAX_RESULTS`/`MAX_ESV_PAGES`
   (search_grouping.py:20-23); the SWORD branch hard-codes
   `truncated=False` and returns every match (views.py:807-811).
   Common one-letter queries produce multi-MB responses.
5. **Cross-page ordering not stabilized for grouped DBT fetch** —
   `fetch_dbt_matches` pages 2..N without forwarding `sort_by` or
   any explicit order (search_grouping.py:197-211). If DBT's
   default ordering is unstable, rows can be silently missed
   (dedup in `group_verses_by_book` hides duplicates only).
6. **Silent chapter drop in SWORD scan** — `except Exception:
   continue` (search_grouping.py:315-317) swallows any
   `get_chapter_verses` failure, not just `PassageNotFoundError`;
   a corrupt module degrades results with no log.
7. **Minor**: `data.meta == {}` when DBT omits pagination
   (views.py:852-865) breaks the documented `meta.pagination`
   shape. `normalize_dbt_verse` passes `chapter`/`verse_start`
   through uncoerced (search_grouping.py:66-73); a mixed
   int/string batch would crash `group_verses_by_book`'s sort with
   `TypeError` → 502. `int(chapter)`-style dead guards
   (`if limit else 1`, views.py:771, 884) survive though limit is
   already validated >= 1.

### Security & auth

- **No throttling on an amplification-prone endpoint.** No DRF
  throttle classes exist anywhere (`settings.py:368-382`), and
  this endpoint is anonymous by design. One grouped ESV request
  fans out to up to 30 upstream calls (search_grouping.py:271-286)
  and any SWORD request is a full-Bible scan — a cheap way to burn
  provider quota and worker CPU.
- No injection surface: `query`/`books`/`fileset_id` only travel
  as URL-encoded GET params or feed a local substring scan — no
  SQL, no shell. No query-length cap means multi-KB queries go into
  upstream URLs (414/400 → `provider_error` 502); minor.
- Positive: `provider_error_response` never embeds `str(exc)`;
  credential-leak regression tests exist
  (test_search_view.py:844-909).
- ESV `verse_text` carries upstream HTML (`content`), while
  SWORD/DBT/API.Bible return plain text — if the frontend renders
  it raw, it is third-party markup; worth pinning semantics.

### Performance

- **SWORD search scans the entire module per request.** Both
  `_sword_search` (paged) and grouped mode call
  `scan_sword_matches` (views.py:876, 807), which walks every
  chapter via per-verse `bible.get` calls (sword/client.py:128-143)
  — ~1,189 chapters for LVSGLU8. No early exit once `page*limit`
  matches are found, no caching between requests. Likely the
  slowest path in the API.
- **DBT `session.get` has no timeout** (dbt/client.py:368-370;
  same gap as `get_timestamps`, flagged in the sibling report).
  Grouped mode compounds it inside a multi-page loop
  (search_grouping.py:197-211).
- Good: shared provider clients reuse warm keep-alive sessions
  (dbt/client.py:384-397, esv/client.py:298-304); ESV grouped
  fan-out is bounded and parallel (search_grouping.py:271-288);
  grouped DBT fetch requests the effective upstream page size so
  rows are not skipped (search_grouping.py:188-207).

### Code quality & consistency

- Provider dispatch (`is_esv_fileset`/`is_apibible_fileset`/
  `is_sword_fileset`/else DBT) is reimplemented inline in every
  view (views.py:707-720, 796-816); `translation_service.py` owns
  routing for passages but has no search hook — a small shared
  `resolve_provider(fileset_id)` would unify five copies.
- Search logic is well factored into `search_grouping.py` (pure
  helpers, docstrings, safety caps) and `views.py` keeps thin
  per-provider responders — good separation overall.
- `@extend_schema` documents all seven params (views.py:536-606);
  docstring matches behavior, including the grouped-mode
  pagination caveat (views.py:656-658).
- Pagination semantics differ per provider (ESV echoes upstream
  page/total_pages; API.Bible/SWORD compute; DBT passes through
  possibly-empty meta) — acceptable but undocumented divergence.
- `format=None` unused arg and duplicated limit/page parsing are
  house-consistent but could be one `parse_pagination()` helper.

### Test coverage

Covered (test_search_view.py, ~35 tests): missing params → 400,
bad `group_by` → 400, DBT paging passthrough, SWORD substring
filtering + `books` scoping, ESV result parsing (ranges,
numbered books, malformed refs skipped), grouped mode for all of
DBT/ESV/SWORD (canonical order, dedup, `truncated`, books filter,
page-size follow-up), provider errors → 502 with no
`str(exc)`/`?key=` leak, upstream 429 → `rate_limited`, grouped
mode ignoring `page=abc`, anonymous access → 400
(test_public_read_permissions.py:50-52).

Missing: **API.Bible branch entirely** (no `ENGNIV_API` test —
`_apibible_search` and `fetch_apibible_matches` are uncovered);
`limit`/`page` non-integer → 400 and `<1` → 400; whitespace-only
`query`; `page` beyond `total_pages`; unknown `fileset_id`
falling to DBT; SWORD paged slicing (`page=2`); DBT response with
no pagination (`meta == {}`); `limit` over provider max.

## Refactoring recommendations

1. **P0 — Strip and bound `query`** (views.py:642-649):
   `query = (query or '').strip()` → 400 when empty; cap at ~200
   chars. Why: `' '` currently returns every verse in the SWORD
   module per request; trivial guard closes it. Effort: S.
2. **P0 — Add a `timeout` to DBT `session.get` calls**
   (dbt/client.py:368-370 and siblings; match the 10s used by
   ESV/API.Bible). Why: a stalled socket pins a Gunicorn/App
   Engine worker forever. Effort: S.
3. **P1 — Cap `limit`** (views.py:675-705): reject `limit > 100`
   with 400, or clamp and report the clamped `per_page`. Why:
   fixes the ESV `per_page` lie (finding 2) and absurd upstream
   page sizes. Effort: S.
4. **P1 — Throttle the search view**: `AnonRateThrottle`/`UserRate-
   Throttle` (or a scoped throttle) on `BibleSearchView`. Why: one
   anonymous request can cost 30 upstream calls or a full local
   Bible scan. Effort: M (needs DRF throttle config + rates).
5. **P1 — Bound the SWORD scan**: for paged mode, stop once
   `page*limit` matches are collected; for grouped mode, honor
   `MAX_RESULTS` and set `truncated`. Optionally cache the
   verse list per fileset (it is immutable) so repeat searches
   skip `bible.get` per verse. Why: removes the endpoint's worst
   latency path. Effort: M.
6. **P2 — Emit `error_code` on the 400 validation responses**
   (views.py:646-705), e.g. `"error_code": "invalid_param"`, via a
   shared `bad_request(msg)` helper. Why: contract consistency;
   frontend branches on `error_code`. Effort: S.
7. **P2 — Narrow the SWORD scan catch**
   (search_grouping.py:315-317): catch `PassageNotFoundError` only;
   log other exceptions with book/chapter context. Effort: S.
8. **P2 — Stabilize grouped DBT paging**: forward `sort_by` (or a
   deterministic order) in `fetch_dbt_matches` page 2+ calls and
   in `client.search` (search_grouping.py:197-211). Effort: S.
9. **P2 — Normalize response shape**: coerce `chapter`/
   `verse_start` to `int` in `normalize_dbt_verse`, and always
   emit `meta.pagination` (possibly nulls) for DBT instead of
   `meta: {}` (views.py:852-865, search_grouping.py:66-73).
   Effort: S.
10. **P2 — Fill test gaps**: API.Bible paged + grouped branches,
    limit/page validation 400s, whitespace query, out-of-range
    page, unknown `fileset_id`. Effort: M.
11. **P2 — Extract provider dispatch** (`resolve_provider` /
    per-provider search table) shared by `BibleSearchView` and the
    passage/timestamps views to kill the repeated is_esv/apibible/
    sword/else-DBT chains. Effort: M.
