# 09 — Bible discovery: translations, timestamps, copyright

Scope: inventory items 2, 3, 4 from `00-base.md` §7.

- `GET /api/v1/bible/translations/?language_iso=` —
  `TranslationListView` (bible/views.py:646-695)
- `GET /api/v1/bible/timestamps/?fileset_id=&book=&chapter=` —
  `AudioTimestampView` (bible/views.py:220-296)
- `GET /api/v1/bible/copyright/?bible_id=` —
  `CopyrightView` (bible/views.py:299-335)

Auth: all three are plain `APIView`s — prod default permission is
`IsAuthenticated`, satisfied by device auto-provisioning
(`00-base.md` §4.2); tests run anonymous. Rust: `Principal`
extractor (`auth:tok`), no `AuthUser` gate.

Paths below are relative to the Django repo root. Main-branch
behaviour is the baseline; `[BRANCH]` tags mark in-flight deltas
from `feat/fileset-groups` and `feat/apibible-niv`.

## 1. Endpoint contracts

### 1.1 `GET /api/v1/bible/translations/`

Query params:

- `language_iso` — optional, ISO 639-3 (`eng`, `lvs`). Forwarded
  to DBT upper-cased as `language_code`; compared lower-cased
  against registry `iso` fields. Absent = no filtering.

200 body (main):

```json
{
  "results": [
    {
      "abbr": "GLU8",
      "name": "Latvian Glück 8th edition",
      "language": "Latvian",
      "language_iso": "lvs",
      "filesets": [
        {"id": "LVSGLU8", "type": "text_plain", "size": "C"},
        {"id": "LVSGLU8C1DA", "type": "audio", "size": "C"}
      ]
    }
  ]
}
```

[BRANCH `feat/fileset-groups`] each result gains two arrays —
`text_options` and `audio_options` — while `filesets` stays
unchanged, plus optional `codec`/`bitrate` passthrough keys on
individual fileset entries:

```json
{
  "abbr": "LAVNLI",
  "text_options": [
    {
      "id": "LAVNLI:text:1",
      "kind": "text",
      "by_testament": {
        "OT": {"text": "LATBSLO_ET"},
        "NT": {"text": "LATBSLN_ET"}
      },
      "coverage_label": "Full Bible",
      "partial": {"OT": false, "NT": false}
    }
  ],
  "audio_options": [
    {
      "id": "LAVNLI:audio:1",
      "kind": "audio",
      "by_testament": {
        "OT": {"mp3": "LATBSLP1DA", "opus16": "LATBSLP1DA-opus16"},
        "NT": {"mp3": "LATBSLN1DA", "opus16": "LATBSLN1DA-opus16"}
      },
      "coverage_label": "Full Bible (partial)",
      "partial": {"OT": true, "NT": false}
    }
  ]
}
```

- `id` is synthetic: `{abbr}:{kind}:{n}` where `n` is a per-kind
  counter in encounter order (`LAVNLI:audio:1`,
  `LAVNLI:audio_drama:1`, `GLU8:generated:1`).
- `kind` ∈ `audio` | `audio_drama` | `generated` | `text`.
- `by_testament` codec keys are an OPEN set: `mp3`, `opus16`,
  `generated` (SWORD TTS), `api` (ESV), `text` (merged text).
- `coverage_label` ∈ `Full Bible` | `New Testament only` |
  `Old Testament only` | `No coverage`, with ` (partial)`
  appended when any winning member is partial.
- `partial` always contains BOTH `"OT"` and `"NT"` booleans.
- Entries whose `abbr`/`name`/`language`/`iso`/`filesets` keys
  are absent crash the view (KeyError → 502; see below).

Errors:

- DBT leg failure is SWALLOWED (see §2.1) — never surfaces.
- Exception in registry listing or the view's field mapping →
  `provider_error_response`: `{"error", "error_code"}` with the
  §4.4 mapping (429 → `rate_limited`, 404 → `not_found`, else
  502 `provider_error`). A missing `t['iso']` is a `KeyError`
  (no status) → 502 `{"error": "Bible provider request failed
  (KeyError)", "error_code": "provider_error"}`.
