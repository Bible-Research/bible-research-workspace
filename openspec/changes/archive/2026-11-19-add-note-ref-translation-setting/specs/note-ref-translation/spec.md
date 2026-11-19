# Spec Delta

## Purpose

Controls which Bible translation note-reference text surfaces
— the `@…` scripture-ref popover in notes/comments and
note-linked verse text in note dialogs — render in: by default
the reader's selected translation, optionally a pinned
translation chosen in Settings.

## ADDED Requirements

### Requirement: Preview follows the selected translation

Unless an override is set, a scripture-reference preview SHALL
render passage text in the translation the reader currently
has selected, resolving testament-split filesets per book and
keeping the bundled offline translation working offline.

#### Scenario: Preview matches the reader's translation

- **WHEN** a user whose selected text translation is not ESV
  opens a `@…` scripture reference
- **THEN** the previewed verses come from that translation

#### Scenario: Preview updates when the translation changes

- **WHEN** the user switches their selected text translation
  with no override set
- **THEN** subsequently opened previews use the new
  translation

#### Scenario: Offline translation still works

- **WHEN** the selected translation is the bundled offline one
  and the device is offline
- **THEN** the preview still renders verse text

#### Scenario: Note dialogs follow the same translation

- **WHEN** a note create/edit dialog lists linked verses with
  no override set
- **THEN** the shown verse text comes from the selected
  translation

### Requirement: Pinned preview translation override

The settings page SHALL offer a note-reference-translation
control with a "selected translation" choice (the default) and
a way to pin one specific translation via the translation
selector. A pinned override SHALL persist and SHALL NOT change
the reader's selected translation.

#### Scenario: Pin a translation for previews

- **WHEN** the user picks a specific translation on the
  settings page
- **THEN** previews render in that translation while the
  reader keeps its own selected translation

#### Scenario: Choosing the pin does not retune the reader

- **WHEN** the user finishes picking a pinned preview
  translation
- **THEN** the main Bible reader still uses the text and audio
  selections it had before

#### Scenario: Picker opens on a sensible seed

- **WHEN** the pin picker opens with a pin already set
- **THEN** it is seeded with the pinned translation; with no
  pin it is seeded with the selected translation

#### Scenario: Pinned translation covers note dialogs too

- **WHEN** a pin is set and a note dialog lists linked verses
- **THEN** the shown verse text comes from the pinned
  translation

#### Scenario: Reset follows the selection again

- **WHEN** the user switches the control back to "selected
  translation"
- **THEN** previews again follow the reader's translation and
  the pinned choice is forgotten

### Requirement: Last-resort preview fallback

When no text fileset id is resolved at all (neither an override
nor a selection is set), the preview SHALL fall back to the
previous fixed behavior (ESV). Unknown or drifted ids pass
through to the API exactly as the reader handles them today.

#### Scenario: No selection resolved falls back to ESV

- **WHEN** neither an override nor a selected text fileset
  exists
- **THEN** the preview requests the passage in the ESV fileset

#### Scenario: Drifted pinned id errors like the reader

- **WHEN** the pinned id no longer resolves to a known text
  fileset
- **THEN** it is sent to the API verbatim and any error
  surfaces through the normal localized error path

### Requirement: Preview errors stay surfaced

Switching preview translations SHALL preserve the existing
error contract: provider failures and
book-not-in-fileset responses render as localized error text
inside the preview, never as silently empty verses.

#### Scenario: Book missing from pinned translation

- **WHEN** the pinned translation lacks a referenced book
- **THEN** the preview shows the localized
  book-not-in-fileset error
