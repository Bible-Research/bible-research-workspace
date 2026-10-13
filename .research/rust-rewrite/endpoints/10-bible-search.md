# 10 — `GET /api/v1/bible/search/` (BibleSearchView)

Size: L. Sources: `bible/views.py:338-643` (view + OpenAPI
params), `bible/services/search_grouping.py` (whole file),
`bible/services/dbt/client.py:318-373`,
`bible/services/esv/client.py:130-173`,
`bible/services/sword/client.py:98-178`,
`bible/utils/{bible_books,provider_errors}.py`,
`bible/tests/test_search_view.py` (24 tests).

## 1. Endpoint contract

`GET /api/v1/bible/search/` (`bible/urls.py:21-25`). Auth:
`tok` — `Principal` extractor (prod auto-provisions; tests run
anonymous under AllowAny, see 00-base §4.2). No DB access.

### Query params

| param       | req | parse / default                                  |
|-------------|-----|--------------------------------------------------|
| `query`     | yes | any non-empty string; `''`/missing → 400         |
| `fileset_id`| yes | any non-empty string; routes provider            |
| `limit`     | no  | `int()`, default `15`. NO cap, NO floor          |
| `page`      | no  | `int()`, default `1`. NO floor                   |
| `sort_by`   | no  | raw string; forwarded to DBT only                |
| `books`     | no  | raw string; DBT verbatim, SWORD split/upper,     |
|             |     | ESV ignored                                      |
| `group_by`  | no  | only `'book'` accepted; anything else → 400;     |
|             |     | `''` treated as absent (paginated path)          |

Validation ORDER (views.py:441-503) — reproduce exactly:

1. `query` falsy → 400 `{"error": "query parameter is required."}`
2. `fileset_id` falsy → 400
   `{"error": "fileset_id parameter is required."}`
3. `group_by` truthy and `!= 'book'` → 400
   `{"error": "group_by must be 'book'."}`
4. `group_by == 'book'` → grouped path; `limit`/`page` are
   NEVER parsed (`page=abc` returns 200, not 400 —
   test_search_view.py:665).
5. Paginated path: `int(limit)` fail → 400
   `{"error": "limit must be an integer."}`; `int(page)` fail →
   400 `{"error": "page must be an integer."}`. `?limit=` (empty)
   400s — `int('')` raises.

Provider dispatch order (both paths): `is_esv_fileset` → ESV;
else `is_sword_fileset` → SWORD; else DBT catch-all
(`ENGESV` without `_API` is a DBT id). ESV/SWORD matching is
case-insensitive; SWORD also accepts abbr `GLU8` and the audio
id `LVSGLU8C1DA` — searching an AUDIO fileset scans the text
module (registry.py:28-45).

### Response — paginated (no `group_by`)

```json
{
  "data": {
    "verses": [
      {
        "book_id": "JHN",
        "chapter": 3,
        "verse_start": 16,
        "verse_text": "For God so loved the world"
      }
    ],
    "meta": {
      "pagination": {
        "total": 100, "count": 15, "per_page": 15,
        "current_page": 1, "total_pages": 7
      }
    }
  }
}
```

Verse fields are provider pass-through — any JSON type may
appear (`normalize_dbt_verse` does `.get()` verbatim, missing →
`null`). DBT: when upstream has no `meta.pagination`, emit
`"meta": {}` (empty object — NOT `{"pagination": null}`). DBT
pagination subfields are `.get()` values → missing keys
serialize as `null`. ESV pagination is synthesized (§2). SWORD
pagination is computed locally.

### Response — `group_by=book`

```json
{
  "data": {
    "groups": [
      {
        "book_id": "GEN",
        "count": 1,
        "verses": [
          {"book_id": "GEN", "chapter": 22,
           "verse_start": 2, "verse_text": "..."}
        ]
      }
    ],
    "meta": {"total": 3, "truncated": false}
  }
}
```

