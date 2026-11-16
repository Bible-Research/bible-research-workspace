# bible-search Specification

## Purpose
Defines the `/api/v1/bible/search/` grouped-by-book contract:
per-book result shape, resumable scanning for SWORD filesets
within a per-request time budget, feature-flag gating of SWORD
search, and how the frontend consumes the resumable contract.

## Requirements

### Requirement: Grouped-by-book search response

When `group_by=book` is passed, the endpoint SHALL return
`data.groups` — a list of `{book_id, count, verses}` objects in
canonical book order with verses sorted by chapter then verse —
plus `data.meta` carrying at least `total` and `truncated`.
Pagination parameters (`page`, `limit`) are not applied in this
mode.

#### Scenario: Grouped results arrive in canonical order

- **WHEN** `GET /api/v1/bible/search/?query=X&fileset_id=F&
  group_by=book` succeeds
- **THEN** `data.groups` lists one entry per book containing
  matches, in canonical book order, and `data.meta.total`
  reports the number of returned matches

#### Scenario: Result cap marks truncation

- **WHEN** a grouped search matches more verses than the
  backend's safety cap
- **THEN** the response sets `data.meta.truncated` to `true`
  and `total` is a lower bound rather than the true match
  count

### Requirement: SWORD resumable grouped search

For SWORD filesets, a grouped search SHALL scan chapters only
within a per-request wall-clock budget (nominally 6 seconds).
When the budget expires mid-scan the response SHALL include
`meta.next_cursor`; when the scan completes the field SHALL be
absent. Supplying a valid `cursor` query parameter SHALL resume
the scan where the previous window stopped, returning only not-
yet-reported matches.

#### Scenario: First window returns a continuation

- **WHEN** a SWORD grouped search cannot finish scanning the
  module within the time budget
- **THEN** the response is 200 carrying the matches found so
  far in `data.groups` and a resume token in
  `data.meta.next_cursor`

#### Scenario: Resume continues from the cursor

- **WHEN** the request supplies a `cursor` returned by a
  previous response for the same query and fileset
- **THEN** the scan resumes at that position and returned
  groups contain no previously reported matches

#### Scenario: Completed scan omits the cursor

- **WHEN** a SWORD grouped search finishes scanning the module
  within the budget
- **THEN** `data.meta.next_cursor` is absent from the response

#### Scenario: Malformed cursor is a client error

- **WHEN** `cursor` is supplied but cannot be decoded into a
  scan position
- **THEN** the API responds 400 with `error_code` of
  `invalid_parameter`

### Requirement: SWORD search feature flag

SWORD-fileset searches SHALL be gated by a database-backed
feature flag named `sword_search`, toggleable at runtime via
Django admin without a redeploy. A missing flag row SHALL mean
enabled. When disabled, the endpoint SHALL return 403 with
`error_code` of `search_disabled` before provider dispatch.
Non-SWORD filesets SHALL NOT be affected by the flag.

#### Scenario: Disabled flag yields coded 403

- **WHEN** the `sword_search` flag is disabled and a search
  request carries a SWORD `fileset_id`
- **THEN** the API responds 403 with `error_code` of
  `search_disabled` and no provider scan runs

#### Scenario: Missing flag row permits search

- **WHEN** no `sword_search` flag row exists and a search
  request carries a SWORD `fileset_id`
- **THEN** search proceeds normally

#### Scenario: Flag does not gate other providers

- **WHEN** the `sword_search` flag is disabled and a search
  request carries an ESV, API.Bible, or DBT `fileset_id`
- **THEN** search proceeds normally for that provider

### Requirement: Iterative client consumption

The frontend search client SHALL issue follow-up requests with
the returned `next_cursor` until a response arrives without
one, merging per-book groups across windows (verses of the
same `book_id` append to that book's group). It SHALL surface
scan progress and remain cancellable between requests.

#### Scenario: Multi-window search merges groups

- **WHEN** a SWORD grouped search spans multiple windows
- **THEN** the client presents one merged, canonically ordered
  result set identical to a single-response result

#### Scenario: Disabled search shows a supportive message

- **WHEN** the API responds `search_disabled`
- **THEN** the search UI displays a translated "search is not
  supported for this translation" message instead of a generic
  error
