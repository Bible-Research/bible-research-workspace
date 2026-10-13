# 08 — `GET /api/v1/bible/` (BiblePassageView)

Status: PLAN. Builds on `00-base.md` (§4.4 error contract, §4.7
provider trait, §4.8 registries, §4.9 GCS, §4.10 SWORD). Sources:
`bible/views.py:60-217` (view), `bible/serializers.py:24-202`
(provider dispatch + body building), `bible/utils/bible_books.py`,
`bible/utils/provider_errors.py`, `bible/services/{dbt,esv,sword,
google_tts}`, `bible/services/storage/gcs.py`, `bible/tests/`.

**This is the most provider-heavy endpoint: one route fans out to
four upstreams (DBT, ESV, SWORD module, GCS audio artifacts) with
per-provider response shapes and a two-layer error contract.**

## 0. In-flight branch warning (read first — affects contract)

Two unmerged branches change THIS endpoint's contract; main is the
baseline below, but the Rust code should be built so either contract
can ship.

### `feat/apibible-niv` (unmerged)

Adds a fourth provider, API.Bible (NIV), routed inside
`BiblePassageSerializer.to_representation` between the SWORD and
DBT arms:

- Registry `bible/services/apibible/registry.py` (exists only on
  the branch): `ENGNIV_API` (text) and `ENGNIVC1DA` (audio id)
  canonicalize to `ENGNIV_API`; `is_apibible_fileset`,
  `get_apibible_meta` → `{bible_id: "78a9f6124f344018-01",
  audio_bible_id: "5dd1a42b02d2dee6-01", ...}`.
- Client `apibible/client.py`: `api-key` header auth to
  `https://api.scripture.api.bible/v1`; `get_chapter` →
  `{verses, headings, fums}`; `get_chapter_audio` → `{audio_url
  (presigned resourceUrl), timecodes, fums}`; raises
  `PassageNotFoundError` on empty.
- Serializer adds `'meta': fums` to both text and audio bodies
  when present — a LICENSE requirement (FUMS reporting).
- `BiblePassageView.get` itself is UNCHANGED on the branch.

Rust impact: the `PassageProvider` trait needs an `ApiBible`
variant; route order becomes ESV → SWORD → APIBible → DBT.
Design `providers/` so a `apibible.rs` module drops in without
reworking the handler. `chapter_audio` should return the raw
`timecodes` too (the timestamps endpoint reuses them).

### `feat/fileset-groups` (unmerged, BREAKS the error contract)

Deletes `bible/utils/provider_errors.py` and `test_provider_errors.
py` wholesale and rewrites `to_representation`:

- NEW `error_code` value `book_not_in_fileset` → HTTP 404, body
  `{book, book_name, chapter, error, error_code}` with default
  error `"Book {id} chapter {ch} is not available in fileset
  {fsid}"` (SWORD audio-not-generated keeps error
  `"Audio not yet generated for this chapter"`).
- Mapping changes vs main: DBT upstream 404 → `book_not_in_fileset`
  (not `not_found`); DBT empty `data` → `book_not_in_fileset`;
  ESV `requests.HTTPError` 404 → `book_not_in_fileset`; SWORD
  `ValueError` (PassageNotFoundError is a ValueError) →
  `book_not_in_fileset` (FileNotFoundError still falls through);
  `response_format=audio` on a DBT text fileset →
  `book_not_in_fileset`; `response_format=audio` + audio row with
  falsy `path` → `book_not_in_fileset`.
- CATASTROPHIC REGRESSION TO FLAG: the branch's generic
  `except Exception` returns `{"verses": [], "message": "No verses
  found for the specified passage"}` — a 200-shaped body with NO
  `error_code`. Rate limits / 500s / network failures would
  silently become empty 200s. If the branch merges first, VERIFY
  this is intentional before porting (almost certainly a bug;
  `test_book_not_in_fileset.py` only asserts the 404 paths).
- The view's `audio_url is None` upgrade becomes defensive-only
  (rewrites body to `book_not_in_fileset`).

Recommendation: implement the trait with a `BookNotInFileset`
error kind alongside `RateLimited`/`NotFound`/`Provider`, map it
to 404 either way, and gate the exact `error_code` string
(`not_found` vs `book_not_in_fileset`) + fallback-body behavior
behind whichever contract is live at cutover. Re-check merge
state when this plan is implemented.

