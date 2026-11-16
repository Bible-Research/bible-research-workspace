# Tasks

## 1. Investigation

- [x] 1.1 frontend: Locate the note tag picker
  (`NoteForm.tsx` tag `Select`, used by `AddTagNoteModal` +
  `EditNoteModal`), the tag API (`src/api/tags.ts`), helpers
  (`src/utils/tagTree.ts`), toast system
  (`@mantine/notifications`, `src/App.tsx`) and the old modal
  (`CreateTagModal`, Tag Management only); record paths in
  design.md
- [x] 1.2 both: Check whether tag names may contain `/`
  (backend `Tag.name` CharField — allowed; prod query:
  0 rows) and record handling in design.md

## 2. Path helpers

- [x] 2.1 frontend: Add to `src/utils/tagTree.ts`:
  `tagPath(tags, tag)` (ancestor chain → ` › `-joined label),
  `tagPathSegments`, `normalizeTagPath`, `isExactPathMatch`,
  `isTopLevelNameTaken`, and `matchTagQuery(tags, query)`
  (normalize `/` ⇔ ` › `, substring match on path); unit
  tests
- [x] 2.2 frontend: Add `src/utils/tagCreation.ts`:
  `splitPathQuery` (≥2 non-empty segments → path),
  `resolvePath` (case-insensitive per-level ancestor
  lookup), `createTagPath` (sequential create + deepest-
  first compensating rollback); `isExactPathMatch` lives in
  `tagTree.ts`; unit tests incl. rollback and literal-slash
  edges

## 3. TagCombobox component

- [x] 3.1 frontend: Build `src/components/TagCombobox.tsx`
  (input + inline `Popover` dropdown — `withinPortal` off so
  it stays inside the note modal's accessible tree):
  existing tags filtered by `matchTagQuery` with path
  labels; ARIA combobox/listbox/option contract;
  Up/Down/Enter/Esc; Enter never submits the form;
  component tests
- [x] 3.2 frontend: Append create rows `+ Create "<query>"`
  (hidden on top-level dupe or exact-path match) and
  `+ Create "<query>" under…`; Enter/click on create calls
  the creation service, selects the leaf, closes; tests
- [x] 3.3 frontend: Parent-picker view — Right Arrow/Tab on
  `under…` opens, Esc returns, searchable parent list with
  path labels + `‹ Back` row; pick → create as child;
  tests

## 4. Wire into NoteForm

- [x] 4.1 frontend: Replace the tag `Select` in
  `NoteForm.tsx` with `TagCombobox` (same value/onChange/
  onDropdownOpen contract); keep autosave/draft behavior;
  update NoteForm/AddTagNoteModal/EditNoteModal tests
- [x] 4.2 frontend: After creation refresh `getTags(true)`
  and set the new leaf as `selectedTagId`; tests

## 5. Post-create toast

- [x] 5.1 frontend: `src/components/TagActionToast.tsx` —
  rendered inside the combobox DOM (the notifications
  portal is `aria-hidden` while the note modal is open, so
  `showNotification` cannot satisfy the live-region
  contract): `role="status"`, path message, no focus
  steal, `Undo` + `Move under…` buttons; tests
- [x] 5.2 frontend: Undo — revert selection, then
  deepest-first delete created tags whose fresh `getTags`
  listing shows `note_count` 0 and no children outside
  the deleted set; report kept tags; tests
- [x] 5.3 frontend: Move under… — reopens the dropdown in
  a 'move' pane (same searchable parent list, excluding
  the tag and its descendants; a nested `Modal` would be
  `aria-hidden` under the note modal) → `reparentTag` +
  `getTags(true)`; tests

## 6. End-to-end + docs

- [x] 6.1 frontend:
  `src/__tests__/integration/inline-tag-creation.test.tsx` —
  through `AddTagNoteModal`: Enter-create top-level,
  under… create, `Work/design` path create with ancestor
  reuse, Undo, Move under
- [ ] 6.2 frontend: Update `DEVELOPER_GUIDE.md` (tag
  creation UX section) and `AGENTS.md` if contracts moved;
  verify `awk 'length>79'` clean on all touched files

## 7. Verification

- [ ] 7.1 `npm run lint` (0 errors, 0 warnings),
  `npm run build` (tsc), `npm test -- --run`, and
  `openspec validate improve-inline-tag-creation --strict`
  all pass; tasks.md checkboxes current
