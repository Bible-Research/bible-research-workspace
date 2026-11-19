# Design

## Context

The 66-book Protestant canon already exists as a single source
of truth: `_BIBLE_BOOKS` in `bible/utils/bible_books.py`
(backend) and `BIBLE_BOOKS` in `reactive-bible`'s
`bibleUtils.ts`. Every *book-name-driven* path resolves through
`get_dbt_book_id`, so passages, timestamps, notes, and reading
positions cannot reach apocryphal content today. SWORD filters
module books to the same list; ESV has no apocrypha upstream.

What remains unfiltered is *provider-derived* data: DBT search
returns `book_id` verbatim (deuterocanonical filesets exist in
the ~500-translation catalog), the `books=` search param is only
charset-validated, `get_passage_body` trusts a raw `book` id
from direct callers, and `generate_audio_timestamps._worklist`
iterates `get_fileset_files` coverage unfiltered.

Constraints: search is the most expensive anonymous endpoint
(tighter throttle scope); any canon check must add zero upstream
calls; payload shapes and `error`/`error_code` vocabulary are a
frozen contract the frontend branches on.

## Goals / Non-Goals

**Goals:**

- Search responses never carry non-canonical `book_id`s, with
  correct provider-reported pagination totals.
- `books=` intersected with canon; empty intersection
  short-circuits without an upstream call.
- `get_passage_body` treats non-canonical `book` as
  `book_not_in_fileset`.
- STT worklist skips non-canonical chapters.
- Zero added latency: set membership checks only, no extra
  upstream requests, no per-request canon computation.

**Non-Goals:**

- Per-fileset or multi-canon support (e.g. Catholic 73-book
  canon) — the frontend is hardcoded Protestant; the constant
  can later become a per-fileset lookup without changing the
  contract.
- Frontend changes — its book lists/refs are already canonical
  and it degrades gracefully on unknown `book_id`s.
- The `bible-structure` endpoint — its spec already owns
  book-id filtering.
- Tightening `is_valid_book_id_list` to canon membership — the
  param stays charset-validated; canon is applied at the
  intersection layer instead.

## Decisions

### 1. One exported canon set in `bible_books.py`

Add `CANONICAL_BOOK_IDS = frozenset(code for _, code, _ in
_BIBLE_BOOKS)` next to the existing derived maps. Every
enforcement point reads this constant — no duplicated lists,
no recomputation per request.

*Alternative:* a settings key or DB table for configurability —
rejected (YAGNI; a static constant is the current product
decision and the cheapest possible lookup).

### 2. Push the canon upstream on DBT search; post-filter everywhere

`DbtProvider.search`/`search_grouped` always send `books=` to
DBT — the caller's `books` intersected with canon, or the full
66-id list when no filter was given. This keeps
`meta.pagination.total`/`total_pages` honest: post-filtering a
page would drop rows while the reported total still counted
apocryphal hits, producing phantom pages and short pages.

A post-normalization canon filter still runs on every provider's
search rows (`normalize_dbt_verse` /
`normalize_apibible_verse` outputs) as belt-and-braces — it
covers upstreams that ignore `books` and providers with no
upstream filter at all.

*Cost:* `books` is a ~270-char query param, and filtering by
book can only reduce DBT's scan scope — this replaces, not
adds to, upstream work. The post-filter is one frozenset
membership test per row.

*Alternative:* post-filter only — rejected, totals lie.
*Alternative:* detect per-fileset canon before searching —
rejected, requires an extra upstream call per fileset.

### 3. `books=` intersection + short-circuit at the view boundary

`BibleSearchView` intersects the `books` param with
`CANONICAL_BOOK_IDS` after `is_valid_book_id_list`. Tokens are
`strip().upper()`-normalized first — the charset validator
accepts lowercase (`books=jhn` reaches DBT today) and
`CANONICAL_BOOK_IDS` is uppercase; the surviving ids serialize
back as a deterministic sorted comma-separated string (the
param travels as a string, not a list, into
`client.search(books=...)` and `scan_sword_matches`).

An empty intersection returns the provider-shaped empty
payload with no provider dispatch — `{groups: [], meta:
{total: 0, truncated: false}}` in grouped mode, and
`{verses: [], meta: {pagination: {total: 0, count: 0,
per_page: <limit>, current_page: <page>, total_pages: 0},
truncated: false}}` in paginated mode. The paginated
short-circuit runs *after* limit/page validation so
`page=abc` still 400s and the echoed `per_page`/`current_page`
are real values; `truncated: false` keeps the payload
identical to SWORD's paginated shape.

This is uniform for all providers — SWORD's local
`scan_sword_matches` filter and ESV's name-resolution drop
get the same semantics for free — and keeps "searches a book
the canon lacks" reading like a coverage miss: an empty
result, not a 400.

*Alternative:* 400 `invalid_parameter` for non-canonical ids —
rejected; `TOB` is a well-formed USFM id the app simply doesn't
cover, matching `book_not_in_fileset` semantics.

### 4. Canon guard in `get_passage_body`

Before provider dispatch, `book.upper() ∉ CANONICAL_BOOK_IDS`
returns `_book_not_in_fileset_body(...)` — the existing 404
vocabulary. Unreachable via the public `passage=` param today
(names resolve through `get_dbt_book_id`), so this is pure
defense in depth for direct callers (tests, internal code,
future endpoints). The uppercasing matches the lookup
conventions in `get_book_name_from_id`/`scan_sword_matches`.

### 5. Worklist filter in `generate_audio_timestamps`

`_worklist` drops rows whose `book_id` is non-canonical —
prevents wasted MP3 downloads + STT spend and the
3-consecutive-failures abort that apocryphal chapters would
trigger (the SWORD reference has no such books, so each would
fail alignment). Side effect worth documenting: `--book TOB`
on the command now yields an empty pending list, which is
correct — the book is outside the served canon.

## Risks / Trade-offs

- **DBT `books=` semantics unverifiable offline** → post-filter
  still drops strays, so the worst case is inflated totals, not
  leaked content; unit tests assert the param is sent.
- **API.Bible has no upstream book filter** → post-filter only;
  provider-reported `total` could exceed returned rows on a
  hypothetical deuterocanon bible — accepted, registered
  bibles are NIV-only today; documented in code.
- **`books=` always sent changes every DBT search request** →
  mitigated by deterministic sorted serialization and tests;
  no client-visible change on canonical-only filesets.
- **Direct `get_passage_body` callers passing `TOB` now 404** →
  intended contract; existing tests pass canonical ids.