## 1. Endpoint contract (main)

Route: `bible/urls.py:5` — `GET /api/v1/bible/`, name
`bible-passage`. Auth: `tok` per inventory — `Principal` suffices
(prod provisioning; tests anonymous; no `permission_classes` on
the view).

### Query params

| param             | req | default  | notes                          |
|-------------------|-----|----------|--------------------------------|
| `passage`         | yes | —        | `"<book> <chapter>"`           |
| `response_format` | no  | `text`   | only `text`/`audio` accepted   |
| `fileset_id`      | no  | `ENGESV` | DBT id / `ENGESV_API` / SWORD  |

Param details:

- `passage` is `split()` on whitespace; LAST token is the chapter,
  everything before joins into the book name (`' '.join(parts[:-1])`
  — multi-word books like `2 Chronicles`, `Song of Solomon`).
- The documented example `?passage=John+3:16` FAILS today:
  `int("3:16")` raises → 400. Verse selectors are NOT supported
  (whole chapters only). Reproduce.
- `response_format` is validated BEFORE parsing; anything else →
  `{"error": "Invalid format. Use 'text' or 'audio'"}` 400.
- `fileset_id=` (present but empty) → `''` → DRF `CharField
  required` rejects blank → 400 `{"fileset_id": ["This field may
  not be blank."]}`. Absent → `ENGESV` (a DBT id, NOT the ESV API).

### Chapter parsing quirks (reproduce verbatim)

`views.py:137-152`: `chapter = int(chapter_part)` then `<= 0`
raises `ValueError("Chapter must be a positive number")` — but
that raise is CAUGHT by the enclosing `except ValueError` and
re-raised as `"Chapter must be a valid number"`. Net effect: ANY
bad chapter (`abc`, `0`, `-3`, `3:16`) → 400 `{"error": "Chapter
must be a valid number"}`. The "positive number" string is dead.

Python `int()` accepts `+3`, `007`, and `1_0` (PEP 515 underscores
in `int()` strings). Rust `i32::from_str` accepts `+3`/leading
zeros but NOT underscores — a `1_0` request is a deliberate
divergence candidate (or accept via a pre-strip; see risks).
Python `int()` is arbitrary-precision: `John 99999999999999999`
passes validation and upstream-404s; an `i64` parse overflows →
400. Harmless but observable — pick a wide type or cap early.

### Success bodies (all 200)

Every body starts `{book, book_name, chapter, format}`.
`book` = resolved DBT book id (`JHN`), `book_name` = the caller's
raw book string verbatim (NOT normalized — `"Jhon"` echoes back).

- DBT text (default arm, `data[]` rows lack `path`):
  `{..., "format": "text", "verses": [{"verse": <verse_start>,
  "text": <verse_text>}, ...]}` — rows WITHOUT a `verse_text`
  key are filtered out entirely (serializer.py:186-194); the
  inner `.get('verse_text','')` default is dead code.
- DBT audio (`'path' in data[0]` — data-driven, ignores
  `response_format`):
  `{..., "format": "audio", "audio_url": <path>,
  "duration_seconds": <duration>, "file_size_bytes":
  <filesize_in_bytes>}`.
- ESV text (`ENGESV_API`): `{..., "format": "text", "verses":
  [{"verse","text"}], "headings": [{"before_verse": int, "text":
  str}]}`. Empty parsed verses → `PassageNotFoundError` → 404.
- ESV audio: `{..., "format": "audio", "audio_url": <ESV CDN
  redirect Location>}`. No `duration`/`file_size`.
- SWORD text (`LVSGLU8`/`GLU8`/`LVSGLU8C1DA`): `{..., "format":
  "text", "verses": [{"verse","text"}]}` — no `headings` key.
- SWORD audio (generated-artifact read): `{..., "format":
  "audio", "audio_url": <V4 signed URL>, "duration_seconds":
  <float>, "file_size_bytes": <int>}` from the timestamps JSON.
- SWORD audio not yet generated: serializer returns `{..., "format":
  "audio", "audio_url": null, "message": "Audio not yet generated
  for this chapter"}`; the VIEW upgrades it to **404** (same body).

Critical DBT quirk: `response_format=audio` on a TEXT fileset
(e.g. `ENGESV`) still returns the TEXT body — the param is only a
routing hint for ESV/SWORD; DBT shape is decided by `data[0]`'s
keys. Port faithfully.