- [BRANCH `feat/fileset-groups`] REGRESSES this: view returns
  `{"error": str(e)}` with HTTP 500. Do not port — see §5.

### 1.2 `GET /api/v1/bible/timestamps/`

Query params (all three required):

- `fileset_id` — DBT audio id, SWORD id/abbr/audio id
  (`LVSGLU8`, `GLU8`, `LVSGLU8C1DA`), [BRANCH] API.Bible ids.
- `book` — free-text book name; resolved by `get_dbt_book_id`
  (normalisation + aliases + difflib fuzzy, cutoff 0.8 —
  bible/utils/bible_books.py:195). Bare ids like `JHN` also
  resolve via the fuzzy path (`jhn` → `john`).
- `chapter` — chapter number (string on the wire).

200 body (both sources):

```json
{"data": [{"verse_start": 1, "timestamp": 0.0},
          {"verse_start": 2, "timestamp": 5.34}]}
```

- `verse_start`/`timestamp` pass through verbatim (ints/floats
  as upstream sent them; absent keys → `null`).
- SWORD/GCS path returns ONLY `payload["data"]` — sibling keys
  (`duration_seconds`, `file_size_bytes`) are dropped.

Error bodies:

- Missing any param → 400
  `{"error": "fileset_id, book, and chapter are required."}`
- Unknown book → 400 `{"error": "Unknown book: {book}"}`
- GCS blob absent → 404
  `{"error": "Timestamps not yet generated for this chapter."}`
- Other failures → `provider_error_response` per §4.4:
  upstream 429 → 429 `rate_limited`; upstream 404 → 404
  `not_found`; else 502 `provider_error` with
  `"Bible provider request failed ({ErrorKind})"` — NEVER raw
  exception text (`?key=` leak guard).
- Edge: SWORD path does `int(chapter)` — `chapter=abc` raises
  `ValueError` → caught by outer `except` → 502
  `provider_error`. DBT path interpolates the raw string into
  the URL → upstream decides (usually 404 → `not_found`).
- [BRANCH `feat/apibible-niv`] adds an upfront guard:
  `chapter` non-integer or `< 1` → 400
  `{"error": "chapter must be a positive integer."}`, and
  passes `chapter_num` (int) to the DBT client instead of the
  raw string.
- [BRANCH `feat/fileset-groups`] REGRESSES errors: 502
  `{"error": "Timestamps unavailable: {exc}"}` (leaks exception
  text) and outer catch → 400 `{"error": str(e)}`. Do not port.

### 1.3 `GET /api/v1/bible/copyright/`

Query params:

- `bible_id` — required. Passed to DBT verbatim; any value
  (incl. `ENGESV_API`, `LVSGLU8`) hits DBT on main.

400 when absent: `{"error": "bible_id is required."}`

200 body — `data` is a per-FILESET list (one bible can have
several filesets):

```json
{
  "data": [
    {
      "id": "ENGESV",
      "type": "text_plain",
      "size": "C",
      "copyright": "© 2001 Crossway Bibles",
      "copyright_date": "2001",
      "copyright_description":
        "The Holy Bible, English Standard Version"
    },
    {
      "id": "ENGESVN2DA", "type": "audio", "size": "NT",
      "copyright": "© 2001 Crossway Audio",
      "copyright_date": "2001",
      "copyright_description": "ESV Audio"
    }
  ]
}
```

- `id`/`type`/`size` pass through (may be `null`).
- `copyright`/`copyright_date`/`copyright_description` come
  from the nested `copyright` object; absent → `""`.

Errors: missing `bible_id` → 400 above; DBT failure →
`provider_error_response` (429/404/502 mapping).

[BRANCH `feat/apibible-niv`] `bible_id` matching an API.Bible
fileset (`ENGNIV_API` or `ENGNIVC1DA` — the canonicalizer maps
both to `ENGNIV_API`) routes to API.Bible
`GET /bibles/{bible_id}` and returns a SINGLE-element list:

```json
{"data": [{
  "id": "ENGNIVC1DA", "type": "text_plain", "size": "C",
  "copyright": "<copyrightStatement>",
  "copyright_date": "",
  "copyright_description": "<copyrightStatement>"
}]}
```

