# 02 — Tags CRUD (`TagViewSet`)

Inventory item 6. Phase 1 (validates auth + DB + serialization).
Builds on `00-base.md`: auth extractor §4.2, PKs §4.3, error
contract §4.4, conventions §6.

Django sources (all relative to `bible_research/`):

- `annotations/views.py:59-167` — `TagViewSet`
- `annotations/serializers.py:48-106` — `TagSerializer`,
  `CurrentAuthenticatedUserDefault` (`:28-45`)
- `annotations/models.py:11-75` — `Tag`
- `annotations/urls.py:6` — `router.register('tags', ...)`
- `annotations/tests.py:54-72` — only tag test (serializer-level)
- `bible_research/settings.py:346-359` — `REST_FRAMEWORK`
- DRF 3.17.1 `rest_framework/validators.py:105-203` —
  `UniqueTogetherValidator` semantics quoted below

SCOPE CORRECTION: the inventory note's "`tag_position` float
ordering, partial unique index" belongs to **Note**
(`models.py:123-129, 152-158`, `unique_tag_position_per_tag`),
not Tag. Tags have no position or reorder handling — nothing
to port here. Tag uniqueness is `unique_together =
('user','name','parent_tag')` (`models.py:65`); migration 0006
restored it after 0003's partial constraint experiment.

Also: **the tag list is NOT paginated.** `TagViewSet` sets no
`pagination_class` and `REST_FRAMEWORK` declares no
`DEFAULT_PAGINATION_CLASS` — `GET /api/v1/tags/` returns a flat
JSON array. (`NotesPagination` applies only to `NoteViewSet`.)

## 1. Endpoint contract

Tag object — every body-returning response:

```json
{
  "id": "TAG1A2B3C4D5E6F7G8",
  "name": "Love",
  "parent_tag": "TAG9H8G7F6E5D4C3B2",
  "created_at": "2025-12-28T03:51:55.123456Z",
  "updated_at": "2025-12-28T04:10:02.000001Z"
}
```

- `id`: `TAG` + 15 uppercase hex chars, varchar(18).
- `parent_tag`: string PK or `null` (never a nested object —
  `use_pk_only_optimization`).
- `user`: `HiddenField` — NEVER appears in output.
- `created_at`/`updated_at`: RFC 3339 micros + `Z` (§4.3).

### `GET /api/v1/tags/` — list

- Auth: `Principal` (`tok` per inventory). Effective user:
  token/provisioned user → own tags; anonymous → `guest` user's
  tags; no `guest` row → `[]`.
- Query params: none honored (`page`/`page_size` ignored).
- 200: flat array `[{tag}, …]`, `ORDER BY name ASC`
  (Postgres collation — byte/case-sensitive).

### `POST /api/v1/tags/` — create

Body fields: `name` (required), `parent_tag` (optional,
nullable). Extra keys silently ignored; `id`/`user`/`created_at`/
`updated_at` read-only (ignored if sent).

`name` rules (`CharField(max_length=100)`, `allow_blank=False`):

- missing → 400 `{"name": ["This field is required."]}`
- `""` → 400 `{"name": ["This field may not be blank."]}`
- len > 100 → 400 `{"name": ["Ensure this field has no more
  than 100 characters."]}`
- JSON `true`/`{}`/`[]`/`null` → 400 `{"name": ["Not a valid
  string."]}` (null additionally: `["This field may not be
  null."]` — CharField `allow_null=False` fires first)
- JSON int/float → COERCED to string (`5` → `"5"`), accepted