### Error bodies (verbatim)

View-level 400s (hand-rolled `{"error": ...}` shape):

- Missing `passage`: `{"error": "Passage parameter is required.
  Example: ?passage=John+3:16"}` — note the (broken) `:16`.
- Bad `response_format`: `{"error": "Invalid format. Use 'text'
  or 'audio'"}`.
- `len(parts) < 2`: `{"error": "Invalid passage format. Use
  'Book Chapter' (e.g., 'John 3')"}`.
- Unknown book (`get_dbt_book_id` → None): `{"error": "Unknown
  book: {book_name}"}`.
- Bad chapter: `{"error": "Chapter must be a valid number"}`.
- Serializer invalid (only `fileset_id=""` in practice): DRF
  field map `{"fileset_id": ["This field may not be blank."]}`.
- Outer `except Exception` catch-all: `{"error": str(e)}` 400 —
  only reachable for client-input errors today; keep it as a
  defensive fallback for truly-unexpected handler panics (an axum
  `IntoResponse` on an internal panic variant), NOT for provider
  errors.

Provider errors (serializer-embedded fields + view-upgraded
status, §4.4):

- `{"book","book_name","chapter","error","error_code"}` — no
  `format`, `verses`, `audio_url`, or `message` keys.
- upstream 429 → 429 `{"error": "Bible provider rate limit
  exceeded (HTTP 429)", "error_code": "rate_limited"}`.
- `PassageNotFoundError` or upstream 404 → 404 `{"error":
  "Passage not found" or "Passage not found (HTTP 404)",
  "error_code": "not_found"}`.
- upstream other-status → 502 `{"error": "Bible provider error
  (HTTP {status})", "error_code": "provider_error"}`.
- no upstream status (network/timeout/ValueError/KeyError) → 502
  `{"error": "Bible provider request failed ({ExcTypeName})"}` —
  Rust: the ProviderError variant name, never `err.to_string()`
  (reqwest errors embed the URL incl. `?key=<DBT_KEY>`).

ESV audio redirect: a non-3xx/5xx response or missing `Location`
header → `ValueError` → generic catch → 502
`provider_error` (class name `ValueError` in the message).

## 2. Current Django behavior trace

