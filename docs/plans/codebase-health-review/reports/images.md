# images (Note/CommentImageViewSet + ImageDestroyView) — code review

## Summary

Upload validation is genuinely solid: MIME allowlist plus PIL
magic-byte verification, a size cap enforced before the GCS
round-trip, and a per-parent count cap re-checked under a row lock
with a rollback that deletes the just-uploaded blob. Ownership and
guest read-only handling are correctly enforced. The significant gap
is the storage lifecycle: every delete path other than the dedicated
`DELETE /images/<pk>/` endpoint (note/tag/user cascades) removes the
DB row but leaves the GCS object orphaned forever. Verdict: **fair**.

## Endpoint contract

- `GET /api/v1/notes/<note_pk>/images/` →
  `NoteImageViewSet.list` (views.py:1598-1609): `200 [Image…]`;
  `404` unknown note; `403` existing-but-inaccessible note.
- `POST` same path → `create` (views.py:1611-1672): multipart
  `file` field; `201` serializer body; `400` missing file or
  `IMAGE_MAX_PER_NOTE` (default 10) cap; `403` non-owner, guest,
  or unauthenticated; `413` > `IMAGE_MAX_BYTES` (default 10 MB);
  `415` disallowed MIME or non-image bytes.
- `GET|POST /api/v1/notes/<note_pk>/comments/<comment_pk>/images/`
  → `CommentImageViewSet` (views.py:1701-1778): same contract,
  cap `IMAGE_MAX_PER_COMMENT` (default 5); comment must belong to
  `note_pk` else `404` (views.py:1694-1699).
- `DELETE /api/v1/images/<image_pk>/` → `ImageDestroyView`
  (views.py:1794-1828): `204`; `403` unless uploader, comment
  author, or parent-note owner; `404` unknown id. Deletes the row
  then best-effort deletes the GCS blob. No GET/PATCH anywhere —
  detail routes on the nested viewsets are 405 (only `list` and
  `create` are defined on a bare `GenericViewSet`).
- Image shape (serializers.py:748-769): `id`, `storage_url`,
  `signed_url` (V4 signed, TTL `IMAGE_SIGNED_URL_TTL_SECONDS`=600,
  None on failure), `content_type`, `size_bytes`, `comment`,
  `note`, `uploaded_by`, `created_at`. Also inlined on comments as
  `images` (serializers.py:808, redacted to `[]` when
  `is_deleted`).

## Findings

### Bugs / correctness

1. **Orphaned GCS objects on every cascade delete** — `Image.note`
   and `Image.comment` are `on_delete=CASCADE` (models.py:303-318),
   but `delete_original` is called only from the upload-rollback
   paths (views.py:1663, 1769) and `ImageDestroyView`
   (views.py:1823). `NoteViewSet.perform_destroy` (views.py:436),
   `TagViewSet` cascade incl. `?delete_notes=true` (views.py:270,
   302), and user deletion all remove Image rows with no blob
   cleanup → permanent storage leak. Comment soft-delete keeps the
   row (views.py:1437-1439), so the leak is specifically hard
   cascades.
2. **Existence leak via 403-vs-404** — `_get_note` uses unscoped
   `Note` (views.py:1593-1596) and `_get_comment` unscoped
   `Comment` (views.py:1694-1699), then `list` raises
   `PermissionDenied` when inaccessible (views.py:1601-1604,
   1704-1709). `CommentViewSet` deliberately returns 404 so
   inaccessible notes are indistinguishable (views.py:1350-1359);
   the image endpoints reveal that a note/comment exists. Tests
   codify the 403 (test_images.py:329-336).
3. **Declared MIME not checked against actual format** —
   `upload_original` verifies the bytes parse as *an* image via
   `PILImage.open().verify()` (image_storage.py:134-147) but never
   compares `img.format` to the client-supplied `content_type`
   that picks the extension and the stored blob metadata
   (image_storage.py:149-157). A PNG sent as `image/gif` is stored
   as `source.gif` and served as GIF.
4. **Uploads allowed onto soft-deleted comments** — `create` does
   not check `comment.is_deleted` (views.py:1719-1724). The comment
   tree then hides those images (`images: []`,
   serializers.py:836-838) while the nested endpoint still lists
   them — inconsistent visibility.