(`id` echoes the as-passed query value; `copyright_statement`
missing → `""`.)

## 2. Current Django behaviour trace

### 2.1 Translations — `TranslationService.get_live_translations`

bible/services/translation_service.py:18-70. Assembly order:

1. DBT leg — `DBTClient.get_bibles(limit=500,
   language_code=ISO.upper())` via RAW `requests.Session.get`
   (dbt/client.py:96-124), not the generated OpenAPI client:
   `GET https://b4.dbt.io/api/bibles?limit=500&v=4&key=<DBT_KEY>
   [&language_code=ENG]`. Note `language_code` arrives through
   `**kwargs`, not the `language` parameter.
2. `_process_translations` (:73-134) on `response["data"]`:
   - Iterate `trans["filesets"]` — a DICT keyed by source
     (`dbp`, `dbp-vid`, …); skip source `dbp-vid` entirely.
   - Per fileset, exclude when `'video' in type` or `type ∈
     {video, audio_stream, audio_drama_stream}`.
   - Translations with zero surviving filesets are dropped
     (warning log).
   - Project each surviving fileset to `{id, type, size}`.
     [BRANCH `feat/fileset-groups`] also `codec`/`bitrate` when
     the DBT item carries them.
   - The rest of `trans` is kept verbatim but the VIEW only
     reads `abbr`, `name`, `language`, `iso`, `filesets`.
3. DBT exceptions are caught and logged; `processed = []` — the
   endpoint still returns 200 with registry entries only.
   THIS IS A CONTRACT: DBT outage ⇒ partial success, not 502.
4. SWORD registry entries via
   `SwordClient.get_translation_listing`
   (sword/client.py:180-202 — iterates the static registry, no
   module loading): `LVSGLU8` → `{abbr:"GLU8", name:"Latvian
   Glück 8th edition", language:"Latvian", iso:"lvs",
   filesets:[{LVSGLU8,text_plain,C},{LVSGLU8C1DA,audio,C}]}`.
5. ESV registry via `get_esv_translation_listing`
   (esv/registry.py:24-46): `ENGESV_API` → `{abbr:"ESV",
   name:"English Standard Version", language:"English",
   iso:"eng", filesets:[{ENGESV_API,text_plain,C},
   {ENGESV_API,audio,C}]}` — same id for text AND audio.
6. `language_iso` filter applied per-source: DBT via the
   `language_code` query param; registries via
   `e["iso"].lower() == language_iso.lower()`.
7. Concatenation order: `processed + sword + esv`.
   [BRANCH `feat/apibible-niv`] appends
   `get_apibible_translation_listing()` after ESV:
   `ENGNIV_API` → `{abbr:"NIV", name:"New International
   Version", language:"English", iso:"eng",
   filesets:[{ENGNIV_API,text_plain,C},{ENGNIVC1DA,audio,C}]}`
   — filtered by `language_iso` the same way.
8. [BRANCH `feat/fileset-groups`] after concatenation, each
   entry gets `group_filesets(entry)` →
   `entry["text_options"]`/`["audio_options"]` (see §2.4).
9. View re-projects `{abbr, name, language, language_iso=t.iso,
   filesets}` (+ `text_options`/`audio_options` via
   `t.get(…, [])` on the branch) into `{"results": [...]}`.

Caching: NONE — every request hits DBT live (~300 ms warm
keep-alive via the singleton `requests.Session`). Parity: same.

### 2.2 Timestamps — source selection

bible/views.py:223-296:

1. Param presence check → 400; `get_dbt_book_id(book)` → 400 on
   miss. Book names normalise through lowercase/roman numerals/
   ordinals/aliases/fuzzy (bible_books.py:107-217) — port lives
   in `core::books`.