```
GET /api/v1/bible/?passage=&fileset_id=[&response_format=]
  │
  ├─ Principal resolution (middleware/token — §4.2)
  ├─ read passage / response_format(text) / fileset_id(ENGESV)
  ├─ passage empty            → 400 {"error": "...required..."}
  ├─ format ∉ {text,audio}    → 400 {"error": "Invalid format..."}
  ├─ parts = passage.split(); <2 → 400 invalid format
  ├─ book_name = ' '.join(parts[:-1]); chapter_part = parts[-1]
  ├─ book_id = get_dbt_book_id(book_name)   (bible_books.py)
  │     normalize: lower, collapse ws, roman→arabic prefix,
  │       'revelation of john'→'revelation', strip 1st/2nd/3rd,
  │       first/second/third→1/2/3
  │     → exact DBT_BOOK_NAME_TO_ID hit
  │     → alias table (isiah, canticles, apocalypse, jhon, ...)
  │     → difflib.get_close_matches(cutoff=0.8), digit-locked
  │       ('3 samuel' only tries '3 *' books; unnumbered input
  │       only tries unnumbered books)
  │     None → 400 {"error": "Unknown book: {name}"}
  ├─ int(chapter_part); fail or <=0 → 400 "Chapter must be a
  │     valid number"  (positive-number msg is dead — see §1)
  ├─ data = {book, book_name, chapter, response_format,
  │     fileset_id} → BiblePassageSerializer.is_valid()
  │     (CharField/IntegerField(min_value=1) — fails only on
  │      blank fileset_id) → 400 field map
  └─ to_representation(data) — provider dispatch, in order:
        1. is_esv_fileset(fsid)     — strip().upper() in
           ESV_TRANSLATIONS {ENGESV_API}
           ├─ audio → esv.get_chapter_audio_url(book_id, ch):
           │   GET api.esv.org/v3/passage/audio/?q="<Book> <ch>"
           │   + Authorization: Token <ESV_KEY>,
           │   allow_redirects=False; 3xx → Location header;
           │   else raise (502 path). 200 + audio body.
           └─ text → esv.get_chapter_with_headings(book_id,ch):
               GET api.esv.org/v3/passage/text/?q=…
               &include-headings=true&include-verse-numbers=true
               &include-footnotes=false&include-passage-references=
               false&include-short-copyright=false
               &include-first-verse-numbers=true
               &include-passage-horizontal-lines=false
               &include-heading-horizontal-lines=false  (timeout=10)
               → _parse_passage() (regex \[(\d+)\] markers,
               blank-line blocks, indented-block = continuation
               else heading; headings collect before_verse)
               → empty verses → PassageNotFoundError → 404.
               200 + verses + headings.
        2. is_sword_fileset(fsid) AND format==audio:
           canon = canonical_sword_fileset_id (id | abbr | audio
           id, case-insensitive); voice = get_tts_config(canon)
           ["voice_name"] (KeyError if non-LVSGLU8 → 502 path).
           ├─ gcs.chapter_audio_exists(canon, book, ch, voice)
           │   HEAD gs://{AUDIO_BUCKET}/audio/{canon}/{voice}/
           │   {BOOK}/{CH}_audio.mp3 → False → body audio_url:
           │   null + message → VIEW → 404.
           ├─ gcs.read_timestamps_json(...) →
           │   {data: [{verse_start, timestamp}], duration_s…,
           │   file_size_bytes}. file_size_bytes==None → legacy
           │   fallback blob.get_blob().size HEAD.
           └─ gcs.signed_audio_url(...) → V4 signed GET URL,
               TTL AUDIO_SIGNED_URL_TTL_SECONDS (3600), IAM
               signBlob via runtime SA + OAuth token; cached per
               (fileset, voice, book, ch) for TTL−60s. 200 body.
        3. is_sword_fileset(fsid) (text):
           sword.get_chapter_verses(fsid, book_id, ch) —
           lazy-load LvGluck8.zip via pysword; book-name map
           (module's own names, e.g. "I Kings") resolved per
           canon; iterate verse 1..200 until empty/err; empty →
           PassageNotFoundError. 200 + verses.
        4. else DBT:
           dbt.get_verses(book_id, str(ch), bible_id=fsid) →
           GET https://b4.dbt.io/api/bibles/filesets/{fsid}/
           {book}/{ch}?v=4&key=<DBT_KEY> (generated client,
           auth_settings dbp_key = query param; verify_ssl=False!).
           ├─ data empty/missing → PassageNotFoundError → 404.
           ├─ 'path' in data[0] → audio body (above).
           └─ else text body; rows lacking verse_text dropped.
        catch Exception → {book, book_name, chapter,
            **provider_error_fields(e)} → view maps
            error_code → 429/404/502.
  ├─ 'error' in body → Response(body, status by error_code,
  │     default 502)
  ├─ format==audio && audio_url==None → Response(body, 404)
  └─ else 200 body
```

Upstream calls per request: exactly ONE provider call for text;
audio = one call (ESV redirect / DBT chapter) or THREE GCS ops
(exists HEAD + JSON download + signBlob, minus cache hits).
SWORD = zero network. No DB access. No caching of passage text
anywhere; only the signed-URL cache (gcs.py:151-213).

## 3. Rust implementation plan

### Module map

- `crates/server/src/routes/bible/passage.rs` — handler + DTOs +
  OpenAPI.
- `crates/core/src/books.rs` — `get_dbt_book_id` port (see below;
  likely already landed from Phase 0 — reuse).
- `crates/providers/src/{mod,dbt,esv,sword}.rs` — trait impls.
- `crates/storage/src/gcs.rs` — `chapter_audio_exists`,
  `read_timestamps_json`, `signed_audio_url` + cache (§4.9).
- `ApiError`/`ProviderError` in `core::errors` (§4.4).

### Request DTO

```rust
#[derive(Deserialize)]
struct PassageQuery {
    passage: Option<String>,
    response_format: Option<String>, // validate ∈{text,audio}
    fileset_id: Option<String>,     // default "ENGESV"
}
```

Use `Query<HashMap<String,String>>` or an `Option`-typed struct —
DRF `.get()` treats missing-vs-blank differently, so do NOT
`#[serde(default)]`-coerce `fileset_id=""` into `ENGESV`; keep the
blank→400 distinction (`{"fileset_id": ["This field may not be
blank."]}` exact body via `ApiError::Validation`).

