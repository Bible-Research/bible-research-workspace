# Comment Count Endpoint — Implementation Plan

## Goal

Add a lightweight read-only endpoint that returns the number of
(non-deleted) `Comment` rows per `Note`, scoped by a flexible filter
set. The **primary filter is `tag_id`** — i.e. "give me comment counts
for every note linked to tag X". A secondary `note_ids` filter must
also be supported for arbitrary note subsets.

The response must be:

- **Cheap**: one aggregated SQL query, no serialization of comment
  bodies, no N+1.
- **Stable**: returns 0 for notes that exist but have no comments
  (so the client can render counts without follow-up lookups).
- **Permission-aware**: respects the same note-visibility rules as
  `NoteViewSet.get_queryset()` (user's own notes + public notes).

## Non-goals

- No nested tree, no comment content, no author info.
- No pagination — counts are aggregated, the response is bounded by
  the number of notes matched.
- No write operations.

## Endpoint

```
GET /api/v1/comments/counts/
```

Standalone (not nested under `notes/`) because the resource is a
*set of notes' counts*, not children of a single note.

### Query parameters

| Param      | Type             | Required | Description |
|------------|------------------|----------|-------------|
| `tag_id`   | string (TAG…)    | one of   | Returns counts for every accessible note carrying this tag. |
| `note_ids` | CSV of note PKs  | one of   | Returns counts for the listed notes (capped, see below). |
| `include_deleted` | bool      | no       | Default `false`. When `false`, `is_deleted=True` rows are excluded from the count. |

Rules:

- Exactly **one of** `tag_id` or `note_ids` must be supplied.
  Otherwise → `400 Bad Request` with a clear message.
- `note_ids` is capped at **200 IDs** per request to prevent abuse.
  Exceeding the cap → `400`.

### Response

`200 OK`, content type `application/json`:

```json
{
  "counts": {
    "NOT0123…": 5,
    "NOT0456…": 0,
    "NOT0789…": 12
  }
}
```

- Object keyed by note PK → integer count.
- Every accessible note matching the filter appears in the map,
  including notes with zero comments.
- Notes the requester cannot see are silently omitted (never 403 per
  note — consistent with how `NoteViewSet` filters silently).

### Error responses

- `400` — missing/conflicting filters, `note_ids` over cap, malformed
  `tag_id`.
- `401` — only if/when global auth is enforced; current code allows
  anonymous reads of public notes, so keep that behaviour.

## Backend changes

### 1. View

New `CommentCountView` (APIView, not ModelViewSet) in
`@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/views.py`.

Algorithm:

1. Parse and validate query params (raise `ValidationError` on bad
   input).
2. Build the **base accessible-note queryset** by reusing the same
   visibility logic that `NoteViewSet.get_queryset()` applies:
   - authenticated → user's notes ∪ public notes
   - anonymous → public notes only
   Extract this into a helper `get_accessible_notes_qs(user)` in
   `views.py` (or a new `selectors.py`) so the rule lives in one
   place. Refactor `NoteViewSet.get_queryset()` to call it too.
3. Apply the requested filter:
   - `tag_id` → `.filter(tag_id=tag_id)`
   - `note_ids` → `.filter(id__in=note_ids)`
4. Aggregate in one query:
   ```python
   from django.db.models import Count, Q
   rows = (
       notes_qs
       .annotate(
           comment_count=Count(
               'comments',
               filter=Q(comments__is_deleted=False),
           ),
       )
       .values_list('id', 'comment_count')
   )
   ```
   The `Q(comments__is_deleted=False)` filter keeps soft-deleted
   comments out of the count by default. When `include_deleted=true`
   is passed, drop the inner filter.
5. Materialize into `{id: count}` and return.

This is **one SQL statement** (a `LEFT OUTER JOIN annotations_comment
… GROUP BY annotations_note.id`) and uses the existing
`comment_note_id_idx` index added in migration `0007`.

### 2. URL

Add to
`@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/urls.py`:

```python
urlpatterns = [
    path('', include(router.urls)),
    path(
        'comments/counts/',
        views.CommentCountView.as_view(),
        name='comment-counts',
    ),
]
```

The static path is registered **before** any router includes that
could shadow it (router routes are under `comments/` only when
nested, so no conflict — but documenting the order anyway).

### 3. Serializer / response shape

No DRF serializer needed; the view returns a plain dict via
`Response`. Add an OpenAPI schema decorator
(`@extend_schema` from `drf_spectacular`, already in the project)
so the endpoint shows up in the generated docs with the response
schema documented.

### 4. Permissions

Inherit project defaults. The visibility filter on the queryset is
the security boundary — a user requesting `tag_id=X` only ever gets
counts for notes they can already see.

## Tests

Add to `bible_research/annotations/tests.py` in a new
`CommentCountViewTest` class. All tests use the existing fixtures
plus the `Comment` factory introduced in the previous commit.

1. **Happy path — `tag_id`**: two notes share a tag; counts include
   both with correct non-deleted totals.
2. **Happy path — `note_ids`**: explicit list returns counts only
   for those IDs.
3. **Zero counts present**: a note with no comments still appears
   with `0`.
4. **Soft-deleted comments excluded by default**: deleted rows do
   not inflate the count.
5. **`include_deleted=true`**: deleted rows are counted.
6. **Visibility — anonymous**: only public notes appear; private
   notes silently omitted.
7. **Visibility — owner**: a user sees counts for their own
   private notes plus public notes.
8. **Visibility — other user's private notes**: silently omitted,
   not 403.
9. **Validation — neither filter**: returns 400.
10. **Validation — both filters**: returns 400.
11. **Validation — `note_ids` over cap**: returns 400.
12. **Cross-tag isolation**: notes tagged with a *different* tag are
    not included when filtering by `tag_id`.
13. **Query-count assertion**: wrap the view call in
    `assertNumQueries(...)` to lock in the single-query guarantee
    (allow auth-related queries — establish the baseline once and
    pin it).

## Performance notes

- The aggregation runs against the existing
  `comment_note_id_idx` index → O(log n) per note + sequential read
  of the matching index range.
- `note_ids` cap of 200 keeps the `IN (...)` list small.
- For very large tags (thousands of notes), the response is still
  bounded by the count of notes the user can see for that tag,
  which is the same bound the existing `GET /notes/?tag_id=` already
  exposes. If that ever needs paging, both endpoints would need it
  together — out of scope here.

## Frontend integration (informational)

The reactive-bible client can call this once per tag-view render
to badge each note card with its comment count, instead of either
(a) fetching the full tree per note or (b) embedding the count in
`NoteSerializer` (which would force every note list endpoint to
join `annotations_comment`, even when counts aren't needed).

## Out of scope / follow-ups

- Per-thread (root-comment) counts.
- "Unread comments since timestamp X" — would need a per-user read
  marker table.
- Caching: not needed at current scale; revisit if the endpoint
  shows up in slow-query logs.

## Files touched

- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/views.py`
  — new `CommentCountView`, extracted `get_accessible_notes_qs`.
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/urls.py`
  — new path entry.
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/annotations/tests.py`
  — new `CommentCountViewTest` class (~13 tests).

No model changes, no migrations.
