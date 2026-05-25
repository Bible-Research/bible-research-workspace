# Plan: Inline `images` in `CommentSerializer` (Backend)

## Goal

Return image attachments directly inside the comment tree response so the
`reactive-bible` frontend renders thumbnails without an extra round-trip
per comment. Eliminates the N+1 fetch pattern described in
`@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/COMMENT_IMAGES_PLAN.md`
section 7.

## Non-goals

- Changing the upload / delete endpoints. Writes still go through
  `NoteImageViewSet` / `CommentImageViewSet`.
- Inlining `images` on `NoteSerializer` (separate concern; can follow
  the same pattern in a follow-up).
- Pagination of images per comment (cap is 5, so unnecessary).

---

## 1. Serializer change

File: `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/serializers.py`

Add a nested read-only field to `CommentSerializer` (declared **after**
`ImageSerializer` is defined, or move `ImageSerializer` above
`CommentSerializer` to avoid a forward reference):

```python
class CommentSerializer(serializers.ModelSerializer):
    # …existing fields
    images = ImageSerializer(many=True, read_only=True)

    class Meta:
        model = Comment
        fields = [
            'id',
            'author',
            'note_id',
            'parent_comment',
            'content',
            'timestamp',
            'is_deleted',
            'images',
        ]
        read_only_fields = [
            'id', 'author', 'note_id', 'timestamp', 'images',
        ]
```

`Image.comment` already has `related_name="images"` (see
`@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/models.py` — Image model from
commit `4758ee6`), so the reverse manager is `comment.images.all()`.

### Soft-deleted comments

`build_comment_tree` at
`@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/serializers.py:370`
already redacts `content` to `[deleted]`. For parity, also blank out
images on deleted nodes so the UI doesn't show orphan thumbnails next
to `[deleted]`:

```python
if comment.is_deleted:
    data['content'] = '[deleted]'
    data['images'] = []
```

Images themselves are **not** hard-deleted when a comment is soft-
deleted — this is purely a render-time omission. (If we ever want the
files gone too, that's a separate background sweep, out of scope.)

---

## 2. Queryset — prevent N+1

File: `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/views.py`

`CommentViewSet.get_queryset` (around line 390) currently does
`select_related('user')`. Add `prefetch_related('images')`:

```python
def get_queryset(self):
    return (
        Comment.objects
        .filter(note_id=self.kwargs['note_pk'])
        .select_related('user')
        .prefetch_related('images')
        .order_by('timestamp')
    )
```

`build_comment_tree` iterates the queryset twice; the prefetch cache
covers both passes, so the total cost is:

- 1 query for comments
- 1 query for all images attached to those comments
- 0 extra queries during serialization

regardless of thread size. Verify with a regression test (see §4).

---

## 3. Signed URL cost

`ImageSerializer.get_signed_url` (`serializers.py:360`) calls
`signed_image_url(obj.id, obj.storage_url)`. On GCP that goes through
IAM `signBlob` — one RPC per image at serialize time.

Per the `image-attachments-plan.md` defaults: `IMAGE_MAX_PER_COMMENT=5`,
threads of ~50 comments → worst case 250 sign calls per `GET
/notes/{id}/comments/`. Acceptable for v1 but worth measuring.

Mitigations to keep in the back pocket (not in this PR):

- Cache `(image_id, ttl_bucket) -> signed_url` in `django.core.cache`
  keyed by floor(now / TTL/2). Reuses URLs within the half-life.
- Switch `signed_image_url` to local signing with the service account
  private key (no RPC) if/when we ship a key-based auth path.

Flag in the PR description; no code change unless benchmarks demand it.

---

## 4. Tests

File: `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/tests.py`
(or extend `test_images.py`).

New / updated cases:

- **Inline serialization happy path:** create a comment with 2 images,
  `GET /api/v1/notes/{id}/comments/` returns the comment with an
  `images` array of length 2, each item containing `id`, `signed_url`,
  `content_type`, `size_bytes`, `created_at`. Mock
  `signed_image_url` to return a deterministic value.
- **Empty images array** for comments without attachments (must be
  `[]`, not missing / null — frontend depends on shape).
- **Nested replies** include their own `images` arrays.
- **Soft-deleted comment** returns `images: []` and
  `content: '[deleted]'`.
- **Query budget:** use `assertNumQueries(N)` to lock in the prefetch.
  Expected count = 1 (comments) + 1 (prefetch images) + auth-related
  fixed queries. Concretely: `CommentTreeTest` already has a pattern;
  add a sibling test that asserts `assertNumQueries` doesn't grow with
  image count.
- **Write path unchanged:** POST/PATCH to the comment endpoint with an
  `images` key in the payload is ignored (read-only field); existing
  upload endpoints remain the only way to attach images.

Run only the touched test module per workspace rule:

```
python manage.py test annotations.tests annotations.test_images
```

---

## 5. Frontend follow-up (separate PR in `reactive-bible`)

Once this lands:

1. Drop `fetchCommentImages` from the v1 plan — `Comment.images` is
   populated by `fetchComments` directly.
2. Update `Comment` type in
   `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/types.ts`
   to make `images: CommentImage[]` non-optional (always present, may
   be empty).
3. Remove the per-node `useEffect` image fetch from `CommentNode`.
4. `CommentThread.silentLoad` already refreshes the tree every 30 s —
   that now also refreshes signed URLs for free (TTL is 600 s, so the
   poll comfortably stays ahead).
5. After a successful image upload, optimistically push the returned
   `CommentImage` into the tree node's `images` array via the existing
   `updateNode` helper; `silentLoad` reconciles.

---

## Migration / rollout

- Purely additive: new field on a read response, no DB migration, no
  breaking change to existing clients (current frontend ignores
  unknown fields).
- Deploy backend first, then ship the frontend cleanup PR. No flag
  needed.

## Risks

- **Payload size:** with 5 max images per comment and ~300-byte
  `ImageSerializer` output each, worst case +1.5 KB per comment.
  Threads of 100 comments → +150 KB. Acceptable; gzip flattens it
  further. Revisit only if real threads stay near the cap.
- **Signed URL RPC fan-out:** see §3.
- **Forward-reference order in `serializers.py`:** `ImageSerializer`
  must be defined before `CommentSerializer` (or use a string
  reference). Trivial reorder.
