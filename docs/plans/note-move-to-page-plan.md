# Plan: Move a note to a different pagination page (custom order)

## Goal

On `TagNotesRoute`, when the tag is in custom sort order
(`custom_asc` / `custom_desc`), each note card gets a "move to
page" icon button. Clicking it opens a modal where the user picks
a target page; the note is moved to the **top of that page's
range**.

**Architecture decision**: the reordering math lives in the
backend. The frontend sends one small request and then purges its
cached notes for the tag and refetches — it never computes
positions or sends bulk `updates[]`.

## Why server-side (vs. the earlier client-side renumber)

- **Atomicity**: a client flow of "fetch all notes -> compute ->
  POST reorder" has a race window; a note created between the
  fetch and the POST is missed and either 409s or lands wrong.
  Server-side does it in one `transaction.atomic()`.
- **Payload**: client-side needed up to 10 paginated fetches of
  fully-serialized notes just to read IDs, plus a ≤1000-item
  updates body. Server-side needs one request + one page refetch.
- **Single source of truth**: ordering semantics (`custom` =
  `tag_position` asc nulls-last + `created_at` desc tiebreak)
  stay in `_apply_ordering`; no duplicated index math on the
  client.
- **Cap**: the `notes/reorder/` 100-update limit is a *request
  payload* cap, not a row limit. `move` has no `updates[]` array,
  so renumbering 750+ rows internally is unaffected — the
  100→1000 raise is NOT needed.

## Current state

- In-page drag reorder exists: `TagSection` (dnd-kit) ->
  `store.reorderNotes(tagId, noteIds, page, pageSize)` ->
  `api.reorderNotes` -> `POST /api/v1/notes/reorder/`
  (`reactive-bible/src/api.tsx:350`). Only draggable when custom
  sort + authenticated.
- Backend `reorder` action:
  `bible_research/annotations/views.py:476` — validates
  `tag_id` + `updates[]` (max 100), unique positions, scopes to
  `user=user, tag_id=tag_id`, two-phase negative-position update
  under `select_for_update` + `transaction.atomic()`; maps the
  `unique_tag_position_per_tag` IntegrityError to 409.
- `NotesPagination` (`annotations/pagination.py`): `page_size=25`,
  `max_page_size=100` — no change needed.
- Note ids are `NOT…` strings; `tag_position` float default
  `get_unix_timestamp`, nullable.

## Backend changes (`bible_research/`)

### 1. New `move` action in `NoteViewSet` (`annotations/views.py`)

```
POST /api/v1/notes/move/
{
  "tag_id": "TAG…",
  "note_id": "NOT…",
  "target_page": 3,      # 1-based, in the DISPLAYED direction
  "page_size": 25,
  "direction": "asc"     # or "desc" — matches custom_asc/_desc
}
-> 204 No Content
```

Implementation inside `transaction.atomic()`:

1. Validate fields: `tag_id`, `note_id` required;
   `target_page >= 1`; `page_size >= 1`; `direction` in
   `{'asc', 'desc'}`. Any failure -> 400 `validation_error`
   (same body shape as `reorder`).
2. Fetch the tag's notes in canonical ascending order — the same
   ordering `_apply_ordering` uses for `custom`:
   `F('tag_position').asc(nulls_last=True), '-created_at'`,
   scoped to `user=request.user, tag_id=tag_id`, wrapped in
   `select_for_update()` (locks the rows for the transaction).
   Extract a tiny shared helper so `reorder`/`move` can't drift.
3. Remove `note_id` from the list; if absent -> 400/404 with the
   same "not found or do not belong to tag" message `reorder`
   uses.
4. Compute the insert index in ascending-position space:
   - `asc`: `(target_page - 1) * page_size`
   - `desc`: `len(remaining) - (target_page - 1) * page_size`
   - clamp to `[0, len(remaining)]`, splice the note in.
5. Two-phase renumber identical to `reorder`: set all to
   `-(i + 1)`, `bulk_update`, then `position = index + 1`,
   `bulk_update`. Keeps `unique_tag_position_per_tag` satisfied
   during the shuffle and normalizes null/fractional drift.
6. Catch `IntegrityError` and return the same 409 contract
   (`error: 'position_conflict'` / `database_error`) as
   `reorder` — covers a concurrent insert racing the renumber.
7. Return `Response(status=204)`.

### 2. Reorder cap — no change

`len(updates) > 100` stays. Justification above; raising it is a
one-line follow-up if ever needed.

### 3. Tests (`annotations/tests.py`, new `TestNoteMove` class)

- Move a middle note to `target_page=1` (asc): returns 204, note
  gets `tag_position=1`, others shift +1 (positions dense 1..M).
- The motivating case: ~50 notes, `page_size=25`, move a page-3
  note to `target_page=1` — 204 and full renumber, no cap error.
- `direction='desc'`: note lands at the top of displayed page N
  (verify via the mirrored index: fetch ordering `-custom` page N
  first item == moved note).
- `target_page` beyond the last page clamps (lands at end for
  asc) rather than 500s.
- Invalid payloads: missing fields, `target_page=0`,
  `direction='bogus'` -> 400.
- Foreign note / wrong tag -> 400 (mirrors reorder tests).
- Unauthenticated -> 401/403 (check existing permission tests).

Run: `source venv/bin/activate && python -m pytest
annotations/tests.py::TestNoteMove -q` (never run migrations;
tests use in-memory SQLite).

### 4. `DEVELOPER_GUIDE.md`

Document `POST /api/v1/notes/move/` (request/response, semantics)
per workspace rule 6.

## Frontend changes (`reactive-bible/`)

### `src/api.tsx`