Groups in canonical book order; verses within a group sorted by
`(chapter, verse_start)`. `total` is the provider-reported (or
scan) count BEFORE dedupe; `truncated` = safety cap hit.

### Error bodies

- 400s: `{"error": "<msg>"}` — exact strings above.
- Provider failure (any exception inside the dispatch, incl.
  grouped fetch): `provider_error_fields` mapping — 429
  `rate_limited`, 404 `not_found` (`"Passage not found"` /
  `"Passage not found (HTTP 404)"`), else 502
  `provider_error` (`"Bible provider error (HTTP {s})"` or
  `"Bible provider request failed ({ErrorKind})"`). Never leak
  `str(exc)` — DBT URLs carry `?key=` (test:847).
- 401 DRF bodies per 00-base §4.4.

## 2. Django behavior trace

### Paginated dispatch (views.py:494-503)

`_esv_search` (510-536): `esv_client.search(query, page, limit)`
→ GET `api.esv.org/v3/passage/search/` params `q`, `page`,
`page-size=min(limit,100)`; header `Authorization: Token
{ESV_KEY}`; timeout 10 s (client.py:148-165). Results parsed by
`parse_esv_results` (search_grouping.py:26-63): split reference
on whitespace, last token must contain `:`; `chapter=int(..)`,
`verse_start=int(first of "a-b" range)`; book name via
`get_dbt_book_id` — full fuzzy-matching path (aliases, difflib
0.8 cutoff, roman numerals, ordinals). Unparseable/unknown
refs skipped silently. Pagination synthesized:
`total=total_results`, `count=len(verses)` AFTER filtering,
`per_page=requested limit` (not the capped page-size — limit=200
reports `per_page: 200` while returning ≤100),
`current_page=result.page or requested page`,
`total_pages=result.total_pages or 1`. `books`/`sort_by`
silently ignored.

`_dbt_search` (571-615): `dbt_client.search(fileset_id, query,
limit, page, sort_by, books)` → GET `b4.dbt.io/api/search`
params `v=4`, `key`, `query`, `fileset_id`, `limit`, `page`,
plus `sort_by`/`books` ONLY when not None (`''` IS forwarded).
Response verses from `result['verses']['data']` normalized
4-field; meta from `result['meta']` else
`result['verses']['meta']` (observed b4.dbt.io variance —
comment views.py:586-590); DBT `links` stripped. No pagination
→ `"meta": {}`.

`_sword_search` (617-643): full scan via `scan_sword_matches`
then LOCAL pagination: `start=(page-1)*limit`,
`page_items=matches[start:start+limit]` (Python slice
semantics — see Risks), `total_pages=ceil(total/limit)` or `1`
when `limit==0`.

### SWORD scan (search_grouping.py:219-255)

`client.list_chapters(fileset_id)` → `[(book_id, chapter)]` in
the module's OWN OT→NT versification order (sword/client.py:
143-178). `books` filter: split `,`, strip, `.upper()` — but
`if books:` means `books=''` disables filtering (asymmetry vs
DBT which forwards `''`). Per chapter `get_chapter_verses` —
ANY exception → skip chapter (line 244). Match =
`query.lower() in verse_text.lower()` — case-insensitive
SUBSTRING (Unicode lower). Result order = chapter order,
verse ascending. No ranking.

### Grouped fetch (views.py:538-569 + helpers)

- DBT `fetch_dbt_matches` (132-182): first call
  `limit=MAX_RESULTS(5000)` (no `page`, no `sort_by`; `books`
  forwarded). `total` = `pagination.total or len(items)`.
  Effective `per_page = min(reported per_page or len(items),
  len(items))`; follow-up pages request THAT size (server may
  cap below requested — views comment 151-154, test:688).
  Loop while `per_page and len(items)<5000 and len<total`;
  break on empty page; hard-truncate at 5000.
  `truncated = total > len(items)`.
