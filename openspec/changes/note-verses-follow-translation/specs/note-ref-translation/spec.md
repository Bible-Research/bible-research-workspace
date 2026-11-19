# Spec Delta

## ADDED Requirements

### Requirement: Note cards render verses in the resolved translation

A note card's linked-verse text SHALL render in the resolved
note-reference translation — the pinned override when set,
otherwise the selected translation — refetching the text
rather than trusting the stored creation-time payload.

#### Scenario: Card verses match the pinned translation

- **WHEN** a pin is set and a note card lists linked verses
- **THEN** their text comes from the pinned translation

#### Scenario: Card verses follow a translation switch

- **WHEN** the user switches the selected translation with no
  pin set
- **THEN** subsequently rendered note cards show verse text in
  the new translation

### Requirement: Stored verse text is the resilient fallback

While re-fetched text is loading, offline, or failed, the
stored `note.verses` text SHALL keep rendering — a card never
blanks or error-blocks its verse list because the
re-resolution fetch failed. Fetch errors use the existing
localized error surface only when there is no stored text at
all.

#### Scenario: Offline card keeps stored text

- **WHEN** the device is offline and a note card renders
- **THEN** the stored verse text displays unchanged

#### Scenario: Failed re-fetch keeps stored text

- **WHEN** the re-resolution request fails for a card's verses
- **THEN** the stored verse text remains displayed and no
  error row replaces the verses