```ts
export const moveNoteToPage = async (
  tagId: string,
  noteId: string,
  targetPage: number,
  pageSize: number,
  direction: 'asc' | 'desc'
): Promise<void>
```

`authenticatedFetch` POST to `/api/v1/notes/move/`; same response
handling style as `reorderNotes` (409 -> `ConflictError`, other
non-ok -> `Error`).

### `src/store.tsx`

New action `moveNoteToPage(tagId, noteId, targetPage, direction)`:

1. `await api.moveNoteToPage(tagId, noteId, targetPage,
   notesPageSize, direction)`.
2. On success — purge and trust the server:
   `clearNotesCache(tagId)` then
   `fetchNotes(tagId, { ordering: <current api ordering>,
   page: targetPage, append: false })` so the user lands on the
   target page with the moved note on top. Green "Note moved"
   notification.
3. `ConflictError`/Error -> notification + `fetchNotes(tagId)`
   refresh (same pattern as `reorderNotes`).

Separate small fix while here: `reorderNotes` never clears
`notesCache`, so a cached page can resurrect the old order on
revisit — add `clearNotesCache(tagId)` in its success path.

No changes needed: `getAllNotes` stays unused; no client-side
ordering util.

### `src/components/NoteCard.tsx`

- New optional prop `onMoveToPage?: (note: Note) => void`.
- `ActionIcon` (e.g. `IconArrowsMove`) + `Tooltip` "Move to page"
  in BOTH icon groups (folded block ~line 185, unfolded ~line
  300), gated by `onMoveToPage && note.is_owner !== false`, next
  to Edit. Optional prop keeps it off shared-note/detail views.

### `src/components/SortableNoteCard.tsx`

- Pass `onMoveToPage` through to `NoteCard`.

### `src/components/TagSection.tsx`

- New props `onMoveToPage?: (note: Note) => void` and
  `totalPages?: number`; pass through in both render branches
  (parent decides visibility).

### New `src/components/MoveNoteToPageModal.tsx`

Mantine `Modal` ("Move note to page") + `Select` with options
`1..totalPages` (`Page N`, `searchable`), default = current page,
"Move" button -> `onConfirm(page)`. Mirrors `EditNoteModal`.

### `src/routes/TagNotesRoute.tsx`

- State `noteToMove: Note | null`.
- `canMoveToPage = isAuthenticated && (custom sort) &&
  totalPages > 1` (same predicate as `isDraggable` at lines
  738-742 + more-than-one-page).
- `handleMoveToPageConfirm` -> `moveNoteToPage(tagId,
  noteToMove.id, targetPage, sortOrder === 'custom_desc' ? 'desc'
  : 'asc')` then close the modal; the store refetch updates
  `notesPage` to the target page.
- Pass `onMoveToPage` + `totalPages` into `TagSection`; render
  `<MoveNoteToPageModal …>` next to `EditNoteModal`.

## Semantics / edge cases

- "Top of the page" = insert index `(target_page-1)*page_size` in
  ascending space; desc mirrors it. Verified: desc page 1 top =
  end of the ascending array.
- Same-page target is allowed — acts as "jump to top of page".
- `target_page` beyond the last page: backend clamps the index
  (lands at the tail); the frontend `Select` only offers valid
  pages anyway.
- Concurrent note creation mid-transaction: `select_for_update`
  locks existing rows; a racing INSERT can still collide on a
  position -> IntegrityError -> 409 -> client refreshes.
- Null `tag_position`s order last (asc) and are renumbered to
  clean integers — drift normalized.
- Shared/public tags: button only renders for the owner
  (`is_owner` + `isDraggable` predicate); endpoint scopes
  `user=request.user` regardless.

## Considered and rejected

- **Client-side renumber + bulk reorder**: race window, heavy
  payloads, duplicated ordering math; also the only thing that
  needed the 100→1000 cap.
- **Fractional insert** (one update, midpoint position): drifts
  toward float64 precision limits (~50 bisections per gap).
- **Move endpoint returning the serialized target page**: saves
  one GET but breaks the uniform "purge cache + refetch" flow and
  duplicates pagination/serialization logic; keep 204 + refetch.
- **Raising `max_page_size`**: irrelevant now.

## Testing & verification

Backend: `python -m pytest annotations/tests.py::TestNoteMove -q`
plus existing `TestPartialReordering` (unchanged).

Frontend:
- Store/api test for `moveNoteToPage`: MSW handler for
  `POST /api/v1/notes/move/` (currently unmocked); assert payload
  fields and that cache is cleared + `fetchNotes` re-called with
  the target page.
- `npm run lint` (zero warnings) + `npm run build` (tsc).
- `awk 'length > 79 {print FILENAME":"NR}'` on every touched file.

Manual UI check:
- Tag with >1 page, `Custom: Ascending`: move a page-3 note to
  page 1 -> lands on top; hard-refresh -> order persists (cache
  cleared). Repeat in `Custom: Descending`; confirm drag reorder
  and pagination still work.

## Task checklist

1. Backend: `move` action + shared ordering helper.
2. Backend: `TestNoteMove` tests; run them.
3. Backend: `DEVELOPER_GUIDE.md` — document `notes/move/`.
4. `api.tsx`: `moveNoteToPage`.
5. `store.tsx`: `moveNoteToPage` action + cache-clear fix in
   `reorderNotes`.
6. `MoveNoteToPageModal.tsx` (new).
7. `NoteCard` / `SortableNoteCard` / `TagSection`:
   `onMoveToPage` prop + icon.
8. `TagNotesRoute.tsx`: modal state, handler, props.
9. `reactive-bible/DEVELOPER_GUIDE.md`: document the feature.
10. Lint/build/79-char check; manual UI verification.
