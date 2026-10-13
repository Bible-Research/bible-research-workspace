# 05 — Reading positions (`ReadingPositionViewSet`)

Inventory item 11. Django source: `annotations/views.py:1240-1296`,
`annotations/serializers.py:723-757`,
`annotations/models.py:412-475`,
`bible/utils/bible_books.py` (book validation),
`annotations/tests.py:1115-1149` (`ReadingPositionViewSetTest`).
Registered in `annotations/urls.py:8-12` via `DefaultRouter`,
basename `reading-position`.

Status: PLAN. Depends on Phase 0 (`core::auth`, `core::errors`,
`core::ids`, `core::books` with `get_dbt_book_id`).

## 1. Endpoint contracts

All six routes sit under `/api/v1/reading-positions`. Auth:
`permission_classes = [IsAuthenticated]` on the viewset → every
method uses the `AuthUser` extractor (401
`{"detail": "Authentication credentials were not provided."}` when
Anonymous; under prod `AUTO_PROVISION_USERS` the device middleware
provisions a user, so requests virtually always pass). No object-level
permission class — ownership is enforced purely by
`WHERE user_id = $1` scoping, so cross-user access is **404**, not 403.

### `GET /api/v1/reading-positions/` — list

- Query param `book` (optional): exact, case-sensitive match on the
  stored `book` string (`queryset.filter(book=book)`). NOT
  normalized — `?book=john` will not match a row stored as `John`.
- 200 body: **plain JSON array** — no pagination (the viewset sets no
  `pagination_class` and `settings.REST_FRAMEWORK` defines no global
  `DEFAULT_PAGINATION_CLASS`).
- Ordered `last_accessed DESC` (model `Meta.ordering`).
- Item shape:
  `{"id": "RDP…", "book": "John", "chapter": 3, "verse": 16,
  "last_accessed": "2025-12-28T03:51:55.123456Z"}`.
  `created_at` and `user` are NOT serialized (not in `Meta.fields`).
- Empty result → `[]`.

### `POST /api/v1/reading-positions/` — upsert

- Body: `{"book": "John", "chapter": 3, "verse": 16}`.
- `book`: required, max_length 50, must resolve via
  `get_dbt_book_id` (exact/alias/fuzzy). Stored **verbatim** —
  `validate_book` returns the raw value, so `john`, `John` and
  `JOHN` are three distinct rows for the same user (unique
  constraint is on the literal string).
- `chapter`, `verse`: optional; model defaults `1` are propagated as
  serializer defaults, so omitting either yields 1. DRF
  `IntegerField` coercion: numeric strings (`"3"`) accepted, floats
  truncated toward zero (`3.7` → 3), `true` → 1.
- Semantics: `update_or_create(user, book, defaults={chapter,verse})`.
  Existing `(user_id, book)` row → chapter/verse replaced; absent →
  insert with `RDP…` id. `last_accessed` refreshed in both cases.
- Response: **200** in both create and update cases — the view
  hardcodes `HTTP_200_OK` (never 201, no `created` flag). Body = the
  serialized row (same shape as list item).
- Errors:
  - missing `book` → 400 `{"book": ["This field is required."]}`
  - `book` > 50 chars → 400
    `{"book": ["Ensure this field has no more than 50 characters."]}`
    (length check runs BEFORE `validate_book` — DRF runs field
    validators before `validate_<field>`)
  - unknown book → 400 `{"book": ["Invalid book name: NotABook"]}`
    — echoes the RAW input value.
  - `chapter`/`verse` non-numeric → 400
    `{"chapter": ["A valid integer is required."]}`.

### `GET /api/v1/reading-positions/{id}/` — retrieve

- `{id}`: `[^/.]+` (never contains a dot).
- 200 + item shape. Other user's id or unknown id →
  404 `{"detail": "Not found."}`.

### `PUT /api/v1/reading-positions/{id}/` — full update