5. **Latent writable FKs** — `comment` and `note` are in
   `ImageSerializer.fields` but not `read_only_fields`
   (serializers.py:750-769). No view binds the serializer to
   request data today (`create` uses `Image.objects.create`), so
   harmless now, but the first update endpoint added could
   re-parent attachments without an ownership check.
6. **Docstring/code mismatch** — ImageSerializer says `signed_url`
   "falls back to `storage_url`" (serializers.py:742-744) but
   `get_signed_url` returns `None` on failure
   (serializers.py:771-778).

### Security & auth

- **Guest handling verified correct.** `get_permissions` returns
  `AllowAny` for list and `IsAuthenticated + GuestReadOnly` for
  writes on both viewsets (views.py:1584-1591, 1689-1692);
  `ImageDestroyView` sets `permission_classes` explicitly
  (views.py:1792). `GuestReadOnly` rejects guest writes with 403
  (permissions.py:21-32). Covered by
  `test_guest_cannot_upload_note_image` (tests.py:3063-3067).
- Upload is further gated to note owner / comment author
  (views.py:1614-1617, 1721-1724); delete to uploader, comment
  author, or parent-note owner (views.py:1804-1818) — the
  note-owner-on-comment-image case is regression-tested
  (test_images.py:537-559).
- Middleware: unsafe methods on `/api/v1/notes/...` are outside
  `PUBLIC_READ_PREFIXES` (middleware.py:16-34), so anonymous POSTs
  get auto-provisioned a shared per-device account — such a user
  can legitimately upload to *that shared account's* own notes.
  By design, but worth knowing uploads are not strictly
  "registered users only".
- `storage_url` (internal `gs://` bucket name) is serialized to
  every reader including anonymous viewers of public notes
  (serializers.py:750-753). Low risk; the bucket is presumably not
  public, but the field buys clients nothing.
- `DELETE /images/<pk>/` answers 404 vs 403 to any authenticated
  caller, disclosing image existence (views.py:1795-1818);
  unguessable `IMG_` ids make this cosmetic.
- Signed URLs use IAM `signBlob` (`service_account_email` +
  `access_token`, image_storage.py:212-222) — the correct keyless
  pattern for App Engine; TTL 600 s with a 60 s safety margin and
  a bounded 2048-entry LRU (image_storage.py:88-94, 232-239).

### Performance

- **`ImageDestroyView` does up to 4 lazy-FK queries** —
  `image.note`, `image.comment`, `image.comment.note`,
  `image.comment.user`, `image.uploaded_by` all resolve
  separately (views.py:1800-1812). One
  `select_related('note','comment__note','comment__user',
  'uploaded_by')` fetch would collapse it.
- **`blob.upload_from_file` has no timeout** (image_storage.py:
  154-157); a stalled GCS socket pins the worker — same class of
  issue flagged in the timestamps report.
- **Good:** per-parent cap is double-checked — cheap count before
  upload, authoritative count under `select_for_update`
  (views.py:1624-1653, 1745-1759) — so the expensive GCS
  round-trip is skipped when already at cap.
- **Good:** GCS client, credentials, and signed URLs are all
  cached (image_storage.py:48-94); comment trees prefetch images
  (views.py:1379) with a query-budget test
  (test_images.py:698-715). Lists are unpaginated but bounded by
  the per-parent caps.

### Code quality & consistency

- **~90 lines duplicated** between `NoteImageViewSet` and
  `CommentImageViewSet` (views.py:1598-1672 vs 1701-1778): cap
  check → upload → atomic lock+create → rollback → serialize is
  identical modulo the parent model. A shared base/mixin would
  collapse it.
- `IMAGE_ALLOWED_CONTENT_TYPES` (settings.py:286-291) is dead —
  the allowlist is re-hardcoded as `_ALLOWED_EXTENSIONS`
  (image_storage.py:36-41); the two can silently drift.
- `getattr(settings, 'IMAGE_MAX_PER_NOTE', 10)` (views.py:1621,
  1728) guards settings that are always defined — plain attribute
  access would match house style.
- Magic `status=201` (views.py:1672, 1778) vs
  `drf_status.HTTP_201_CREATED` used elsewhere in the file.
