# Proposal

## Why

Creating a tag while writing a note is impossible without
leaving the note flow. The tag dropdown in `NoteForm` (used by
both `AddTagNoteModal` and `EditNoteModal`) only searches
existing tags; the inline `+ Create` item was removed in
`0ad3502` (July 2026). To file a note under a new tag the user
must abandon the form, open Tag Management, use the
`CreateTagModal` dialog, then come back — and hierarchical
tags ("this belongs under X") take the same detour.

The result is flat, messy tag trees: users pick a near-enough
existing tag instead of creating the one they meant.

## What Changes

- The note's tag dropdown offers two inline create rows when
  the search text doesn't match an existing tag:
  - `+ Create "<query>"` — Enter or click creates a top-level
    tag immediately, attaches it to the note, no dialog.
  - `+ Create "<query>" under…` — swaps the dropdown into a
    searchable parent picker (Right Arrow/Tab opens it, Esc
    goes back) and creates the tag as a child of the chosen
    parent.
- Path shortcut: typing `Parent/child` (e.g. `Work/design`)
  creates the whole chain — existing ancestors are reused,
  missing ones are created, and the operation is atomic
  (all or nothing).
- Existing tags in the dropdown are labelled with their full
  path (`Work › design`) instead of the bare name, so
  same-named tags under different parents are
  distinguishable.
- After creating, a non-focus-stealing toast shows the new
  tag's path with two actions: **Undo** (deletes only the
  tags this action created that aren't used elsewhere) and
  **Move under…** (re-parents the new tag).
- The note-flow creation never opens a modal.
  `CreateTagModal` stays — it is Tag Management's create
  dialog, not part of the note flow (see design.md,
  "Deviations").

## Scope

Frontend only (`reactive-bible`). The backend tag API
(`POST/PATCH/DELETE /api/v1/tags/`, `note_count` in tag
payloads) already covers every write this needs; no backend
changes.

## Out of scope

- Tag creation on the Tag Management page (keeps
  `CreateTagModal`).
- Full-path labels in other tag selectors (`NotesView`,
  `TagNotesToolbar`, `CreateTagModal`, `EditTagModal`) —
  only the note tag dropdown changes.
- A backend bulk/atomic path-creation endpoint — the client
  achieves atomicity via compensating deletes (design.md,
  "Atomicity").