- Default `ModelSerializer.update()` — plain `UPDATE`, NOT the
  upsert path. `book` IS writable here.
- `book` required (partial=False); `chapter`/`verse` reset to their
  serializer default `1` when omitted. `last_accessed` bumps via
  `auto_now` on every save (even PATCH `{}`).
- 200 + serialized row. Unknown/other-user id → 404.
- Same validation errors as POST.
- Edge: changing `book` to a name the user already has a row for
  violates `unique_together (user, book)` → unhandled
  `IntegrityError` → Django 500. Reproduce as a 500 (see §5).

### `PATCH /api/v1/reading-positions/{id}/` — partial update

- Only provided fields validated/updated (`book` optional here);
  `last_accessed` always bumps. 200 + serialized row; 404 as above.

### `DELETE /api/v1/reading-positions/{id}/` — destroy

- Default `destroy()` (no `perform_destroy` override). 204 empty
  body. Unknown/other-user id → 404.

### `POST /api/v1/reading-positions/bulk/` — bulk fetch

- Registered `detail=False` → static segment beats `{id}` (axum
  prefers static segments anyway; register it BEFORE `/{id}`).
- Body: `{"books": ["John", "Mark"]}`.
- `books` missing or `[]` → 200 `{}`.
- `books` not a JSON array → 400 `{"error": "books must be an
  array"}` (hand-rolled body — `error` key, NOT DRF field map).
- Books are NOT validated through `get_dbt_book_id` — the action
  just does `filter(user=…, book__in=books)`, exact/case-sensitive.
- 200 body: JSON object keyed by each requested book (request
  order, duplicates collapsed by first occurrence):
  `{"John": {"chapter": 3, "verse": 16}, "Mark": null}`.
  Only `chapter` + `verse` per hit — no `id`, no `last_accessed`.
- Quirk: `result` is a Python dict built from the raw `books`
  values. JSON scalars (1, true, null) become keys `"1"`, `"true"`,
  `"null"` and match nothing. Objects/arrays are unhashable →
  Django TypeError → 500 (see §5).

## 2. Current Django behavior trace

- `get_queryset` (views.py:1252-1261):
  `SELECT … FROM reading_positions WHERE user_id = %(user)`
  `[AND book = %(book)] ORDER BY last_accessed DESC`. One query;
  `reading_pos_user_book_idx` and `reading_pos_user_time_idx` cover
  both shapes.
- `create` (views.py:1263-1272): `serializer.is_valid()` →
  `serializer.save()` → `ReadingPositionSerializer.create`
  (serializers.py:745-757) →
  `ReadingPosition.objects.update_or_create(user=…, book=…,
  defaults={'chapter': …, 'verse': …})`. Django's
  `update_or_create` runs `SELECT … FOR UPDATE` inside
  `transaction.atomic`, then UPDATE or INSERT. One round-trip pair
  (SELECT FOR UPDATE + UPDATE) or (SELECT + INSERT); Postgres
  `ON CONFLICT` is one statement and strictly stronger.
  Response re-serializes the instance → 200.
- `validate_book` (serializers.py:737-743): `get_dbt_book_id(value)
  is None` → `ValidationError("Invalid book name: {value}")`.
  `get_dbt_book_id` (bible_books.py:195-217): normalize (lowercase,
  whitespace collapse, roman numeral `i/ii/iii` → `1/2/3`, ordinal
  suffixes `1st/2nd/3rd` → `1/2/3`, number words `first/second/
  third` → `1/2/3`, `revelation of john` → `revelation`) → exact
  map hit → `_BOOK_ALIASES` (`isiah`, `canticles`, `apocalypse`,
  …) → `difflib.get_close_matches` cutoff 0.8, with a digit-prefix
  lock (`3 samuel` only competes against `3 *` books; unnumbered
  input only competes against unnumbered names). The RESOLVED id is
  discarded — only None/not-None matters, raw value returned.
