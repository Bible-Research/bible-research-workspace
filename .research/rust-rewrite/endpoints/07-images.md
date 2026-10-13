# 07 — Images (inventory items 13, 14, 16)

Status: PLAN. Scope: `NoteImageViewSet` (13), `CommentImageViewSet`
(14), `ImageDestroyView` (16). Sources (paths relative to
`bible_research/`):

- `annotations/views.py` L987–1237 (three views),
  L46–56 (`get_accessible_notes_qs`)
- `annotations/serializers.py` L577–619 (`ImageSerializer`),
  L643/L685–720 (inline `images` on comment tree — comments plan,
  reused here only via `signed_image_url`)
- `annotations/models.py` L262–359 (`Image`, `generate_image_id`)
- `annotations/services/image_storage.py` (whole file)
- `annotations/urls.py` L18–40 (router registrations)
- `bible_research/settings.py` L247–275 (image settings)
- `annotations/test_images.py` (whole file)
- `bible/services/storage/gcs.py` L158–213 (signed-URL precedent)

## 1. Endpoint contracts

Field order below matches DRF serializer `fields` order; keep it in
the Rust DTO for readability.

### 1a. `GET /api/v1/notes/{note_pk}/images/`

- Auth: `tok` — `Principal` extractor (DRF `AllowAny`; anonymous
  allowed through to the view).
- Checks, in order:
  1. Note exists → else 404 `{"detail": "Not found."}`.
  2. Note is accessible (`user_id == me OR public`; anon →
     `public` only) → else 403
     `{"detail": "You do not have access to this note."}`.
- 200 body: bare JSON array (no pagination wrapper), ordered by
  `created_at DESC`:

  ```json
  [{
    "id": "IMG_XXXXXXXXXXXXX",
    "storage_url": "gs://<bucket>/originals/<id>/source.png",
    "signed_url": "https://storage.googleapis.com/..." ,
    "content_type": "image/png",
    "size_bytes": 100,
    "comment": null,
    "note": "NOTXXXXXXXXXXXXXXX",
    "uploaded_by": 7,
    "created_at": "2025-12-28T03:51:55.123456Z"
  }]
  ```

- `signed_url`: `string | null` — null whenever signing fails (no
  creds, bad `storage_url`); the serializer docstring's
  "falls back to storage_url" claim is stale — code returns null.
- `comment` is `null` for note images; `uploaded_by` is the
  integer `auth_user.id`, nullable (`SET_NULL`).

### 1b. `POST /api/v1/notes/{note_pk}/images/`

- Auth: `auth!` — `AuthUser` extractor (DRF `IsAuthenticated`).
  NOTE: prod Django → 401 `{"detail": "Authentication credentials
  were not provided."}` + `WWW-Authenticate: Token`; the Django
  TEST env → 403 with the same body (first auth class is
  `SessionAuthentication`, which sets no challenge header; DRF
  only consults authenticators[0] — see §5). Rust emits 401 in
  both modes; document as a deliberate deviation in parity runs.
- Request: `multipart/form-data` with a `file` part. DRF counts a
  part as a file only when `Content-Disposition` carries a
  `filename`; a text field named `file` does NOT count. Multiple
  `file` parts: Django `request.FILES.get` returns the LAST one.
