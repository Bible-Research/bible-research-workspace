# Design — improve-inline-tag-creation

## Investigation results (task 1)

### Where things live today (task 1.1)

- **Note tag picker** — `src/components/NoteForm.tsx`, a
  Mantine v6 `<Select searchable>` (~line 462) whose `data`
  is `tags.map(t => ({ value: t.id, label: t.name }))`.
  Rendered by `src/components/AddTagNoteModal.tsx` (new note
  under a tag, opened from `VerseActionToolbar` and the
  notes views) and `src/components/EditNoteModal.tsx`. Both
  pass `onTagDropdownOpen` → `getTags()` (cache-aware).
- **Creation API** — `src/api/tags.ts`:
  `createTag(name, parentTagId)`, `reparentTag(id, parentId)`
  (PATCH `{parent_tag}` only), `deleteTag(id)`, `getTag(id)`
  (public fetch, returns `note_count`). Store refresh:
  `getTags(forceRefresh)` in
  `src/stores/bibleStore/notesSlice.ts`; `tags` prop comes
  from `state.tags`.
- **Tag helpers** — `src/utils/tagTree.ts`:
  `buildTagTree`, `isTagDescendant`, `validateTagName`,
  `TAG_NAME_MAX_LENGTH` (100). Path helpers to add here.
- **Toast system** — `@mantine/notifications`
  `showNotification`; `<Notifications position="top-right">`
  mounted in `src/App.tsx` (~line 105). `message` accepts a
  ReactNode, so Undo/Move buttons render inline.
- **"Old" modal** — `src/components/CreateTagModal.tsx` +
  `.test.tsx`, used ONLY by
  `src/routes/TagManagementRoute.tsx` (New Tag button +
  empty state). The note flow has no create affordance and
  no parent-prompt modal — the spec's "before" state is
  partially inaccurate; see Deviations.
- **Tests** — vitest + `@testing-library` + MSW
  (`src/mocks/handlers.ts` already stubs
  `GET/POST/PATCH/DELETE /tags/`); route-level "e2e"
  convention is `src/__tests__/integration/*.test.tsx` with
  `vi.mock('../../api')` (see `tag-refresh.test.tsx`).
- **Tag model** — `src/types.ts`:
  `{ id, name, parent_tag, public, note_count?,
  created_at, updated_at }`. Backend enforces uniqueness
  only for TOP-LEVEL names per user; same-named siblings
  under a parent are legal.

### Slash in tag names (task 1.2)

- Backend: `Tag.name` is a plain `CharField(100)` — `/` is
  permitted, no charset validation.
- Prod data (read-only Aiven query, 2026-11-16):
  **0 tags contain `/`**.
- Decision: inside the new combobox a query is a **path**
  iff splitting on `/` and dropping empty segments yields
  ≥2 segments (`Work/design`, `a//b` → paths; `Work/`,
  `/x`, `a/b/` (trailing → `a`/`b`)… see edge rules below).
  Otherwise it is a literal name — so `Work/` still creates
  a tag literally named `Work/` (not silently `Work`).
- Consequence: a tag whose name literally contains `/`
  cannot be created inline; `CreateTagModal` on Tag
  Management remains the escape hatch. Documented in the
  PR. Path matching treats query `/` as equivalent to the
  ` › ` display separator, so an existing `Work/design`
  chain is found (and create suppressed) rather than
  duplicated.

## Decisions

### Component: replace `Select` with a purpose-built combobox

New `src/components/TagCombobox.tsx` (or extend NoteForm —
kept separate for testability). Mantine v6 `Select`
supports `creatable` + `getCreateLabel` for exactly ONE
create row and no nested pane; the spec needs two rows plus
a sub-picker, so a hand-rolled listbox over `TextInput` +
`Popover`/`Paper` + `ScrollArea` is simpler than fighting
the component. Local state machine:
`view: 'options' | 'parents' | 'move'` + `highlightIndex`.
The dropdown renders inline (`Popover` `withinPortal` off):
portal'ing it to `document.body` would put it under the
note modal's `aria-hidden` (see Toast).

- **options view** — filtered existing tags (path labels),
  then the two create rows when query non-empty and no
  exact-path match.
- **parentPicker view** — the same input repurposed to
  filter candidate parents (existing tags, path labels),
  a `‹ Back` row for pointer users, header
  `Create "<query>" under:`.
- Enter activates the highlighted row and is
  `preventDefault`ed so it never submits the note form.
- Right Arrow / Tab on `under…` opens the picker; Esc
  unwinds one level (picker → options → close).
- ARIA per the spec's keyboard contract; the create rows
  are `role="option"` like any other row.

### Creation service

`src/utils/tagCreation.ts` (pure-ish, testable):