- `update`/`destroy`/`retrieve`: stock ModelViewSet via the
  user-scoped queryset — the 404 boundary is the queryset, no
  `PermissionDenied` path exists in this viewset.
- `bulk` (views.py:1274-1296): no serializer. `book__in` query,
  then a Python dict keyed by raw request values. Insertion order =
  request order; later rows for the same book overwrite the value
  (impossible: `book` unique per user).

## 3. Rust implementation plan

Module `crates/server/src/routes/annot/reading_positions.rs`
(base layout §3: annot group). Register in `routes/mod.rs` nested
under `/api/v1` AND add `#[utoipa::path]` + `openapi.rs` entries:

```text
route("/reading-positions",        get(list).post(upsert))
route("/reading-positions/bulk",   post(bulk))          // before {id}
route("/reading-positions/{id}",   get(retrieve)
        .put(update).patch(update).delete(destroy))
```

### DTOs (beside the handler, base §6 conventions)

- `ReadingPositionDto { id, book, chapter: i32, verse: i32,
  last_accessed: DateTime<Utc> }` — `Serialize` only;
  `last_accessed` via `to_rfc3339_opts(SecondsFormat::Micros, true)`
  per base §4.3. Do NOT expose `created_at`/`user_id`.
- `UpsertPositionRequest { book: String,
  chapter: Option<JsonValue>, verse: Option<JsonValue> }` —
  take chapter/verse as raw `Value` and run a `drf_int()` helper
  (accept `u64`/`i64`/numeric string with `str().strip()`;
  truncate `f64` toward zero like Python `int()`; `true`/`false` →
  1/0). Absent → 1. Shared helper can live in `core::errors` or a
  `core::serde_drf` module if other plans need it — note the
  collision and extract then.
- `BulkPositionsRequest { books: Option<Vec<JsonValue>> }`.
- Validation is manual (serde alone can't emit the exact DRF field
  maps). Order per base §4.4 / DRF: required → type → max_length(50)
  → `core::books::get_dbt_book_id(&book).is_none()` →
  `ApiError::Validation({"book": ["Invalid book name: {book}"]})`.
  `book` must be a JSON string: non-string →
  `{"book": ["Not a valid string."]}`; null/missing →
  `{"book": ["This field is required."]}` (DRF treats explicit null
  on a required non-null field as
  `"This field may not be null."` — verify and match; see §5).

### Handlers

`list(State, AuthUser, Query<{book: Option<String>}>)`:

```sql
SELECT id, book, chapter, verse, last_accessed
FROM reading_positions
WHERE user_id = $1
  AND ($2::text IS NULL OR book = $2)
ORDER BY last_accessed DESC
```

(or two static queries — same plan). → `Json<Vec<ReadingPositionDto>>`.

`retrieve / update precheck / delete precheck` all resolve the row
with `WHERE user_id = $1 AND id = $2`; `None` →
`ApiError::NotFound` → `{"detail": "Not found."}`.

`upsert(State, AuthUser, Json<UpsertPositionRequest>)`:

```sql
INSERT INTO reading_positions
    (id, user_id, book, chapter, verse, last_accessed, created_at)
VALUES ($1, $2, $3, $4, $5, now(), now())
ON CONFLICT (user_id, book) DO UPDATE SET
    chapter = EXCLUDED.chapter,
    verse = EXCLUDED.verse,
    last_accessed = now()
RETURNING id, book, chapter, verse, last_accessed
```

- `$1` = `core::ids::reading_position_id()`
  (`"RDP" + Uuid::new_v4().simple().to_uppercase()[..15]`,
  varchar(18)).
- Single statement — no txn needed (replaces Django's
  atomic + SELECT FOR UPDATE; strictly stronger under races).
- Response `StatusCode::OK` + dto — 200 ALWAYS, even on insert.

`update` (PUT and PATCH share the handler; PUT = require `book`,
apply chapter/verse defaults 1; PATCH = apply only provided fields,
both via the same validation then):

