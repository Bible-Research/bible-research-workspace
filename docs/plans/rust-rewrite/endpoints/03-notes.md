# 03 — Notes CRUD (inventory #7)

Status: PLAN. Covers `NoteViewSet` standard CRUD ONLY:
`GET/POST /api/v1/notes/` and `GET/PUT/PATCH/DELETE
/api/v1/notes/{id}/`. The `reorder`, `bulk`, and `linked`
@action methods (views.py:453-747) are a SEPARATE agent's scope
— understood here only insofar as they share the serializer.

Sources:

- `annotations/views.py:170-451` — NoteViewSet (context,
  perform_create/update/destroy, `_apply_ordering`,
  get_queryset); `get_accessible_notes_qs` at :46-56.
- `annotations/serializers.py:28-45` —
  CurrentAuthenticatedUserDefault; :109-123
  NoteVerseReferenceSerializer; :138-410 NoteSerializer
  (create + to_representation enrichment/degradation).
- `annotations/models.py:78-169` — Note (`NOT` PK, tag FK,
  public, tag_position float, verses M2M through NoteVerse);
  :362-409 — NoteVerse (`NVE` PK, PROTECT verse FK).
- `bible/models.py:9-23` — Verse (`VER` PK; book, chapter,
  verse, dbt_book_id).
- `annotations/pagination.py` — NotesPagination (page 1,
  page_size 25, max 100).
- `bible/services/dbt/client.py:214-239` + generated
  `dbt_client/.../bibles_api.py:730-849` — `get_verses` →
  `GET /bibles/filesets/{fileset_id}/{book}/{chapter}`
  `?verse_start&verse_end&v=4&key=`.
- `bible/services/esv/client.py:32-128,176-` —
  `fetch_chapter_raw` + `_parse_passage`; registry
  `is_esv_fileset` (`esv/registry.py:20-21`) — only
  `ENGESV_API` (strip+upper) routes to ESV.
- `bible/utils/provider_errors.py:36-74` —
  `provider_error_fields` (the degradation contract).
- `bible/utils/bible_books.py:79-82` — BOOK_ORDER_MAP
  (title-cased name → 1-66) for `ordering=verse`.
- `annotations/tests.py:74-107` — `test_note_serializer`
  (@expectedFailure); :1151-1272 — NoteProviderErrorTest
  (the degradation contract tests); :943-973 — serialized
  note shape reference (LinkedNotes).

Auth: no `permission_classes` on the viewset → prod default
`IsAuthenticated` + device provisioning ⇒ every prod request
has a user (`tok` semantics, base §4.2). Test world (conftest)
= DRF defaults `AllowAny` ⇒ anonymous branches reachable —
they contain THREE quirks enumerated below. Use `Principal` on
reads; see §1.5/§5 for the write-side decision.

## 1. Endpoint contract

Base path `/api/v1/`. Note PKs: `NOT` + 15 uppercase hex
chars of a uuid4 (18 chars, `core::ids`). Path `{id}` is a
single `[^/.]+` segment.

### 1.0 Shared serialized-note shape (all 5 methods)

`NoteSerializer.Meta.fields` order on output:
`id, note_text, public, created_at, updated_at, tag,
tag_position` — `user` (HiddenField) and `verse_references`
(write_only) never appear. `to_representation` then appends
`verses`, `headings`, and on failure `error`, `error_code`
(in that key order: error fields merge in the except block
BEFORE verses/headings are assigned):

```json
{
  "id": "NOT1A2B3C4D5E6F7A",
  "note_text": "text",
  "public": false,
  "created_at": "2025-12-28T03:51:55.123456Z",
  "updated_at": "2025-12-28T03:51:55.123456Z",
  "tag": {"id": "TAG9F8E7D6C5B4A3F2", "name": "Faith",
          "parent_tag": null,
          "created_at": "...", "updated_at": "..."},
  "tag_position": 1767093456.789012,
  "verses": [
    {"book": "John", "chapter": 3, "verse": 16,
     "text": "For God so loved ..."}
  ],
  "headings": [
    {"before_verse": 16, "text": "For God So Loved the World"}
  ]
}
```

