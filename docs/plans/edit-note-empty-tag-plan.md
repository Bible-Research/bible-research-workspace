# Fix: Edit modal shows empty tag for verse-less notes

**Status: partial** — backend root cause fixed on main:
`NoteSerializer.to_representation` emits the nested tag plus
empty `verses`/`headings` for verse-less notes (#92). The
frontend defensive hardening is still unmerged on the
`test-quality/integration` branch (367e12e).

- **Date:** 2026-10-21
- **Repos touched:** `bible_research` (root cause),
  `reactive-bible` (defensive hardening)

## Symptom

Clicking "edit" on an existing note opens `EditNoteModal`, but the
Tag `Select` is empty even though the note has a tag. Submitting is
also impossible without manually re-picking a tag (`NoteForm`
blocks with "Cannot save note / Please choose a tag"), and autosave
never fires for these notes (the `!tagId` guard skips every tick).

## Root cause (backend)

`NoteSerializer.to_representation` in
`bible_research/annotations/serializers.py`:

- `tag` is declared as `PrimaryKeyRelatedField`, so the base
  representation serializes it as the raw id string.
- The nested expansion (`representation['tag'] =
  TagSerializer(instance.tag).data`) only happens near the end of
  `to_representation` (~line 566).
- But notes with **no verse links** hit an early return at
  ~line 343 (`if not verses: return representation`) — long before
  the tag expansion. Their `tag` field therefore stays the raw id
  string, e.g. `"tag": "TAG4B71DC37F99A475"`.
- The early return also skips `representation['verses']` /
  `['headings']` entirely — the keys are missing, not `[]`
  (frontend papers over this with `note.verses ?? []`).

Verified live (Tester5, read-only GET `/api/v1/notes/`): note
`NOT440C7ADE900C44E` returns `"tag": "TAG4B71DC37F99A475"` (a
string) and no `verses` key; every tagged note *with* verses
returns `tag` as the nested object. So the reproduction is: edit
any tagged note that has no verse references (e.g. a standalone
note created from a tag page).

## Frontend impact chain

`reactive-bible`'s `Note` type declares `tag: Tag | null` — the
string shape violates the contract everywhere `note.tag` is read:

- `src/components/EditNoteModal.tsx:252` —
  `tagId: note.tag?.id ?? ""`. On a string, `?.id` is `undefined`
  → `""` → `NoteForm`'s `selectedTagId` starts empty → Mantine
  `Select` shows nothing. **This is the reported bug.**
- `EditNoteModal.tsx:172` (`refreshViewedNotes`) —
  `note?.tag?.id` is `undefined` for string tags → falls back to
  `UNTAGGED_TAG_ID` → post-edit refetch loads the wrong list.
- `src/components/NoteCard.tsx:132` — heading falls back to
  `'Note'` instead of the tag name; share title (:185) falls back
  to `'Shared note'`.
- `src/routes/NoteDetailRoute.tsx:88,148` — same `'Shared note'`
  fallback for verse-less public notes.
- `src/components/AddTagNoteModal.tsx:47,334` — linked-note label
  is `''` and the tag badge is hidden.
- `src/store.tsx:412` / `src/routes/TagNotesRoute.tsx:325` —
  `n.tag?.id === tagId` is false for string tags (masked today by
  the `tags.some(t => t.id === tagId)` ownership branch).

## Prior art

Branch `test-quality/integration` (unmerged) already carries a
frontend-only fix: commit `367e12e` "Fix: Preserve tag on
verse-less notes in edit modal" adds a `noteTagId` helper
accepting `Tag | string | null` and uses it at both consumption
sites above. Worth cherry-picking/porting rather than rewriting.

Note on that commit's claim that edits "PATCH `tag:''` — silently
un-tagging": current code can't reach that — `NoteForm`'s submit
guard blocks on empty `selectedTagId`, and even if it did, DRF
3.17 `PrimaryKeyRelatedField` rejects `''` with 400
`does_not_exist` (verified in source). The real user impact is an
unsaveable edit, not data loss.

## Plan

### Step 1 — backend fix (root cause), `bible_research`

In `NoteSerializer.to_representation`, expand the tag **before**
the verse-less early return and always emit `verses`/`headings`:

```python
representation = super().to_representation(instance)
# Tag shape must not depend on verse enrichment.
if instance.tag:
    representation['tag'] = TagSerializer(instance.tag).data

verses = sorted(instance.verses.all(), key=lambda v: v.verse)
if not verses:
    representation['verses'] = []
    representation['headings'] = []
    return representation
```

Then remove the now-duplicated tag block near the end of the
method (~line 565-567).

- Add a regression test in `annotations/tests.py`: a note with a
  tag and no `NoteVerse` rows serializes `tag` as the nested object
  (with `id`/`name`) and includes `verses: []`, `headings: []`.
- Existing note tests must still pass
  (`python -m pytest annotations/tests.py -q`; known pre-existing
  failures on main: `test_note_serializer`,
  `test_single_query_assertion`, `test_position_already_occupied`
  — do not chase those).
- No migration needed — serializer-only change. Do NOT run any
  migration commands (workspace rule); none are required anyway.

### Step 2 — frontend hardening, `reactive-bible`

Port commit `367e12e`'s approach onto a fresh branch from `main`:

- Add `noteTagId(note): string | null` helper (accept
  `Tag | string | null`) in `EditNoteModal.tsx` or a small shared
  util (`src/utils/noteTag.ts`) so other consumers can reuse it.
- Use it at `EditNoteModal.tsx:252` (NoteForm `tagId` seed) and
  `:172` (`refreshViewedNotes` fallback).
- Keep the helper even after the backend ships: the notes cache
  (`bible_notes_cache_v2`) never expires and persisted
  `state.notes` may still hold string-shaped tags, so stale-client
  notes keep hitting the string path until their pages refetch.
- Optional (same helper family): string-tag fallbacks in
  `NoteCard` heading / `NoteDetailRoute` title /
  `AddTagNoteModal` badge. A string tag has no name client-side,
  so the best degraded display is tag-id-as-label or the existing
  fallbacks — low value; safe to defer once the backend fix lands.
- Extend `src/components/EditNoteModal.test.tsx`: a note whose
  `tag` is a raw id string still seeds the Select; the existing
  object-shape cases must keep passing.

### Step 3 — docs

- `bible_research/DEVELOPER_GUIDE.md`: document the `tag` contract
  (`Tag | null`, always expanded, independent of verses).
- `reactive-bible/DEVELOPER_GUIDE.md`: note the `noteTagId`
  helper / string-tag tolerance for cached data.

## Verification

1. Backend: new pytest case + spot-check the live response after
   deploy (tag dict on a verse-less note).
2. Frontend: `npm test -- --run` (new EditNoteModal case) +
   `npm run lint` on touched files.
3. Manual: create a standalone (verse-less) note under a tag →
   open the tag's notes page → Edit → Tag select shows the tag;
   Submit saves; closing refetches the correct tag list.
4. Line-length rule: `awk 'length > 79 {print FILENAME":"NR}'` on
   every touched file.