```sql
UPDATE reading_positions
SET book = $3, chapter = $4, verse = $5, last_accessed = now()
WHERE user_id = $1 AND id = $2
RETURNING id, book, chapter, verse, last_accessed
```

PATCH builds the SET list dynamically but ALWAYS sets
`last_accessed = now()` (auto_now on save). Unique violation
(`is_unique_violation()`) → 500-equivalent `ApiError` — parity
with Django's unhandled `IntegrityError` (see §5).

`destroy`: `DELETE … WHERE user_id = $1 AND id = $2`;
`rows_affected == 0` → `NotFound`; else `204` empty.

`bulk(State, AuthUser, Json<Value>)` — manual parse: `books`
absent/null → treat as `[]`? NO — Django does
`request.data.get('books', [])`: absent → `[]` → `{}`, but
explicit `null` → `books=None` → `isinstance(None, list)` False →
400. Reproduce exactly. If `books` is not a JSON array → 400
`{"error": "books must be an array"}`. For entries: string → key =
value; other JSON scalars → key = their JSON rendering (`"1"`,
`"true"`, `"null"`); object/array entries → deviation decision,
default: 400 `{"error": "books must be an array"}` (see §5).

```sql
SELECT book, chapter, verse FROM reading_positions
WHERE user_id = $1 AND book = ANY($2::varchar[])
```

then assemble the map iterating REQUEST entries so key order =
request order. Use `serde_json::Map` with the `preserve_order`
feature (indexmap) — `BTreeMap` would reorder keys alphabetically
and break the parity diff.

### `core::books` dependency

`get_dbt_book_id` must exist before this endpoint ships (Phase 0 or
this plan — whichever lands first; the base doc assigns
`books.rs` to `core`). Port bible_books.py verbatim:
`_BIBLE_BOOKS` 66-row table → `DBT_BOOK_NAME_TO_ID`,
`_BOOK_ID_TO_NAME` via `_book_title`, `BOOK_ORDER_MAP` (also needed
by notes `ordering=verse`), `normalize_sword_book_name`,
`_normalize_for_lookup`, `_BOOK_ALIASES`, `_NUMBER_WORDS`,
`_ORDINAL_RE`, `_fuzzy_book_id` (cutoff 0.8, digit lock) via the
`difflib` crate's `SequenceMatcher`/`get_close_matches`. For this
endpoint only `get_dbt_book_id(...) -> Option<&'static str>` is
consumed; the resolved id is discarded.

Timestamps: `last_accessed`/`created_at` are `timestamptz`;
`now()` in SQL keeps them server-side, matching `auto_now`.

## 4. Test plan

Mirror `annotations/tests.py::ReadingPositionViewSetTest`
(tests.py:1115-1149) as `tests/annot_reading_positions.rs` with
`axum-test` + scratch Postgres, `AUTO_PROVISION_USERS=false`, token
seeded in `authtoken_token`:

Django-mirrored:

1. `test_create_position_valid_book` — POST `{book:"John",
   chapter:3, verse:16}` → 200, body book/chapter/verse echo +
   `id` prefixed `RDP`, `last_accessed` present.
2. `test_create_position_invalid_book` — POST `NotABook` → 400
   `{"book": ["Invalid book name: NotABook"]}`.

Coverage the Django tests lack but the contract needs:

3. POST twice same `(user, book)` → second returns 200 with same
   `id`, updated chapter/verse (upsert, not 201/dup row).
4. POST `{book:"John"}` → chapter=1, verse=1 (defaults).
5. POST `{"chapter":"4"}` string coercion → 200 chapter 4;
   `{"chapter": 3.7}` → 3; `{"chapter":"x"}` → 400
   `"A valid integer is required."`.
6. POST book >50 chars → 400 max-length message; invalid AND long
   → length error wins (ordering check).
7. List → array ordered `last_accessed` DESC; `?book=john` does not
   match stored `John` (verbatim storage); `?book=John` filters.
