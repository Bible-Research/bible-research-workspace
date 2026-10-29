# Error Contract Specification

## Purpose

Defines the stable error payload shape and code taxonomy the
backend returns for provider and client-input failures, the
HTTP status each code maps to, and how the frontend translates
coded errors into typed error classes it branches on.

## Requirements

### Requirement: Error payload shape

API failures SHALL return a JSON body carrying `error` plus an
`error_code` field (the frontend also reads `message`). Field
names MUST stay stable because the React app branches on
`error_code`. Provider errors are built by
`bible/utils/provider_errors.py`; client-input errors by
`bible/utils/error_responses.py`.

#### Scenario: Provider failure returns coded body

- **WHEN** an upstream provider fails on a passage request
- **THEN** the response body contains `error` and `error_code`
  fields naming the failure category

### Requirement: error_code to HTTP status mapping

The backend SHALL map codes to statuses as follows:
`rate_limited` → 429; `not_found` (missing passage, upstream
404, `PassageNotFoundError`) → 404; `book_not_in_fileset`
(every provider's fileset-lacks-book coverage miss) → 404;
`provider_error` → 502 for all other provider failures;
`internal_error` → 500 as the catch-all for unexpected server
exceptions; `missing_parameter`/`invalid_parameter`
(`bad_request`) → 400.
Routing ids SHALL be charset-validated at the view boundary.

#### Scenario: Rate-limited upstream yields 429

- **WHEN** an upstream provider rejects a request for
  rate limiting
- **THEN** the API responds 429 with `error_code` of
  `rate_limited`

#### Scenario: Book missing from fileset yields coded 404

- **WHEN** a fileset does not contain the requested book or
  chapter on the passage endpoint
- **THEN** the API responds 404 with `error_code` of
  `book_not_in_fileset`

#### Scenario: Unclassified provider failure yields 502

- **WHEN** a provider fails for a reason outside the named
  codes
- **THEN** the API responds 502 with `error_code` of
  `provider_error`

#### Scenario: Malformed routing id yields coded 400

- **WHEN** `fileset_id`, `bible_id`, `language_iso`, or `book`
  fails charset validation or a required parameter is missing
- **THEN** the API responds 400 with `missing_parameter` or
  `invalid_parameter`

### Requirement: No raw exception text in error field

The `error` field MUST NOT embed `str(exc)` or other raw
exception text, because exception messages can contain request
URLs with credentials. Error text SHALL be a safe, stable
category description.

#### Scenario: Upstream exception with URL is sanitized

- **WHEN** a provider raises an exception whose message embeds
  a credentialed request URL
- **THEN** the returned `error` field contains no part of that
  exception text

### Requirement: Graceful note-enrichment degradation

Note serialization SHALL degrade the same way as passage
fetching: when provider lookups fail during verse enrichment,
verse references are kept and `text` is returned empty rather
than failing the whole notes response.

#### Scenario: Provider outage during note listing

- **WHEN** notes are listed while the text provider is down
- **THEN** the notes response succeeds with verse refs intact
  and empty `text` fields

### Requirement: Frontend typed error mapping

The frontend SHALL map `error_code` to typed classes in
`src/api/errors.ts`: `RateLimitError`, `ClientInputError`,
`ProviderError`, and `BookNotInFilesetError`. Coded HTTP 400
bodies MUST become `ClientInputError`, deliberately not
`ProviderError`. Provider failures MUST NOT be silently
swallowed into empty verse lists.

#### Scenario: Coded 400 surfaces as client-input error

- **WHEN** the backend returns a 400 with a coded error body
- **THEN** the frontend raises `ClientInputError` rather than
  `ProviderError`

#### Scenario: Provider failure remains visible

- **WHEN** a passage fetch fails with any coded provider error
- **THEN** the frontend surfaces the typed error instead of
  rendering an empty verse list
