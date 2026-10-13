# 06 — Comments tree + counts (inventory #12, #15)

Status: PLAN. Covers `CommentViewSet` (nested under notes) and
`CommentCountView`. Sources:

- `annotations/views.py:750-861` (CommentViewSet),
  `annotations/views.py:864-984` (CommentCountView)
- `annotations/serializers.py:568-720` (CommentAuthorSerializer,
  CommentSerializer, build_comment_tree)
- `annotations/models.py:172-260` (Comment `CMNT_` PK, self-FK,
  note FK)
- `annotations/urls.py:13-17, 31-35` (router + counts path)
- `annotations/tests.py:110-673` (comment + counts tests)

Auth per base doc §4.2: prod = `IsAuthenticated` default +
device-provisioning ⇒ every request has a user (`tok` semantics).
Test world (conftest) = DRF defaults `AllowAny` ⇒ anonymous reads
work. No permission_classes are set on either view.

No mention/@-notification logic exists anywhere in the backend
(grep `mention` = 0 hits). Nothing to port there.

## 1. Endpoint contracts

Base path: `/api/v1/`. Comment PKs: `CMNT_` + 13 uppercase hex
(18 chars total). Path segments are `[^/.]+` (no dots).

### 1.1 `GET /notes/{note_pk}/comments/` — tree list

Auth: `Principal` (anonymous OK in test world; prod provisioned).

Response 200 — **bare JSON array** (no envelope, no pagination;
the custom `list()` bypasses DRF pagination entirely):

```json
[
  {
    "id": "CMNT_ABCDEF0123456",
    "author": {"id": 3, "username": "testuser"},
    "note_id": "NOT1234567890ABCD",
    "parent_comment": null,
    "content": "Root text",
    "timestamp": "2025-12-28T03:51:55.123456Z",
    "is_deleted": false,
    "images": [],
    "replies": [
      {"id": "CMNT_...", "author": {"id": 4, "username": "u2"},
       "note_id": "NOT1234567890ABCD",
       "parent_comment": "CMNT_ABCDEF0123456",
       "content": "Reply", "timestamp": "...",
       "is_deleted": false, "images": [], "replies": []}
    ]
  }
]
```

- Roots and `replies` arrays are ordered `timestamp` ASC.
- `author.id` = `auth_user.id` (`BigAutoField` → JSON integer).
- `timestamp` = RFC 3339 µs + `Z` (base doc §4.3).
- Soft-deleted nodes: `"content": "[deleted]"`, `"images": []`,
  `is_deleted: true`; their replies stay attached.
- **No note-existence or accessibility check**: unknown or
  inaccessible `note_pk` → `200 []`, NOT 404/403. Comments on
  private notes owned by others are readable by any user. Contract
  (also a security quirk — see §5).
- Orphan rule: a reply whose `parent_comment_id` is not among the
  fetched comments is silently DROPPED from the tree (not promoted
  to root). Only reachable via data corruption, but reproduce it.

`images` element shape (shared `ImageDto`, ordering `-created_at`
= newest first per `Image.Meta`):

```json
{"id": "IMG_...", "storage_url": "gs://.../originals/IMG_.../s.jpg",
 "signed_url": "https://storage.googleapis.com/..." ,
 "content_type": "image/jpeg", "size_bytes": 12345,
 "comment": "CMNT_...", "note": null, "uploaded_by": 3,
 "created_at": "..."}
```

`signed_url` is `null` on ANY signing error (try/except → None).

### 1.2 `POST /notes/{note_pk}/comments/` — create

Auth: effectively authenticated (`AuthUser`). In prod IsAuthenticated
gates it; a truly-anonymous POST in test world hits
`serializer.save(user=AnonymousUser)` → 500. Rust returns 401 —
deliberate deviation, prod-identical.

Request:

```json
{"content": "text", "parent_comment": "CMNT_..." }
```

- `content`: required, non-blank (`allow_blank=False` — model
  `TextField` has no `blank=True`).
- `parent_comment`: optional, nullable. Must (a) exist → else
  `{"parent_comment": ["Invalid pk \"X\" - object does not exist."]}`;
  (b) have `note_id == {note_pk}` → else
  `{"parent_comment": ["parent_comment must belong to the same note."]}`.
- Replying to a soft-deleted comment is allowed (no check).
- Depth is unlimited.

