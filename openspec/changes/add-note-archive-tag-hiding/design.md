# Design

## Context

See proposal.md — Why. Key facts from investigation:

- `Tag` (annotations/models.py:22) and `Note`
  (annotations/models.py:103) are per-user models with existing
  boolean-flag precedent (`public`, `Comment.is_deleted`).
  `Note.tag_position` (FloatField, unique per tag) orders notes
  and is independent of any visibility flag.
- `TagViewSet.get_queryset` (views.py:243–281) annotates
  `note_count=Count('notes')`; `TagViewSet.set_public`
  (views.py:484–561) is the precedent for an atomic flag
  cascade onto notes. `NoteViewSet.get_queryset`
  (views.py:792–940) already parses query params;
  `get_accessible_notes_qs` (views.py:69–79) is the shared
  visibility scope for list/retrieve/comments/counts.
- Frontend tag options come from the Zustand store `state.tags`
  (populated by `GET /api/v1/tags/`); the note-creation dropdown
  is the single Mantine `Select` in `NoteForm.tsx:469–485`, shared
  by `AddTagNoteModal`, `AddStandaloneNoteModal`, `EditNoteModal`.
- `/tags` is `TagManagementRoute.tsx` + recursive `TagTree.tsx`;
  `EditTagModal.tsx` already hosts a "Public tag" `Switch`.
- `/notes/tag/:id` lists come from `useTagNotesData.ts` →
  `fetchNotes` → `GET /notes/?tag_id=`; note pages are cached in
  localStorage and invalidated via `clearNotesCache()`.
- Agents never run migrations; pytest builds its SQLite DB from
  migration files, so the new columns need a migration file in
  the change.

## Goals / Non-Goals

**Goals:**

- Server-side archive exclusion so pagination `count` stays
  correct and every consumer (list, linked notes, order index)
  behaves identically.
- Effective-archive semantics (`note.archived OR tag.archived`)
  plus a dumb write cascade on the tag flag: archive tag → all
  notes `archived=true`; unarchive tag → all notes
  `archived=false`, including previously individual ones.
- Order preservation for free: `tag_position` is never modified
  by any archive path.
- One-step single-note rescue: retagging clears `archived`.
- Owner-only visibility of effectively archived notes, including
  public ones on the anonymous/shared surface.
- Hidden and archived tags filtered in one place (`NoteForm`)
  covering all three note modals.
- Reversible toggles only — no deletes.

**Non-Goals:**

- A dedicated "archive" page; `/tags` toggle + tag note lists
  are the only browse path for archived notes.
- Provenance tracking ("archived individually vs via tag") —
  the cascade is deliberately dumb per product decision:
  unarchiving a tag unarchives ALL its notes.
- Hiding tags from other tag selectors (parent-tag pickers in
  `CreateTagModal`/`EditTagModal`, the `TagNotesToolbar`
  switcher, `NotesView` legacy select, `defaultTag` landing
  resolution) — they keep working unchanged. Archived tags are
  likewise still usable there.
- Dedicated `set-archived`/`set-hidden` endpoints (the cascade
  rides on plain PATCH per product decision).
- Bulk archive operations, comment-count changes for archived
  notes, archived handling in `bulk`/`reorder`/`move` actions.
- Blocking API writes that assign a note into an archived tag —
  allowed; the note is simply hidden (effective rule).

## Decisions

### D1: Plain PATCH for all three flags; tag PATCH carries the
cascade

`TagSerializer.Meta.fields` gains `hidden` and `archived`;
`NoteSerializer` gains `archived` (all writable). Unlike
`public`'s dedicated `set-public` action, `hidden` needs no
cascade and `archived` cascades from inside the regular
`partial_update` path per product decision — when the request
changes `archived`, wrap the flag write + notes bulk-update in
`transaction.atomic` (same mechanics `set_public` uses).
`updateTag`/`editNote` in the frontend API layer already issue
PATCHes, so no new endpoints.

Alternative considered: dedicated `set-archived` action —
rejected; user wants the cascade on the same endpoint.

### D2: Dumb write cascade, not computed-only visibility

Two mechanisms combine:

- `TagViewSet.partial_update` cascades `archived` onto
  `tag.notes.update(archived=<value>)` when the flag changes —
  both directions, no provenance. Unarchiving a tag wipes
  individual archive state (accepted product trade-off).
- The *visibility* rule is still `note.archived OR
  note.tag.archived` ("effective archive"), because the flag
  writes alone don't cover: notes individually unarchived while
  their tag is archived (must stay hidden — the UI disables
  Unarchive there), and notes assigned into an archived tag via
  API.

Alternatives considered: computed-only visibility (no writes) —
rejected by user, who wants the flags literally set and
unarchive-tag to resurrect individually archived notes;
cascade-only visibility (note.archived alone) — rejected, a
single-note unarchive under an archived tag would wrongly
surface it.

### D3: Archive exclusion points

- `NoteViewSet.get_queryset`: for `list`, apply the effective-
  archive exclusion `~(Q(archived=True) | Q(tag__archived=True))`
  unless `include_archived=true` parses truthy.
- `linked()` and `order_index()`: always apply the same
  exclusion (no opt-in — linked notes and reorder indices never
  need archived rows).
- `get_accessible_notes_qs(user)`: scope effectively archived
  rows to the owner — `Q(user=user) | (Q(archived=False) &
  (Q(tag__isnull=True) | Q(tag__archived=False)))`; anonymous
  callers get only the non-archived half. Owner keeps full
  retrieve/PATCH access for unarchiving and retagging.

### D4: Retagging clears `archived` (API semantic)

