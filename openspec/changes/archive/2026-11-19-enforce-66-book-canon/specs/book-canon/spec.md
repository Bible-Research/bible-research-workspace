# book-canon Specification (delta)

## Purpose

Defines the boundary that limits every provider-facing surface —
passage/audio fetches, verse search, and provider-derived
worklists — to the 66-book Protestant canon, regardless of
what an upstream fileset actually contains.

## ADDED Requirements

### Requirement: Only the 66-book canon is exposed

All user-facing verse content (chapter/passage text, audio,
timestamps, search results) SHALL be limited to the 66-book
Protestant canon. A book outside that set SHALL be treated as
not covered by the fileset, even when the upstream provider
carries it.

#### Scenario: Deuterocanonical fileset serves only canon

- **WHEN** a fileset contains deuterocanonical books upstream
  (e.g. a Catholic DBT bible)
- **THEN** every content surface behaves as if those books do
  not exist in the fileset

#### Scenario: Canonical books unaffected

- **WHEN** a deuterocanonical fileset is queried for a
  canonical book
- **THEN** it behaves exactly as before this change

### Requirement: Search results are canon-filtered

Every provider's search — paginated and `group_by=book` —
SHALL return only canonical `book_id`s. Where the provider
supports an upstream book filter, the canon restriction SHALL
be pushed upstream so reported totals count canonical matches
only; normalized rows SHALL additionally be filtered after
fetch.

#### Scenario: Grouped search on a deuterocanonical fileset

- **WHEN** a `group_by=book` search runs against a fileset
  containing deuterocanonical books
- **THEN** every returned group's `book_id` is canonical

#### Scenario: Paginated totals count canon only

- **WHEN** a paginated search runs against a provider with an
  upstream book filter
- **THEN** `meta.pagination.total` and `total_pages` reflect
  canonical matches only, with no phantom pages

#### Scenario: Provider without an upstream book filter

- **WHEN** a provider lacks an upstream book filter
- **THEN** non-canonical rows are dropped during response
  normalization

### Requirement: books parameter intersects with the canon

The search `books` request parameter SHALL be intersected
with the canonical set before reaching a provider; an empty
intersection SHALL return an empty result set without an
upstream call or an error.

#### Scenario: Only non-canonical ids requested

- **WHEN** a search carries `books=TOB`
- **THEN** the response is an empty result set and no upstream
  call is made

#### Scenario: Mixed ids requested

- **WHEN** a search carries `books=TOB,JHN`
- **THEN** it behaves exactly like `books=JHN`

### Requirement: Non-canonical passages are coverage misses

The passage fetch path (`get_passage_body`) SHALL treat a
non-canonical `book` id as `book_not_in_fileset` (HTTP 404)
— the coverage-miss vocabulary — without dispatching to a
provider.

#### Scenario: Direct call with a non-canonical book id

- **WHEN** the passage fetch path receives `book='TOB'`
- **THEN** it returns the `book_not_in_fileset` error body
  without dispatching to a provider

### Requirement: Provider-derived worklists are canon-scoped

Worklists derived from upstream fileset coverage (e.g. the
generated-timestamps chapter list) SHALL skip non-canonical
books.

#### Scenario: Fileset coverage includes apocrypha

- **WHEN** the fileset's upstream file list contains rows for
  a non-canonical book
- **THEN** the worklist excludes them