8. Verbatim-storage quirk: POST `john` then `John` → 2 rows, both
   valid (fuzzy passes lowercase).
9. GET/PATCH/DELETE on another user's id → 404 `{"detail": "Not
   found."}`; on unknown id → 404.
10. PUT `{book:"Mark"}` on a John row → renames book, 200; PUT
    omitting `book` → 400 `{"book": ["This field is required."]}`.
    PATCH `{}` → 200 and `last_accessed` bumps.
11. DELETE → 204 empty; second DELETE → 404.
12. bulk: `{"books":["John","Mark"]}` with only John stored →
    `{"John":{"chapter":3,"verse":16},"Mark":null}`; `{}` (missing
    books) → `{}`; `{"books":"John"}` → 400
    `{"error":"books must be an array"}`; dup keys collapse;
    response key order == request order.
13. Anonymous (no token, AUTO_PROVISION_USERS=false) → all six
    methods 401 `{"detail": "Authentication credentials were not
    provided."}`; bad token → 401 `{"detail": "Invalid token."}`
    + `WWW-Authenticate: Token`.
14. Fuzzy path: `genisis`, `isiah` (alias), `1st Kings`, `I Samuel`,
    `Revelation of John` all → 200 (book-name unit tests live in
    `core::books` tests — here just one or two through the HTTP
    layer).

Unit tests: `drf_int` helper edge cases; id-gen length/charset
(covered by core::ids tests).

Parity corpus (`tests/parity/`, base §4.15): add a
reading-positions sequence — POST create → POST update same book →
list → `?book=` filter → PUT rename → PATCH → bulk happy + bulk
non-array + bulk missing → DELETE. Normalize `id` and
`last_accessed` before diffing.

## 5. Open questions / risks

- **`book` stored verbatim** (no normalization on write): keep
  parity — 'john' and 'John' coexist and `?book=`/`book__in` are
  exact-match. If the frontend only ever sends title case this is
  invisible; flag to the frontend owner whether to normalize in v2.
- **PUT/PATCH book collision → IntegrityError → Django 500.** Rust
  options: (a) reproduce 500 — honest parity; (b) 409 with a
  reorder-style error — nicer but new contract. Recommend (a) with
  a logged warning; the frontend can't hit it without sending a
  duplicate book anyway.
- **`bulk` non-string entries**: dict/list entries crash Django
  (unhashable → 500); JSON scalars become string keys that match
  nothing. Plan accepts scalars (stringified as keys/lookups) and
  rejects objects/arrays with the 400 `books must be an array`
  body — deliberate deviation, document it; a 500 reproduction has
  no consumer value.
- **`books: null`** → Django 400s (`get` returns None, not `[]`).
  Absent → `{}`. Reproduce both.
- **Explicit `null` for `book`/`chapter`/`verse`**: DRF emits
  `"This field may not be null."` for non-nullable fields —
  verify against a live Django instance and match the message;
  a bare serde `Option` would silently map null→default and
  diverge.
- **int4 overflow**: `chapter: 10**30` passes Django validation then
  fails at INSERT → 500. Rust: keep `i32` and let the DB error
  surface as 500, or 400 early — parity says let it 500; cheap
  either way, decide at impl.
- **`update_or_create` vs `ON CONFLICT`**: equivalent-or-stronger;
  the only observable difference would be lock-ordering deadlocks
  under concurrent upserts of the same (user, book) — Postgres
  `ON CONFLICT` avoids the SELECT-then-INSERT race entirely. No
  contract change.
- **Trailing slashes**: DRF/`DefaultRouter` emit `/bulk/` and
  `/…/` with `APPEND_SLASH` redirects for slash-less requests.
  Router-wide decision belongs to the base plan — note it here so
  `/reading-positions/bulk` (no slash) parity is handled
  consistently (register both or add a redirect layer).
- **`AutoField` `user_id` width**: `auth_user.id` is bigint —
  bind `i64`, not `i32`, in all `user_id` params.
