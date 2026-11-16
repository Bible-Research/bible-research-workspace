# Spec Delta

## Purpose

Defines the `/api/v1/bible/structure/` endpoint: the
provider-aware contract by which clients learn which books and
chapters a translation actually contains, with book names
localized to the translation's language.

## ADDED Requirements

### Requirement: Structure endpoint shape

`GET /api/v1/bible/structure/?fileset_id=<id>` SHALL return a
JSON body `{books: [{book_id, name, name_short, testament,
chapters}]}` where `book_id` is a USFM code, `chapters` is the
ordered list of chapter numbers the fileset covers, and `name`/
`name_short` are localized to the fileset's language.

#### Scenario: DBT-backed fileset returns own coverage

- **WHEN** a request carries a DBT-backed `fileset_id` whose
  bible has a partial canon (e.g. `LAVLVR`, Psalms+NT only)
- **THEN** the response lists only the books that bible covers,
  each with its real chapter list and names in that bible's
  language

#### Scenario: Full-canon fileset returns all books

- **WHEN** a request carries `RUSSYN` (or its text fileset id)
- **THEN** the response lists all 66 Protestant books with
  Russian names (e.g. `GEN` -> `Бытие`)

#### Scenario: Missing fileset_id is a client error

- **WHEN** a request omits `fileset_id`
- **THEN** the response is HTTP 400 with `error_code`
  `missing_parameter`, per the error contract

### Requirement: Provider-aware sourcing with language fallback

Structure SHALL resolve through the same provider dispatch as
passage requests: DBT filesets read their own bible's book
list; SWORD filesets read the local module's structure;
`ENGKJV` reads the bundled/registry structure; any fileset
whose provider cannot answer falls back to a curated canonical
bible for its `language_iso` (`eng`->`ENGESV`,
`lvs`->`LAVNLI`, `rus`->`RUSSYN`), then to English.

#### Scenario: SWORD fileset reads module structure

- **WHEN** a request carries `LVSGLU8`
- **THEN** coverage comes from the local SWORD module, and
  names are Latvian (module names if usable, else the `lvs`
  canonical source)

#### Scenario: Canonical fallback for a language

- **WHEN** a fileset's provider cannot supply structure and its
  `language_iso` is `rus`
- **THEN** coverage and names come from `RUSSYN`

### Requirement: Canonical naming within a language

Book `name`/`name_short` values SHALL come from the fileset's
own bible when it is DBT-backed, and otherwise from the
canonical per-language bible — never mixed orthographies (e.g.
a partial-canon DBT bible shows its own names; a SWORD/bundled
fileset shows the canonical source's names for its language).

#### Scenario: Bundled KJV shows English names

- **WHEN** a request carries `ENGKJV`
- **THEN** `GEN` is named `Genesis` (not the all-caps
  `GENESIS` DBT reports for `ENGKJV`)

### Requirement: Book-id filtering

The response SHALL include only book ids in the recognized
USFM canon set; non-book entries a provider reports (e.g.
video chapter ids such as `C01`) SHALL be filtered out.
Deuterocanonical ids outside the 66-book set MAY be filtered
in v1.

#### Scenario: Non-USFM entries dropped

- **WHEN** a bible's book list contains non-USFM entries such
  as `C01`
- **THEN** they do not appear in the response

### Requirement: Structure caching posture

The endpoint SHALL be anonymous-identical and edge-cacheable
like other catalog endpoints (`Vercel-CDN-Cache-Control`
marking, `VercelEdgeCacheMixin`), and SHALL be backed by a
process-wide cache so repeat requests do not re-hit providers.
No request SHALL forward a `Vercel-CDN-Cache-Control` body for
non-2xx or degraded responses.

#### Scenario: Repeat request served from cache

- **WHEN** two requests carry the same `fileset_id` within the
  cache TTL
- **THEN** the second is answered without a new provider call

### Requirement: Provider failures follow the error contract

Upstream failures SHALL surface through the existing
`error`/`error_code` contract (`rate_limited` -> 429,
`not_found` -> 404, else `provider_error` -> 502); exception
text SHALL never be embedded in `error`.

#### Scenario: Upstream outage

- **WHEN** the provider for a `fileset_id` fails
- **THEN** the response carries `error` + `error_code`
  `provider_error` with HTTP 502