Ordering: body validation (incl. parent lookups) runs BEFORE the
note-404 check in `perform_create` → `_get_note`. POST to a
nonexistent note: valid body → `404 {"detail": "Not found."}`;
invalid body → 400 first.

Response 201 — same flat comment shape, NO `replies` key,
`images: []`, `is_deleted: false`, `parent_comment` = raw PK or
null.

### 1.3 `GET /notes/{note_pk}/comments/{pk}/` — retrieve

Auth: `Principal`. `get_object` = queryset (`note_id={note_pk}`)
filtered `pk={pk}` → a comment on a DIFFERENT note → 404
`{"detail": "Not found."}`. Flat shape, NO `replies` key.

Quirk: a soft-deleted comment retrieved directly returns its real
DB `content` (`""` after DELETE) and its REAL `images` — the
`[deleted]`/`[]` redaction lives only in `build_comment_tree`.

### 1.4 `PATCH /notes/{note_pk}/comments/{pk}/` — partial update

Auth: `AuthUser` + author check. `comment.user != request.user` →
403 `{"detail": "You do not have permission to edit this
comment."}` (checked AFTER the 404 fetch — a comment on another
note is 404, not 403).

Writable fields: `content`, `parent_comment`, **`is_deleted`** —
the serializer does not mark `is_deleted` read-only, so the author
can soft-delete via PATCH (`{"is_deleted": true}`, content kept in
DB) and can RESTORE (`{"is_deleted": false}`) — restore does not
recover cleared content. `parent_comment` may be re-pointed to any
same-note comment (or `null` → becomes a root); the same-note
validator runs on PATCH too. Read-only: `id`, `author`, `note_id`,
`timestamp` (auto_now_add only — PATCH does NOT bump it; Comment
has no `updated_at`).

Response 200 — flat comment shape.

### 1.5 `DELETE /notes/{note_pk}/comments/{pk}/` — soft delete

Auth: `AuthUser` + author check → 403 `{"detail": "You do not have
permission to delete this comment."}`. Sets `is_deleted=true`,
`content=''` (row retained; replies and image rows survive).
Response `204` empty. Deleting an already-deleted comment → 204
(idempotent).

### 1.6 `PUT` — not supported

`http_method_names` excludes `put` → 405
`{"detail": "Method \"PUT\" not allowed."}` on BOTH collection and
detail URLs. HEAD/OPTIONS allowed (DRF metadata; axum gets these
free — OPTIONS body parity is best-effort).

### 1.7 `GET /comments/counts/` — CommentCountView

Auth: `Principal`. Query params (mutually exclusive):

- `tag_id={TAG...}` — every accessible note with this tag.
- `note_ids={id1,id2,...}` — comma-separated, ≤ 200 IDs
  (`NOTE_IDS_CAP`), whitespace stripped, empties dropped.
- `include_deleted=true` (case-insensitive; anything else = false).

Visibility = `get_accessible_notes_qs(user)`: authenticated → own
notes ∪ public; anonymous → public only. Inaccessible notes are
SILENTLY omitted (never 403).

Response 200: `{"counts": {"NOT_X": 3, "NOT_Y": 0}}` — every
accessible matching note appears, including zero counts (LEFT
JOIN). Object key order irrelevant.

400s — `ValidationError` raised in the VIEW (not a serializer) ⇒
DRF emits a **bare JSON array**, not `{"detail": …}`:

- both params: `["Provide either tag_id or note_ids, not both."]`
- neither: `["One of tag_id or note_ids is required."]`
- >200 ids: `["note_ids may contain at most 200 IDs."]`

Edge: `note_ids=",,,"` → `has_ids` true, parsed list empty →
`{"counts": {}}` (200). Empty-string `tag_id`/`note_ids` count as
absent (`bool('')` is False).

Standard bodies for all routes: 401 no-creds `{"detail":
"Authentication credentials were not provided."}` (prod only);
403/404/405 `{"detail": …}`; serializer 400 = field→`["msg"]` map.

## 2. Current Django behavior trace

### CommentViewSet.list (GET tree)

`get_queryset()` (views.py:793-806) — ONE query for comments +
authors via `select_related('user')` (INNER JOIN), plus ONE
prefetch query for images:

