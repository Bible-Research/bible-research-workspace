# 04 — Note custom actions (reorder / bulk / linked)

Inventory items 8, 9, 10 — `NoteViewSet` `@action` methods.
Django source: `annotations/views.py:453-476` (`bulk_create_notes`),
`views.py:478-565` (`linked`), `views.py:567-747` (`reorder`),
`annotations/serializers.py:109-135` (`NoteVerseReferenceSerializer`,
`LinkedNotesRequestSerializer`), `serializers.py:138-410`
(`NoteSerializer`), `serializers.py:413-565` (`BulkNoteItemSerializer`,
`BulkNoteCreateSerializer`), `annotations/models.py:82-169` (`Note`),
`models.py:366-409` (`NoteVerse`), `bible/models.py:9-23` (`Verse`),
`annotations/tests.py:676-1112` (`BulkNoteCreateTest`,
`LinkedNotesTest`, `TestPartialReordering`).

Status: PLAN. Depends on Phase 0 (`core::auth` `Principal`,
`core::errors`, `core::ids`) and shares `NoteDto` / `TagDto` /
`VerseRef` serialization with the notes-CRUD plan (item 7) — see
§5, "Shared DTO dependency".

## Auth note (applies to all three actions)

`NoteViewSet.permission_classes` is commented out
(`views.py:187`), so DRF's default applies: prod =
`IsAuthenticated` (`bible_research/settings.py:355-357`), tests =
`AllowAny` (`conftest.py` defines no `REST_FRAMEWORK`). Combined
with the device middleware, prod requests are virtually always
authenticated. Per base §4.2: use `Principal`, NOT `AuthUser`,
for all three handlers — each action has its own anonymous
branch that must remain reachable under
`AUTO_PROVISION_USERS=false`:

- `linked` explicitly returns `200 {"count": 0, "results": []}`
  for anonymous (`views.py:529-535`) — an `AuthUser` extractor
  would wrongly emit 401.
- `bulk` scopes the `tag_id` queryset to the `guest` user and
  saves notes as `guest` (or `user=NULL` if guest is missing)
  (`serializers.py:453-464`, `525-531`).
- `reorder` filters `user=request.user`; for a Django
  `AnonymousUser` this raises `TypeError`
  (`int(AnonymousUser())` inside the FK `get_prep_lookup` —
  confirmed against `related_lookups.py`) → unhandled 500.
  Deviation decision in §5.

## 1. Endpoint contracts

### 1.1 `POST /api/v1/notes/reorder/`

Request (verbatim example from the docstring, `views.py:574-579`):

```json
{
  "tag_id": "TAG123",
  "updates": [
    {"note_id": "NOT456", "position": 50.0},
    {"note_id": "NOT789", "position": 51.0}
  ]
}
```

Success: **204 No Content**, empty body (`views.py:712`).

Errors — every 400 uses the hand-rolled shape
`{"error": "validation_error", "message": "<msg>"}` (NOT a DRF
field map). Validation order is observable — replicate exactly:

1. `tag_id` missing/falsy (`""`, `0`, `false`, `[]`, `{}`,
   `null`) → `tag_id is required.`
2. `updates` missing, not a list, or an empty list →
   `updates array is required.`
3. `len(updates) > 100` → `Cannot reorder more than 100 notes
   at once. Received {n} notes.`
4. `note_id` falsy (missing/`""`/`0`/`null`) OR
   `position is None` (missing or `null`) → `Update at index
   {i} is missing note_id or position.`
5. `note_id` already seen → `Duplicate note_id in request:
   {note_id}`
6. `float(position)` fails (`"abc"`, list, dict) → NO catch →
   Django **500** (see §5)
7. duplicate positions after `float()` → `Duplicate positions
   in request.`
8. locked-row count != `len(note_ids)` → `Notes not found or
   do not belong to tag {tag_id}: {missing}` — `{missing}` is
   a Python set repr, e.g. `{'NOT1', 'NOT2'}` (order unstable)

Steps 4-6 iterate `updates` in order, so the FIRST bad index is
named. `position: 0` and `position: false` are VALID (float() →
`0.0`); `note_id: 0`/`""` are invalid. A non-dict `updates[i]`
(e.g. `5`, `"x"`) → `update.get` → `AttributeError` → 500.

Conflict (inside `except IntegrityError`, `views.py:714-747`):

- **409** when the constraint name appears in the error text:

