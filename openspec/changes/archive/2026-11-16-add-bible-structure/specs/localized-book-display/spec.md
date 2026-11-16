# Spec Delta

## Purpose

Defines how the frontend resolves and renders localized book
names — a display-only layer keyed on the active translation's
language — while English book names remain the API, URL and
storage identity.

## ADDED Requirements

### Requirement: Display names follow translation language

Book names rendered in the UI SHALL come from the active
translation's structure (language-matched names), not from a
fixed English table. This applies to the book picker, passage
header, audio titles/playlists, verse-action labels and note
headings.

#### Scenario: Latvian translation shows Latvian names

- **WHEN** the active text fileset's language is Latvian
- **THEN** the book picker lists `1. Mozus grāmata` for `GEN`,
  not `Genesis`

#### Scenario: Language switch swaps names

- **WHEN** the user switches from an English to a Russian
  translation
- **THEN** rendered book names change to Russian without
  reloading the page

#### Scenario: Uncovered book keeps the translation's language

- **WHEN** a partial-canon translation's structure omits a
  book (e.g. `LAVLVR` lacks `EXO`) and a bundled catalog
  exists for the translation's language
- **THEN** the uncovered book renders the bundled catalog's
  same-language name (`2. Mozus grāmata`), not the English
  `Exodus` — the catalog doubles as the same-language
  fallback for books the structure does not cover

### Requirement: English names remain the contract identity

Book names used in API parameters (`passage=`), stored fields
(`book` on notes/reading positions) and external-link slugs
SHALL stay English; localization SHALL be display-only and
SHALL NOT change what is sent to or stored by the backend.

#### Scenario: Passage fetch stays English

- **WHEN** the UI displays `1. Mozus grāmata 1` while fetching
  chapter text
- **THEN** the request `passage=` parameter still resolves via
  the English name (`Genesis 1`)

### Requirement: Stale-while-revalidate structure display

On translation change the picker SHALL keep rendering the
previously loaded structure until the new one arrives, then
swap names and coverage atomically in one render. Bundled
`en`/`lv`/`ru` name catalogs SHALL cover the shipped locales
when no cache entry exists; per-bible structure SHALL be cached
in localStorage keyed on bible abbreviation — except degraded
(`degraded: true`) or empty-`books` anomaly bodies, which
SHALL NOT be persisted and are retried on the next request.

#### Scenario: Uncached non-shipped language

- **WHEN** the user selects a translation whose language has no
  bundled catalog and no cached structure
- **THEN** the picker shows the previous structure's names
  until the fetch resolves

#### Scenario: Atomic swap

- **WHEN** a fetched structure arrives
- **THEN** book labels and coverage state update together in a
  single render — never new coverage with old names

### Requirement: Uncovered books stay reachable but inert

Books the active translation lacks SHALL be shown greyed out
in the picker and remain clickable; clicking SHALL NOT navigate
or fall back to another translation — it SHALL show a
notification naming the translation and the uncovered book.
Text content SHALL never auto-fallback to another translation;
audio keeps its existing alternates walk and substitution
notice.

#### Scenario: Greyed book click

- **WHEN** the user clicks a greyed book on an NT-only
  translation
- **THEN** a notification states the translation does not
  include that book, and the reader does not navigate

#### Scenario: Deep link to uncovered book

- **WHEN** the user opens a URL for a book the active
  translation lacks
- **THEN** the existing `book_not_in_fileset` handling shows
  the coverage hint (unchanged behavior)

### Requirement: Name matching accepts localized input

Book-name matching in mention parsing and legacy URL segments
SHALL accept the displayed localized names additively —
localized names resolve to USFM codes alongside the existing
English names and codes, never replacing them.

#### Scenario: Localized mention parses

- **WHEN** a Latvian UI user types `Jāņa 3` in the mention
  parser under a Latvian translation
- **THEN** it resolves to `JHN` chapter 3; `John` and `JHN`
  inputs still resolve as before