- ESV `fetch_esv_matches` (185-216): page 1 at `page_size=100`;
  `total_pages` learned, then pages `2..min(total_pages,30)`
  fetched on a 4-worker `ThreadPoolExecutor` (`pool.map`
  preserves page order). `truncated = total_pages > 30`.
- SWORD `scan_sword_matches` again; `total=len(verses)`,
  `truncated=False` — dedupe happens later so `total` can
  exceed the sum of group counts.

### `group_verses_by_book` (87-129) — exact algorithm

1. Iterate input order; drop verse if `book_id` is not a
   non-empty `str`.
2. Dedupe key `(book_id, chapter, verse_start)` — keep FIRST
   occurrence (range-normalized dupes like 3:16-18 → 16).
3. Bucket into `by_book` preserving first-sight order.
4. Sort each bucket by `(chapter or 0, verse_start or 0)`
   — falsy → 0; mixed str/int values would raise TypeError →
   outer catch → 502 (edge, see Risks).
5. Emit `{book_id, count, verses}`; sort groups by
   `_book_order`: `get_book_name_from_id(book_id)` →
   `BOOK_ORDER_MAP[title_name]` (1-66); any failure or unknown
   → `len+1 = 67`. Python `sort` is stable — equal-rank groups
   (multiple unknowns) keep insertion order.

### Query preprocessing

NONE for DBT/ESV (verbatim upstream). SWORD: `query.lower()`
substring. No trimming anywhere — `query=' '` is valid input.

## 3. Rust implementation plan

Module: `crates/server/src/routes/bible/search.rs` +
`crates/providers/src/grouping.rs` (per 00-base §3).

### Handler

```text
async fn bible_search(
    State(s): State<AppState>,
    p: Principal,                       // tok
    Query(q): Query<SearchParams>,      // ALL Option<String>
) -> ApiResult<Json<Value>>
```

`SearchParams` fields all `Option<String>` — parse `limit`/
`page` manually (in validation order!) so the 400 bodies and
the "grouped path never parses page" quirk are reproduced.
`str::trim().parse::<i64>()` approximates `int()`; deviations
in Risks.

Provider resolution: `s.providers.for_fileset(&fileset_id)`
→ `ProviderKind::{Esv, Sword, Dbt}` per 00-base §4.7 routing
order.

### DTOs

```rust
pub struct SearchVerse {            // serde pass-through
    pub book_id: serde_json::Value, // DBT fields are verbatim;
    pub chapter: Value,             // wrong types/nulls must
    pub verse_start: Value,         // round-trip unchanged
    pub verse_text: Value,
}
pub struct SearchPagination {       // all Option<i64> — emit
    pub total: Option<i64>,         // null, never skip
    pub count: Option<i64>,
    pub per_page: Option<i64>,
    pub current_page: Option<i64>,
    pub total_pages: Option<i64>,
}
pub struct SearchMeta {             // {"pagination": {...}}
    #[serde(skip_serializing_if = "Option::is_none")]
    pub pagination: Option<SearchPagination>, // or {} empty
}
pub struct BookGroup {
    pub book_id: String,
    pub count: usize,
    pub verses: Vec<SearchVerse>,
}
pub struct GroupedMeta { pub total: i64, pub truncated: bool }
```

### Provider trait surface (refines 00-base §4.7)

```rust
async fn search(&self, q: &SearchQuery)
    -> Result<SearchPage, ProviderError>;
async fn search_all(&self, q: &SearchQuery)
    -> Result<AllMatches, ProviderError>;

pub struct SearchQuery {            // validated values only
    pub query: String, pub fileset_id: String,
    pub limit: i64, pub page: i64,  // unused in search_all
    pub sort_by: Option<String>, pub books: Option<String>,
}
pub struct SearchPage { verses: Vec<SearchVerse>,
                        meta: SearchMeta }
pub struct AllMatches { verses: Vec<SearchVerse>,
                        total: i64, truncated: bool }
```