In `NoteSerializer.update` (or `perform_update`): when the
incoming `tag` differs from the current value, force
`archived=False` in the same write. API-level so every client
behaves identically; a PATCH that doesn't change `tag` never
touches `archived`. This is the single-step rescue from an
archived tag: retag → note is live (its flag cleared AND the new
tag isn't archived).

### D5: `note_count` annotation filters archived

`TagViewSet.get_queryset` changes `Count('notes')` →
`Count('notes', filter=Q(notes__archived=False))` in the
authenticated branch. Counts a note's own flag only — for an
archived tag every note was cascade-archived anyway, so the
count lands at 0 without needing a join in the annotation.

### D6: Hand-written `0014` migration ships with the change

One `annotations/migrations/0014_*.py` with three `AddField`
operations (`Note.archived`, `Tag.hidden`, `Tag.archived`),
modeled on `0013` (which added `tag.public`). Agents can't run
`makemigrations`, and pytest's DB is built from migration files —
without the file the new tests error. The user verifies
afterwards that `makemigrations annotations` reports no changes.

### D7: Dropdown filtering inside `NoteForm`'s Select data

`data={tags.filter(t => (!t.hidden && !t.archived) ||
t.id === selectedTagId).map(...)}`. Filtering by the *current
selection* (not a prop) means unavailable tags are never offered
for new selection, while an edit form pre-selected to one still
displays it — and the option drops out once the user picks
another tag. Covers all three modals in one line.

### D8: Show-archived state lives in the frontend store

A persisted Zustand flag `showArchivedNotes` toggled by a
`Switch` on `TagManagementRoute`. `useTagNotesData`/`fetchNotes`
pass `include_archived=true` when set; tag note lists render
archived rows with an "Archived" badge. `NoteCardActions` shows
Archive on live rows; on archived rows it shows Unarchive —
disabled with a tooltip ("unarchive the tag from /tags, or
retag the note") when `note.tag?.archived`. Every toggle calls
`clearNotesCache()` + refetch, mirroring `setTagPublic`/
`deleteNote` invalidation.

Alternative considered: a URL param — rejected; the toggle is
per-user UI preference, persisted state matches existing
patterns.

### D9: Tag-flag affordances in `EditTagModal` + tree indicators

- `EditTagModal`: "Hidden tag" and "Archived" `Switch`es beside
  the existing "Public tag" switch; `updateTag` PATCHes the
  flag(s), then `clearNotesCache()` +
  `getTags(forceRefresh)` (nested tag payloads on cached notes
  carry the flags, and the archive cascade rewrote notes).
- `TagTree`: subtle indicators near the public badge — eye-off
  for hidden, archive/box icon for archived — so both states are
  distinguishable on `/tags`; controls stay in the edit modal
  only.
- TS types: `Tag.hidden?`, `Tag.archived?`, `Note.archived?`
  (optional for additive persistence, `?? false` reads —
  same pattern as `tag.public ?? false`).

### D10: Backend tests pin the contract

- `test_tag_isolation.py` pinned tag-key set gains `hidden` and
  `archived`.
- New/extended tests modeled on `test_tag_public.py`:
  effective-archive exclusion on list/`linked`/`order_index`,
  `include_archived=true` owner opt-in, anonymous 404 on
  effectively archived public note, PATCH round-trips for all
  three flags, tag-archive cascade both directions (incl.
  wiping individual archive state), retag-clears-archived,
  `note_count` excluding archived.
- Frontend: MSW fixtures get the new fields + cascade behavior;
  `NoteForm`/`EditTagModal`/`TagManagementRoute`/
  `useTagNotesData`/`NoteCardActions` tests cover filtering,
  switches, the toggle, badges, and the disabled Unarchive
  tooltip.

## Risks / Trade-offs

- Cached note pages hold stale archived state → Mitigation:
  `clearNotesCache()` on every archive/hidden write and on the
  show-archived toggle (established pattern).
- `order_index` exclusion changes reorder numbering for tags
  with archived notes → acceptable: positions among visible
  notes stay consistent, and `tag_position` restoration on
  unarchive is exact.
- Unarchiving a tag loses per-note individual archive state →
  accepted product decision (documented in spec).
- Anonymous deep links to effectively archived public notes 404
  → accepted: archiving withdraws the note; spec'd as
  owner-only visibility.
- Cascade PATCH on a huge tag does one `UPDATE` per flag change
  → bounded by per-user tag sizes; `update()` is a single query,
  wrapped in `transaction.atomic`.
- Note assigned into an archived tag via API is silently hidden
  → spec'd behavior; the dropdown never offers archived tags so
  only raw API hits it.
- Hand-written migration could drift from `makemigrations`
  output → Mitigation: three plain `AddField`s mirroring `0013`;
  user runs `makemigrations annotations` to confirm no diff.
- Hidden/archived flags in nested tag payloads expose owner
  metadata to anyone who can see the note → harmless; accepted.

## Migration Plan

1. Merge backend PR (fields + migration + endpoints) — flags
   default false, existing rows unaffected.
2. User runs `python manage.py migrate` on environments that
   need it (and confirms `makemigrations annotations` produces
   nothing new).
3. Merge frontend PR — it only reads/sets the new fields; old
   payloads simply lack them (`?? false`).

Rollback: revert the PRs; the columns are additive
`default=False`, so no data migration is needed either way.

## Open Questions

None — resolved with the user: archive hides notes everywhere
incl. linked notes; unarchive browse path is the `/tags`
show-archived toggle; tag archive uses dumb cascade both
directions; retagging clears `archived` (one-step rescue);
hidden/archived tag controls are `EditTagModal` switches;
Unarchive is disabled with a tooltip while the tag is archived;
the edit dropdown keeps the note's current unavailable tag.