```json
{
  "error": "position_conflict",
  "message": "<see below>",
  "constraint": "unique_tag_position_per_tag"
}
```

`message` verbatim (single JSON string):

> Position conflict: one or more positions are already
> occupied by other notes in this tag. Please refresh and try
> again.

- **409** any other `IntegrityError`:

```json
{
  "error": "database_error",
  "message": "A database constraint was violated. Please try again."
}
```

Note: the code checks
`'unique_tag_position_per_tag' in str(e).lower()` — on Postgres
the constraint name is always in the error message; in Rust use
`db_err.constraint() == Some("unique_tag_position_per_tag")`.

### 1.2 `POST /api/v1/notes/bulk/`

Request (`BulkNoteCreateSerializer`, `serializers.py:431-565`):

```json
{
  "tag_id": "TAG123",
  "notes": [
    {
      "note_text": "First note",
      "public": false,
      "verse_references": [
        {"book": "John", "chapter": 3, "verse": 16}
      ]
    }
  ]
}
```

All three `notes[]` item fields are optional; defaults:
`note_text=""`, `public=false`, `verse_references=[]`
(`serializers.py:413-428`). `book` = CharField max 50;
`chapter`/`verse` = DRF IntegerField coercion — `int(re.sub(
r'\.0*\s*$', '', str(v)))`: `3`, `"3"`, `3.0`, `"3.0"` → 3;
`3.7`, `"x"`, `true` → 400 `"A valid integer is required."`.

Success: **201 Created**, body = **bare JSON array** of
`NoteSerializer` outputs in request order (NOT `{count,results}`):

```json
[
  {
    "id": "NOT…",
    "note_text": "First note",
    "public": false,
    "created_at": "2025-12-28T03:51:55.123456Z",
    "updated_at": "2025-12-28T03:51:55.123456Z",
    "tag": "TAG123",
    "tag_position": 1748512345.678912
  }
]
```

**Polymorphic `tag` quirk** (`NoteSerializer.to_representation`,
`serializers.py:252-410`): when a note has NO verses the
serializer early-returns before the tag-expansion block, so
`"tag"` stays a **PK string** and the `verses`/`headings` keys
are ABSENT. When it has ≥1 verse: `"tag"` becomes the nested
object `{"id","name","parent_tag","created_at","updated_at"}`,
plus `"verses": [{"book","chapter","verse","text"}]` and
`"headings"`. Bulk responses with verse refs trigger ONE
provider enrichment call per note (`include_verse_text` is NOT
disabled here — unlike `linked`); on provider failure
`error`/`error_code` are merged into the note object and HTTP
stays 201 (`serializers.py:388-402`).

Errors (400, DRF field-map shape):

```json
{"tag_id": ["This field is required."]}
{"tag_id": ["This field may not be null."]}
{"tag_id": ["Invalid pk \"<v>\" - object does not exist."]}
{"tag_id": ["Incorrect type. Expected pk value, received list."]}
{"notes": ["This field is required."]}
{"notes": ["At least one note is required."]}
{"notes": {"0": {"note_text": ["Not a valid string."]}}}
{"non_field_errors": [
  "Expected a list of items but got type \"dict\"."]}
{"non_field_errors": ["<verses-not-found message, below>"]}
```

`non_field_errors[0]` verbatim (single JSON string):

> Verses not found: john 99:99. Please ensure the verses
> exist in your database.

- `Invalid pk` fires for unknown AND for another user's tag
  (queryset is user-scoped, `serializers.py:445-464`).
- `Verses not found:` (raised inside `create()` →
  `non_field_errors`): the book is echoed LOWERCASED
  (key = `ref['book'].lower()`), entries sorted by
  `(book_lower, chapter, verse)`, comma-joined as `b c:v`
  (`serializers.py:503-513`).
- Atomicity: a bad verse / DB failure rolls back ALL notes —
  the view wraps `serializer.save()` in `transaction.atomic()`
  (`views.py:466-467`).

Anonymous: `tag_id` validated against guest's tags; notes owned
by `guest` (or `user_id NULL` if no guest row).

### 1.3 `POST /api/v1/notes/linked/`

Request (`LinkedNotesRequestSerializer`, `serializers.py:125-135`):

```json
{"verse_references": [{"book": "John", "chapter": 3, "verse": 16}]}
```

