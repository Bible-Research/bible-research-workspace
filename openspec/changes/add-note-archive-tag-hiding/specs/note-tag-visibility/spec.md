# Spec Delta

## Purpose

Defines the lifecycle visibility flags for user-owned annotations:
`archived` on notes and tags (with a tag→notes cascade and
effective-archive semantics) and `hidden` on tags (cosmetic
exclusion from the note-creation dropdown only).

## ADDED Requirements

### Requirement: Note archive flag

A note SHALL expose a boolean `archived` field in its API
representation, defaulting to `false`. The note's owner SHALL be
able to set or clear it via `PATCH /api/v1/notes/<id>/`. Archiving
MUST NOT delete the note, its comments, its verse links, or its
`tag_position` ordering value.

#### Scenario: Archive a note

- **WHEN** the owner PATCHes `/api/v1/notes/<id>/` with
  `{"archived": true}`
- **THEN** the response shows `"archived": true` and the note is
  excluded from default note enumerations

#### Scenario: Unarchive a note

- **WHEN** the owner PATCHes `/api/v1/notes/<id>/` with
  `{"archived": false}` on an archived note whose tag is not
  archived
- **THEN** the note returns to normal enumeration surfaces

### Requirement: Retagging unarchives a note

Changing a note's `tag` SHALL clear `archived` in the same write,
so moving an archived note to a live tag makes it visible again in
one step. Writes that leave `tag` unchanged MUST NOT clear
`archived`.

#### Scenario: Retag archived note to live tag

- **WHEN** the owner PATCHes an archived note with a different
  `tag` value (including `null`)
- **THEN** the response shows the new tag and `"archived": false`

#### Scenario: Other updates keep archive state

- **WHEN** the owner PATCHes an archived note changing only
  `note_text`
- **THEN** `archived` remains `true`

### Requirement: Tag archive flag with note cascade

A tag SHALL expose a boolean `archived` field, defaulting to
`false`, writable via `PATCH /api/v1/tags/<id>/`. Setting it
SHALL atomically set `archived` on every note in the tag to the
same value; clearing it SHALL clear `archived` on all of them,
including notes that were archived individually.

#### Scenario: Archive tag cascades to notes

- **WHEN** the owner PATCHes `/api/v1/tags/<id>/` with
  `{"archived": true}` on a tag containing notes
- **THEN** the tag and all its notes report `archived: true` and
  the notes leave default enumerations

#### Scenario: Unarchive tag restores all notes

- **WHEN** the owner PATCHes `/api/v1/tags/<id>/` with
  `{"archived": false}` on an archived tag whose notes include
  some archived individually before the tag was archived
- **THEN** every note reports `archived: false`, and all reappear
  in their tag's list with unchanged `tag_position` ordering

### Requirement: Effective archived state

A note SHALL be treated as archived — excluded from enumerations
and hidden from non-owners — when `note.archived` is true OR its
assigned tag's `archived` is true. Untagged notes depend only on
their own `archived` flag.

#### Scenario: Note under archived tag is hidden

- **WHEN** a note with `archived=false` belongs to an archived
  tag
- **THEN** it is excluded from default enumerations exactly as an
  archived note is

### Requirement: Archived notes excluded from enumeration

`GET /api/v1/notes/`, `POST /api/v1/notes/linked/`, and
`GET /api/v1/notes/order-index/` SHALL exclude effectively
archived notes by default for every caller, including the owner.

#### Scenario: Note list omits archived

- **WHEN** the owner lists `GET /api/v1/notes/?tag_id=<id>` and
  one note under that tag is effectively archived
- **THEN** that note is absent and `count` excludes it

#### Scenario: Linked notes omit archived

- **WHEN** the owner POSTs `/api/v1/notes/linked/` for verses an
  effectively archived note is linked to
- **THEN** that note is not returned

#### Scenario: Order index omits archived

- **WHEN** the owner requests `GET /api/v1/notes/order-index/
  ?tag_id=<id>` for a tag containing an effectively archived note
- **THEN** that note has no entry in the order index

### Requirement: Archived notes opt-in via include_archived

The notes list endpoint SHALL accept `include_archived=true` so an
authenticated owner can list effectively archived notes (alone or
mixed). Non-owner callers MUST NOT gain access to other users'
archived notes through this parameter.

#### Scenario: Owner lists with archived included

- **WHEN** the owner requests `GET /api/v1/notes/?tag_id=<id>
  &include_archived=true`
- **THEN** effectively archived and live notes are both returned

#### Scenario: Parameter never leaks other users' notes