### Handler skeleton

```rust
async fn bible_passage(
    State(s): State<AppState>, _p: Principal,
    Query(q): Query<PassageQuery>,
) -> ApiResult<Json<Value>>
```

Steps mirror the view exactly: missing passage →
`ApiError::BadRequest(error_obj)`, format check → same, split +
`<2` → same, `books::get_dbt_book_id` → same, chapter parse (see
quirks below) → same, blank `fileset_id` → Validation field map.
Then `provider = s.providers.for_fileset(&fileset_id)` and call
`chapter_text`/`chapter_audio` per the dispatch order below.

Chapter parse: `chapter_part.parse::<i64>()` — replicate Python
`int()`: `+3`/` 007` OK in both; Python also accepts `1_0` —
either pre-strip underscores inside the digits or accept the
divergence (a `John 1_0` request 400s either way; only the ERROR
STRING would differ — both emit "valid number", so identical
observable behavior; the difference is `int("1_0")==10` succeeds
in Django → fetches ch 10. Decide: strip `_` before parse for
parity — cheap). Any failure or `<=0` → "Chapter must be a valid
number".

### Provider dispatch (preserve ORDER)

```
if is_esv_fileset(fsid)      → EsvProvider
elif is_sword_fileset(fsid)  → SwordProvider (audio arm → GCS)
elif is_apibible_fileset     → ApiBibleProvider   [branch only]
else                         → DbtProvider
```

`PassageProvider` trait usage (§4.7): `chapter_text` returns
`TextChapter { verses: Vec<{verse:i64,text:String}>,
headings: Option<Vec<{before_verse,text}>> }` — `headings` only
populated for ESV (key PRESENT-but-[] for ESV, ABSENT for
DBT/SWORD — serde `skip_serializing_if=Option::is_none` does NOT
emit it; verify JSON key presence matches: ESV emits
`"headings":[]`, others omit).
`chapter_audio` returns `AudioChapter { audio_url:
Option<String>, duration_seconds: Option<f64>,
file_size_bytes: Option<i64>, message: Option<String> }` — SWORD
audio-miss returns `audio_url: None` + `message`, which the
handler turns into 404.

Because provider bodies share `{book,book_name,chapter,format}`
but diverge after, build the final `serde_json::Value` in the
handler from a per-provider `PassagePayload` enum — do NOT try to
force a single struct.

### reqwest clients

- DBT (`providers/dbt.rs`): `GET {base}/bibles/filesets/{fsid}/
  {book}/{ch}?v=4&key={DBT_KEY}` — hand-rolled reqwest, NOT the
  generated client. Deserialize `data` as `Vec<Map>`; detect audio
  via `data[0].contains_key("path")`. Map non-2xx →
  `ProviderError::upstream(status)`. IMPORTANT: `verify_ssl=False`
  in Django — Rust uses rustls with proper verification; this is
  a deliberate security IMPROVEMENT, keep it (log as deviation).
- ESV (`providers/esv.rs`): one `reqwest::Client` with default
  `Authorization: Token {ESV_KEY}` header. Text → GET
  `/v3/passage/text/` with the 9 params verbatim; audio → GET
  `/v3/passage/audio/` with `redirect::Policy::none()` on a
  SEPARATE client builder (policy is per-client), accept 3xx →
  `headers[LOCATION]` else `ProviderError::provider` (never
  `not_found` for missing Location — Django maps ValueError →
  provider_error). Timeout 10s both.
- ESV `_parse_passage` port (`providers/esv.rs` or `core`):
  `regex` crate, `\[(\d+)\]`, split on `\n{2,}`, exact
  block/pending-heading/pending-verse state machine from
  client.py:180-319 including `is_indented` handling. This is
  THE subtlest pure function in the codebase — port line-by-line
  and replay every fixture in `test_esv_client.py`
  (incl. the Daniel 7 + Luke 4 regression passages).
- SWORD (`providers/sword.rs`): per §4.10 — `rsword_chirho` eval,
  else ~400-line pysword ztext port. Keep `OnceCell`-loaded
  module + `Mutex`, book-name map built from the module's own
  structure (Roman-numeral regression coverage), verse loop
  1..=200, empty → `ProviderError::not_found`. Client is
  `Arc<Mutex>` inside the provider.