- `verse_references`: ListField, `min_length=1`, `max_length=500`,
  child = `{book: str ≤50, chapter: int, verse: int}` (same int
  coercion as §1.2).
- `?fileset_id=` query param is accepted for parity but ignored
  (schema doc, `views.py:483-489`).

Success: **200**, custom envelope:

```json
{
  "count": 1,
  "results": [
    {
      "id": "NOT…",
      "note_text": "On John 3:16",
      "public": false,
      "created_at": "…Z",
      "updated_at": "…Z",
      "tag": {"id": "TAG…", "name": "LinkedTag",
              "parent_tag": null,
              "created_at": "…Z", "updated_at": "…Z"},
      "tag_position": 50.0,
      "verses": [
        {"book": "John", "chapter": 3, "verse": 16, "text": ""}
      ],
      "headings": []
    }
  ]
}
```

- `count` == `len(results)`; ordering `-created_at`; a note
  matching multiple refs appears ONCE (`distinct()`).
- `verses[].text` is ALWAYS `""` and `headings` always `[]` —
  the view sets `context['include_verse_text'] = False`, so ZERO
  provider calls (`views.py:553-557`).
- Anonymous → `200 {"count": 0, "results": []}` (never 401).
- Results only include the requester's OWN notes
  (`filter(user=user)`), regardless of `public` flags on others'.

Errors (400, DRF field-map shape):

```json
{"verse_references": ["This field is required."]}
{"verse_references": ["Ensure this field has at least 1 elements."]}
{"verse_references": ["Ensure this field has no more than 500 elements."]}
{"verse_references": ["Expected a list of items but got type \"str\"."]}
{"verse_references": {"0": {"book": ["This field is required."]}}}
{"verse_references": {"0": {"chapter": ["A valid integer is required."]}}}
```

Child errors are index-keyed dicts (`run_child_validation`,
`fields.py:1714-1726`); a note-level `non_field_errors` key can
also appear inside a child dict.

## 2. Current Django behavior trace

### 2.1 `reorder` (`views.py:567-747`)

1. Hand validation steps 1-7 (§1.1 numbered list) — all before
   the DB. `position_map[note_id] = float(position)` coerces
   Python-side; `1`/`1.0`/`true`/`"3.5"` all → valid f64.
2. `transaction.atomic()`:
   a. Lock + scope query (`views.py:675-681`):
      `SELECT … FROM annotations_note WHERE id IN (…)
       AND user_id = %s AND tag_id = %s FOR UPDATE`
      (`select_for_update()`; on Postgres real row locks, on the
      SQLite test DB a no-op). No ORDER BY — lock order is the
      scan order.
   b. `len(notes) != len(note_ids)` → 400 with the
      `{'NOT…', …}` set repr of `set(note_ids) - found`.
   c. Phase 1 (`views.py:701-704`): assign
      `tag_position = -(idx+1)` over the fetched list, then
      `bulk_update` → ONE statement:
      `UPDATE annotations_note SET tag_position =
       CASE id WHEN %s THEN %s … END WHERE id IN (…)`
   d. Phase 2 (`views.py:706-710`): same CASE UPDATE with
      `position_map[note_id]` finals.
   e. Commit → 204.
3. `IntegrityError` → rollback → branch on constraint name in
   message → the two 409 bodies above.

Collision mechanics: the partial unique index
`unique_tag_position_per_tag` on `(tag_id, tag_position)
WHERE tag_id IS NOT NULL` (`models.py:152-158`, migration
`annotations/migrations/0012`) means a requested position that
is already taken by a note OUTSIDE the update set fails only at
phase-2 flush. The two-phase swap inside the set always works
because temps `-1..-n` cannot collide with real positions.
`test_position_already_occupied` exercises exactly this → 409.

Lock/edge notes: all-or-nothing ownership check — one foreign
or missing id fails the whole request at step (b). Rows in the
update set are locked `FOR UPDATE` until commit; two concurrent
reorders on the same tag serialize on the FIRST lock waiter,
then the second may hit the unique violation → 409 (matches
the "refresh and try again" message).

### 2.2 `bulk_create_notes` (`views.py:453-476` + serializer)

1. `is_valid(raise_exception=True)` — field validation incl.
   `tag_id` `get()` on the user-scoped queryset and
   `validate_notes` non-empty.