- Non-multipart Content-Type → 415
  `{"detail": "Unsupported media type \"<ct>\" in request."}`
  (DRF parser rejection, not the view's own 415).
- Check order (exact; each listed error aborts):
  1. `AuthUser` (permission layer runs BEFORE note lookup).
  2. Note exists → 404 `{"detail": "Not found."}`.
  3. `note.user == user` → 403
     `{"detail": "Only the note's owner may upload images."}`
     (anonymous never reaches this — permission layer got it).
  4. `file` part present → 400 `{"file": ["No file provided."]}`.
  5. Cheap cap check: `COUNT(images WHERE note_id) >=
     IMAGE_MAX_PER_NOTE` (10) → 400
     `{"file": ["Maximum 10 images per note."]}`.
  6. GCS-side validation inside `upload_original`:
     a. part Content-Type ∈ {image/jpeg, image/png, image/webp,
        image/gif} → else 415 `{"detail": "Unsupported image type
        '<ct>'. Allowed: image/jpeg, image/png, image/webp,
        image/gif."}` (allowlist order fixed: dict insertion
        order, NOT `settings.IMAGE_ALLOWED_CONTENT_TYPES` — that
        setting is dead code).
     b. size > `IMAGE_MAX_BYTES` (10 MiB default) → 413
        `{"detail": "File too large (<n> bytes). Maximum allowed:
        <max> bytes."}` — the *custom* message wins, NOT
        `FileTooLarge.default_detail` ("File exceeds maximum
        allowed size."). Base doc §4.9 gets this wrong; correct it
        or override locally.
     c. Pillow `verify()` fails on content → 415 `{"detail":
        "File content is not a valid image. Allowed: image/jpeg,
        image/png, image/webp, image/gif."}`.
  7. Object written to GCS, then txn: `SELECT … FOR UPDATE` the
     note row → recount → same 400 cap error → `INSERT` image row.
     ANY error inside the txn (incl. the cap 400) → best-effort
     GCS delete of the just-uploaded object, then propagate.
  8. GCS upload/network failure → 500 (no special mapping today).
- 201 body: the same single-image object as 1a (not a list).

### 1c. `GET /api/v1/notes/{note_pk}/comments/{comment_pk}/images/`

- Auth: `tok`. Checks: comment `pk AND note_id` must match → 404
  `{"detail": "Not found."}` (comment under a different note →
  404, not 403). Then accessibility of the PARENT NOTE → 403
  `{"detail": "You do not have access to this comment."}`.
- 200: bare array, `-created_at`, same fields; `note: null`,
  `comment: "CMNT_…"`.
- NO `is_deleted` filter: images on soft-deleted comments are
  still listed here (the comment TREE redacts them — different
  code path).

### 1d. `POST /api/v1/notes/{note_pk}/comments/{comment_pk}/images/`

- Auth: `auth!`. Same flow as 1b with these diffs:
  - Permission check: `comment.user == user` → 403 `{"detail":
    "Only the comment's author may upload images."}` (NOTE:
    comment AUTHOR, not note owner).
  - Cap: `IMAGE_MAX_PER_COMMENT` (5); messages
    `"Maximum 5 images per comment."` (both occurrences).
  - Row lock is on the COMMENT row; insert sets `comment_id`,
    `note_id = NULL`.

### 1e. `DELETE /api/v1/images/{image_pk}/`

- Auth: `auth!` (`IsAuthenticated`; same 401-vs-403 nuance as 1b).
- Checks:
  1. Image exists → 404 `{"detail": "Not found."}`.
  2. `image.uploaded_by == user` OR (`image.note` non-null AND
     `image.note.user == user`) OR (`image.comment` non-null AND
     `image.comment.user == user`) → else 403 `{"detail": "You do
     not have permission to delete this image."}`.
     Quirk to preserve: for a comment image, `image.note` is NULL
     (XOR constraint), so the NOTE owner cannot delete a comment
     image on their own note — only the uploader or the comment
     author can.
- On success: delete DB row, then best-effort GCS delete
  (all errors swallowed, incl. malformed `storage_url`), →
  204 empty body. GCS failure does NOT change the response.

### 1f. Router artifacts worth noting

- DRF `DefaultRouter` also registers detail routes
  `…/images/{pk}/` on both viewsets, mapped to
  retrieve/update/destroy — none exist on the GenericViewSets →
  EVERY method (incl. GET/POST) returns 405
  `{"detail": "Method \"<M>\" not allowed."}`. Cheapest parity:
  register a `fallback` on `…/images/{pk}` returning 405 with that
  body; skipping it entirely yields 404 — flag in the parity
  corpus if we skip.
- `PUT`/`PATCH`/`DELETE` on the collection URLs → 405 same shape.

## 2. Current Django behavior trace

Upload (`views.py` L1028–1089 note / L1133–1195 comment):

1. `get_object_or_404` on the parent; ownership/author check.
2. `request.FILES.get('file')` — absent → 400 field error.
3. Cheap `COUNT(*) >= max` pre-check (comment: "Cheap pre-check
   before the expensive GCS round-trip" L1041–1042).
4. `generate_image_id()` → `"IMG_"` + first 13 chars of
   `uuid4().upper().replace('-','')` → `IMG_` + 13 upper-hex
   (`models.py` L262–263).
5. `upload_original(image_id, file)` (`image_storage.py` L97–165):
   - `content_type` from the uploaded file's own attribute
     (client-supplied part header); `""` if missing → fails
     allowlist → 415.
   - `seek(END)`/`tell()` → `size_bytes`; `> IMAGE_MAX_BYTES` →
     `FileTooLarge` (413, custom message per §1b.6b).
   - `PILImage.open(file).verify()` — failure → 415 "not a valid
     image". NOTE: Pillow auto-detects format from magic bytes and
     IGNORES the declared content-type for validation — a JPEG
     declared `image/png` passes and is stored as `source.png`
     with `content_type=image/png`. Keep this quirk (see §5).
   - Object name: `originals/{image_id}/source{ext}` where ext =
     `.jpg|.png|.webp|.gif` by content-type; `upload_from_file`
     with `content_type`; returns `gs://{bucket}/{object}`,
     `size_bytes`, `content_type`.
6. `transaction.atomic()`: `select_for_update().get(pk)` on the
   parent row → recount → `Image.objects.create(...)`. Any
   exception → `delete_original(image_id, gs_uri)` (swallows its
   own errors) then re-raise.
7. 201 with `ImageSerializer` output.

Image row columns (`annotations_image`): `id` varchar(18) PK,
`storage_url` varchar(1024), `content_type` varchar(64) (blank ''),
`size_bytes` integer NULL, `comment_id`/`note_id` FK (XOR via
CHECK `image_exactly_one_parent`), `uploaded_by_id` int FK NULL
(`SET_NULL`), `created_at` timestamptz (`auto_now_add` —
Django supplies the timestamp client-side; Rust must too).

Signed URL (`signed_image_url`, `image_storage.py` L168–240):
parse `gs://bucket/object` from `storage_url`; V4 GET signed URL,
TTL `IMAGE_SIGNED_URL_TTL_SECONDS` (600); SA email + OAuth access
token → IAM `signBlob` (scopes `devstorage.read_only`, `iam`).
Module-level LRU: key `storage_url`, cap 2048, expiry =
`TTL − 60 s` margin, `move_to_end` on hit. ANY failure → `None`
(logged) → serialized `signed_url: null`. Same cache/margin
pattern as audio (`gcs.py` L158–213) but keyed differently —
audio: `(fileset, voice, book, chapter)`; images: `storage_url`.

Delete (`views.py` L1209–1237): `get_object_or_404` → three-way
permission OR → `image.delete()` (single `DELETE` SQL) →
`delete_original` best-effort → 204. NOT wrapped in a txn; a GCS
failure still yields 204 and the row is already gone.

Delete helper (`image_storage.py` L243–267): parses
`gs://bucket/object`, `blob.delete()`; malformed URI and all GCS
errors tolerated (logged).

Settings (`settings.py` L247–275): `IMAGE_BUCKET_ORIGINALS`
(default `{project}-images-originals`), `IMAGE_BUCKET_THUMBNAILS`
(`{project}-images-thumbnails` — UNUSED today; no thumbnail code),
`IMAGE_MAX_BYTES` (10 MiB), `IMAGE_SIGNED_URL_TTL_SECONDS` (600),
`IMAGE_MAX_PER_NOTE` (10), `IMAGE_MAX_PER_COMMENT` (5),
`IMAGE_ALLOWED_CONTENT_TYPES` (dead — real allowlist is the
`_ALLOWED_EXTENSIONS` dict L36–41). Views use `getattr(settings,
…, 10/5)` fallbacks — Config fields suffice.

Read permission source: `get_accessible_notes_qs` (`views.py`
L46–56): authenticated → `Q(user=me) | Q(public)`; anon →
`public` only.

## 3. Rust implementation plan

Crate placement per base §3/§6:

- `crates/storage/src/images.rs` — port of `image_storage.py`:
  `ImageStorage` trait `{ upload_original, delete_original,
  signed_image_url }`, `GcsImageStorage` impl on
  `google-cloud-storage` + `google-cloud-auth` `Signer`, plus
  `FakeImageStorage` (in-memory, for handler tests). Signed-URL
  LRU: `Mutex<` small LRU `>` — hand-roll `HashMap`+`VecDeque` or
  `lru` crate; key `storage_url`, cap 2048, TTL−60s via
  `Instant`. Share the signer/credentials plumbing with the audio
  module (`storage::gcs` equivalent) — one `google-cloud-auth`
  `Signer`, cached token.
- `crates/server/src/routes/annot/images.rs` — handlers + DTOs.
  Register in `routes/mod.rs`:
  - `GET/POST /api/v1/notes/{note_pk}/images/`
  - `GET/POST
    /api/v1/notes/{note_pk}/comments/{comment_pk}/images/`
  - `DELETE /api/v1/images/{image_pk}/`
  - `#[utoipa::path]` entries; multipart request bodies via
    `utoipa` `request_body(content = …, content_type =
    "multipart/form-data")`.

### DTO

```text
ImageDto { id, storage_url, signed_url: Option<String>,
           content_type, size_bytes: Option<i64>,
           comment: Option<String>, note: Option<String>,
           uploaded_by: Option<i64>, created_at: DateTime<Utc> }
```

`created_at` → RFC 3339 micros `Z` (base §4.3). All serde
snake_case, no renames.

### Handlers

`list_note_images(State, Path(note_pk), Principal)`:

```sql
-- exists + accessibility in one round trip:
SELECT EXISTS(SELECT 1 FROM annotations_note WHERE id = $1),     -- note_exists
       EXISTS(SELECT 1 FROM annotations_note WHERE id = $1
              AND (user_id = $2 OR public));                     -- accessible
```

Or two queries to mirror Django exactly (fetch note 404 first —
needed anyway for the 404-vs-403 distinction). For anon, the
`user_id = $2` term is simply never bound (skip it / bind NULL —
`user_id = NULL` is never true, matching `public`-only semantics).

Then:

```sql
SELECT id, storage_url, content_type, size_bytes, comment_id,
       note_id, uploaded_by_id, created_at
FROM annotations_image WHERE note_id = $1
ORDER BY created_at DESC;
```

`list_comment_images`: same, but first
`SELECT note_id, user_id FROM annotations_comment
WHERE id = $1 AND note_id = $2` (404 on miss), accessibility on
`note_id`, then `WHERE comment_id = $1`.

`create_*` (one generic fn parameterized on parent kind):

1. `AuthUser` extractor → `user`.
2. Parent fetch (404) + owner/author check (403) — before
   touching the body, matching Django where `request.FILES` is
   lazily evaluated but the view reads `.get('file')` only after
   the owner check. Multipart parsing happens first in axum
   though (extractor order) — that's fine: Django parses the body
   during `request.FILES` access AFTER the checks too, and the
   observable contract (status code per bad request) is identical
   since an absent `file` only matters when earlier checks pass.
   EXCEPT content-type 415: DRF rejects non-multipart BEFORE
   running the view (parser selection precedes permission checks).
   So: non-multipart POST → 415 even for anon/non-owner. Handle by
   extracting `Multipart` conditionally: read `Content-Type`
   header manually; if not `multipart/form-data` → 415 body
   `{"detail": "Unsupported media type \"<ct>\" in request."}`.
   Then run auth/parent/owner checks, THEN walk fields.
3. Extract `file` field: iterate `Multipart::next_field()`,
   keep the LAST field where `name() == "file"` AND
   `file_name().is_some()` (DRF filename rule; see §1b). Buffer
   via `field.chunk()` into `Vec<u8>` capped at
   `IMAGE_MAX_BYTES + 1`; exceeding → remember `oversized` and
   stop reading (can abort the rest of the stream). Grab
   `field.content_type()` (`""` if absent). No file → 400
   `{"file": ["No file provided."]}`.
4. Pre-cap `SELECT COUNT(*) …` → 400 cap error.
5. `storage.images.upload_original(image_id, bytes, content_type)`:
   - allowlist map → `ext`; else `ApiError::UnsupportedMediaType`
     415 with the exact detail strings from §1b.6a/6c.
   - `size > IMAGE_MAX_BYTES` → 413 (custom message §1b.6b).
     Note order: content-type check precedes size check (match:
     a 20 MiB `application/pdf` → 415, not 413).
   - decode check: `image::ImageReader::new(Cursor::new(&bytes))
     .with_guessed_format()` → `.decode()`; any error → 415.
     Do NOT compare decoded format to declared type (§5 quirk).
   - PUT object `originals/{image_id}/source{ext}` with
     `content_type`; return `gs://` URI + size + content_type.
   - `image_id` via `core::ids` (`IMG_` + 13 upper-hex, base §4.3).
6. `pool.begin()`: `SELECT id FROM annotations_note WHERE id=$1
   FOR UPDATE` (or `annotations_comment`), recount, then

   ```sql
   INSERT INTO annotations_image
     (id, storage_url, content_type, size_bytes, comment_id,
      note_id, uploaded_by_id, created_at)
   VALUES ($1,$2,$3,$4,$5,$6,$7,$8);
   ```

   Commit. On ANY error (cap 400, sqlx error): call
   `delete_original(image_id, &gs_uri)` (best-effort — never
   propagate its error), then return the original error.
7. 201 `Json(ImageDto)` — sign the URL for the response (calls
   `signed_image_url`; null on failure).

`delete_image(State, Path(image_pk), AuthUser)`:

```sql
SELECT i.id, i.storage_url, i.uploaded_by_id,
       n.user_id AS note_owner_id, c.user_id AS comment_author_id
FROM annotations_image i
LEFT JOIN annotations_note n    ON n.id = i.note_id
LEFT JOIN annotations_comment c ON c.id = i.comment_id
WHERE i.id = $1;
```

One query covers all three permission arms (Django lazy-loads the
FKs — same result, one round trip). Perm OR → 403. Then
`DELETE FROM annotations_image WHERE id = $1`; then
`delete_original`; 204 (empty body, no JSON).

### Error mapping (`ApiError` additions)

- `Validation(fields: map)` → 400 `{"file": ["…"]}` (DRF wraps
  scalar detail into a list — reproduce: always a JSON array
  under `file`).
- `PayloadTooLarge(detail)` → 413 `{"detail": "…"}`.
- `UnsupportedMediaType(detail)` → 415 `{"detail": "…"}` — also
  used for the non-multipart-parser rejection.
- `Forbidden(detail)` / `NotFound` — existing §4.4 shapes.
- Storage-layer `Err` on upload (network/quota) → 500; Django has
  no mapping — propagate as internal error, log via tracing. Do
  NOT leak GCS error bodies (may contain bucket/object paths —
  same rule as §4.4 for provider URLs).

## 4. Test plan

Mirror `annotations/test_images.py` 1:1 as `tests/annot_images.rs`
(axum-test + scratch Postgres + `AUTO_PROVISION_USERS=false`,
per base §4.15/§6):

Model/unit-level (no HTTP):
- `IMG_` prefix, ≤18 chars (id gen).
- XOR check constraint: both-parents and neither-parent inserts
  fail (`is_check_violation` / constraint name
  `image_exactly_one_parent`).
- `upload_original` unit tests against `FakeImageStorage`
  plumbing + a real `image`-crate decode check: allowed type
  happy path → `gs://` uri/size/content_type; disallowed type →
  415; `max+1` bytes → 413; non-image bytes declared `image/png`
  → 415; object path convention `originals/<id>/source.png`.
  Reuse the `_SMALL_PNG` 1×1 fixture bytes verbatim (Pillow-11-
  strict-valid PNG also decodes under `image`).

HTTP-level (DB seeded via SQL; `FakeImageStorage` records calls):
- note happy path: 201, `id` starts `IMG_`, row count 1.
- non-owner POST → 403, no row.
- `application/pdf` → 415; `IMAGE_MAX_BYTES+1` → 413; spoofed
  bytes → 415; no `file` field → 400 `{"file": ["No file
  provided."]}`; text field named `file` (no filename) → 400.
- anon POST → 401 (Rust; Django test says 403 — deviation,
  assert 401 + note in parity entry).
- cap: seed `IMAGE_MAX_PER_NOTE` rows → next upload 400; comment
  cap 5 likewise.
- list: 200 array len 1; other user's private note → 403; anon on
  private → 403; anon on public → 200 len 1.
- comment images: author happy path 201 + `comment_id` set,
  `note` null; non-author → 403; comment under different
  `note_pk` → 404; images on soft-deleted comment still listed.
- delete: anon → 401; non-owner → 403 row survives; owner → 204,
  row gone, `FakeImageStorage::delete` called once; uploader-vs-
  note-owner-vs-comment-author each accepted; note owner CANNOT
  delete another author's comment image (403) — quirk test.
- ordering: two images → `-created_at` order in list.
- `signed_url` null when fake returns `None`.
- GCS-failure-on-upload → 500 (fake returns Err).
- GCS-failure-on-delete → still 204.

GCS mocking choice: `FakeImageStorage` behind the trait for all
handler tests (mirrors Django's `mock.patch` of
`_originals_bucket`/`_get_client`/`signed_image_url`). wiremock is
only worth it as a smoke test if the `google-cloud-storage` crate
supports a custom endpoint — investigate; NOT required for parity
since no client-visible behavior depends on GCS wire details.
Signed-URL correctness is verified in a storage-crate unit test
asserting the V4 URL shape/params produced by `Signer` against a
recorded signBlob response (offline signature verification is
possible: V4 signature is deterministic given fixed creds+time —
feed `Signer` a fake `signBlob` and check `X-Goog-*` params).

Parity corpus entries (`tests/parity/`): list/empty-list,
upload happy path (normalize `id`, `storage_url`, `signed_url`,
`created_at`), each error branch (400/403/404/413/415 bodies
byte-compared), delete 204 + 403/404. Normalize the anon-write
401-vs-403 case explicitly.

## 5. Open questions / risks

- **401 vs 403 for anon writes.** Prod Django → 401
  (`TokenAuthentication` first, sets `WWW-Authenticate`); test
  Django → 403 (`SessionAuthentication` first, DRF checks only
  `authenticators[0].authenticate_header`). `AuthUser` emits 401
  always — matches prod; `test_images.py` asserts 403/401-∨-403
  because it runs in the test auth world. Deliberate deviation;
  keep, and normalize in parity runs.
- **413 detail message.** Base §4.9 states the body is
  `{"detail": "File exceeds maximum allowed size."}` — that is
  `FileTooLarge.default_detail`, never used; the call site passes
  `"File too large ({size} bytes). Maximum allowed: {max}
  bytes."` (`image_storage.py` L129–132). Plan uses the actual
  message. Worth fixing the base doc.
- **Format/content-type mismatch.** Pillow `verify()` auto-detects
  the real format; a JPEG uploaded as `image/png` succeeds and is
  stored as `source.png`. `image::decode` likewise ignores the
  declared type. Keep parity (accept); note for a follow-up
  hardening flag (would break parity, so gate behind config if
  ever wanted). Also `image`-crate full decode is STRICTER than
  Pillow `verify()` (e.g. truncated IDAT may verify under Pillow
  but fail full decode) — accept the small drift; only the
  "clearly not an image" case is contract-tested.
- **Multipart streaming.** Django buffers the upload to disk then
  measures size; axum `Multipart` streams. We accumulate with a
  `IMAGE_MAX_BYTES+1` cap — fine up to 10 MiB. Ensure no
  `DefaultBodyLimit`/`RequestBodyLimit` rejects earlier with a
  different status (tower-http body-limit 413 body shape differs
  — avoid or match DRF 415/413 shapes explicitly).
- **Detail routes → 405.** Router-artifact routes (`…/images/{pk}/`)
  exist in Django and return 405 for every method. Register a
  405 fallback or accept 404 — decide; low risk either way.
- **`IMAGE_BUCKET_THUMBNAILS` unused.** No thumbnail pipeline
  exists; config carries the name for future use only. Don't
  invent thumbnails.
- **`IMAGE_ALLOWED_CONTENT_TYPES` dead in Django.** Real allowlist
  lives in `image_storage.py`; keep one Rust source of truth but
  note the setting is config-shaped anyway (`Config` may expose it
  later — today hardcode jpeg/png/webp/gif → `.jpg/.png/.webp/
  .gif` order-preserved).
- **Orphaned GCS objects.** Note/comment deletion (other plans)
  cascades image ROWS but nothing deletes the GCS objects —
  pre-existing leak; flag for the notes/comments plan owner, do
  not fix here.
- **Concurrent uploads beyond cap.** The `FOR UPDATE` row lock
  + recount is the authoritative guard — port verbatim, including
  the pre-check (avoids the GCS write for the common over-cap
  case).
- **Decode limits.** `image` crate default `Limits` cap
  dimensions/allocations; a maliciously large-dimensioned but
  small-byte PNG could be rejected by limits where Pillow
  accepts. Configure generous limits (or `Limits::no_support`)
  after size-check to match Django's accept-anything-verifiable
  stance; document choice.
