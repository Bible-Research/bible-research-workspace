# comments (CommentViewSet + CommentCountView) — code review

## Summary

`CommentViewSet` (annotations/views.py:1293-1439) and
`CommentCountView` (views.py:1492-1566) are well-built: nested
scoping, ownership, soft-delete semantics, and the aggregate
count query are all correct, with strong test coverage. The
real problems live at the edges — soft-deleted comments still
serve their images via `CommentImageViewSet.list`
(views.py:1701-1714), note owners cannot moderate comments on
their own public notes, and PATCH/reply operations ignore the
`is_deleted` tombstone. Verdict: **good** (with one P1 leak in
an adjacent endpoint and a P1 moderation gap).

## Endpoint contract

`notes/<note_pk>/comments/` — DRF router (annotations/urls.py:13-17):

- `GET .../comments/` — nested tree, roots with `replies`
  → 200 / 404.
- `POST .../comments/` — create; optional `parent_comment`
  → 201 / 400 / 403 / 404.
- `GET .../comments/<pk>/` — flat single comment
  → 200 / 404.
- `PATCH .../comments/<pk>/` — author-only content edit
  → 200 / 400 / 403 / 404.
- `DELETE .../comments/<pk>/` — author-only soft-delete
  → 204 / 403 / 404.

- PUT disabled via `http_method_names` (views.py:1337-1339).
- Response shape: `{id, author{id,username}, note_id,
  parent_comment, content, timestamp, is_deleted, images[],
  replies[]}` (serializers.py:816-831). Deleted nodes:
  `content='[deleted]'`, `images=[]`
  (serializers.py:833-839). List is an unpaginated bare array.
- Reads are `AllowAny`; visibility is enforced by
  `get_accessible_notes_qs` (own ∪ public). Writes require
  `IsAuthenticated` + `GuestReadOnly` (views.py:1341-1348).
  Any authenticated user may comment on a *public* note; only
  the comment author may edit/delete.
- 404 is uniform for nonexistent vs inaccessible notes —
  existence is not leaked (views.py:1350-1359, 1361-1381).

`GET /api/v1/comments/counts/` (urls.py:31-35):

- Params: `tag_id` XOR `note_ids` (comma-separated, cap 200 =
  `NOTE_IDS_CAP`, views.py:47); `include_deleted=true`;
  `tag_id=untagged` counts comments on tag-less notes
  (views.py:1538-1539).
- Response: `{"counts": {"<NOTE_PK>": <int>, ...}}`; notes
  with zero comments still appear with 0; inaccessible notes
  are silently omitted.
- Errors: 400 when both/neither filter given, 400 over cap.
- Permissions: none declared → defaults to
  IsAuthenticated+GuestReadOnly (settings.py:377-380), i.e.
  open read for the middleware-provisioned device/guest users.

## Findings

### Bugs / correctness

- **P1 — soft-deleted comment images still downloadable.**
  `CommentImageViewSet.list` (views.py:1701-1714) returns
  `Image` rows with signed URLs for any accessible comment
  without checking `comment.is_deleted`. The comment payload
  redacts `images` to `[]` (serializers.py:836-838), so a
  "deleted" comment's attachments remain fetchable —
  contradicting the redaction contract.
- **P2 — PATCH on a tombstoned comment silently accepts
  writes.** `perform_update` (views.py:1407-1420) has no
  `is_deleted` guard; new content is stored but permanently
  masked by `to_representation` — a confusing no-op that
  also repopulates redacted data in the DB.
- **P2 — replies to deleted comments allowed.**
  `validate_parent_comment` (serializers.py:841-865) checks
  same-note only; `is_deleted` parents still accept replies.
  Arguably intentional (threads stay alive), but undeclared.
- **P2 — orphaned replies silently dropped from the tree.**
  `build_comment_tree` discards nodes whose parent is
  missing/foreign (serializers.py:894-898). The API prevents
  cross-note parents, but admin/shell edits can create
  orphans that vanish from `list` while still counting in
  `/comments/counts/` → tree/count drift.
- **P3 — no content constraints.** `content` is an unbounded
  `TextField` with no `max_length` or whitespace check —
  `' '` passes `blank=False` (models.py:223).
- **Nested-route scoping is correct.** `get_queryset`
  filters `note_id=note_pk` AND `note__in` accessible notes
  (views.py:1369-1381), so a comment fetched under a wrong —
  but accessible — `note_pk` 404s, and inaccessible-note
  comments 404 on every verb. `perform_create` re-fetches
  the note through the accessible qs (views.py:1399).

### Security & auth

- **P1 — no moderation: note owners cannot delete comments
  on their own notes.** `perform_destroy` is author-only
  (views.py:1428-1432). Any authenticated (or
  auto-provisioned device) user can comment on a public
  note, and the note's owner cannot remove it. Inconsistent
  with `ImageDestroyView`, which *does* let note owners
  delete comment-attached images (views.py:1804-1818).
- **P2 — existence oracle via `parent_comment` PKRF.**
  `queryset=Comment.objects.all()` (serializers.py:809-814)
  distinguishes "does not exist" from "must belong to the
  same note" → any authenticated user can probe arbitrary
  `CMNT_` ids and infer their note association. Low risk
  (unguessable ids) but avoidable by scoping the queryset.
- **P2 — 404-vs-403 oracle in adjacent image viewsets.**
  `CommentImageViewSet._get_comment` (views.py:1694-1699)
  and `NoteImageViewSet._get_note` (views.py:1593-1596)
  fetch unscoped objects then raise 403, while nonexistent
  targets 404 — revealing private note/comment existence.
  `CommentViewSet` itself avoids this correctly.