2. Inside `transaction.atomic()` (`views.py:466`):
   a. Resolve requester: `request.user` if authenticated else
      `guest` user else `None` (`serializers.py:525-531`).
   b. `_resolve_verses` (`serializers.py:473-514`): dedupe
      `(book.lower(), chapter, verse)` across all items → ONE
      OR-chain query:
      `SELECT id, book, chapter, verse, dbt_book_id
       FROM bible_verse WHERE (UPPER(book)=UPPER(%s) AND
       chapter=%s AND verse=%s) OR …`
      (`book__iexact` → `UPPER()` both sides on Postgres).
      Any missing key → `ValidationError` → 400
      `non_field_errors`, transaction aborts.
   c. `Note.objects.bulk_create` → ONE multi-row INSERT with
      app-generated `NOT`+15-hex ids, `tag_position =
      time.time()` per row (sequential calls — distinct),
      `created_at`/`updated_at` set (Django ≥4.1 sets
      auto_now* on bulk_create).
   d. `NoteVerse.objects.bulk_create` → ONE multi-row INSERT,
      `NVE` ids, `verse_id` from the map.
3. Serialize `note_instances` in request order → 201.
   `NoteSerializer` re-reads `instance.verses` (fresh prefetch-
   less query per note with verses) and calls the provider per
   verse-bearing note — N upstream calls possible.

### 2.3 `linked` (`views.py:478-565`)

1. Validate request; anonymous → `200 {count:0,results:[]}`.
2. Query 1 (the only notes query):
   `SELECT DISTINCT n.*, t.* FROM annotations_note n
    LEFT JOIN annotations_tag t ON n.tag_id = t.id
    INNER JOIN annotations_noteverse nv ON nv.note_id = n.id
    INNER JOIN bible_verse v ON v.id = nv.verse_id
    WHERE n.user_id = %s AND (
      (UPPER(v.book)=UPPER(%s) AND v.chapter=%s AND v.verse=%s)
      OR … )
    ORDER BY n.created_at DESC`
   — all OR'd refs share ONE join alias (Django reuses the join
   within a single `filter()`), so `DISTINCT` is required for
   dedupe; `select_related('tag')` adds the tag columns
   (LEFT JOIN — `tag_id` is nullable).
3. Query 2 (`prefetch_related('verses')`):
   `SELECT v.*, nv.note_id FROM bible_verse v
    INNER JOIN annotations_noteverse nv ON nv.verse_id = v.id
    WHERE nv.note_id IN (…)`
