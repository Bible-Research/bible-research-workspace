# Tag Public Toggle — Implementation Plan

## Goal

Let the user toggle a tag between private and public from the UI:

- A button on every tag node in Tag Management (`/tags`, `TagTree`)
- The same button in the tag view (`/notes/tag/:tagId`, `TagNotesRoute`)

Toggling a tag public:

1. Flips all existing notes under that tag to `public=True`.
2. Persists `public=True` on the tag so that, from that point on,
   new notes created under it are marked public automatically.

Toggling back to private flips the flag and the tag's existing
notes to `public=False`.

## Confirmed decisions

- **Scope: just the clicked tag.** Descendant tags are unaffected.
- **Toggle both ways.** The same button makes the tag public or
  private; making it private flips existing notes back.
- **Tag flag is a default only.** New notes under a public tag
  default to `public=True`, but an explicit per-note `public`
  value sent by the client is still honored.
- **Button on every tag node** in the tree (not just roots).

## Current state

- `Note.public` (BooleanField) already exists and controls
  anonymous visibility (`get_accessible_notes_qs`,
  `annotations/views.py`). `Tag` has no public flag.
- The frontend never sends `public` — note creation
  (`addTagNote` → `POST /notes/`, and `POST /notes/bulk/`)
  doesn't include it, and no UI toggles it. `NoteCard` only
  *reads* `note.public` to show a share button.
- `TagTree` renders hierarchy in `TagManagementRoute` with
  view/edit/delete `ActionIcon`s — the new button slots in there.
- `TagNotesRoute` header has Refresh + Share `ActionIcon`s
  (`TagNotesRoute.tsx` ~lines 676–710) — the toggle goes beside
  them, rendered only for the tag owner.

## Backend (`bible_research`)

### 1. Model — `annotations/models.py`

Add to `Tag`, mirroring `Note.public`:

```python
public = models.BooleanField(
    default=False,
    help_text=(
        "If True, notes under this tag are public by default."
    ),
)
```

**Migration needed** — per workspace rules the agent must not run
it. After the model change, the user runs:

```bash
python manage.py makemigrations annotations
python manage.py migrate
```

### 2. Serializer — `annotations/serializers.py`

- `TagSerializer.Meta.fields`: add `'public'` (writable, so it is
  returned in tag payloads).
- `NoteSerializer`: declare
  `public = serializers.BooleanField(required=False)` with no
  default, so the key is absent from `validated_data` when the
  client omits it (an inherited model default could otherwise
  mask "not sent"). In `create()`:

  ```python
  if 'public' not in validated_data:
      tag = validated_data.get('tag')
      validated_data['public'] = bool(tag and tag.public)
  ```

- `BulkNoteItemSerializer`: drop `default=False` from `public`
  (keep `required=False`). In `BulkNoteCreateSerializer.create()`
  build notes with `public=nd.get('public', tag.public)`.

`update()`/`PATCH` unchanged: moving a note into a public tag
does not auto-flip it; the flag only applies at creation.

### 3. Endpoint — `annotations/views.py`

New `TagViewSet` action (consistent with `notes/reorder/` style):

```
POST /api/v1/tags/{id}/set-public/
body: { "public": true | false }
200: { "tag": <TagSerializer>, "notes_updated": <int> }
```

Implementation sketch:

```python
@action(detail=True, methods=['post'], url_path='set-public')
def set_public(self, request, pk=None):
    tag = self.get_object()  # scoped to request.user's tags
    make_public = bool(request.data.get('public', False))
    with transaction.atomic():
        tag.public = make_public
        tag.save(update_fields=['public', 'updated_at'])
        notes_updated = Note.objects.filter(
            tag=tag, user=request.user
        ).update(public=make_public)
    return Response({
        'tag': TagSerializer(tag, context={'request': request}).data,
        'notes_updated': notes_updated,
    })
```

Ownership comes free: `get_queryset()` already limits tags to
`request.user` (or `guest`), so `get_object()` 404s on foreign
tags. Validate that `public` is a bool → 400 otherwise
(`{"error": "validation_error", "message": ...}` style).

### 4. Backend tests — `annotations/tests.py`

- set-public flips flag + all tag notes, both directions.
- Returns correct `notes_updated` count; idempotent re-toggle.
- Another user's tag → 404; their notes untouched.
- Note create under public tag without `public` → `public=True`.
- Note create under public tag with `public=false` → stays
  private (override honored).