Handler applies `grouping::group_by_book(verses)` to
`AllMatches` — grouping is provider-agnostic, so it must NOT
be duplicated per provider (adjust the §4.7 sketch
accordingly; it returned `SearchAll`).

- `providers::dbt`: reqwest GET `{base}/search`, query params
  as in client.py:347-361 (omit `None`s, forward `''`). Parse
  `verses.data[]` + meta fallback chain; keep `Value`s.
- `providers::esv`: GET `…/v3/passage/search/` with
  `page-size=min(limit,100)`; `parse_esv_results` port uses
  `core::books::get_dbt_book_id` (difflib crate — same
  `SequenceMatcher` semantics per 00-base §2).
- `providers::sword`: `list_chapters` + `chapter_verses`
  (already required for passage plan + audio_job); scan is the
  same code path for both `search` (then local slice) and
  `search_all`.
- `fetch_dbt_matches` port: sequential loop (each iteration
  needs prior `per_page`); `fetch_esv_matches` port: page 1
  then `futures::stream::iter(2..=last).map(..).buffered(4)`
  — preserves page order like `pool.map`.

### Grouping port notes

- `book_order`: `BOOK_ORDER_MAP` as a `static`/`OnceLock`
  `HashMap<&str, u8>` over the title-cased names in
  `core::books`; unknown → 67.
- Sort keys: extract `as_i64().unwrap_or(0)` for
  chapter/verse_start (see Risks for the str/int deviation).
- Dedupe on `(book_id, chapter, verse_start)` `Value` tuples —
  `HashSet` on the JSON triple; keep first occurrence.
- Stable `sort_by_key` on groups (Rust sort is stable —
  matches Python).

### SWORD search feasibility

pysword has NO search API — Django brute-forces
`list_chapters × get_chapter_verses`. Rust options:

1. Same shape over whichever reader the passage plan picks:
   `rsword_chirho` 0.3 (verify it exposes book/chapter/verse
   iteration — it loads from an extracted module dir) or the
   ~400-line ztext port (00-base §4.10). Per-verse `get()` is
   fine in-process (no IPC).
2. BETTER in the ztext port: decompress each `.bzz` block
   ONCE and emit all its verses — a full-module scan is one
   sequential pass over the zip (LvGluck8 ≈ 1,189 chapters,
   ~31k verses, a few MB of text).
3. Cache the flattened corpus per fileset:
   `OnceCell<Vec<(book_id, chapter, verse, String)>>` inside
   the `Arc<SwordProvider>` — first search builds it
   (~10-100 ms expected), later searches are pure in-memory
   substring scans. Django re-reads the zip every request;
   caching is a deliberate perf improvement, same results.
4. Scan is sync CPU work → `tokio::task::spawn_blocking`
   (or `block_in_place`) so it never stalls the runtime; a
   request holds no lock while scanning the shared corpus.
5. Do NOT build an inverted index — substring semantics +
   tiny corpus make it pointless; revisit only if measured.

Matching: `text.to_lowercase().contains(&needle_lower)` —
both Python `str.lower` and Rust `to_lowercase` implement
Unicode default lowercase; add a parity test with Latvian
diacritics (ā č ē ģ ī ķ ļ ņ ŗ š ū ž).

## 4. Test plan + parity

Django tests to mirror (`bible/tests/test_search_view.py`,
classes/fns → `tests/bible_search.rs`):

- 400s: missing `query`, missing `fileset_id`, bad
  `group_by`, `limit=abc`, `page=abc` (non-grouped),
  `limit=''` → 400.
- DBT: forwards `limit/page/sort_by/books`; meta passthrough;
  meta absent → `"meta": {}`; meta under `verses.meta`
  fallback.
- ESV: ref parsing — plain `John 3:16`, range `Gen 1:1-2` →
  verse_start 1, numbered book `1 John 4:8` → `1JN`,
  malformed `Invalid Reference` skipped; pagination fields
  incl. `per_page` = requested limit.
