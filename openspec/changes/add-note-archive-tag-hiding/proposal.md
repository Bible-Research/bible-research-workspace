# Proposal

## Why

Users accumulate notes and tags they no longer actively use but do
not want to delete. There is no way to retire a note without
deleting it, and no way to keep a tag out of the note-creation
dropdown without removing it. Archiving notes and tags, plus
hiding tags, let users declutter while keeping data recoverable.

## What Changes

- `Note` gains an `archived` boolean, writable via
  `PATCH /api/v1/notes/<id>/`. Retagging a note (changing its
  `tag`) clears `archived` in the same request, so moving a note
  to a live tag restores its visibility in one step.
- `Tag` gains two booleans, both writable via
  `PATCH /api/v1/tags/<id>/`:
  - `hidden` — cosmetic only; the frontend omits hidden tags from
    the note-creation dropdown. No cascade.
  - `archived` — PATCHing it cascades `archived` to every note in
    the tag atomically, both directions (unarchiving a tag
    unarchives ALL its notes, including ones archived
    individually). Note `tag_position` values are never touched,
    so unarchiving restores the previous order exactly.
- A note is *effectively archived* when `note.archived` OR its
  tag's `archived` is set. Effectively archived notes are
  excluded from every enumeration surface — `GET /api/v1/notes/`,
  `POST /api/v1/notes/linked/`, `GET /api/v1/notes/order-index/`
  — unless the owner passes `include_archived=true`. They are
  visible only to their owner: archiving (directly or via tag)
  removes even public notes from the anonymous/shared surface.
- The `note_count` annotation on tags excludes
  `note.archived=true` rows.
- Frontend: note row actions gain Archive/Unarchive (Unarchive
  disabled with a tooltip while the note's tag is archived);
  a "Show archived" toggle on `/tags` reveals effectively
  archived notes (badged) in tag note lists; `EditTagModal` gains
  "Hidden tag" and "Archived" switches; hidden and archived tags
  are visually marked in the `/tags` tree; the note-form tag
  dropdown filters hidden and archived tags except the note's
  currently assigned tag.
- One new migration `annotations/0014` adds all three fields.

## Capabilities

### New Capabilities

- `note-tag-visibility`: lifecycle visibility flags on notes
  (`archived`) and tags (`hidden`, `archived`) — the tag→notes
  archive cascade, effective-archived semantics, which API
  surfaces enumerate archived notes and the `include_archived`
  opt-in, owner-only visibility of archived notes, exclusion of
  hidden/archived tags from the note-creation dropdown, and the
  frontend affordances that expose the flags.

### Modified Capabilities

- None. The `auth` spec pins anonymous list/comment-counts
  behavior, which archiving does not alter (anonymous tag/note
  lists were already empty; archived public notes leaving the
  shared surface is new behavior owned by
  `note-tag-visibility`).

## Impact

- Backend (`bible_research/`): `annotations/models.py` (three
  fields), `annotations/serializers.py` (`TagSerializer`,
  `NoteSerializer` + retag-clears-archived in update),
  `annotations/views.py` (`TagViewSet` PATCH cascade,
  `note_count` annotation; `NoteViewSet.get_queryset`, `linked`,
  `order_index`; `get_accessible_notes_qs`),
  `annotations/admin.py`, new `annotations/migrations/0014_*.py`,
  tests (incl. the pinned tag-key test in
  `test_tag_isolation.py`), `DEVELOPER_GUIDE.md`.
- Frontend (`reactive-bible/`): `src/types.ts`,
  `src/components/NoteForm.tsx`, `EditTagModal.tsx`,
  `TagTree.tsx`, `TagManagementRoute.tsx`, `NoteCardActions.tsx`,
  `src/hooks/useTagNotesData.ts`, `src/stores/bibleStore/
  notesSlice.ts`, `src/api/notes.ts`, `src/api/tags.ts`, MSW
  handlers, related tests, `DEVELOPER_GUIDE.md`.
- Migration: agents never run migrations; the change ships a
  hand-written `0014` migration (three `AddField` ops mirroring
  `0013`) so tests have the columns, and the user verifies with
  `python manage.py makemigrations annotations` that no further
  migration is generated.