- **WHEN** any caller requests `include_archived=true`
- **THEN** the response still contains only notes that caller
  could otherwise access

### Requirement: Archived notes visible only to owner

An effectively archived note SHALL be retrievable and mutable only
by its owner. Anonymous callers and other authenticated users
SHALL NOT see it even when it is public.

#### Scenario: Anonymous cannot retrieve archived public note

- **WHEN** an anonymous caller GETs `/api/v1/notes/<id>/` for a
  public note that is effectively archived
- **THEN** the API responds as though the note does not exist
  (404)

#### Scenario: Owner can still retrieve archived note

- **WHEN** the owner GETs `/api/v1/notes/<id>/` for their own
  effectively archived note
- **THEN** the API returns the note so it can be inspected and
  managed

### Requirement: Tag note counts exclude archived notes

The `note_count` reported for a tag SHALL count only notes whose
own `archived` flag is false.

#### Scenario: Count drops after archiving

- **WHEN** the owner archives a note under a tag that had
  `note_count` N
- **THEN** subsequent `GET /api/v1/tags/` reports `note_count`
  N-1 for that tag

### Requirement: Tag hidden flag

A tag SHALL expose a boolean `hidden` field in its API
representation, defaulting to `false`, writable via
`PATCH /api/v1/tags/<id>/`. Hiding MUST NOT affect the tag's
notes, hierarchy, or availability to API consumers.

#### Scenario: Hide a tag

- **WHEN** the owner PATCHes `/api/v1/tags/<id>/` with
  `{"hidden": true}`
- **THEN** the response shows `"hidden": true` and the tag remains
  in `GET /api/v1/tags/` results

### Requirement: Unavailable tags excluded from note-creation dropdown

The tag selector shown when creating or editing a note SHALL omit
hidden and archived tags, except that a note already assigned to
such a tag SHALL still show it as the selected value.

#### Scenario: Hidden tag absent from dropdown

- **WHEN** the user opens the note-creation tag dropdown and one
  tag is hidden
- **THEN** the hidden tag is not among the selectable options

#### Scenario: Archived tag absent from dropdown

- **WHEN** the user opens the note-creation tag dropdown and one
  tag is archived
- **THEN** the archived tag is not among the selectable options

#### Scenario: Current unavailable tag stays selectable while editing

- **WHEN** the user edits a note whose tag is hidden or archived
- **THEN** the dropdown shows that tag as the note's current tag,
  and choosing another tag removes it from the options

### Requirement: Hidden tags remain manageable on /tags

The `/tags` page SHALL list hidden tags alongside visible ones,
visually distinguish them, and offer a control to hide or unhide
each tag.

#### Scenario: Toggle hides tag

- **WHEN** the user sets the hidden control on a tag in `/tags`
- **THEN** the tag is marked hidden, stays listed, and disappears
  from the note-creation dropdown

#### Scenario: Toggle unhides tag

- **WHEN** the user clears the hidden control on a hidden tag
- **THEN** the tag is no longer marked hidden and reappears in the
  note-creation dropdown

### Requirement: Archived tags remain listed and restorable

The `/tags` page SHALL continue to list archived tags, visually
distinguish them, and offer a control to archive or unarchive each
tag; unarchiving restores all of the tag's notes per the cascade
requirement.

#### Scenario: Archived tag shown marked

- **WHEN** the user views `/tags` while a tag is archived
- **THEN** the tag is listed with a visible archived indicator

#### Scenario: Unarchive from /tags restores notes

- **WHEN** the user clears the archived control on an archived tag
- **THEN** the tag loses its indicator and all its notes reappear
  in note lists in their previous order

### Requirement: Archived notes browsable via show-archived toggle

The `/tags` page SHALL offer a toggle to reveal effectively
archived notes. While active, tag note lists SHALL include them,
visually marked, with an Unarchive action — disabled with an
explanatory hint when the note's tag is archived. Note rows SHALL
offer Archive on unarchived notes.

#### Scenario: Toggle reveals archived notes

- **WHEN** the user enables the show-archived toggle on `/tags`
  and opens a tag's note list containing an archived note
- **THEN** the archived note is listed and visibly marked as
  archived

#### Scenario: Unarchive restores note

- **WHEN** the user invokes Unarchive on an archived note whose
  tag is not archived
- **THEN** the note reappears in default note lists

#### Scenario: Unarchive disabled under archived tag

- **WHEN** a listed archived note's tag is archived
- **THEN** its Unarchive action is disabled and explains that the
  tag must be unarchived (or the note retagged) instead
