# Spec Delta

## MODIFIED Requirements

### Requirement: error_code to HTTP status mapping

The backend SHALL map codes to statuses as follows:
`rate_limited` → 429; `book_not_in_fileset` (fileset
coverage miss on the passage endpoint — includes
`PassageNotFoundError` and upstream 404s there) → 404;
`not_found` (coverage miss on other endpoints) → 404;
`search_disabled` (SWORD-fileset search rejected because
the `sword_search` feature flag is off) → 403;
`provider_error` → 502 for other provider failures;
`internal_error` → 500 catch-all;
`missing_parameter`/`invalid_parameter` (`bad_request`) →
400. Routing ids SHALL be charset-validated at the view
boundary.

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

#### Scenario: Passage-endpoint miss normalizes the code

- **WHEN** a `PassageNotFoundError` or an upstream 404
  escapes a provider on the passage endpoint
- **THEN** the API responds 404 with `error_code` of
  `book_not_in_fileset`, not `not_found`

#### Scenario: Non-passage provider miss yields not_found

- **WHEN** a provider lookup misses on a non-passage
  endpoint such as copyright, timestamps, search, or note
  verse enrichment
- **THEN** the API responds 404 with `error_code` of
  `not_found`

#### Scenario: Disabled SWORD search yields coded 403

- **WHEN** the `sword_search` feature flag is disabled and
  a search request carries a SWORD `fileset_id`
- **THEN** the API responds 403 with `error_code` of
  `search_disabled`

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
