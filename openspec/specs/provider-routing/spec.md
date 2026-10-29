# Provider Routing Specification

## Purpose

Defines how the backend dispatches every passage, audio, search,
and copyright request to an upstream Bible provider based on the
`fileset_id` routing key shared by both repos, including the
chapter-text caching layer and per-provider reporting duties.

## Requirements

### Requirement: fileset_id dispatch order

Every passage/audio request SHALL carry a `fileset_id` that
selects the provider through the facade
(`bible/services/providers.py` `provider_for`). Dispatch order
MUST be: (1) ESV registry filesets → ESV API; (2) API.Bible
registry filesets → api.scripture.api.bible; (3) SWORD registry
filesets → local SWORD modules; (4) all other ids → DBT API v4.

#### Scenario: ESV fileset routes to ESV API

- **WHEN** a request carries an ESV fileset id such as
  `ENGESV_API`
- **THEN** the facade returns the ESV provider and text is
  fetched from api.esv.org

#### Scenario: API.Bible fileset routes to API.Bible

- **WHEN** a request carries an API.Bible fileset id such as
  `ENGNIV_API` or `ENGNIVC1DA`
- **THEN** the facade returns the API.Bible provider and text is
  fetched from api.scripture.api.bible

#### Scenario: SWORD fileset routes to local modules

- **WHEN** a request carries a SWORD fileset id such as
  `LVSGLU8` or `LVSGLU8C1DA`
- **THEN** the facade returns the SWORD provider serving local
  SWORD modules

#### Scenario: Unknown fileset falls through to DBT

- **WHEN** a request carries a fileset id present in no registry
- **THEN** the facade returns the DBT provider backed by DBT
  API v4

### Requirement: Uniform provider surface

`provider_for` SHALL return an object exposing the uniform
surface `get_chapter`, `get_passage`, `get_audio`,
`get_timestamps`, `search`, `search_grouped`, `get_copyright`,
and `translation_listing`, so callers never branch on provider
type. `get_passage_body()` in the same module is the
`/api/v1/bible/` fetch-and-shape path the view calls.

#### Scenario: View fetches a passage without provider branches

- **WHEN** `/api/v1/bible/` is called with any valid
  `fileset_id`
- **THEN** the response is produced through the uniform provider
  surface with no provider-specific code in the view

### Requirement: Chapter cache wrapper

The facade SHALL wrap ESV, SWORD, and DBT providers in
`_CachingProvider`: whole-chapter text served from a
process-wide LRU (24h TTL, 500-entry bound), 5-minute negative
caching of `PassageNotFoundError`, and per-translation verse
budgets for licensor caps. Verse-range hints MUST NOT be
forwarded upstream on a cache miss. Returned payloads are
shared and MUST NOT be mutated.

#### Scenario: Repeated chapter request hits the cache

- **WHEN** two requests ask for the same fileset/book/chapter
  within the TTL
- **THEN** only one upstream call is made and both responses
  come from the shared payload

#### Scenario: Missing passage is negatively cached

- **WHEN** an upstream answers `PassageNotFoundError` for a
  chapter
- **THEN** repeated requests within 5 minutes do not re-hit the
  upstream

#### Scenario: Verse budget enforced per translation

- **WHEN** requests for `ENGESV_API` accumulate verses
- **THEN** they count against the shared `esv` budget, while
  public-domain `ENGKJV` and `LVSGLU8` requests are exempt

#### Scenario: Narrowed fetch does not poison the chapter entry

- **WHEN** a request carries a verse-range hint and the chapter
  is not cached
- **THEN** the hint is not forwarded upstream, so the full
  chapter fills the cache entry

### Requirement: API.Bible FUMS reporting path

`ApiBibleProvider` SHALL be returned unwrapped by the caching
layer. API.Bible text responses carry a `meta` block with FUMS
fields that the frontend MUST report per display, as required
by the API.Bible license; caching would defeat per-display
reporting.

#### Scenario: API.Bible response preserves meta block

- **WHEN** the frontend renders text from an API.Bible fileset
- **THEN** the response includes the `meta` block and the
  frontend reports the FUMS fields for that display

### Requirement: Audio fileset id convention

Fileset ids ending in `DA` SHALL denote audio filesets. Known
ids: `ENGKJV` (KJV), `ENGESV_API` (ESV), `LVSGLU8` (Latvian
Glück text), `LVSGLU8C1DA` (generated audio, verse-synthesis
mode), `LVSGLU8C2DA` (same audio, chapter-synthesis via SSML
marks), `ENGNIV_API` (API.Bible NIV text), `ENGNIVC1DA` (NIV
audio). Frontend-chosen ESV audio ids are
`ENGESVO1DA`/`ENGESVN1DA` by testament.

#### Scenario: Latvian audio fileset selects synthesis mode

- **WHEN** audio is requested for `LVSGLU8C1DA` versus
  `LVSGLU8C2DA`
- **THEN** verse-synthesis audio is served for `C1DA` and
  chapter-synthesis audio for `C2DA`

### Requirement: Copyright and enrichment routing

Copyright lookups SHALL use the same facade: ESV/SWORD ids
answer from registry `license` metadata, API.Bible ids fetch
the live statement (audio fileset ids read the audio bible's
entry), other ids proxy DBT. Note enrichment SHALL batch
upstream calls by (fileset, book, chapter) — one call per
chapter per page, not per note.

#### Scenario: SWORD copyright answered locally

- **WHEN** copyright is requested for `LVSGLU8`
- **THEN** the registry license metadata is returned without an
  upstream call

#### Scenario: Note list enriches verses in batches

- **WHEN** a page of notes spans multiple verses in one chapter
- **THEN** at most one upstream chapter fetch is made for that
  (fileset, book, chapter) tuple
