# Spec Delta

## Purpose

Defines how new tags are created inline from the note's tag
search (`NoteForm`'s tag dropdown, used by both the add-note
and edit-note flows), replacing the absent/moved creation
path with a dialog-free combobox flow.

## ADDED Requirements

### Requirement: Inline create options in the note tag dropdown

When the tag search field contains text that does not exactly
match the full path of an existing tag, the dropdown SHALL
offer, after any matching tags, the rows
`+ Create "<query>"` and `+ Create "<query>" under…` in that
order. `+ Create "<query>"` SHALL be hidden when a top-level
tag with that exact name (case-insensitive) already exists,
since the backend rejects duplicate top-level names.

#### Scenario: Typing a new name shows both create rows

- **WHEN** the user types `design` and no tag's path ends in
  an exact `design` match
- **THEN** the dropdown shows `+ Create "design"` followed
  by `+ Create "design" under…`

#### Scenario: Top-level duplicate hides the direct create row

- **WHEN** the user types `Work` and a top-level tag `work`
  already exists
- **THEN** `+ Create "Work"` is not offered, while
  `+ Create "Work" under…` remains available (same-named
  children under a parent are allowed)

### Requirement: Immediate top-level creation

Activating `+ Create "<query>"` (Enter while highlighted, or
click) SHALL create a top-level tag named `<query>`,
select it as the note's tag, and close the dropdown without
opening any dialog. A query containing `/` instead triggers
path creation (see Path shortcut).

#### Scenario: Enter creates and selects the tag

- **WHEN** the user types `design`, highlights
  `+ Create "design"`, and presses Enter
- **THEN** a top-level tag `design` is created, it becomes
  the note's selected tag, the dropdown closes, and no
  modal opens

### Requirement: In-dropdown parent picker

Activating `+ Create "<query>" under…` (click, Right Arrow,
or Tab while highlighted) SHALL replace the dropdown's
contents with a searchable list of existing tags — labelled
by full path — to choose as the parent. Esc SHALL return to
the normal options view with the query intact. Choosing a
parent SHALL create the tag as a child of it and select it
for the note. With a `/` query, the resolved chain is
created beneath the chosen parent.

#### Scenario: Create under a chosen parent

- **WHEN** the user types `design`, opens `under…`, searches
  `work`, and picks `Work`
- **THEN** a tag `design` is created with parent `Work` and
  selected for the note

#### Scenario: Esc abandons the picker, not the query

- **WHEN** the user is inside the parent picker and presses
  Esc
- **THEN** the dropdown returns to the options view showing
  the create rows for the unchanged query

### Requirement: Path shortcut

A query containing `/` between non-empty segments SHALL be
treated as a `Parent/child` path. `+ Create "<query>"`
resolves the chain from the top level; `under…` resolves it
beneath the chosen parent. Each existing ancestor matched
case-insensitively by name at its level SHALL be reused;
each missing segment SHALL be created. If the resolved path
already exists, the create rows are suppressed and the
existing tag is offered.

#### Scenario: Typing a path shows a single create row

- **WHEN** no `Work/design` chain exists and the user types
  `Work/design`
- **THEN** the dropdown shows `+ Create "Work/design"` and
  `+ Create "Work/design" under…`

### Requirement: Path creation atomicity

Path creation SHALL be atomic from the user's perspective:
if any creation fails, tags created by the same action are
deleted deepest-first, and reused ancestors are never
deleted. The user SHALL be told when creation failed.

#### Scenario: Reuse and create mixed chain

- **WHEN** tag `Work` exists at top level and the user types
  `Work/design` and activates `+ Create "Work/design"`
- **THEN** `Work` is reused, `design` is created under it,
  and `design` is selected for the note

#### Scenario: Failure rolls back partial creation

- **WHEN** creating `a/b/c` fails after `a` and `b` were
  created (neither existed before)
- **THEN** `b` and `a` are deleted so no partial chain
  remains, and the user is told creation failed

### Requirement: Full-path labels

Every existing tag listed in the note tag dropdown SHALL be
labelled with its full ancestor path using ` › ` as the
separator (e.g. `Work › design`). Queries SHALL match
against that path, with `/`-separated input treated as
equivalent to ` › ` separators.

#### Scenario: Distinguishing same-named tags

- **WHEN** tags `Work/design` and `School/design` both exist
- **THEN** the dropdown lists `Work › design` and
  `School › design` as separate selectable options

#### Scenario: Slash query matches existing paths

- **WHEN** the user types `Work/design` and that exact path
  exists
- **THEN** the existing `Work › design` tag is offered for
  selection and the create rows are suppressed

### Requirement: Post-creation toast

After a successful creation a toast SHALL announce the
created tag's full path. It SHALL NOT steal keyboard focus,
SHALL be rendered in a polite live region, and SHALL offer
**Undo** and **Move under…** actions for the duration it
is visible.

#### Scenario: Toast announces the new path

- **WHEN** `design` is created under `Work`
- **THEN** a toast showing `Work › design` appears with
  Undo and Move under… actions, and focus stays on the
  tag field

### Requirement: Undo reverts the creation

Undo SHALL revert the note's tag selection to its
pre-creation value, then delete deepest-first every tag the
action created that is still unused. A created tag that has
since gained notes or child tags outside the created set
SHALL be kept and reported as kept.

#### Scenario: Undo removes a fresh top-level tag

- **WHEN** the user creates `design` and activates Undo
  before any note is saved under it
- **THEN** `design` is deleted and the note's tag selection
  reverts

#### Scenario: Undo keeps a tag now in use

- **WHEN** the user creates `design`, another note is saved
  under it, and the user then activates Undo
- **THEN** `design` is kept and the toast reports it stayed
  because it is in use

### Requirement: Move under re-parents the new tag

Move under… SHALL open a searchable chooser of existing
tags (full-path labels); choosing one SHALL re-parent the
created tag beneath it without renaming it.

#### Scenario: Move under re-parents the new tag

- **WHEN** the user creates top-level `design` and activates
  Move under… then picks `Work`
- **THEN** `design` becomes a child of `Work`

### Requirement: Keyboard and ARIA contract

The tag field SHALL be an ARIA combobox (`role="combobox"`,
`aria-expanded`, `aria-controls`, `aria-activedescendant`)
with the dropdown as `role="listbox"` and each row as
`role="option"`. Up/Down SHALL move the highlight, Enter
SHALL activate it without submitting the note form, Right
Arrow or Tab on the `under…` row SHALL open the parent
picker, and Esc SHALL step back (picker → options → closed
dropdown). The creation toast SHALL NOT move focus.

#### Scenario: Full keyboard flow

- **WHEN** the user focuses the tag field, types `design`,
  presses Down to `under…`, presses Right Arrow, arrows to
  `Work`, and presses Enter
- **THEN** `design` is created under `Work` and focus
  remains on the tag field

#### Scenario: Esc inside the picker does not close the note

- **WHEN** the user presses Esc while the parent picker is
  open
- **THEN** the dropdown returns to the options view and the
  note form stays open