- SWORD: substring filter, `books` limits scanned chapters,
  local pagination math, `limit=0` → `total_pages=1`,
  empty page.
- Grouped: canonical order (GEN before JHN), verse sort,
  dedupe first-wins, missing/non-str `book_id` dropped,
  unknown book_id → trailing group; `meta` {total,truncated};
  `page=abc` ignored; `books` forwarded (DBT) / applied
  (SWORD).
- Fetch loops: DBT effective `per_page` follow-up
  (test:688-756); ESV 30-page cap → `truncated: true`,
  exactly 30 calls; DBT empty page 2 → truncated by `total`.
- Errors: provider raise → 502 `provider_error`, body never
  contains exception text/`?key=`; upstream 429 → 429
  `rate_limited`; upstream 404 → 404 `not_found`.
- Test doubles: `wiremock` for DBT/ESV; `FakeProvider` impl or
  fixture module for SWORD scan; `AUTO_PROVISION_USERS=false`.
- Parity corpus (00-base §4.15): add entries —
  `?query=love&fileset_id=ENGESV`,
  `&fileset_id=ENGESV_API`, `&fileset_id=LVSGLU8`,
  `&group_by=book` for all three, `?query=love&limit=abc`
  (400), `&group_by=chapter` (400), `&books=JHN,ROM`,
  `&page=2&limit=5`. Normalize only volatile fields.

## 5. Open questions / risks

- **Python slice semantics** in `_sword_search`: negative
  `page`/`limit` wrap (`page=-1&limit=15` → `matches[-30:-15]`).
  Options: (a) ~15-line `py_slice` helper for exact parity —
  cheap, recommended; (b) clamp and accept deviation.
  `page=0` → `start=-limit` → empty page either way.
- **`int()` fidelity**: Python accepts surrounding whitespace,
  `+`/`-`, and even unicode digits / `1_0`. `trim()+parse()`
  covers whitespace/sign; underscores/unicode digits → accept
  deviation (would 400 where Django accepts).
- **Non-numeric `chapter`/`verse_start`** from DBT: Django
  echoes verbatim AND `sort` raises TypeError → 502 on mixed
  types. Rust port uses `as_i64().unwrap_or(0)` — emits the
  same values but sorts instead of 502ing. Accepted deviation
  (DBT returns ints in practice); flagged.
- **`total` vs dedupe**: grouped `meta.total` counts
  pre-dedupe matches; `sum(group.count)` can be smaller.
  Keep verbatim.
- **SWORD audio-id search**: `LVSGLU8C1DA` scans the text
  module — surprising but contractual (registry canonicalizes
  audio ids). Keep.
- **Empty-string asymmetries**: `books=''` → DBT receives
  `books=`, SWORD ignores the filter; `sort_by=''` forwarded;
  `group_by=''` → paginated. Reproduce.
- **ESV `count`** = post-filter verse count, can be < upstream
  `count` — reproduce.
- **rsword_chirho** iteration surface unverified — if it
  can't enumerate books/chapters by the module's own
  versification, the ztext port (option 2/3) is REQUIRED for
  both this endpoint and the audio job. Decide with the
  passage plan (item 1), not twice.
- **ESV parallel fetch**: `buffered(4)` preserves order; a
  failing page aborts the whole grouped request →
  `provider_error` (same as Django — `pool.map` propagates).
- **Timeouts**: ESV client uses 10 s; DBT session has none.
  Suggest reqwest `timeout(10s)` for ESV parity; leave DBT
  unbounded or pick a generous default — decision for
  `providers/dbt.rs` owner; document whatever is chosen.
- **Full-scan latency**: Django's per-request SWORD scan is
  O(module); Rust corpus cache makes later searches fast but
  the FIRST search still decompresses everything — consider
  warming it at startup behind a flag.