`parent_tag` rules (`PrimaryKeyRelatedField`, queryset scoped to
effective user's tags):

- `null`/absent → `null`
- existing tag of the effective user → accepted
- missing/foreign/nonexistent id → 400
  `{"parent_tag": ["Invalid pk \"<v>\" - object does not exist."]}`
- bool/dict/list → 400 `{"parent_tag": ["Incorrect type.
  Expected pk value, received <pytype>."]}` (`<pytype>` =
  `bool`/`dict`/`list`; a JSON int is queried as pk →
  "Invalid pk" path, not incorrect_type)
- self-reference allowed (no cycle check)

Uniqueness — `unique_together (user, name, parent_tag)`:

- checked ONLY when user, name AND parent_tag are all non-null
  on create (DRF 3.17 skips when any value is `None`) →
  **duplicate top-level (null-parent) names are ALLOWED**
- duplicate child (same user+name+parent_tag) → 400
  `{"non_field_errors": ["The fields user, name, parent_tag must
  make a unique set."]}`

Success → **201** + tag object. Owner = effective user (token or
provisioned; anonymous → `guest`; no `guest` → `user_id` NULL —
row created but invisible to all subsequent scoped queries).

### `GET /api/v1/tags/{id}/` — retrieve

- 200 + tag object. `{id}` matches `[^/.]+`.
- Other user's / missing id → 404 `{"detail": "Not found."}`
  (queryset-scoped `get_object_or_404` — never 403).

### `PUT /api/v1/tags/{id}/` — full update

- Full replace semantics: `name` required (same 400 as create);
  absent `parent_tag` → field default `null` (NOT "keep
  existing") — a PUT without `parent_tag` detaches the tag.
- Uniqueness checked on the effective (user, name, parent_tag)
  triple — missing fields filled from the instance — excluding
  the row's own pk; violation → same `non_field_errors` 400.
  Check runs iff some field changed AND no changed value is
  `None`. NOTE the asymmetry: renaming a TOP-LEVEL tag to an
  existing top-level name DOES 400 (the filled filter uses
  `parent_tag IS NULL` matching), even though creating the same
  duplicate would have succeeded.
- 200 + tag object; `updated_at` bumped. Owner cannot change
  (`user` is hidden/read-only).

### `PATCH /api/v1/tags/{id}/` — partial update

- Absent fields skipped entirely (keep DB values). Explicit
  `"parent_tag": null` sets it null — and then uniqueness is
  SKIPPED (changed value is `None`).
- Otherwise identical to PUT: 200 + object, same error bodies.

### `DELETE /api/v1/tags/{id}/` — destroy

- 204, empty body.
- HARD DELETE with DB cascades: `parent_tag` FK CASCADE removes
  descendant tags recursively; `Note.tag` FK CASCADE removes
  EVERY NOTE under the tag (and via note cascades: note-verses,
  comments, images rows — GCS objects not touched by Django
  either). No soft-delete, no confirmation flag.

### Shared error bodies

- 401 bad token: `{"detail": "Invalid token."}` +
  `WWW-Authenticate: Token`
- 401 anonymous under prod `IsAuthenticated` (unreachable —
  provisioning; tests run AllowAny):
  `{"detail": "Authentication credentials were not provided."}`
- 404: `{"detail": "Not found."}`
- 405 (e.g. `PUT /api/v1/tags/`):
  `{"detail": "Method \"PUT\" not allowed."}` — router-level, §4.4

## 2. Django behavior trace

`get_queryset` (`views.py:67-107`):

- `request.user.is_authenticated` →
  `Tag.objects.filter(user=user).order_by('name')`:
  ```sql
  SELECT id, user_id, name, parent_tag_id, created_at, updated_at
  FROM annotations_tag WHERE user_id = $1 ORDER BY name ASC;
  ```
- anonymous → `User.objects.get(username='guest')` then the same
  filter with guest's id; `User.DoesNotExist` →
  `Tag.objects.none()` → `[]` (still 200).
- The `logger.debug` calls evaluate `queryset.count()` eagerly —
  a second `SELECT COUNT(*)` per list/detail. Non-contract; keep
  the log line, count from row len or skip (§4.14 observability).

`create` (`perform_create` `:109-138` + `TagSerializer.create`
`serializers.py:97-106`):

1. Field validation as §1 (name, parent_tag scoped pk lookup:
   `SELECT … FROM annotations_tag WHERE id=$1 AND user_id=$2`,
   missing → `does_not_exist` 400).
2. `UniqueTogetherValidator` on `Tag.objects.all()` (NOT scoped;
   user is part of the filter): build `filter(user=…, name=…,
   parent_tag=…)`; skip if any checked value is `None`; exists →
   400 non_field_errors. (`validators.py:152-203`.)
3. `serializer.save(user=<effective user>)` → INSERT with
   `generate_tag_id()` (`TAG`+uuid4-hex[:15], uppercase) and
   `created_at = updated_at = now()`. Django sets timestamps in
   Python — `annotations_tag` has NO column defaults; Rust MUST
   supply both on INSERT.
4. If effective user is `None` (anon + no guest): `create()` drops
   `user` → `user_id` NULL INSERT.

`update` (`perform_update` `:140-151`): get object from scoped
queryset → `serializer.save()`. PUT = `partial=False`, PATCH =
`partial=True`; validator fills missing attrs from the instance,
then compares each field to the instance — only changed fields
enter `checked_values`; `None` in `checked_values` (or no
changes) skips the check; `exclude_current_instance` drops own
pk. Then UPDATE, `updated_at = now()`.

`destroy` (`perform_destroy` `:153-167`): `instance.delete()` —
single DELETE + FK cascades. No owner check beyond queryset
scoping (unlike notes' explicit `PermissionDenied`).

DB constraint truth: plain `UNIQUE (user_id, name,
parent_tag_id)` — Postgres treats NULLs as distinct → DB does
NOT back the validator for null parents. A true unique-violation
race on child tags → `IntegrityError` → Django 500 (HTML, not
JSON).

Auth detail: prod `IsAuthenticated` + middleware ⇒ every request
is a user (token or provisioned; §4.2). conftest drops
`REST_FRAMEWORK` → AllowAny + no provisioning ⇒ anonymous hits
the guest branches. `Principal`/`AUTO_PROVISION_USERS` covers
both; do NOT use `AuthUser` here (would break the guest tests).

## 3. Rust implementation plan

Module: `crates/server/src/routes/annot/tags.rs` (§6 layout).
Register in `crates/server/src/routes/mod.rs` under `/api/v1`:

```rust
.route("/tags/", get(list_tags).post(create_tag))
.route("/tags/{id}",
    get(get_tag).put(replace_tag).patch(patch_tag)
    .delete(delete_tag))
```

plus `#[utoipa::path]` per handler + `openapi.rs` entries
(request schema, tag DTO, `Token`/`Bearer` security).

Row + DTOs (serde snake_case already — §6 naming):

```rust
struct TagRow {                      // query_as target
    id: String, user_id: Option<i32>, name: String,
    parent_tag_id: Option<String>,
    created_at: DateTime<Utc>, updated_at: DateTime<Utc>,
}
struct TagDto {                      // Serialize — output only
    id: String, name: String,
    parent_tag: Option<String>,      // = row.parent_tag_id
    created_at: String, updated_at: String, // Micros + 'Z'
}
struct TagWrite {                    // Deserialize — input
    name: Option<serde_json::Value>, // custom: coerce int/float
    parent_tag: Option<Option<String>>, // absent/null/"TAG…"
}
```

- `name` as raw `Value` then `coerce_name()` →
  `Result<String, ApiError>`: String ok; `Number` →
  `to_string()`; `Bool`/`Object`/`Array`/`Null` → the §1 400s.
  `len > 100` → max-length 400 (check `chars().count()` — DRF
  counts Python str length = chars, not bytes).
- `parent_tag` tri-state drives PUT-vs-PATCH merge:
  `None` outer → partial-skip (PATCH) / default-null (PUT);
  `Some(None)` → explicit null; `Some(Some(pk))` → resolve.

Shared helper:

```rust
async fn effective_user(pool, &Principal)
    -> Result<Option<i64>, ApiError>
// p.user → Some(id); anonymous → SELECT id FROM auth_user
// WHERE username='guest' → Some/None
```

Handlers (`ApiResult<T>`; `never unwrap`; `tracing::info!` the
same lifecycle lines as Django):

- `list_tags`: `effective_user` → `Some(id)` →
  `query_as::<TagRow>(… WHERE user_id=$1 ORDER BY name ASC)`;
  `None` → `Json(vec![])`.
- `create_tag`: validate name; resolve parent_tag — when
  effective user is `None` Django's queryset is literally
  `Tag.objects.none()` → fail `does_not_exist` WITHOUT a query;
  else `SELECT … WHERE id=$1 AND user_id=$2` (plain `=` — a
  NULL-user row must NOT resolve); unique check
  `SELECT EXISTS(SELECT 1 FROM annotations_tag WHERE
  user_id=$1 AND name=$2 AND parent_tag_id=$3)` ONLY when all
  three non-null; INSERT via `core::ids` `TAG` id + one `now()`
  timestamp for both columns; `RETURNING` → `(201, Json(dto))`.
- `get_tag`: scoped `SELECT … WHERE id=$1 AND user_id=$2` →
  `Option` → `ApiError::NotFound` (`{"detail":"Not found."}`).
- `replace_tag`/`patch_tag`: fetch scoped row (404) → merge per
  PUT/PATCH rules → validator-equivalent: fill from row, compute
  changed fields, skip when none-or-None, else `EXISTS … AND
  id<>$4` → 400 `{"non_field_errors":[…]}` → UPDATE +
  `RETURNING` → 200.
- `delete_tag`: scoped existence check (404 via a `SELECT id` —
  must 404, not silently succeed) → `DELETE FROM
  annotations_tag WHERE id=$1` → `StatusCode::NO_CONTENT`.
  Cascades are DB-owned — nothing else to do.

`ApiError` variants used: `Validation(fields)` for the §1 400s
(`{"name":[…]}` / `{"parent_tag":[…]}` /
`{"non_field_errors":[…]}`), `NotFound`, `Unauthorized` (from
`Principal` on bad token). Map a residual
`is_unique_violation()` on INSERT/UPDATE to the same
`non_field_errors` 400 — see Risks.

No pagination import, no transactions needed (each action is one
statement; the 3-step validate→insert is read-check-write —
validator + DB constraint are both advisory anyway).

## 4. Test plan

`tests/annot_tags.rs`: `axum-test` + scratch Postgres
(`schema.sql`), `AUTO_PROVISION_USERS=false`, SQL-seeded
`auth_user` rows (`testuser`, `other`, `guest`) + tokens —
mirrors conftest.

Django coverage today: ONLY `SerializerTestCase.
test_tag_serializer` (`tests.py:54-72`) — creates via
`TagSerializer`, asserts `id` set + name echo. No HTTP-level tag
tests exist; this suite is new coverage, not a port.

Cases:

1. POST happy → 201; `id` matches `^TAG[0-9A-F]{15}$`, name
   echo, `parent_tag: null`, NO `user` key, both timestamps.
2. GET list → 200 flat array, own tags only (other user's
   absent), `ORDER BY name`.
3. Anonymous (no token) list → guest's tags only; anonymous
   POST → row owned by guest (list again shows it).
4. No `guest` row in DB → anonymous list `[]`; anonymous POST →
   201, row has `user_id` NULL (assert via SQL).
5. Foreign id → 404 `{"detail":"Not found."}` on GET/PUT/PATCH/
   DELETE.
6. POST missing name → `{"name":["This field is required."]}`;
   `""` → blank msg; 101 chars → max-length msg; `true` →
   `["Not a valid string."]`; `5` → 201 name `"5"`.
7. POST `parent_tag` = other user's tag / garbage →
   `Invalid pk` 400; `parent_tag: {}` → `incorrect_type`
   400; `parent_tag` = own id → 201 (self-parent allowed).
8. Child duplicate (same name+parent) → 400
   `{"non_field_errors":["The fields user, name, parent_tag must
   make a unique set."]}`.
9. Top-level duplicate name → **201** (DRF None-skip) — pin the
   quirk.
10. PATCH rename top-level → existing top-level name → 400
    non_field_errors (update-time asymmetry vs #9); PATCH
    `parent_tag: null` on a child → 200, skips uniqueness.
11. PUT without `parent_tag` → row's parent detached to null;
    PUT without `name` → 400 required.
12. PATCH name only → `parent_tag`/`id`/`created_at` unchanged,
    `updated_at` advanced.
13. DELETE → 204; child tags and their notes gone (SQL counts);
    second DELETE → 404.
14. `Authorization: Token bogus` → 401
    `{"detail":"Invalid token."}` + `WWW-Authenticate: Token`.
15. With `AUTO_PROVISION_USERS=true`: no creds → device user
    sees ONLY its own (empty) tags — prod parity.

Parity corpus entry (`tests/parity/`): `GET tags/` (token +
anon), `POST` (minimal; child; dup-top-level; dup-child; bad
parent; 101-char; int name), `GET/PUT/PATCH/DELETE tags/{id}`
(own, foreign, missing). Normalize `id`, `created_at`,
`updated_at` (volatile, §4.15).

## 5. Open questions / risks

- **Flat array confirmed** — but if a
  `DEFAULT_PAGINATION_CLASS` ever lands in settings.py the
  envelope changes; flag in code comment, re-check at parity
  time.
- **Tag delete is destructive**: cascade wipes child tags AND
  all notes under them (and comments/image-rows). Faithful port;
  confirm the frontend confirms before delete. No soft-delete
  exists to preserve.
- **Unique-race divergence**: validator is advisory; a real
  `UNIQUE (user_id,name,parent_tag_id)` violation in Django →
  HTML 500. Planned Rust: same `non_field_errors` 400.
  Deviation — safer and unreachable in practice; parent agent
  may instead prefer JSON 500 `{"detail":"Internal server
  error."}` (still deviates from Django's HTML).
- **Name coercion** of JSON numbers is DRF-correct but odd
  (`5`→`"5"`); replicated for parity — tighten only if the
  frontend never sends non-strings.
- **Parsers**: DRF default also accepts form-encoded/multipart
  bodies; plan is JSON-only. Frontend sends JSON — low risk,
  documented deviation.
- **Guest-missing anonymous create** writes `user_id` NULL rows
  that are invisible to every scoped query (guest lookup misses,
  authed queries filter `user_id = self`). Replicated verbatim;
  harmless dead rows.
- **Ordering** is plain `ORDER BY name` under the column's
  Postgres collation (C-ish, case-sensitive — `'Zebra'` before
  `'apple'`). Keep identical; don't "fix" to `LOWER(name)`.
- **Router polish deferred to base**: trailing-slash 301
  (`APPEND_SLASH`), 405 body shape (§4.4), `OPTIONS`.
- **`parent_tag` pk type**: column is varchar — a JSON int
  reaches "Invalid pk" (queried) while bool/dict/list reach
  `incorrect_type`; replicate the type→error mapping exactly.
- Timestamps: column has no DB default — forgetting
  `created_at` on INSERT is a runtime NULL failure; the
  `RETURNING` + fixture schema test (#1) catches it.
