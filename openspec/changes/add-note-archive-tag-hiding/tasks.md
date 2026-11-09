# Tasks

## 1. Backend: model fields + migration

- [ ] 1.1 backend: Add `archived = BooleanField(default=False)`
  to `Note`, and `hidden` + `archived` BooleanFields to `Tag` in
  `annotations/models.py` (help_text mirroring `public`); verify
  no line exceeds 79 chars (`awk 'length>79{print
  FILENAME":"NR}' annotations/models.py`)
- [ ] 1.2 backend: Hand-write
  `annotations/migrations/0014_note_archived_tag_hidden_
  archived.py` with three `AddField` ops modeled on `0013`;
  verify it lists `dependencies = [("annotations",
  "0013_...")]` matching the real latest migration filename and
  `python manage.py check` passes (never run
  makemigrations/migrate)

## 2. Backend: serializers, cascade, views, admin

- [ ] 2.1 backend: Add `archived` to `NoteSerializer.Meta.fields`
  and `hidden` + `archived` to `TagSerializer.Meta.fields`
  (writable); verify via API that PATCH sets each flag and it
  appears in responses
- [ ] 2.2 backend: In `NoteSerializer.update` (or
  `perform_update`), force `archived=False` when the incoming
  `tag` differs from the current value; verify retagging an
  archived note returns `archived: false` while a
  `note_text`-only PATCH keeps `archived: true`
- [ ] 2.3 backend: In `TagViewSet.partial_update`, when
  `archived` changes, cascade `tag.notes.update(archived=value)`
  inside `transaction.atomic` (mirroring `set_public`); verify
  archiving then unarchiving a tag sets/clears the flag on all
  its notes including individually archived ones
- [ ] 2.4 backend: In `NoteViewSet`, exclude effectively
  archived notes (`Q(archived=True) | Q(tag__archived=True)`)
  from `list` unless `include_archived=true`, and always in
  `linked` and `order_index`; verify manual requests show them
  absent by default and present with the opt-in
- [ ] 2.5 backend: In `get_accessible_notes_qs`, scope
  effectively archived notes to their owner (`Q(user=user) |
  live`); verify anonymous GET of an archived public note — and
  of a live note under an archived public tag — returns 404
  while the owner still retrieves and PATCHes it
- [ ] 2.6 backend: Change `TagViewSet.get_queryset`
  `note_count` to `Count('notes',
  filter=Q(notes__archived=False))`; verify a tag's count drops
  when a note is archived
- [ ] 2.7 backend: Surface all three flags in
  `annotations/admin.py` (`list_display`/`list_filter`/
  fieldsets, mirroring `CommentAdmin.is_deleted`); verify admin
  loads for Tag and Note

## 3. Backend: tests + docs

- [ ] 3.1 backend: Update `test_tag_isolation.py`'s pinned tag
  response key set to include `hidden` and `archived`; verify
  the test passes
- [ ] 3.2 backend: Add tests modeled on `test_tag_public.py`
  covering: effective-archive exclusion on list/`linked`/
  `order_index`, `include_archived=true` owner opt-in, anonymous
  404 on effectively archived public note, PATCH round-trips for
  all three flags, tag-archive cascade both directions (incl.
  wiping individual archive state), retag-clears-archived,
  `note_count` excluding archived; verify with
  `python -m pytest` on touched test files only
- [ ] 3.3 backend: Update `DEVELOPER_GUIDE.md` notes/tags
  endpoint docs with `archived`, `hidden`, `include_archived`,
  the tag cascade, and retag-clears-archived; verify docs match
  the code

## 4. Frontend: types, API, store

- [ ] 4.1 frontend: Add `archived?: boolean` to `Note` and
  `hidden?` + `archived?` to `Tag` in `src/types.ts`; extend
  `api/notes.ts` fetch params with `include_archived` and
  `api/tags.ts` `updateTag` payload to carry the flags; verify
  `npm run lint` and `tsc` pass
- [ ] 4.2 frontend: Add persisted `showArchivedNotes` state +
  setter in the Zustand store and a `setNoteArchived`-style
  action in `stores/bibleStore/notesSlice.ts` that PATCHes,
  calls `clearNotesCache()`, and refetches (mirroring
  `deleteNote`); verify persistence round-trips across reload
- [ ] 4.3 frontend: Update MSW handlers/fixtures
  (`src/mocks/handlers.ts`) with the new fields, tag-archive
  cascade behavior, and retag-clears-archived; verify existing
  tests still pass (`npm test -- --run`)

## 5. Frontend: archive UX

- [ ] 5.1 frontend: Add Archive/Unarchive actions to
  `NoteCardActions.tsx` (owner-gated like edit/delete);
  Unarchive disabled with an explanatory tooltip when
  `note.tag?.archived`; verify component tests cover enabled,
  disabled, and badge states
- [ ] 5.2 frontend: Thread `showArchivedNotes` into
  `useTagNotesData.ts` so lists pass `include_archived=true` and
  render archived rows with an "Archived" badge; verify archived
  notes hide by default and appear (badged) when the flag is on
- [ ] 5.3 frontend: Add a "Show archived notes" `Switch` to
  `TagManagementRoute.tsx` bound to the store flag with
  `clearNotesCache()` + refetch on toggle; verify toggling on
  `/tags` reveals archived notes in tag note lists

## 6. Frontend: tag flags UX + tests + docs

- [ ] 6.1 frontend: Filter hidden and archived tags in
  `NoteForm.tsx`'s Select data (`(!t.hidden && !t.archived) ||
  t.id === selectedTagId`); verify tests show them absent for
  new selection but present as the current tag while editing
- [ ] 6.2 frontend: Add "Hidden tag" and "Archived" `Switch`es
  to `EditTagModal.tsx` beside "Public tag", saving via
  `updateTag` + `clearNotesCache()` + `getTags(forceRefresh)`;
  verify the archive switch cascades notes end-to-end in tests
- [ ] 6.3 frontend: Add hidden (eye-off) and archived
  (archive/box) indicators to `TagTree.tsx` rows near the public
  badge; verify flagged tags are visually distinct on `/tags`
- [ ] 6.4 frontend: Update affected tests
  (`TagManagementRoute`, `TagTree`, `NoteForm`, `EditTagModal`,
  `NoteCardActions`, `useTagNotesData` consumers) and add
  coverage for the new behavior; verify `npm test -- --run` is
  green and no line exceeds 79 chars
- [ ] 6.5 frontend: Update `DEVELOPER_GUIDE.md` with archive /
  hide behavior incl. retag-rescue and the disabled-Unarchive
  rule; verify docs match shipped UI

## Workflow follow-up

- User runs `python manage.py makemigrations annotations` to
  confirm the hand-written `0014` matches (no new migration),
  and `python manage.py migrate` on environments that need it.
- Archive the change after PRs merge (`openspec archive`).
- Verify `openspec validate --all --strict` after archiving.