- `is_uploader` and `is_comment_author` in `ImageDestroyView`
  (views.py:1804-1814) are provably equivalent — `create` requires
  `comment.user == user` and sets `uploaded_by=user`. Harmless
  redundancy.
- `PermissionDenied` for list-vs-create split means list on an
  inaccessible note is 403 while `CommentViewSet` uses 404 —
  posture inconsistency beyond the leak itself.
- Decompression-bomb rejections from PIL surface as `415`
  UnsupportedMediaType (image_storage.py:139-146) rather than
  413 — cosmetic mislabel.

### Test coverage

Covered (test_images.py + tests.py:2888-3075): upload happy paths
both parents; non-owner/non-author 403; 415 spoofed/disallowed
MIME; 413 oversize; missing file 400; anon/private/public list
matrix; delete authz incl. note-owner-deletes-comment-image
(issue #56); XOR `CheckConstraint`; inline `images` in the comment
tree incl. soft-delete redaction and a 2-query budget.

Missing: per-note/per-comment **cap enforcement**; **wrong-nesting**
404 (comment_pk under a different note_pk); guest `POST` on
comment images and guest `DELETE` on `/images/<pk>/`; upload onto
a soft-deleted comment; `signed_url=None` fallback in serialized
output; cascade-delete blob cleanup (would pin finding 1);
declared-MIME-vs-actual-format mismatch.

## Refactoring recommendations

1. **P0 — Delete the GCS object whenever an Image row is
   removed.** Add a `post_delete` receiver on `Image` calling
   `delete_original` (Django emits delete signals for
   cascade-collected instances), or a
   `delete_images_for_notes(note_ids)` service invoked from
   `NoteViewSet.perform_destroy`, `TagViewSet.perform_destroy`/
   `destroy`, and user deletion. Why: today every note/tag/user
   delete permanently leaks blobs — finding 1. Effort: M.
2. **P1 — Fetch the parent through the accessible queryset.**
   `_get_note`/`_get_comment` should resolve via
   `get_accessible_notes_qs` (and `Comment` filtered to it) so
   inaccessible parents return 404, matching `CommentViewSet`.
   Fixes the existence leak (finding 2) and removes the separate
   `accessible.filter(pk=…).exists()` query in `list`.
   views.py:1593-1609, 1694-1709. Effort: S.
3. **P1 — Verify declared MIME against `img.format`** in
   `upload_original` (map PIL format → expected content_type) and
   reject a mismatch with 415; let decompression-bomb errors
   surface as 413. image_storage.py:134-157. Effort: S.
4. **P1 — `select_related` the delete lookup** in
   `ImageDestroyView`: `get_object_or_404(
   Image.objects.select_related('note','comment__note',
   'comment__user','uploaded_by'), pk=image_pk)`. Collapses ~5
   queries to 1. views.py:1795-1812. Effort: S.
5. **P2 — Extract a shared image-attachment base.** A
   `_BaseImageViewSet` (or one `upload_to_parent` helper taking
   parent instance, cap setting name, and FK name) removes ~90
   duplicated lines between the two viewsets. Effort: M.
6. **P2 — Tighten `ImageSerializer`:** move `comment`/`note` into
   `read_only_fields` (finding 5), fix the "falls back to
   `storage_url`" docstring (finding 6), and consider dropping
   `storage_url` from the serialized payload. Effort: S.
7. **P2 — Reject uploads onto `is_deleted` comments** (403 with a
   clear message) so the tombstone contract holds on the nested
   route too. views.py:1719-1724. Effort: S.
8. **P2 — Add a `timeout` (and ideally `retry`) to
   `blob.upload_from_file`.** image_storage.py:154-157. Effort: S.
9. **P2 — Reconcile the MIME settings:** consume
   `settings.IMAGE_ALLOWED_CONTENT_TYPES` in `_ALLOWED_EXTENSIONS`
   or delete the unused setting. Effort: S.
10. **P2 — Fill the test gaps:** cap enforcement, wrong-nesting
    404, guest POST/DELETE matrix, soft-deleted-comment upload,
    cascade cleanup, `signed_url=None`. Effort: M.