- `resolvePath(tags, query)` → segments, walking level by
  level, `{ existing: Tag[], missing: string[] }`.
- `createTagPath(query, parentId?)` → sequential
  `createTag` calls; on failure, delete the ids created in
  this call deepest-first (compensating rollback — see
  Atomicity) and rethrow.
- Returns `{ leaf: Tag, created: Tag[] }` — `created` is
  what Undo may delete.

### Attach + refresh

On success the combobox calls `getTags(true)` (store
refresh so path labels/counts update everywhere), reports
`onTagSelected(leaf.id)` to NoteForm which sets
`selectedTagId`, closes the dropdown, and fires the toast.

### Toast — in-modal, not the notifications portal

**Implementation finding:** Mantine v6 `Modal` applies
`aria-hidden` to every other top-level element while open —
including the `<Notifications>` portal and any `withinPortal`
dropdown. Inside a note modal that makes a
`showNotification` toast unreachable to assistive tech (and
invisible to `role=` queries), so the spec's "polite live
region" contract cannot be met through the notification
system. Instead `src/components/TagActionToast.tsx` renders
a `Notification`-styled `Affix` inside the combobox's own
DOM: created path + `Undo` / `Move under…` buttons +
auto-close timer (10 s with actions, 4 s for result/error
toasts). `role="status"` makes it a polite live region and
nothing auto-focuses.

**Undo**: revert `selectedTagId` to the pre-create value;
then for each created tag deepest-first re-list `getTags`
and delete only when `note_count` is 0 and it has no
children outside the deleted set; kept tags are reported in
the same toast. (The current note's own pending attachment
is excluded by reverting the selection first — a note
autosaved onto the tag makes `note_count` > 0 and the tag
is correctly treated as in use.)

**Move under…**: for the same `aria-hidden` reason a
nested `Modal` is unreachable inside the note modal, so
Move reopens the combobox dropdown in a `'move'` view —
the same searchable, path-labelled parent list as the
create `under…` picker, minus the tag itself and its
descendants (`isTagDescendant` — a re-parent under its own
child would cycle). Pick → `reparentTag(leaf.id, chosen)` +
`getTags(true)`, reported in the toast.

### Atomicity (path creation)

The API has no bulk endpoint. Sequential creates +
compensating deletes gives user-visible all-or-nothing: a
mid-chain failure deletes what this action created. The
one uncovered case is rollback itself failing (network
drops between create and delete) — surfaced as an error
toast naming the tags that may remain; Tag Management can
delete them. A true `POST /tags/path/` transaction is a
possible later backend task; deliberately out of scope.

## Deviations from the proposal text

1. **"Delete the old parent-prompt modal and its tests"** —
  No parent-prompt modal exists in the note flow (the old
  `creatable` row was removed in `0ad3502`). The only modal
  is `CreateTagModal`, Tag Management's create dialog; user
  confirmed (2026-11-16) it stays. The spec instead
  requires: creation from the note flow opens no modal.
2. **"e2e for the main flows"** — no Playwright/Cypress
  infra exists; the repo's e2e convention is
  `src/__tests__/integration/` (testing-library + mocked
  api layer). E2E coverage lands there.

## Risks

- Mantine v6 `Popover` + combobox focus management inside a
  `Modal` (both modals render NoteForm) — mitigated by an
  inline (non-portal) dropdown so options stay inside the
  modal's accessible tree, plus explicit focus handling;
  tested via user-event keyboard flows.
- Undo window vs note autosave: a tag autosaved under by
  the current note reads `note_count > 0` and is kept —
  intended ("used elsewhere"), and reverting the selection
  first lets the next save move the note back anyway.
- Duplicate path race (two tabs creating `Work/design`) —
  backend 400 on the top-level duplicate surfaces as the
  creation error; rollback cleans partial state.

## Test plan

- `src/utils/tagCreation.test.ts` — path parse, ancestor
  reuse, sibling matching, rollback on mid-chain failure,
  `Work/`/`/x` literal-name edges.
- `src/utils/tagTree.test.ts` additions — `tagPath`,
  `matchTagQuery` (slash⇔` › ` equivalence), exact-path
  suppression of create rows.
- `src/components/TagCombobox.test.tsx` — rendering both
  create rows, hiding direct-create on top-level dupe,
  keyboard nav (Down/Up/Enter/RightArrow/Tab/Esc), ARIA
  roles, Enter not submitting the form, parent picker
  create-under.
- `src/components/NoteForm.test.tsx` — wiring: created tag
  becomes `selectedTagId`, `onSubmit` receives it.
- `src/__tests__/integration/inline-tag-creation.test.tsx` —
  the main flows end-to-end through `AddTagNoteModal`:
  Enter-create top-level, under… picker create,
  `Work/design` path create with reuse, Undo deletes,
  Move under re-parents.