- GCS audio (`storage/gcs.rs`): §4.9 — `chapter_object_paths`,
  exists check, `read_timestamps_json`, V4 `signed_audio_url`
  with `(fileset, voice, book, chapter)`-keyed cache (TTL−60s),
  legacy `file_size_bytes` HEAD fallback — DECIDE: keep for
  compat (one release) or drop since Rust deploys fresh; plan
  keeps it behind a small branch (matches Django until old
  objects are regenerated).
- API.Bible (branch-gated): `providers/apibible.rs` —
  `api-key` header client; `get_chapter` JSON `content` walker →
  verses+headings+`fums` meta passthrough; `get_chapter_audio` →
  presigned `resourceUrl` + `timecodes`. Add `meta: Option<Value>`
  to the response enum for FUMS.

### Error mapping

`ProviderError` (§4.4) → `error_fields()` JSON; handler checks
`body.get("error")` after building → status map
`rate_limited→429, not_found→404, default→502` — preserve the
two-layer shape (fields embedded in the success-shaped body AND
the status upgrade). `audio_url: null` + `format: "audio"` → 404.
If `fileset-groups` merged: add `book_not_in_fileset→404` to the
status map and emit branch bodies; keep an audit list of every
`not_found`→`book_not_in_fileset` site (§0).

Timeouts: ESV/DBT/API.Bible reqwest `timeout(10s)` parity; GCS
uses SDK defaults. Network/timeout errors carry no status →
`"Bible provider request failed ({VariantName})"` — implement
`ProviderError::kind_name()` returning a stable string per variant
(`Timeout`, `Connect`, `Decode`, `Upstream {status}`…) rather than
`Display` (credential leak risk, §4.4).

Logging: `tracing::info!` per request (params), `debug!` for
parse steps, `warn!` for 400s, `error!` w/ `?err` for provider
failures — mirrors the view's generous logging (only observability
this app has).

## 4. Test plan

Django tests covering this endpoint (mirror 1:1):

- `bible/tests/test_provider_errors.py` — `ProviderErrorSerializer
  Tests` (429/500/ESV-429/statusless-leak/404/empty-data/ESV-empty/
  SWORD-missing → error_code) + `ProviderErrorViewTests`
  (view-level 429/502/404). → `tests/bible_passage_errors.rs`
  with wiremock bodies + a `FakeProvider`.
- `bible/tests/test_audio_streaming.py` — SWORD audio happy path
  (mocked gcs → signed URL, duration, file_size via legacy blob
  fallback). → axum-test + fake storage.
- `bible/tests/test_sword.py` — `test_bible_passage_endpoint_
  glucks`, `test_bible_passage_accepts_abbr_gl8` (real module!);
  `SwordClientTests` (roman numerals, abbr, list_chapters). →
  fixture `LvGluck8.zip` test against the real reader, or
  FakeProvider at HTTP layer.