2. `is_sword_fileset(fileset_id)` → GCS arm (ESV ids are NOT
   special-cased — `ENGESV_API` falls to DBT; preserve this
   quirk):
   - `canon = canonical_sword_fileset_id(fileset_id)` — accepts
     canonical id, `abbr`, or audio id, case-insensitive
     (sword/registry.py:28-41).
   - `voice_name = get_tts_config(canon)["voice_name"]` —
     `GOOGLE_TTS_VOICE_NAME` setting
     (`lv-LV-Chirp3-HD-Sadachbia`); `KeyError` for non-LVSGLU8.
   - `gcs.read_timestamps_json(canon, book_id, int(chapter),
     voice_name)` → blob
     `audio/{canon}/{voice_name}/{book_id}/{int(chapter)}
     _timestamps.json` in `AUDIO_BUCKET_NAME`
     (storage/gcs.py:41-52, 140-148).
   - `google.api_core.exceptions.NotFound` → 404 "Timestamps
     not yet generated…"; any other exception →
     `provider_error_response` (502 typically).
   - Success → `{"data": payload.get("data", [])}`.
3. Else DBT arm — `GET {base}/timestamps/{fileset_id}/{book_id}
   /{chapter}?v=4&key=…` (raw requests path, dbt/client.py:
   241-279; `chapter` passed as the raw string on main). Each
   `data` item projected to `{verse_start, timestamp}` — other
   upstream keys dropped.
4. Outer `except Exception` → `provider_error_response`.

[BRANCH `feat/apibible-niv`] inserts, between the SWORD and DBT
arms:

- Upfront `int(chapter)` validation → 400
  `"chapter must be a positive integer."` (applies to ALL
  fileset kinds, incl. DBT).
- `is_apibible_fileset(fileset_id)` →
  `ApiBibleClient.get_chapter_audio(meta["audio_bible_id"],
  book_id, chapter_num)` → `timecodes` mapped by
  `_apibible_timecode` to `{verse_start:int|None,
  timestamp:float|None}` (parses `verseId` like `JHN.3.16`,
  range tails stripped; numbers may arrive as strings and are
  coerced). Empty timecodes → the same 404 "not yet generated"
  body as the GCS arm.

### 2.3 Copyright

bible/views.py:302-335 → `DBTClient.get_copyright`
(dbt/client.py:281-316): `GET {base}/bibles/{bible_id}
/copyright?v=4&key=…`. NOTE: the upstream body is a bare JSON
LIST (no `data` envelope — verified by test_dbt_client.py:64-
84). View iterates `result or []`, projects
`{id, type, size, copyright, copyright_date,
copyright_description}` with `""` defaults for missing nested
fields. Any exception → `provider_error_response`.

[BRANCH `feat/apibible-niv`] `is_apibible_fileset(bible_id)` →
`ApiBibleClient.get_bible(meta["bible_id"])` →
`data.copyrightStatement` → single-element response (§1.3).
`get_bible` hits `GET {apibible}/bibles/{bible_id}` with an
`api-key` header, base `https://api.scripture.api.bible/v1`.

### 2.4 [BRANCH] `group_filesets` algorithm