- `POST /notes/bulk/` inherits `tag.public` per omitted flag.
- Note create under private tag → `public=False` (unchanged).

### 5. Docs — `bible_research/DEVELOPER_GUIDE.md`

Document `POST /tags/{id}/set-public/` and the `public` field on
Tag; note the new-note default behavior.

## Frontend (`reactive-bible`)

### 1. Types — `src/types.ts` and `src/api.tsx`

Two `Tag` interfaces exist (both files) — add `public: boolean`
to each so they stay in sync.

### 2. API — `src/api.tsx`

```typescript
export const setTagPublic = async (
  tagId: string,
  makePublic: boolean
): Promise<{ tag: Tag; notes_updated: number }> =>
  authenticatedFetch(`${API_BASE_URL}/api/v1/tags/${tagId}/` +
    `set-public/`, {
      method: 'POST',
      body: JSON.stringify({ public: makePublic }),
    }).then(...)
```

(same ok/error handling pattern as `updateTag`).

### 3. `src/components/TagTree.tsx`

- New prop `onTogglePublic: (tag: Tag) => void`.
- New `ActionIcon` in the actions `Group`, before the view icon:
  `IconWorld` when public (e.g. `color="teal"`), `IconWorldOff`
  / gray when private; tooltips "Make tag public" /
  "Make tag private"; `e.stopPropagation()` like siblings.

### 4. `src/routes/TagManagementRoute.tsx`

`handleTogglePublic(tag)`:

- `window.confirm` matching the delete style:
  `"Make all notes under "X" public? Anyone with the link can
  view them."` / private variant.
- `await setTagPublic(tag.id, !tag.public)`
- Success notification including `notes_updated`.
- `await loadTags(true)` and `clearNotesCache(tag.id)` so the tag
  view refetches fresh flags.

### 5. `src/routes/TagNotesRoute.tsx`

- Same `ActionIcon` in the header controls `Group`, next to
  Refresh/Share (~line 700).
- Render only when the visitor owns the tag:
  `isAuthenticated && storedTags.some((t) => t.id === tag.id)`
  (store tags are already the caller's own).
- Handler mirrors the management route, then:
  `setTag(updatedTag)`, `clearNotesCache(tagId)`,
  `fetchNotes(tagId, …)` to refresh note flags.
- Optional: a subtle "Public" badge next to the tag name so
  owners see state at a glance.

### 6. `src/mocks/handlers.ts`

Add MSW handler for `POST /api/v1/tags/:id/set-public/`.

### 7. Optional polish

`AddTagNoteModal`: when the chosen tag is public, hint that the
new note will be publicly visible. No payload change needed —
the backend applies the default.

### 8. Frontend tests

- `TagTree`: button renders per node, calls `onTogglePublic`.
- Route-level test for the toggle via MSW (follow existing
  `src/routes/__tests__/` patterns).

## Edge cases

- Note with no tag / moved between tags: keeps its own flag;
  only creation inherits.
- Per-note privacy inside a public tag still possible (explicit
  `public=false` honored).
- Anonymous viewers: unchanged — they see notes with
  `public=True` on `/notes/tag/:id` as today.
- Persisted/cached tags from older clients lack `public` →
  falsy → treated private until a tag refresh; no store schema
  bump required.
- `guest` user's tags can technically be toggled anonymously
  (existing TagViewSet scoping); harmless, but the buttons are
  only rendered for authenticated owners anyway.

## Task order

1. Backend: `Tag.public` field → tell user to run
   `makemigrations` + `migrate` (never run them ourselves).
2. Backend: serializer changes + `set-public` action.
3. Backend: tests in `annotations/tests.py`
   (`pytest annotations/tests.py -q`; note the three known
   pre-existing failures on `main`).
4. Frontend: types + `setTagPublic` + MSW handler.
5. Frontend: `TagTree` + `TagManagementRoute` wiring.
6. Frontend: `TagNotesRoute` button.
7. Frontend: tests; `npm run lint`, `npm run build`,
   `npm test -- --run`.
8. Docs: update `bible_research/DEVELOPER_GUIDE.md` and
   `reactive-bible/DEVELOPER_GUIDE.md`.