- `bible/tests/test_esv_client.py` — registry tests, session/
  header tests, `ParsePassageVersesTests`, `ParsePassageHeadings
  Tests`, `ParsePassagePoetryAndQuotesTests` (Daniel 7, Luke 4 —
  the issue-#66 regression), audio redirect tests. → unit tests
  for `esv::parse_passage` verbatim fixtures + wiremock for
  redirect Location.
- `bible/tests/test_bible_books.py` — exact/alias/fuzzy/unresol
  vable name tables. → `core::books` unit tests (difflib crate
  must reproduce `get_close_matches` cutoff=0.8 + n=1 semantics —
  VERIFY tie-breaking matches CPython's, which returns the FIRST
  max-ratio candidate in order).
- On `feat/fileset-groups`: `test_book_not_in_fileset.py` (422
  lines) — port if merged.
- On `feat/apibible-niv`: `test_apibible_client.py` (882 lines) —
  port if merged.

New Rust tests beyond mirroring:

- wiremock DBT: `/bibles/filesets/ENGESV/JHN/3?v=4&key=…` text
  rows, audio rows (`path`), `{"data":[]}` → 404, 429/500/404
  status mapping, malformed JSON → provider_error.
- wiremock ESV: text params assertion, `passages:[""]` → 404,
  audio 302+Location, audio 200 → 502 (ValueError-class
  provider_error), audio 302-no-Location → 502.
- DBT quirk: `response_format=audio` + text fileset → text body.
- `fileset_id=` blank → 400 field-map body verbatim.
- Passage parse: `"John 3"`, `"2 Chronicles 14"`, `"john 3"`,
  `"Jhon 3"` (alias), `"xyz 3"` → "Unknown book: xyz",
  `"John"`, `"John abc"`, `"John -3"`, `"John 0"`,
  `"John 1_0"` (parity decision), `"John 3:16"` → 400s.
- SWORD abbr/audio-id canonicalization (`GLU8`, `LVSGLU8C1DA`).
- SWORD audio: exists→signed URL cached; not-exists → 404
  `audio_url: null` + `message` body; `file_size_bytes` legacy
  HEAD fallback.
- Status map: `error_code` absent-but-`error` present → default
  502 (matches `PROVIDER_ERROR_STATUSES.get(...)` default).
- Parity corpus (`tests/parity/`): replay
  `?passage=John 3&fileset_id={ENGESV,ENGESV_API,LVSGLU8,GLU8}`
  ×`{text,audio}` plus the error matrix, diff after normalizing
  `audio_url` (signed-URL signatures) and `duration_seconds`.

## 5. Open questions / risks

1. **fileset-groups merge state** (§0): if merged, the entire
   provider-error surface changes (`not_found`→`book_not_in_
   fileset`, `provider_errors.py` deleted, serializer fallback
   becomes a 200 `{"verses":[],"message":…}`). The 200-empty
   fallback looks like a REGRESSION — confirm intent before
   porting; the workspace root AGENTS.md already documents
   `book_not_in_fileset`+`alternates` as live contract, so this
   likely merges first. GATE: re-run `git merge-base`/`grep` at
   implementation time.
2. **apibible merge state** (§0): adds provider #4 + `meta`/
   FUMS passthrough + `timecodes` reuse; trait design must not
   need rework. `ENGNIVC1DA` canonicalization mirrors SWORD.
3. **difflib parity**: `difflib` crate's `SequenceMatcher` must
   match CPython `ratio`/tie-order for `get_close_matches`.
   Verify against the test table (e.g. `jud`→JUD, `eze`→None).
   If it diverges, hand-port the matcher (~100 lines).
4. **ESV parser state machine**: `_parse_passage` has subtle
   flush ordering (pending_heading vs pending_verse_text vs
   is_indented). Rust port must replay the committed fixtures
   byte-for-byte; poetry/quote blocks are the known failure mode.
5. **Python `int()` vs Rust parse**: underscores/`+`/whitespace —
   decide strip-underscore parity or documented divergence.
6. **SWORD reader risk** (§4.10): `rsword_chirho` output must be
   diffed per-verse against pysword for all LvGluck8 books before
   adopting; fallback is the ztext port. The `book.name` mapping
   (`_get_book_name_map`) is module-internal — whatever reader we
   pick must expose the module's own book names + structure.
7. **`verify_ssl=False` on DBT**: Django disables TLS
   verification for b4.dbt.io. Rust won't — deliberate hardening;
   if DBT's cert chain is actually broken this becomes a
   functional regression — TEST against live DBT early.
8. **GCS ops per audio request**: exists-HEAD + JSON read +
   signBlob = ~3 calls; the signed-URL cache only covers signing.
   Latency parity fine; consider an `exists`→`read` consolidation
   later (timestamps JSON presence implies MP3 by upload order —
   actually JSON is written FIRST, so JSON-exists does NOT imply
   MP3. Keep both checks).
9. **`book_name` echo**: response echoes the RAW caller string
   (`book_name` field), not normalized — easy to "fix"
   accidentally; keep verbatim.
10. **Non-LVSGLU8 TTS config**: `get_tts_config` raises KeyError
    for any other canon → 502 provider_error (`KeyError` class
    name in the message). Only reachable if registry adds a
    SWORD fileset without TTS config — guard anyway.
11. **`fileset_id` case**: `is_esv_fileset`/`is_sword_fileset` are
    case-insensitive (`strip().upper()`); DBT ids pass through
    VERBATIM (`engesv` ≠ `ENGESV` to DBT — upstream 404s →
    `not_found` 404). Reproduce the asymmetry.
12. **No verse-level passage support**: `John 3:16` 400s today;
    frontend may rely on this failing loudly. Do not "improve".