- **Context — auto-provisioned writers.** Device middleware
  (middleware.py:125-159) creates a user per
  language+User-Agent pair, and those users *can* post
  comments on any public note — an unauthenticated spam
  channel that GuestReadOnly does not cover (it only blocks
  the literal `guest` account).

### Performance

- Comment list does one query + `select_related('user')` +
  `prefetch_related('images')` (views.py:1370-1381); tree
  assembly is O(n) in Python (serializers.py:868-901). No
  DB N+1. **But** each image triggers a GCS signed-URL call
  via `ImageSerializer.get_signed_url`
  (serializers.py:771-778); `signed_image_url` caches per
  `storage_url` (image_storage.py:168+), so repeat views are
  cheap but cold trees still fan out per image.
- `_get_note()` + `note__in` subquery duplicate the access
  check on `list` (views.py:1387 vs 1374) — one redundant
  notes query per request (needed on detail routes, not
  here).
- No pagination or comment-count/depth cap: a note with
  thousands of comments returns the full serialized tree;
  unbounded nesting depth produces deeply recursive JSON.
- `CommentCountView` is a single `COUNT ... FILTER` grouped
  query over the accessible-notes qs (views.py:1550-1559) —
  no N+1, correct semantics. The `tag_id` path is unbounded
  in note count but bounded in practice by per-user data.
- Middleware tax: `/api/v1/comments/counts/` is not in
  `PUBLIC_READ_PREFIXES` (middleware.py:16-20), so every
  unauthenticated/token request provisions or looks up a
  device user + `login()` session write — the likely cause
  of the `@expectedFailure` `test_single_query_assertion`
  (tests.py:954-964). Cookie-less clients may create a new
  session row per request.

### Code quality & consistency

- `validate_parent_comment` reaches into
  `request.parser_context['kwargs']` for `note_pk` because
  `CommentViewSet` never puts it in serializer context
  (serializers.py:854-860). Works, but fragile — an
  unreachable `self.context.get('note_pk')` branch hints at
  the intended design.
- Naming drift: `Comment.timestamp` vs `Note.created_at`;
  no `updated_at`/`edited` flag, so edits are invisible to
  clients (models.py:226-228).
- `order_by('timestamp')` duplicates `Meta.ordering`
  (views.py:1380, models.py:241) — harmless redundancy.
- `CommentCountView` raises plain `ValidationError` strings
  (views.py:1513-1520) — response is a bare list, not the
  `{error, error_code}` contract used elsewhere in the API.
- `_get_note`/`_get_comment` helpers are duplicated across
  three viewsets with *different* scoping (views.py:1350,
  1593, 1694) — the unscoped variants are what produced the
  oracle bug above.
- Docstrings are thorough and accurate — the N+1 and
  soft-delete claims match the code.
- Mentions: no backend mention/notification logic exists;
  comment `content` is opaque TipTap JSON the frontend
  parses (scripture mentions are client-side only).

### Test coverage

Strong: `CommentTreeTest`, `CommentViewSetTest`,
`CommentNoteAccessTest` (private/public/anonymous × all
verbs), `CommentCountViewTest` (both filters, validation,
cap, visibility, untagged via test_untagged_notes.py),
guest-write denial, inline-images tests.
Gaps:

- No test for detail fetch under a *wrong but accessible*
  note_pk (comment B via note A's URL).
- No tests for PATCH/reply targeting `is_deleted` comments.
- No test that a deleted comment's `.../images/` endpoint is
  empty — would have caught the P1 leak.
- `test_single_query_assertion` is `@expectedFailure`
  (tests.py:954-964) — known unenforced budget.

## Refactoring recommendations

1. **P0 — Filter deleted comments' images.**
   In `CommentImageViewSet.list`, `raise Http404` (or return
   `[]`) when `comment.is_deleted`; add a regression test.
   Closes the redaction bypass. Effort: S.
2. **P1 — Let note owners moderate.** Allow
   `note.user == request.user` in `perform_destroy` (and
   optionally `perform_update`), mirroring
   `ImageDestroyView`'s note-owner rule. Effort: S.
3. **P1 — Uniform 404 in image viewsets.** Scope
   `_get_comment`/`_get_note` through
   `get_accessible_notes_qs`-filtered querysets instead of
   fetch-then-403, matching `CommentViewSet._get_note`.
   Effort: S.
4. **P2 — Guard the tombstone.** Reject PATCH on
   `is_deleted` comments (400) and decide+document whether
   replies to deleted parents are allowed; enforce in
   `validate_parent_comment`. Effort: S.
5. **P2 — Bound the payload.** `max_length` on
   `CommentSerializer.content`, optional depth/count cap in
   `build_comment_tree`. Effort: S.
6. **P2 — Pass `note_pk` via `get_serializer_context`**
   instead of parsing `request.parser_context`; drop the
   dead `self.context.get('note_pk')` fallback. Effort: S.
7. **P2 — De-duplicate the access check on `list`.**
   Keep `_get_note` for 404 semantics and drop the
   `note__in` subquery there, or make `list` rely on the
   queryset and 404 on empty. Effort: S.
8. **P2 — Fix the query-budget test.** Either move
   `/comments/counts/` into `PUBLIC_READ_PREFIXES` (it only
   exposes public-note data to anonymous callers anyway) or
   rewrite `test_single_query_assertion` to exclude
   middleware queries like the images test does. Effort: S.
9. **P2 — Scope `parent_comment`'s queryset** to
   comments on the URL's note (a custom
   `PrimaryKeyRelatedField.get_queryset`), removing the
   existence oracle and the manual same-note check.
   Effort: S.
10. **P3 — Add `updated_at`/`edited` on Comment** if the
    frontend should mark edited comments; rename `timestamp`
    → `created_at` for consistency (migration needed).
    Effort: M.