```sql
SELECT c.*, u.*
FROM annotations_comment c
INNER JOIN auth_user u ON u.id = c.user_id
WHERE c.note_id = %s
ORDER BY c.timestamp ASC;
-- prefetch_related('images'):
SELECT * FROM annotations_image
WHERE comment_id IN (...) ORDER BY created_at DESC;
```

Then `build_comment_tree` (serializers.py:685-720) — pure-Python
O(n), two passes over the ordered rows:

1. Serialize each comment (flat dict); if `is_deleted` →
   `content='[deleted]'`, `images=[]`; add `replies: []`; index in
   `comment_map[id]`.
2. Second pass: `parent_comment_id` set and found in map → append
   node to `parent['replies']`; parent set but NOT found → dropped;
   no parent → append to `roots`. Return `roots`.

Query count: 2 DB queries total (comments+users JOIN, images IN).
`len(queryset)` in the log line reuses the evaluated queryset.

### retrieve / partial_update / destroy

`get_object()` filters the SAME queryset by `pk` → 404 when missing
or on another note. `perform_update`/`perform_destroy` then compare
`instance.user` to `request.user` → `PermissionDenied` → 403.
Destroy issues `UPDATE annotations_comment SET is_deleted=true,
content=''` (update_fields). PATCH issues UPDATE of changed fields.

### create

`is_valid` → field-level `parent_comment` fetch + same-note check
→ `perform_create` → `_get_note` (`SELECT … FROM annotations_note
WHERE id=%s` → 404) → `INSERT` with generated `CMNT_` id,
`timestamp` = `auto_now_add` (Django-side `now()`), `user_id` =
request user, `note_id` = URL note.

### CommentCountView.get

Exactly ONE aggregate query (plus whatever auth/provisioning does):

```sql
SELECT n.id,
       COUNT(c.id) FILTER (WHERE c.is_deleted = FALSE) AS cnt
FROM annotations_note n
LEFT OUTER JOIN annotations_comment c ON c.note_id = n.id
WHERE (n.user_id = %s OR n.public = TRUE)   -- anon: n.public=TRUE
  AND n.tag_id = %s                          -- or: n.id IN (%s,…)
GROUP BY n.id;
```

`include_deleted=true` drops the FILTER → plain `COUNT(c.id)`.
On SQLite tests Django renders `COUNT(CASE WHEN … THEN 1 END)`;
Postgres prod uses `FILTER` — we write FILTER.

`annotations/tests.py:663` `test_single_query_assertion` is
`@expectedFailure`: it asserts exactly 2 queries but the endpoint
runs exactly 1 under `force_authenticate` (no auth/session query).
The 1-query aggregate IS the contract; the test is aspirational —
do not chase the count, document parity instead.

## 3. Rust implementation plan

Module: `crates/server/src/routes/annot/comments.rs` (+ counts
handler alongside or `comments/counts.rs`); DTOs beside handlers
per §6 conventions.

### Routes (in `routes/mod.rs`, under `/api/v1`)

```text
.route("/notes/{note_pk}/comments",
       get(list_comments).post(create_comment))
.route("/notes/{note_pk}/comments/{pk}",
       get(get_comment).patch(patch_comment).delete(delete_comment))
.route("/comments/counts", get(comment_counts))
```

- axum `Path<(String, String)>` extracts `note_pk`/`pk`; axum 0.8
  `{param}` syntax matches DRF `[^/.]+`-style single segments.
- PUT absent → axum 405; set `method_not_allowed_fallback` (router-
  level, §4.4 body `{"detail": "Method \"PUT\" not allowed."}`) —
  core/shared concern, flag to the base agent if missing.
- Register `comments/counts` BEFORE any `/comments/{pk}`-shaped
  route (none exist, but keep it a static segment anyway).

### DTOs

```rust
struct CommentAuthorDto { id: i64, username: String }
struct CommentDto {
    id: String, author: CommentAuthorDto, note_id: String,
    parent_comment: Option<String>, content: String,
    timestamp: DateTime<Utc>,           // RFC3339 µs Z
    is_deleted: bool, images: Vec<ImageDto>,
}
struct CommentNodeDto {
    #[serde(flatten)] base: CommentDto,
    replies: Vec<CommentNodeDto>,
}
struct CommentCreateReq { content: String,
                          parent_comment: Option<String> }
struct CommentPatchReq { content: Option<String>,
    parent_comment: Option<Option<String>>,   // absent vs null
    is_deleted: Option<bool> }
struct CountsQuery { tag_id: Option<String>,
    note_ids: Option<String>, include_deleted: Option<String> }
struct CountsResp { counts: Map<String, i64> }
```

