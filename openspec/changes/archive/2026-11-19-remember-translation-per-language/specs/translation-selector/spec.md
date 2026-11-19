# Spec Delta

## Purpose

Defines how the translation selector modal remembers the last used
Bible version per language client-side and restores it when the
user switches back to a language they have used before.

## ADDED Requirements

### Requirement: Per-language selection is recorded on save

When the user saves a translation choice, the client SHALL record
that selection (version identity plus its staged text and audio
fileset ids) under the chosen translation's language, persisted so
it survives reloads. Browsing or picking without saving SHALL NOT
update the record.

#### Scenario: Saving updates the record for that language

- **WHEN** the user selects a Latvian version in the selector and
  presses Save
- **THEN** the persisted per-language record for Latvian holds
  that version and its text/audio fileset ids

#### Scenario: Cancelling leaves the record untouched

- **WHEN** the user picks a different version for a language but
  closes the selector without saving
- **THEN** the previously recorded selection for that language is
  unchanged

### Requirement: Last used translation is restored on language switch

When the viewed language changes inside the selector and a
recorded selection exists for that language, the picker SHALL
stage it — showing that version as selected with its recorded
text and audio choices — once the language's translation list is
loaded. The staged restore applies only on Save.

#### Scenario: Switching back restores the remembered version

- **WHEN** the user has previously saved English → ESV and
  Latvian → GLÜK, is viewing the Latvian tab, and switches to the
  English tab
- **THEN** the version picker shows ESV selected with its recorded
  text/audio choices, without changing the active translation
  until Save

#### Scenario: Restore works for Other-tab languages

- **WHEN** the user re-picks on the Other tab a language with a
  recorded selection
- **THEN** the remembered version is staged once that language's
  translations load

### Requirement: Missing or stale memory falls back gracefully

When no selection is recorded for the viewed language, or the
recorded version no longer appears in the fetched translation
list, the selector SHALL behave as it does today — an empty
picker, or the mirror of the active selection when it belongs to
that language.

#### Scenario: Language never used before

- **WHEN** the user switches to a language with no recorded
  selection
- **THEN** no version is pre-selected beyond the existing
  active-selection mirror

#### Scenario: Remembered version is gone

- **WHEN** a recorded version for a language is absent from the
  freshly fetched translation list
- **THEN** the selector falls back to the same no-memory behavior
  instead of staging a stale selection