4. `NoteSerializer` per note with `include_verse_text=False`:
   `verses` = Python-sorted by `v.verse`, each
   `{book, chapter, verse, text: ""}` (book/chapter taken from
   the FIRST sorted verse's row), `headings: []`, `tag`
   expanded via `TagSerializer`. Asserted exactly 2 queries in
   `test_no_upstream_calls_and_no_n_plus_1`.

## 3. Rust implementation plan

File: `crates/server/src/routes/annot/note_actions.rs`
(new module; notes CRUD module belongs to the item-7 plan —
keep these handlers in their own file to avoid merge churn).

### Routes (`routes/annot/mod.rs` or `routes/mod.rs`)

```text
Router::new()
    .route("/notes/reorder", post(reorder_notes))
    .route("/notes/bulk",    post(bulk_create_notes))
    .route("/notes/linked",  post(linked_notes))
```

Register BEFORE/alongside `"/notes/{id}"`; axum prefers static
segments, matching DRF `DefaultRouter` ordering (base §4.1).
All three take `Principal`, NOT `AuthUser` (§Auth note).

### DTOs — prefer `Json<serde_json::Value>` + manual validation

Python's `.get` / truthiness / `float()` semantics cannot be
expressed by strict serde typing without changing the error
contract, so take the raw `Value` body and validate manually:

- `reorder`: pull `tag_id` (truthiness: null/`""`/`0`/`false`/
  empty array/empty object → step-1 error) and `updates`
  (`as_array` → step-2 error). Per item: `as_object` (non-object
  → see §5 deviation), `note_id` falsy-check, `position` —
  `is_null()` → error; coerce to `f64` accepting JSON numbers,
  `true`→1.0/`false`→0.0, numeric strings via `f64::from_str`,
  everything else → the §5 float-error decision. Track
  `HashMap<note_id_string, f64>` for the dup checks. `note_id`
  must be a string for the map; a JSON number `123` is truthy
  and hashable — keep it as the JSON-stringified key
  (`"123"`), which then matches nothing in SQL → step-8 400.
- `bulk`: `tag_id` as `Value` → `null`→`may not be null`,
  non-scalar (array/object) → `incorrect_type`, string/number →
  tag existence check → `does_not_exist`. `notes` →
  `as_array` or `Expected a list of items but got type "X"` under
  `non_field_errors`; empty → `At least one note is required.`.
  Items validated into:
  `BulkItem { note_text: String /*default ""*/,
  public: bool /*default false*/,
  verse_references: Vec<VerseRef> /*default []*/ }`.
- `linked`: `LinkedRequest { verse_references: Vec<VerseRef> }`
  with manual min=1/max=500 checks emitting the ListField
  messages, then per-item `VerseRef` validation producing
  index-keyed error dicts.
- `VerseRef { book: String /*max 50*/, chapter: i64,
  verse: i64 }` — implement the DRF int coercion:
  strip `\.0*$` from the string form then `i64::from_str`;
  `3.7`/`true`/`"x"` → `"A valid integer is required."`.
  `book` non-string → `"Not a valid string."`, >50 chars →
  `"Ensure this field has no more than 50 characters."`.

Emit errors via a new `ApiError` variant (extend `core::errors`):
`ApiError::Django400(serde_json::Map)` → serialize the map
verbatim; `ApiError::ReorderValidation(String)` →
`{"error":"validation_error","message":…}`; `ApiError::Conflict{
error, message, constraint: Option }` → the two 409 bodies.

### `reorder_notes` handler

```text
1. Json<Value> body; run validation steps 1-7 → position_map.
2. let mut tx = pool.begin().await?;
3. SELECT id FROM annotations_note
   WHERE id = ANY($1) AND user_id IS NOT DISTINCT FROM $2
     AND tag_id = $3
   FOR UPDATE
   ($1: String[] of note_ids; $2: Option<i64> principal.user.id —
   see §5 for the anonymous deviation).
4. count mismatch → 400 missing-set message; build the repr
   manually: "{'A', 'B'}" (order unstable upstream too).
5. Phase 1 — single CASE UPDATE via sqlx::QueryBuilder:
   UPDATE annotations_note SET tag_position =
     CASE id WHEN 'NOT..' THEN -1.0 … END
   WHERE id = ANY($n)
   (assign temps -1..-n over the FETCHED list order, mirroring
   enumerate(notes)).
6. Phase 2 — same shape, final positions from position_map.
7. tx.commit() → StatusCode::NO_CONTENT (empty body).
8. On sqlx::Error::Database(e):
   e.is_unique_violation() &&
   e.constraint() == Some("unique_tag_position_per_tag")
     → 409 position_conflict body
   e.is_unique_violation() / other IntegrityError kinds
     → 409 database_error body
   non-integrity errors (lock/deadlock) → 500 (Django same).
```

### `bulk_create_notes` handler

```text
1. Principal; resolve owner id: user.id if authenticated else
   SELECT id FROM auth_user WHERE username='guest' (miss → NULL).
   Tag scope id = same user-or-guest id; when None use a
   never-matching id so the tag check still 400s correctly.
2. Validate tag_id (DRF messages) → SELECT id FROM
   annotations_tag WHERE id=$1 AND user_id=$2 (guest scope when
   anonymous). Miss → does_not_exist 400.
3. Validate notes list → Vec<BulkItem>.
4. Dedupe (book.to_lowercase(), chapter, verse) → ONE
   QueryBuilder SELECT on bible_verse with OR'd
   (UPPER(book)=UPPER($x) AND chapter=$y AND verse=$z).
   Key map by (book.to_lowercase(), chapter, verse).
   Missing → 400 non_field_errors with the sorted, joined list.
5. tx = pool.begin():
   - QueryBuilder INSERT INTO annotations_note
     (id, user_id, tag_id, note_text, public, tag_position,
      created_at, updated_at) — one push_values() row per item;
     id = core::ids::note_id() ("NOT"+15 hex upper);
     tag_position = SystemTime::now() as f64 per row;
     timestamps = Utc::now() per row.
   - If any refs: INSERT INTO annotations_noteverse
     (id, note_id, verse_id, created_at) via push_values;
     id = core::ids::note_verse_id() ("NVE"+15).
   - commit.
6. Serialize with the shared NoteDto helper (include_verse_text
   = TRUE, fileset_id from ?fileset_id or "ENGESV" default) —
   provider enrichment per verse-bearing note, error fields
   merged on failure, HTTP stays 201. Until Phase 3 lands the
   provider, enrichment may stub to the failure path
   (empty text + error fields) per base §5.
7. 201 + bare array in request order.
```

### `linked_notes` handler

```text
1. Principal. If user is None → Json({"count":0,"results":[]}).
2. Validate request → Vec<VerseRef>.
3. Query 1: DISTINCT notes + LEFT JOIN tag + INNER JOIN
   noteverse + INNER JOIN verse with the OR'd
   (UPPER(v.book)=UPPER($n) AND v.chapter=$n+1 AND v.verse=$n+2)
   predicate chain; WHERE n.user_id = $1;
   ORDER BY n.created_at DESC. Use QueryBuilder (params are
   dynamic triples).
4. Query 2: verses for the returned ids
   (nv.note_id = ANY($1)) → group by note_id, sort by verse.
5. Assemble NoteDto with include_verse_text=false:
   text="", headings=[], tag nested, verses Python-sorted.
6. Json({"count": results.len(), "results": results}).
```

Keep the 2-query discipline — the parity/test query count is
part of the contract (`assertNumQueries(2)`).

### Shared pieces (coordinate with the item-7 agent)

- `NoteDto`/`TagDto`/`VerseRef` structs + a
  `note_to_dto(row, tag, verses_sorted, enrichment)` helper —
  put them in `routes/annot/mod.rs` or `core` so both plans
  share ONE serializer. Must reproduce: hidden `user`,
  write-only `verse_references`, `tag` PK-string-vs-nested
  quirk, RFC 3339 `Z` micros timestamps, f64 `tag_position`.
- `Verses not found` message builder and the guest-lookup
  helper also fit `core` (single-note create has the same
  guest fallback in `NoteSerializer.create`).

## 4. Test plan

Mirror `annotations/tests.py` classes →
`tests/annot_note_actions.rs` (or three files). Fixtures: seed
`auth_user`, `annotations_tag`, `bible_verse`,
`annotations_note`, `annotations_noteverse` rows directly in
SQL; `AUTO_PROVISION_USERS=false`; authenticate via seeded
`authtoken_token` (or guest fixture for the anon branches).

`TestPartialReordering` → reorder tests:

- `test_partial_reorder`: 10 notes positions 1..10, move 3 to
  50-52 → 204, positions written.
- `test_duplicate_position_conflict`: same position twice →
  400 `Duplicate positions in request.`
- `test_position_already_occupied`: occupy 1.0 with a note NOT
  in the set → **409**, `error == "position_conflict"`,
  `constraint == "unique_tag_position_per_tag"` (NOTE: Django
  test asserts `'conflict'` and is `@expectedFailure` — we
  match implemented behavior; see §5).
- `test_reorder_notes_from_different_tags` /
  `test_reorder_unauthorized_notes`: → 400 containing
  `"not found"`.
- NEW (validation order): missing tag_id; missing updates;
  updates `{}`/`"x"`/`[]`; 101 updates; item missing note_id;
  item missing position; `position: null` at index 1 →
  `…index 1…`; `position: 0` accepted; duplicate note_id.
- NEW: swap positions within the set → 204 (two-phase works);
  verify final positions persisted.

`BulkNoteCreateTest` → bulk tests:

- `test_bulk_create_happy_path`: 201, bare array len 2, texts
  present, tag is PK string (no verses), row count 2.
- `test_bulk_create_missing_verse_returns_400`: 400 +
  `non_field_errors[0]` starts `"Verses not found: john 99:99"`
  + zero rows.
- `test_bulk_create_wrong_tag_returns_400`: other user's tag →
  400 `Invalid pk`.
- `test_bulk_create_empty_notes_returns_400`.
- `test_bulk_create_atomicity`: mixed good/bad → 400, zero rows.
- NEW: verse-linked note → `tag` becomes nested object,
  `verses` present, `text` from the (stubbed) provider path,
  `error`/`error_code` merged when provider fails — HTTP 201.
- NEW: anonymous + seeded guest tag → notes owned by guest;
  anonymous + other-user tag → 400.

`LinkedNotesTest` → linked tests:

- `test_returns_own_linked_notes` (count 1, note_text).
- `test_excludes_other_users_notes`.
- `test_note_matching_multiple_refs_listed_once` (DISTINCT).
- `test_no_match_returns_empty` (`{count:0,results:[]}`).
- `test_book_name_matching_is_case_insensitive` (`"john"`).
- `test_empty_references_returns_400` +
  `verse_references` message exactness.
- `test_unauthenticated_returns_empty` → 200 `{count:0}`.
- `test_no_upstream_calls_and_no_n_plus_1`: assert result shape
  (`verses[].text == ""`, `headings == []`, nested `tag.name`);
  track sqlx query count via a counting test pool if practical
  (else rely on implementation review — the 2-query discipline
  is asserted, not timed).
- NEW: >500 refs → 400 max_length message; ordering
  `-created_at` with 2+ notes; chapter `"3"`/`3.0` coercion.

Parity corpus (`tests/parity/`): three entries —

- `POST /api/v1/notes/reorder/` — happy 204; occupied-position
  409 (compare `error`/`message`/`constraint` verbatim); a 400
  `validation_error` body.
- `POST /api/v1/notes/bulk/` — 201 (normalize `id`,
  `tag_position`, `created_at`/`updated_at`); 400 wrong-tag and
  missing-verse bodies verbatim.
- `POST /api/v1/notes/linked/` — 200 (normalize volatile
  fields); anonymous-empty 200; empty-refs 400.

## 5. Open questions / risks

- **`error: 'conflict'` vs `'position_conflict'`** —
  `test_position_already_occupied` (`tests.py:1045-1061`,
  `@expectedFailure`) asserts `'conflict'`; the code emits
  `'position_conflict'` since `views.py:722`. Match implemented
  behavior. Frontend must be grepped for which key it branches
  on before final sign-off — this may be a latent contract bug.
- **Python set repr in the missing-notes 400** — `{'NOT1',
  'NOT2'}` ordering is hash-order (unstable even in Python).
  Reproduce the `{'…', '…'}` shape; exact-order parity is
  impossible — parity corpus should regex-normalize this body.
- **`float(position)` failure → Django 500**: `"abc"`, `{}`,
  `[]`, unhashable note_id, non-dict update item, non-dict body
  all escape as 500s (HTML). Recommendation: `ApiError::Internal`
  (log it) — same status, different body. Deliberate deviation;
  alternative is coercing to 400 (changes the contract).
- **Anonymous reorder**: Django crashes (TypeError inside
  `get_prep_lookup` for `user=AnonymousUser`). In prod this is
  unreachable. Recommended Rust behavior: bind
  `Option<i64>` user id and use `user_id IS NOT DISTINCT FROM
  $1` — matches NULL-user notes (closest real semantic); a
  stricter alternative is returning the 400 missing-notes path.
  Flagged as deliberate deviation either way.
- **NaN/Infinity positions**: Python `json` accepts `NaN`,
  `Infinity`, `1e999` (→ inf); serde_json rejects non-finite.
  Django would let `float('nan')` through to Postgres
  (`NaN = NaN` violates the unique index → 409 path). Rust
  rejects at parse → different behavior; document.
- **Shared `NoteDto` dependency** on the item-7 (notes CRUD)
  plan — whoever lands second must reuse the first's DTO, incl.
  the `tag` PK-vs-nested quirk and the
  `include_verse_text=False` mode.
- **Provider calls in bulk responses**: verse-bearing bulk
  notes make one upstream call each (Phase-3 dep). Stubbed
  provider must go through the failure path (empty `text` +
  `error`/`error_code`, HTTP still 201), NOT panic/skip.
- **Deadlocks**: `FOR UPDATE` lock order is scan order;
  concurrent reorders on the same tag can deadlock → sqlx
  error is NOT `is_unique_violation()` → 500. Django has the
  same exposure (uncaught `OperationalError`). Optional
  hardening: `ORDER BY id` in the lock select would reduce
  deadlock risk but deviates from Django's no-ORDER-BY query —
  leave as-is, note it.
- **`bulk` tag_position collision**: theoretical only — two
  rows with identical `time.time()` f64 in one tag would 500
  (unique index). Python's sequential calls make it rarer than
  a tight Rust loop computing `now` per row; if worried, add
  a monotonic epsilon per row.
- **`id = ANY($1)` vs `id__in`**: `ANY` with a `String[]` param
  is the clean sqlx equivalent; `tag_id`/`note_id` params stay
  `&str`/`String` (varchar). Numbers sent as JSON ints become
  string keys — they simply never match (step-8 400), same as
  Django.