bible/services/fileset_groups.py (exists only on
`feat/fileset-groups`; spec:
`.research/translation-selector/implementation-plan.md` §"Data
model"). Pure function; port verbatim:

- `_split_codec`: id ending `-opus16` → codec `opus16`,
  base = stripped; else `mp3`.
- `_size_coverage`: `C` in size → OT+NT; `NT`/`OT` substrings
  mark coverage; `P` anywhere → covered testaments partial.
  Empty/unparseable size → fileset SKIPPED entirely.
- `AUDIO_ID_RE = ([CONPS])(\d)DA$` on the BASE id → `letter`,
  `digit`.
- Per fileset dispatch (in order):
  1. `type == "text_plain"` → `text_members` (codec `text`).
  2. `type` not in `{audio, audio_drama}` → skip.
  3. `is_sword_fileset(base_id)` → singleton, kind `generated`,
     codec `generated`.
  4. `is_esv_fileset(base_id)` → singleton, kind `audio`,
     codec `api`.
  5. `digit` matched → group keyed `(type, digit)` — `digit=2`
     or `fs_type==audio_drama` → kind `audio_drama`, else
     `audio`. NOTE: ENGNIVC1DA matches this rule (`C1DA`) when
     both branches merge — see §5.
  6. else → singleton, kind `audio` (codec from `_split_codec`).
- `_build_option` merges members per (testament, codec) slot;
  `_member_rank` — complete beats partial, non-`P` letter beats
  `P` — applied PER CODEC SLOT, so a partial `opus16` winner can
  mark a testament partial beside a complete `mp3`.
- `pending` list preserves encounter order → stable option
  ordering; per-kind counter `n` forms `{abbr}:{kind}:{n}`.
- `partial` map always emits both `OT` and `NT`; `by_testament`
  only emits covered testaments, ordered OT then NT.

### 2.5 [BRANCH] error-contract divergence — IMPORTANT

`feat/fileset-groups` DELETES `bible/utils/provider_errors.py`
and rewires every view's catch-all to `{"error": str(e)}` at
400/500 — this reintroduces the credential-leak bug the helper
exists to prevent, and regresses timestamps/copyright statuses
(502→400, 429→500). The `book_not_in_fileset` contract it adds
is passage-endpoint only (inventory item 1 — the passage plan's
concern). DECISION FOR THIS PLAN: implement the main-branch
error contract verbatim plus the branch's ADDITIVE fields
(`text_options`/`audio_options`/`codec`/`bitrate`, the
`chapter` int guard from `feat/apibible-niv`, the API.Bible
arms). Flag the regression upstream — do NOT port it.

## 3. Rust implementation plan

Module: `crates/server/src/routes/bible/discovery.rs` — three
handlers in one file (one endpoint group per §6 conventions).
DTOs beside the handlers. Register routes in
`routes/bible/mod.rs`:

```text
GET /api/v1/bible/translations/ → list_translations
GET /api/v1/bible/timestamps/   → get_timestamps
GET /api/v1/bible/copyright/    → get_copyright
```

### 3.1 Query DTOs — all fields `Option<String>`

`axum::extract::Query<T>` rejects missing declared fields with
a generic 400 — wrong body. Declare everything optional and
validate by hand to reproduce the exact `{"error": …}` bodies:

```rust
struct TranslationsQuery { language_iso: Option<String> }
struct TimestampsQuery {
    fileset_id: Option<String>, book: Option<String>,
    chapter: Option<String>,
}
struct CopyrightQuery { bible_id: Option<String> }
```

### 3.2 Response DTOs

```rust
struct TranslationDto {
    abbr: String, name: String, language: String,
    language_iso: String,
    filesets: Vec<FilesetDto>,
    text_options: Vec<OptionDto>,   // [] on main
    audio_options: Vec<OptionDto>,  // [] on main
}
struct FilesetDto {
    id: Option<String>, #[serde(rename="type")] kind: Option<String>,
    size: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    codec: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    bitrate: Option<String>,
}
struct OptionDto {
    id: String, kind: String,
    by_testament: ByTestament,
    coverage_label: String,
    partial: PartialMap,
}
// key order MUST be OT then NT (Python dict insertion order,
// TESTAMENTS tuple) — serde struct fields serialize in order:
struct ByTestament {
    #[serde(rename="OT", skip_serializing_if = "Option::is_none")]
    ot: Option<BTreeMap<String, String>>,
    #[serde(rename="NT", skip_serializing_if = "Option::is_none")]
    nt: Option<BTreeMap<String, String>>,
}
struct PartialMap { #[serde(rename="OT")] ot: bool,
                    #[serde(rename="NT")] nt: bool }
struct TimestampDto { verse_start: serde_json::Value,
                      timestamp: serde_json::Value }
struct CopyrightDto {
    id: Option<String>, #[serde(rename="type")] kind: Option<String>,
    size: Option<String>, copyright: String,
    copyright_date: String, copyright_description: String,
}
```

`TimestampDto` uses `Value` fields deliberately — Django
`.get()` passes through whatever type upstream sent and yields
`null` when absent; typing `i64`/`f64` would break parity on
unexpected upstream shapes. (Upstream in practice: int +
float.)

### 3.3 `list_translations` handler

1. `state.providers.translations(dbt, q.language_iso)` — port
   `get_live_translations` into a new
   `providers::listing::live_translations` async fn:
   - `dbt.get_bibles(limit=500, language_code=iso.to_uppercase())`
     → `providers::dbt` reqwest call; `Deserialize` only
     `data[].{abbr,name,language,iso,filesets}` where `filesets`
     is `HashMap<String, Vec<RawFileset>>` — iterate the MAP in
     serde_json order (preserve_order feature or `Vec<(String,
     Vec<…>)>`; `dbp` precedes `dbp-vid` upstream but ordering
     among surviving sources must match Python dict order —
     safest: parse into `serde_json::Map` and iterate its
     preserved order).
   - `_process_translations` port: same exclusion sets,
     `{id,type,size[,codec,bitrate]}` projection, drop empties.
   - DBT `Err` → `tracing::error!`, continue with `vec![]`.
   - Registry listings: `providers::sword::registry::
     translation_listing()` + `providers::esv::registry::
     translation_listing()` as `const`/fn statics (no client
     needed — Django's is on `SwordClient` but only reads the
     registry). [BRANCH] + `apibible::registry::
     translation_listing()`.
   - `language_iso` filter: `e.iso.eq_ignore_ascii_case(iso)`.
2. `providers::fileset_groups::group_filesets(&entry)` per
   entry — pure port of §2.4 into `providers/fileset_groups.rs`
   (new module; NOT `grouping.rs` — that is search grouping).
   Needs `is_sword_fileset`/`is_esv_fileset` (and, once
   `feat/apibible-niv` lands, decide `is_apibible_fileset`
   handling — §5).
3. Re-project and return `Json(json!({"results": …}))`.
   `ApiError::Provider(ProviderError)` for the outer catch —
   reproduce the KeyError→502 case as an internal error
   (`ProviderError::no_status(Kind::Internal)` → "Bible
   provider request failed ({kind})").

### 3.4 `get_timestamps` handler

1. Missing params → `ApiError::bad_request` emitting
   `{"error": "fileset_id, book, and chapter are required."}`.
2. [BRANCH apibible] `chapter.parse::<i64>()` < 1 → 400
   `"chapter must be a positive integer."` — adopt now; on
   main this diverges only for SWORD (`int()` crash → 502) and
   malformed DBT ids (upstream error); the 400 is strictly
   better and matches the merged future. FLAG in plan: if
   `feat/apibible-niv` has NOT merged by implementation time,
   decide whether to hold this parity delta; recommend adopting.
3. `core::books::get_dbt_book_id(book)` → `None` → 400
   `{"error": "Unknown book: {book}"}`.
4. `canonical_sword_fileset_id(fileset_id)` → `Some(canon)`:
   `voice = state.config.tts_config(canon).voice_name`
   (`get_tts_config` port — `KeyError` for non-LVSGLU8 stays an
   internal error path);
   `state.storage.read_timestamps_json(canon, book_id, ch,
   voice)` — `storage` crate, `google-cloud-storage` read of
   `audio/{canon}/{voice}/{book}/{ch}_timestamps.json`:
   - `NotFound` → 404 `{"error": "Timestamps not yet generated
     for this chapter."}`
   - other error → `ApiError::Provider` (§4.4; map the GCS
     error KIND, never `Display` — same leak rule)
   - `Ok(payload)` → `Json({"data": payload["data"] or []})`.
5. [BRANCH apibible] `is_apibible_fileset` →
   `apibible.get_chapter_audio(meta.audio_bible_id, book_id,
   ch)` → map `timecodes` via `_apibible_timecode` port;
   empty → the same 404 body.
6. else `dbt.get_timestamps(fileset_id, book_id, chapter)` →
   `GET /timestamps/{fs}/{book}/{ch}`; project `data[]` to
   `TimestampDto`.
7. Provider errors → `ApiError::Provider`.

Trait seam (§4.7): `PassageProvider::timestamps` exists, but
the SWORD arm is a GCS read and the ESV fallthrough to DBT is
a quirk — implement the dispatch inside the provider router
(`fileset_id → ProviderKind`), keeping `Esv` aliased to `Dbt`
for this method so the quirk is explicit, not dropped.

### 3.5 `get_copyright` handler

1. `bible_id` missing → 400 `{"error": "bible_id is
   required."}`.
2. [BRANCH apibible] `is_apibible_fileset` →
   `apibible.get_bible(meta.bible_id)` → single-element list;
   `copyright`/`copyright_description` =
   `copyrightStatement.unwrap_or("")`.
3. else `dbt.get_copyright(bible_id)` — upstream body is a
   BARE LIST: `Vec<RawCopyright>`; project each item,
   `copyright` sub-object absent → `""` defaults.
4. Errors → `ApiError::Provider`.

### 3.6 Client additions (`providers/dbt.rs`)

One shared `reqwest::Client` (keep-alive parity):
- `get_bibles(limit: u32, language_code: Option<&str>)` →
  `GET {base}/bibles`, params `v=4, key, limit,
  [language_code]`.
- `get_timestamps(fs, book, chapter)` →
  `GET {base}/timestamps/{fs}/{book}/{chapter}`.
- `get_copyright(bible_id)` →
  `GET {base}/bibles/{bible_id}/copyright` → bare `Vec`.
- [BRANCH apibible] `providers/apibible.rs`: `Client` with
  `api-key` header, `get_bible`, `get_chapter_audio`,
  `_apibible_timecode` port.

Error taxonomy: upstream `StatusCode` →
`ProviderError::Upstream(status)`; transport/JSON →
`ProviderError::Kind(ErrorKind::Reqwest|Deserialize|…)` —
`IntoResponse` maps per §4.4; the `?key=` param must never
enter `error` text (reqwest error `Display` embeds URLs —
forbidden).

### 3.7 Caching — none (parity)

No response caching anywhere on this surface today. The only
reuse is process-level client singletons → `Arc<DbtClient>` /
`reqwest::Client::new()` once in `AppState`. A TTL cache for
`/translations/` is tempting (~300 ms upstream) but is a
post-parity optimisation — record as non-goal.

## 4. Test plan + parity

Unit (pure logic, `cargo test`):

- `providers::fileset_groups`: port `test_fileset_groups.py`
  verbatim — the full matrix
  (C/NT/OT/NTP/OTP/NTPOTP × digit 1/2 × ±opus16), opus16/mp3
  sibling merge, drama/plain split, EN1WEB mixed prefixes,
  complete-beats-partial + non-P-beats-P precedence, NTPOTP
  both-partial, SWORD `generated` + ESV `api` singletons,
  odd-id singleton, `text_plain`-only merge (text_usx/format
  ignored), empty/absent-size skip.
- `core::books` lookups used here (already covered by the
  base agent's `test_bible_books.py` port).
- `_size_coverage`/`_split_codec`/`_member_rank` edge cases.

Handler tests (`axum-test`, `AUTO_PROVISION_USERS=false`,
explicit token or force-auth fixture — mirror conftest):

- `test_timestamp_view.py` equivalents: success projection
  (2 items), missing params 400, unknown book 400, DBT
  exception → 502 `provider_error` without `str(exc)`, 429 →
  `rate_limited`, `?key=` credential non-leak.
- `test_audio_timestamps_endpoint.py` equivalents: SWORD id
  (`GLU8`!) → GCS read path (storage trait fake returning a
  payload), GCS failure → 502 without URL/token leakage;
  NotFound → 404.
- `test_copyright_view.py` equivalents: success projection
  (nested `copyright` flattening), missing `bible_id` 400,
  502/429 mapping, credential non-leak.
- `test_dbt_client.py` timestamp/copyright URL+`v=4` assertions
  → wiremock request matching.
- `ESVTranslationsEndpointTests` +
  `test_translations_listing_includes_glucks` +
  `test_translations_response_includes_options` /
  `test_service_groups_dbt_filesets` /
  `test_process_translations_passes_codec_and_bitrate`
  (branch): registry entries present under `language_iso`
  filters, `results` projection shape, options attached, DBT
  outage → 200 with registry-only results.
- [BRANCH apibible] chapter-validation 400; `ENGNIVC1DA`
  timestamps arm (timecode coercion, empty → 404); copyright
  arm single-element body.

Integration: `wiremock` for `b4.dbt.io` (`/bibles`,
`/timestamps/*`, `/bibles/*/copyright`) and
`api.scripture.api.bible`; fake the `storage` read behind a
trait object so GCS never loads in tests.

Parity corpus (`tests/parity/`, `00-base.md` §4.15):

- `GET /api/v1/bible/translations/` (no param), `?language_iso=
  eng`, `?language_iso=lvs` — diff `results` verbatim incl.
  `text_options`/`audio_options` key order and `partial` maps.
- `GET /api/v1/bible/timestamps/?fileset_id=LVSGLU8&book=John
  &chapter=3` (GCS path — requires generated audio to exist;
  else expect the 404 body on both sides) and
  `?fileset_id=<DBT audio fs>&book=John&chapter=3`.
- `GET /api/v1/bible/copyright/?bible_id=ENGESV`; 400/404/502
  cases: missing params, `book=Zxqw`, upstream-404 fileset.
- Volatile normalization: none needed for these three (no
  timestamps/URLs in bodies) — direct diff.

## 5. Open questions / risks

1. **`feat/fileset-groups` regresses the error contract.** It
   deletes `provider_errors.py` and returns `{"error": str(e)}`
   at 400/500 — reintroducing the `?key=` credential-leak the
   helper prevents, and changing status codes (502→400 on
   timestamps/copyright failures, 429→500). RECOMMENDATION:
   implement main's contract + the branch's additive fields;
   raise the regression with the user before that branch
   merges. If it merges as-is, a fix-up commit restoring
   `provider_error_response` is required.
2. **Branch merge collision.** Both `feat/fileset-groups` and
   `feat/apibible-niv` rewrite the tail of
   `translation_service.py` and parts of `views.py`. If both
   merge, `ENGNIVC1DA` hits `group_filesets` rule 5 (`C1DA`
   digit) → kind `audio`, codec `mp3` — probably acceptable
   (the audio is real MP3 over API.Bible), but an
   `is_apibible_fileset` check → codec `api` (mirroring ESV)
   may be intended. Decision needed at merge; Rust should
   implement whatever the merged `group_filesets` does — if
   neither has merged, implement `is_apibible_fileset → api`
   as the safer forward-compatible default? NO — parity first:
   implement exactly the merged state, and record the choice.
3. **`chapter` typing drift.** Main passes the raw string to
   DBT timestamps; `feat/apibible-niv` validates `<1 → 400` and
   passes an int. The URL is identical (`/…/JHN/3`), so int is
   safe; adopt the branch's 400 guard unconditionally (§3.4).
4. **KeyError parity.** View maps `t['abbr']` etc. by direct
   indexing — a DBT entry missing `iso`/`abbr` 500s today
   (as a 502 `provider_error` via `provider_error_response`,
   `KeyError` kind). Reproduce as internal-error `Kind` or make
   fields optional and emit `null`? Prefer reproducing the
   failure — silent nulls would hide upstream schema drift.
5. **DBT `filesets` map ordering.** Python iterates the dict in
   JSON insertion order across sources; Rust must preserve the
   same member ordering inside the flattened `filesets` list —
   use `serde_json::Map` (preserve_order) rather than
   `HashMap`, or flatten `Vec<(String, Vec<RawFileset>)>`.
6. **SWORD timestamp voice coupling.** The GCS object path
   embeds `GOOGLE_TTS_VOICE_NAME` — artifacts generated by the
   job under a different voice name 404. Same on both sides;
   just keep `get_tts_config` as the single source.
7. **`book` accepts display names AND ids** via fuzzy
   (`JHN`→`john`). Any change to `core::books` semantics
   affects this endpoint — dependency on the base agent's
   port being verbatim (difflib `SequenceMatcher` parity).
8. **ESV quirk.** `ENGESV_API` timestamps route to DBT on main
   and would upstream-fail; `feat/apibible-niv` does not add
   an ESV arm either. Preserve — do not add one.
9. **`bitrate`/`codec` passthrough** is branch-only and
   optional upstream — DTO uses `Option` + skip so the field
   never appears when absent (matches branch behaviour where
   the keys are only added when DBT sends them).