- `tag` is the NESTED `TagSerializer` object when set
  (`{id,name,parent_tag,created_at,updated_at}` — `user`
  hidden), `null` when `tag_id IS NULL`. Never a bare PK on
  output.
- `tag_position`: `f64` (float8). Default on create =
  `time.time()` (Unix seconds float).
- `verses`: sorted by `verse` ASC (Python sort on verse
  number only). Elements: `{book, chapter, verse, text}` —
  `book`/`chapter` are ALWAYS copied from the FIRST (lowest
  verse-number) linked verse; only `verse` varies per entry.
  `text` is `""` when the provider row lacks the verse or on
  degradation.
- `headings`: `[{before_verse:int, text:str}]` filtered to
  `before_verse ∈ note's verse numbers`; `[]` when none or on
  degradation/`include_verse_text=False`.
- **`verses` and `headings` keys are ABSENT ENTIRELY when the
  note links zero verses** (early return in
  to_representation, serializers.py:259-260).
- `error`/`error_code`: present only after a provider
  failure — merged via `provider_error_fields` (§4.4):
  `rate_limited` (upstream 429, "Bible provider rate limit
  exceeded (HTTP 429)"), `not_found` (PassageNotFoundError →
  "Passage not found"; upstream 404 → "Passage not found
  (HTTP 404)"), `provider_error` ("Bible provider error
  (HTTP {status})" or "Bible provider request failed
  ({ExceptionClassName})"). HTTP status stays 200/201 —
  degradation lives INSIDE the body. All verses keep their
  refs with `text:""`, `headings:[]`.
- Comment counts are NOT embedded — they come from
  `GET /comments/counts/` (comments plan, #15). No
  `mentions` logic exists anywhere.
- Timestamps RFC 3339 µs + `Z` (base §4.3).

### 1.1 `GET /api/v1/notes/` — list

Query params (all optional, `str` — parse manually, not
serde-typed, to reproduce edge behavior):

- `tag_id={TAG…}` — switches the base queryset to the
  ACCESSIBLE set (own ∪ public for auth; public-only for
  anon) then `AND tag_id = $x`. Empty string `tag_id=` is
  falsy → ignored entirely (falls to the no-tag branch).
- `public` — truthy iff `value.lower() == 'true'`. Auth:
  `true` → own ∪ public; anything else → own only. ANON
  (test world only): `Note.objects.filter(public=<bool>)` —
  **`?public=true` → all public notes; absent/other →
  `WHERE public = FALSE` → returns all PRIVATE notes to an
  anonymous caller.** Inverted-logic exposure bug; reproduce
  verbatim (unreachable in prod — middleware provisions) and
  flag in §5.
- `ordering` ∈ `verse|-verse|custom|-custom|created|
  -created`; unknown/absent → `custom`.
- `fileset_id` — serializer context only; see §1.6.
- `page` (default 1), `page_size` (default 25, max 100).

Response 200 — DRF envelope:

```json
{"count": 137, "next": "https://host/api/v1/notes/?page=2",
 "previous": null, "results": [ {<note>}, … ]}
```

`next`/`previous` = absolute URLs (scheme+Host, honor
`X-Forwarded-Proto`) preserving all other query params;
`null` at the ends. `page=last` → last page;
non-integer/0/negative/out-of-range page → **404
`{"detail": "Invalid page."}`**. `page_size` non-integer, 0,
or negative → silently falls back to 25; >100 → clamped 100.

### 1.2 `POST /api/v1/notes/` — create

Request body (all fields optional; DRF semantics):

```json
{
  "note_text": "...",            // blank ok; model default ''
  "public": false,               // default false
  "tag": "TAG9F8E7D6C5B4A3F2",   // PK or null
  "tag_position": 123.45,        // float; default time.time()
  "verse_references": [
    {"book": "John", "chapter": 3, "verse": 16}
  ]
}
```

- `tag` validated against the CALLER's tags (anon → guest
  user's tags): bad/out-of-scope PK → 400
  `{"tag": ["Invalid pk \"X\" - object does not exist."]}`.
  `null`/`absent` → NULL.
- `note_text` non-string → `{"note_text": ["Not a valid
  string."]}`; `null` → `["This field may not be null."]`.
- `public` non-bool → `["Must be a valid boolean."]`; `null`
  → `["This field may not be null."]`.
- `tag_position` non-numeric → `["A valid number is
  required."]` (numeric strings coerce — `float(data)`).
- `verse_references` items: `book` ≤50 chars
  (`["Ensure this field has no more than 50 characters."]`),
  `chapter`/`verse` ints (`["A valid integer is required."]`),
  missing keys → `["This field is required."]`; non-list →
  `["Expected a list of items but got type \"...\"."]`.
- Verse resolution is per-ref `Verse.objects.get(
  book__iexact, chapter, verse)` INSIDE `create()` — AFTER
  the note row is already inserted and NOT in a transaction.
  Missing verse → 400 with a **bare JSON array** body
  `["Verse not found for 'John 3:16'. Please ensure the verse
  exists in your database."]` AND the note row persists
  orphaned (no links). `MultipleObjectsReturned` → generic
  arm: `["Error processing verse reference: {exc}"]` —
  embeds `str(exc)` (one place where it is allowed; it's a
  Django ORM message, no secrets). Reproduce.
- `user`: auth → request.user. ANON quirk: `perform_create`
  logs a warning and NEVER saves → `serializer.data` runs
  `to_representation(validated_data-dict)` → AttributeError
  → **500** in test world. Rust plan: 401 via `AuthUser`
  (deviation — prod-identical; §5).
- `?fileset_id=` on the POST also flows to context.
- Response 201 = full serialized note — **the response
  serialization performs the provider call** (degrades to
  `error` fields, never 5xx on provider failure).
- `Location` header: DRF `get_success_headers` — absent (no
  `url` field).

### 1.3 `GET /api/v1/notes/{id}/` — retrieve

`get_queryset` sees `kwargs['pk']` → accessible qs (own ∪
public auth; public-only anon) + `id={id}`; `get_object`
re-filters pk. Invisible/nonexistent → 404
`{"detail": "Not found."}` (never 403 — a private note owned
by someone else is indistinguishable from missing).
Response 200 = serialized note; provider degradation applies.

### 1.4 `PUT/PATCH /api/v1/notes/{id}/` — update

Order of gates: `get_object` 404 → body validation 400 →
`perform_update` ownership 403. All serializer fields are
`required=False`, so PUT behaves like PATCH (`PUT {}` is a
no-op success).

- Ownership: `if user.is_authenticated and instance.user !=
  user` → 403 `{"detail": "You do not have permission to
  update this note."}`. Reachable only for a PUBLIC note
  owned by someone else (privates 404 first). **ANON quirk:
  `is_authenticated` is False → check skipped → anonymous
  PUT/PATCH/DELETE on any public note succeeds in test
  world.** Reproduce or gate — decision in §5.
- `user` rewrite quirk: HiddenField default re-populates
  `validated_data['user']` on update → `user_id` is rewritten
  to request.user (auth: no-op; anon: guest user or NULL).
- `verse_references` accepted but SILENTLY IGNORED on
  update (`setattr` on non-model field — no relinking). The
  only way to change verse links is delete+create.
- `tag` present+non-null → same caller-scoped PK validation;
  `null` → clears to NULL; absent → unchanged.
- `updated_at` bumps (auto_now); `created_at` immutable;
  `id` read-only.
- Response 200 = serialized note (provider call again).

### 1.5 `DELETE /api/v1/notes/{id}/` — destroy

404 gate → same `is_authenticated && owner-mismatch` → 403
`{"detail": "You do not have permission to delete this
note."}` → `instance.delete()` → 204 empty. ON DELETE
CASCADE removes `annotations_noteverse`, `annotations_comment`,
`annotations_image` ROWS — GCS image objects are orphaned
(never deleted here; flag, matching Django).

### 1.6 `fileset_id` enrichment semantics — READ CAREFULLY

`fileset_id` (query param, context default `'ENGESV'`) is a
BINARY switch, not true provider routing:

- `is_esv_fileset(fileset_id)` (strip+upper == `ENGESV_API`)
  → ESV path: `GET api.esv.org/v3/passage/text/
  ?q={BookName} {chapter}&include-headings=true&…`
  (`Authorization: Token $ESV_KEY`), whole chapter parsed,
  verses matched by `verse_start == verse.verse`, headings
  filtered to the note's verse numbers.
- EVERYTHING ELSE (incl. `LVSGLU8`, absent, garbage) → DBT
  path calling `get_verses(dbt_book_id, chapter,
  verse_start=first, verse_end=last)` — whose `bible_id`
  parameter **defaults to hardcoded `"ENGESV"` and the
  request's `fileset_id` is NEVER forwarded**. Upstream URL:
  `GET b4.dbt.io/api/bibles/filesets/ENGESV/{dbt_book_id}/
  {chapter}?verse_start={first}&verse_end={last}&v=4
  &key=$DBT_KEY`. Reproduce verbatim — notes always show
  ENGESV text unless `fileset_id=ENGESV_API`.
- Empty `parsed.verses` (ESV) or `data` rows lacking ANY
  `verse_text` key (DBT) → PassageNotFoundError →
  `error_code:"not_found"`, `error:"Passage not found"`.
- Enrichment uses ONLY `verses[0].dbt_book_id` /
  `.chapter` / `.book` — a note spanning books/chapters
  fetches just the lowest-verse's chapter; other-chapter
  verses get `text:""` silently. Reproduce.

## 2. Current Django behavior trace

### get_queryset (views.py:350-451)

```text
note_pk = kwargs['pk']            # detail routes only
tag_id  = ?tag_id
if note_pk or tag_id:
    qs = accessible = auth ? (user==U OR public) : public
else:
    pub = (?public).lower()=='true'
    qs = auth ? (pub ? (user==U OR public) : user==U)
              : Note.filter(public=pub)     # ← inverted bug
if tag_id: qs &= tag_id=…
if note_pk: qs &= id=…
_apply_ordering(qs, ?ordering or 'custom')
```

### Ordering SQL (views.py:281-348)

- `custom` (default/unknown): `ORDER BY tag_position ASC
  NULLS LAST, created_at DESC`.
- `-custom`: `tag_position DESC NULLS FIRST, created_at ASC`.
- `created` / `-created`: `created_at ASC` / `DESC`.
- `verse` / `-verse`: annotate `MIN(v.book) first_book`,
  `MIN(v.chapter) first_chapter`, `MIN(v.verse) first_verse`
  (three INDEPENDENT column mins, not one row), and
  `book_order = CASE MIN(v.book) WHEN 'Genesis' THEN 1 …
  ELSE 999 END` (66-entry BOOK_ORDER_MAP on title-cased
  names, e.g. 'John'→32); order by `book_order,
  first_chapter, first_verse` (DESC each for `-verse`).
  Verse-less notes → NULLs → `book_order=999` → LAST under
  `verse`, FIRST under `-verse`. LEFT JOINs; Postgres GROUP
  BY `n.id` only (functional-dep optimization). Paginator
  count on the grouped qs ⇒ `COUNT(*)` over subquery —
  write `COUNT(DISTINCT n.id)`.

### Query counts

- **list**: 1 COUNT + 1 page SELECT (LIMIT/OFFSET) + **N×(
  `SELECT v.* FROM bible_verse v JOIN annotations_noteverse
  nv … WHERE nv.note_id=…`)** — no prefetch ⇒ per-note verse
  query + per-note tag lazy load (`SELECT … FROM
  annotations_tag WHERE id=…` when tag_id non-null) + **1
  upstream provider HTTP call per note**. A 100-row page =
  ~202 DB queries + 100 blocking sequential provider calls.
- **retrieve**: 1 note + 1 verses + ≤1 tag + 1 provider.
- **create**: tag check (1, caller-scoped SELECT) → INSERT
  note → per-ref Verse SELECT (`book__iexact` →
  `UPPER(book)=UPPER($1)` on PG) → one multi-VALUES
  NoteVerse INSERT (`bulk_create`) → serialize (verses + tag
  + provider).
- **update**: get_object SELECT → UPDATE (writes user_id,
  tag_id, note_text, public, tag_position, updated_at=now())
  → serialize (verses + tag + provider).
- **delete**: get_object SELECT → DELETE (cascade).

## 3. Rust implementation plan

Module `crates/server/src/routes/annot/notes.rs`; DTOs beside
handler per §6.

### Routes (`routes/mod.rs`, under `/api/v1`)

```text
.route("/notes",      get(list_notes).post(create_note))
.route("/notes/{id}", get(get_note).put(put_note)
                      .patch(patch_note).delete(delete_note))
```

`notes/bulk`, `notes/linked`, `notes/reorder` are registered
by the actions agent — axum resolves static segments before
`{id}` regardless, but keep registration order explicit.

### DTOs

```rust
struct NotesListQuery {            // all Option<String>
    tag_id: Option<String>, public: Option<String>,
    ordering: Option<String>, page: Option<String>,
    page_size: Option<String>, fileset_id: Option<String>,
}
struct VerseRefReq { book: String, chapter: i32, verse: i32 }
struct NoteWriteReq {              // POST + PUT/PATCH shared
    note_text: Option<String>,     // null → 400 "may not be null"
    public: Option<bool>,
    tag: Option<Option<String>>,   // absent vs explicit null
    tag_position: Option<f64>,
    verse_references: Option<Vec<VerseRefReq>>, // ignored on update
}
struct TagDto { id: String, name: String,
    parent_tag: Option<String>,
    created_at: DateTime<Utc>, updated_at: DateTime<Utc> }
struct NoteVerseDto { book: String, chapter: i32,
    verse: i32, text: String }
struct NoteHeadingDto { before_verse: i32, text: String }
struct NoteDto {                   // field order = §1.0
    id: String, note_text: String, public: bool,
    created_at: DateTime<Utc>, updated_at: DateTime<Utc>,
    tag: Option<TagDto>,           // null or nested; never PK
    tag_position: f64,
    #[serde(skip_serializing_if = "Option::is_none")]
    verses: Option<Vec<NoteVerseDto>>,      // None = key absent
    #[serde(skip_serializing_if = "Option::is_none")]
    headings: Option<Vec<NoteHeadingDto>>,
    #[serde(skip_serializing_if = "Option::is_none")]
    error: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    error_code: Option<String>,
}
```

Serializer helper `serialize_note(row, verses, tag, ctx:
{fileset_id, include_verse_text}) -> NoteDto` lives in
`routes::annot::notes` and is EXPORTED — the linked/bulk
agent reuses it with `include_verse_text=false` (context flag
exists in Django for exactly this).

### Shared fetch helpers

```text
load_verses(pool, note_ids) -> Map<note_id, Vec<VerseRow>>
  SELECT v.id, v.book, v.chapter, v.verse, v.dbt_book_id,
         nv.note_id
  FROM bible_verse v
  JOIN annotations_noteverse nv ON nv.verse_id = v.id
  WHERE nv.note_id = ANY($1)
  -- group by note_id in Rust; sort each group by verse ASC
load_tags(pool, tag_ids) -> Map<tag_id, TagRow>
  SELECT id, name, parent_tag_id, created_at, updated_at
  FROM annotations_tag WHERE id = ANY($1)
```

This replaces Django's 2 per-note queries with 2 `ANY`
queries — allowed per §6 "same count discipline" (Django's
N+1 here is incidental, not contractual; contract is the
response shape). Note count stays observably identical.

### Provider enrichment (the core piece)

`enrich_note(verses_sorted, fileset_id) -> (verse_texts,
headings) or degrades in-place`:

```text
if verses empty -> verses=None, headings=None   # keys absent
first = verses[0]                                # lowest verse
book=first.book, chapter=first.chapter,
dbt=first.dbt_book_id
if !include_verse_text -> text="" per verse, headings=[]
elif is_esv_fileset(fileset_id):                 # ENGESV_API
    ch = esv.chapter_text(dbt, chapter, "ENGESV_API")
    # needs verses+headings from whole chapter
else:
    ch = dbt.chapter_verses(dbt, chapter, "ENGESV",   # HARDcoded
        verse_start=first.verse, verse_end=last.verse)
    # DBT must receive the range params — they affect the
    # empty-data → not_found check (see §4.7 note below)
match ch:
  Ok(c) if c has ≥1 row carrying text ->
      per note-verse: text = row where
      row.verse_start == verse.verse else ""
      headings = c.headings.filter(before_verse ∈ verse_nums)
  Ok(empty/all-textless) -> ProviderError::NotFound(None)
  Err(e) -> e
on any Err: rep.error/rep.error_code = e.error_fields();
      verses kept, text=""; headings=[]
```

**Trait note (coordinate with passage agent #1):** the
`PassageProvider::chapter_text` signature in §4.7 has no
verse range. The DBT arm MUST send `verse_start`/`verse_end`
upstream (a populated chapter whose requested range is empty
yields `data:[]` → `not_found` in Django; client-side
filtering of a full chapter would instead return rows → all
`text:""` with NO error — different contract). Options:
add optional `(verse_start, verse_end)` to `chapter_text`,
or a `chapter_verses_range` method. The ESV arm ignores the
range (whole chapter parsed; handler filters). The note
never calls `resolve(fileset_id)` for the DBT arm — it is
ALWAYS DBT/`ENGESV`; only `is_esv_fileset` consults the id.
SWORD filesets silently take the DBT+ENGESV path — parity.

`ProviderError::error_fields()` differences from Python:
the no-status arm embeds the Rust error-kind enum name
(e.g. `(Timeout)`) where Django embeds the exception class
name (`(ConnectTimeout)`). Documented deviation per §4.4 —
never `err.to_string()` (URLs carry `?key=`).

### Handlers

`list_notes(State, Principal, Query(q))`:

1. Resolve WHERE arm exactly per §2 decision tree:
   `tag_id` non-empty or detail → accessible qs; else auth/
   `public` flag logic incl. the anon `public=<bool>` bug.
   `Principal.anonymous` → `via == Anonymous`.
2. Ordering: build ORDER BY per the table; for `verse` arms
   emit the LEFT JOIN + `MIN(...)` subselects + 66-arm CASE
   (generate from `core::books::BOOK_ORDER_MAP`) in ORDER BY
   — or subquery `SELECT … , (SELECT MIN(b.book) …) AS
   first_book …` then `CASE first_book …`. GROUP BY n.id.
3. `page`/`page_size` per §1.1 (manual parse: `parse::<u32>`
   fail → 404 `{"detail": "Invalid page."}`; `page=="last"`
   → num_pages; `page_size` invalid/<1 → 25, clamp 100).
   COUNT (DISTINCT for joined arms) → page SELECT.
4. `load_verses` + `load_tags` (2 queries for the page).
5. Serialize each note → `enrich_note` per item → **N
   sequential upstream calls in Django**. Rust: use
   `futures::stream::iter(items).buffered(8)` — order-
   preserving bounded concurrency; still N calls. Rate-limit
   risk documented in §5 — keep parity (no caching) for v1.
6. Envelope `{count,next,previous,results}`; build links
   from Host + `X-Forwarded-Proto`, preserving raw query
   pairs, replacing `page`.

`create_note(State, AuthUser-or-Principal, Query(q), Json)`:

- Validate in field order (DRF iterates fields in Meta
  order: note_text, public, tag, tag_position,
  verse_references — first-error field map; collect ALL
  field errors like DRF before returning 400).
- `tag` → `SELECT 1 FROM annotations_tag WHERE id=$1 AND
  user_id=$2` (caller or guest) → miss → 400 pk message.
- INSERT with `core::ids::note_id()`, `Utc::now()` for both
  timestamps, `tag_position` = provided or
  `SystemTime::now()` float secs.
- Per verse_ref IN ORDER: `SELECT id FROM bible_verse WHERE
  UPPER(book)=UPPER($1) AND chapter=$2 AND verse=$3` → miss
  → return 400 `["Verse not found for '{book} {ch}:{v}'.
  Please ensure the verse exists in your database."]` —
  **note already committed: NO transaction, parity orphan**.
  DB-level `ErrorKind`≠NotFound (e.g. multiple rows) →
  `["Error processing verse reference: {msg}"]` (PgError msg
  — verify no sensitive content; Django embeds ORM text).
- Then `INSERT INTO annotations_noteverse (id,note_id,
  verse_id,created_at) VALUES …` multi-row.
- Serialize (with provider call) → 201.
- Anonymous: see §5 — recommended `AuthUser` → 401.

`get_note`: `SELECT n.* FROM annotations_note n WHERE
(accessible) AND n.id=$1` → 404 → serialize → 200.

`put_note`/`patch_note` (share `update_note`, `partial` flag
is meaningless since all fields optional — same handler):

- get_object (404) → body validation (400) →
  `if user.is_authenticated && note.user_id != Some(user.id)`
  → 403 detail string. Preserve the anon-skip quirk or gate
  — decision §5.
- UPDATE: `user_id` ← request user (guest/NULL if anon —
  quirk), `tag_id` if key present (null clears), `note_text`,
  `public`, `tag_position` if present, `updated_at=now()`.
  `verse_references` parsed (so invalid items still 400 —
  DRF validates write-only fields!) then dropped.
- Serialize → 200.

`delete_note`: get_object → same ownership conditional →
`DELETE FROM annotations_note WHERE id=$1` → 204.

### Validation/error plumbing

- Field-map 400s via `ApiError::Validation(Map)` (§4.4).
- The bare-array 400 (`["Verse not found for …"]`) needs the
  same `ApiError::ValidationList(Vec<String>)` variant the
  comments plan requested — coordinate so both land in
  `core::errors` once.
- `Principal` anonymous + writes: see §5 decision.
- HEAD/OPTIONS free via axum; OPTIONS body parity
  best-effort (§4.13).

## 4. Test plan

Mirror `annotations/tests.py` → `tests/annot_notes.rs`
(`AUTO_PROVISION_USERS=false`, seeded users/tokens/verses;
`bible_verse` rows seeded in SQL since there is no verse
CRUD).

Unit (pure):

- `serialize_note`: no-verse note → `verses`/`headings` keys
  ABSENT; multi-verse sort; `text=""` fill; headings filter
  (`before_verse` ∈ note verses only); error merge order.
- Ordering SQL: 6 variants incl. unknown → custom;
  `verse`/`±verse` emit book_order CASE w/ default 999.
- Param parsing: `public` case variants; `tag_id=""` ignored;
  `fileset_id=""` → default; page `last`/`abc`/`0` → 404 or
  resolved; page_size `0`/`abc`/`101` → 25/25/100.
- `core::ids::note_id` → `NOT` prefix, len 18, upper-hex.

HTTP (axum-test + scratch Postgres + `wiremock` DBT/ESV):

- CRUD happy: POST 201 (`NOT` id, `user` absent from body,
  tag nested, verse linked, `text` filled from wiremock);
  GET detail 200; list envelope; PATCH single field; PUT
  full; DELETE 204 + row gone.
- Visibility matrix: owner sees own private; other-user
  private → 404 detail (not 403); other-user PUBLIC →
  visible in detail, PUT→403, DELETE→403; anon+`public=true`
  → public only; **anon+no param → private notes exposed**
  (bug parity — assert it, flag loudly);
  `?tag_id=` → accessible-scoped filter.
- Ordering: seed 3+ notes w/ different tag_position/created/
  verses; assert each ordering incl. verse-less notes' 999
  placement both directions.
- Pagination: 2 pages, `next`/`previous` URLs carry
  `ordering`/`tag_id`/`page_size`; `page=last`;
  `page=99`/`abc`/`0` → 404 `{"detail": "Invalid page."}`;
  `page_size=0` → 25 rows.
- Create validation: bad tag pk 400 exact message; tag of
  another user 400; `verse_references` missing verse → 400
  bare-array + **assert note row persisted** (orphan
  parity); invalid chapter type → field error;
  `note_text:null` → 400; `tag:null` create → NULL ok.
- Update: `verse_references` ignored on PATCH (links
  unchanged, 200); `tag:null` clears; `user_id` rewrite
  no-op for owner; `updated_at` bumps.
- Provider degradation (mirror NoteProviderErrorTest):
  wiremock 429 → `error_code:"rate_limited"`, `text:""`,
  HTTP 200; 503 → `provider_error` "…(HTTP 503)"; 404 →
  `not_found` "Passage not found (HTTP 404)"; `data:[]` →
  `not_found` "Passage not found" (no suffix); rows without
  `verse_text` → `not_found`; ESV empty verses → `not_found`;
  ESV happy → headings filtered + text matched by
  verse_start; **`?fileset_id=LVSGLU8` still hits the DBT
  URL with fileset ENGESV** — assert wiremock path/query.
- `test_note_serializer` equivalent: authenticated create
  via serializer path succeeds and links verses (the Django
  version is `@expectedFailure` — ours must PASS; document
  that Django's failure is a test-harness auth artifact).
- List N+1 guard: instrument sqlx — assert verse/tag queries
  are the 2 `ANY` queries (not per-note); upstream call
  count == page size.
- Parity-corpus entries: list with `ordering=verse` +
  `tag_id`; detail degraded (wiremock → real diff vs Django
  needs provider stubbed identically); create with verses;
  update 403 on other-user public note.

## 5. Open questions / risks

- **Anonymous-write parity (DECIDE)**: three test-world
  quirks — (a) anon POST → Django skips save → 500; (b) anon
  PUT/PATCH/DELETE on public notes → ownership check skipped
  → succeeds + owner rewritten to guest; (c) anon PUT sets
  `user_id` to guest/NULL. Recommended: `AuthUser` on POST
  (401 — prod-identical, deviates only from the crash); for
  PUT/PATCH/DELETE keep `Principal` + mirror the
  `is_authenticated && owner-mismatch` conditional exactly so
  the parity corpus matches, flagging it as a live security
  bug to fix post-migration. Alternative: `AuthUser` on all
  writes (safer, but drifts from test-world Django on anon
  writes). Confirm with base agent.
- **Anon list exposure bug** (`?public` absent → private
  notes): reproduce-for-parity like comments' missing
  checks; file follow-up to invert the default.
- **N+1 upstream calls**: page of 100 notes = 100 sequential
  ~300ms DBT calls in Django (potential 30s list). `buffered
  (8)` cuts wall time; calls still N — keep parity, evaluate
  per-(fileset,book,chapter) memoization within a single
  request later (adjacent notes often share a chapter —
  Django does NOT dedupe, so this is a deviation; measure
  first).
- **Trait signature**: needs DBT verse-range passthrough
  (§3) — coordinate with passage plan agent; until `dbt.rs`
  lands (Phase 3) the notes agent can ship enrichment behind
  a `FakeProvider` per §5 Phase 2.
- **`ErrorKind` naming drift** in `provider_error` free-text
  vs Python exception class names — parity diff must
  normalize or accept; never revert to `err.to_string()`.
- **Cross-book/chapter notes**: enrichment uses first verse
  only — verbatim, but worth a follow-up (provider could
  fetch each distinct chapter).
- **Orphan note on 400**: create() commits before verse
  resolution fails — reproduce; consider a txn in a future
  contract-breaking pass.
- **Image GCS orphans on note DELETE**: cascade removes
  image rows; blobs leak — same as Django today.
- **DBT `headings` key**: rarely present upstream; keep the
  `get('headings')` conditional shape verbatim.
- **`Verse` table is pre-seeded data** (no CRUD) — scratch
  DB fixture must seed `bible_verse` rows incl.
  `dbt_book_id`.