`ImageDto` is shared with the images plan (#13/14/16) — import
from `routes::annot::images` (or `core` if both agents need it
first; coordinate).

### Handlers

`list_comments(State, Principal, Path(note_pk))`:

1. One `query_as` — explicit column list, JOIN `auth_user`,
   `WHERE c.note_id = $1 ORDER BY c.timestamp ASC`.
2. One images query — `WHERE comment_id = ANY($1)`; group rows by
   `comment_id`, sort each group `created_at DESC`, build
   `ImageDto`s incl. `signed_image_url` (storage crate; error →
   `null`, same try/except→None contract).
3. In-memory assembly mirroring `build_comment_tree` exactly:
   `HashMap<String, usize>` id→index into a `Vec<Node>` arena;
   second pass appends child indexes to parent.replies (order =
   row order = timestamp ASC); missing parent → skip (orphan drop);
   collect roots in order; serialize `Vec<CommentNodeDto>`.
4. `is_deleted` rows → `content="[deleted]"`, `images=[]` BEFORE
   tree attach.

`create_comment(State, AuthUser, Path(note_pk), Json(req))`:

- Validate IN DJANGO ORDER: `content` present/non-blank → 400
  `{"content": ["This field is required."]}` / `["This field may
  not be blank."]`; `parent_comment` → `SELECT note_id FROM
  annotations_comment WHERE id=$1` → miss: 400 `{"parent_comment":
  ["Invalid pk \"{id}\" - object does not exist."]}`; `note_id !=
  note_pk` → 400 same-note field error.
- THEN `SELECT id FROM annotations_note WHERE id=$1` → 404
  `{"detail": "Not found."}`.
- `INSERT` (`core::ids::comment_id()` = `CMNT_` + 13 upper hex;
  `timestamp` = `Utc::now()`; `is_deleted=false`). Return 201
  `CommentDto` (author = caller, `images=[]`; `replies` absent).

`get_comment / patch_comment / delete_comment`:

- Shared fetch: `SELECT … WHERE note_id=$1 AND id=$2` → 404.
- patch/delete: `AuthUser`; `row.user_id != user.id` → 403 with
  the exact `detail` strings from §1.4/§1.5.
- patch: apply only provided fields; re-run the `parent_comment`
  existence+same-note validation when the key is present and
  non-null; `null` clears it (becomes root); allow `is_deleted`
  both directions (parity quirk); `content` non-blank if present.
  `UPDATE … SET col=$…` (dynamic SET or full-row update of the
  three mutable cols); return 200 `CommentDto`.
- delete: `UPDATE … SET is_deleted=true, content=''` → `204`
  (empty body, `StatusCode::NO_CONTENT`).

`comment_counts(State, Principal, Query(q))`:

- Parse exactly as Django: `has_tag = tag_id.is_some_and(|s| !s
  .is_empty())`; same for `note_ids`; `include_deleted` =
  `eq_ignore_case("true")`.
- Both → 400 `["Provide either tag_id or note_ids, not both."]`;
  neither → 400 `["One of tag_id or note_ids is required."]`;
  split `,`→trim→drop empty; >200 → 400 cap message. These need a
  `ApiError` variant emitting a bare `["msg"]` array — ADD
  `ApiError::ValidationList(Vec<String>)` (or similar) to
  `core::errors` — the existing `Validation(fields)` map shape
  does NOT match this endpoint.
- Single query (§2 SQL). `WHERE` arm chosen by auth: authenticated
  → `(n.user_id = $1 OR n.public)`; anonymous → `n.public`.
  `tag_id` arm → `n.tag_id = $2`; `note_ids` arm → `n.id =
  ANY($2)`. `FILTER (WHERE NOT c.is_deleted)` unless
  include_deleted. GROUP BY n.id. Fold rows into `{"counts":{…}}`.

### Query counts (parity with §2)

- list: 2 (comments+users JOIN; images ANY) — same count
  discipline as `select_related`+`prefetch_related`.
- detail/patch/delete: 1 fetch + 1 write.
- create: parent check (≤1) + note check (1) + INSERT (1).
- counts: 1 aggregate.

### OpenAPI

`#[utoipa::path]` on all 6 handlers; `CommentNodeDto` needs a
recursive `ToSchema` (use `schema(no_recursion)`-style boxing or a
documented `replies: Vec<serde_json::Value>` fallback — utoipa
handles recursive schemas via `Ref` fine; verify in build).

## 4. Test plan

Mirror `annotations/tests.py` classes → `tests/annot_comments.rs`
with `AUTO_PROVISION_USERS=false` + seeded users/tokens.

Unit (pure):
- `build_comment_tree` port: empty→`[]`; single root;
  3-level nesting; deleted → `[deleted]` + `images:[]` + replies
  kept; multiple roots ordered by timestamp; orphan reply dropped.
- `core::ids::comment_id` → `CMNT_` prefix, len ≤ 18.
- Counts param parsing: both/neither/cap/empty-parts/
  `include_deleted` case variants.

HTTP (axum-test + scratch Postgres):
- `CommentViewSetTest` parity: POST top-level 201 (CMNT_ prefix,
  note_id, parent null); POST reply 201; GET list nested tree;
  DELETE 204 + row `is_deleted` + replies retained; non-owner
  DELETE 403; non-owner PATCH 403; cross-note parent 400;
  nonexistent note POST 404 (valid body) vs 400 (invalid body,
  ordering!); missing/blank `content` 400; nonexistent
  `parent_comment` pk 400 exact message; PUT 405; GET list on
  unknown note → `200 []`; GET detail cross-note → 404; GET detail
  of deleted comment → `content:""` NOT `[deleted]`; PATCH
  `is_deleted`/`parent_comment` quirks (author only).
- `CommentCountViewTest` parity: tag_id happy (2 counts correct);
  note_ids happy (only requested keys); zero-count present;
  deleted excluded by default; `include_deleted=true`; anonymous →
  public only; owner sees own private; other-user private omitted
  (200, key absent); neither → 400 bare-array body; both → 400;
  201 ids → 400; cross-tag isolation.
- Query-count guard: instrument the test pool (log/assert
  `sqlx` query count) for list = 2 and counts = 1 — replaces the
  known-failing `test_single_query_assertion`.
- Parity-corpus entries: tree list with mixed deleted/live
  nesting; counts `tag_id` incl. a zero-count note; 400 bodies
  verbatim.

## 5. Open questions / risks

- **Bare-array 400 bodies** for view-raised `ValidationError` —
  derived from DRF `exception_handler` (`exc.detail` list is used
  verbatim); confirm against live Django in the parity run before
  shipping `ValidationList`.
- **Missing visibility checks (security, reproduce-for-parity)**:
  `get_queryset` ignores note accessibility — any authenticated
  user can READ comments on any private note and POST replies to
  notes they cannot see. `CommentCountView` DOES scope via
  `get_accessible_notes_qs` — asymmetric. Reproduce exactly in v1;
  file a follow-up to align comment access with note visibility
  (likely intended: private-note comments = note-owner-visible or
  author-visible).
- **Anonymous POST** maps to 401 (`AuthUser`) vs Django's
  theoretical 500 — deliberate deviation; prod behavior identical.
- **Sequencing**: comments land in Phase 2 (base doc §5) but
  `ImageDto.signed_url` needs the storage crate (Phase 4).
  Mitigations: `signed_url: null` is already a legal value (Django
  emits it on ANY signing error), so Phase 2 may ship a stub
  signer → `null` and wire the real signer later; or pull the
  signed-URL helper forward. Coordinate with the images agent.
- **`author.id` type**: `BigAutoField` ⇒ `i64`. If any environment
  ever ran `AutoField`, values still fit — keep `i64`.
- **PATCH quirk surface** (`is_deleted` writable, reparenting
  allowed, `timestamp` immutable): verify frontend doesn't depend
  on the quirks before any future tightening; keep parity now.
- **OPTIONS/HEAD**: DRF metadata responses are not replicated;
  acceptable per §4.13 best-effort stance.
- **`test_single_query_assertion` stays failing** upstream by
  design — our single-aggregate-query implementation satisfies the
  intent; do not "fix" Django.
