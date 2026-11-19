# Proposal

## Why

The app intends to serve only the 66-book Protestant canon, and
almost every surface already enforces it (`_BIBLE_BOOKS` /
`get_dbt_book_id` gate passage names, timestamps, SWORD book
maps, reading positions, and the frontend book list). The hole
is search: `GET /api/v1/bible/search/` passes provider
`book_id`s through verbatim, so DBT filesets containing
deuterocanonical books (the translations endpoint lists ~500
translations, including Catholic/Orthodox ones) return
apocryphal verses the UI renders as unreadable raw-id groups
("TOB", "WIS", ...). The `books` filter param also accepts and
forwards non-canonical ids. Secondary gaps: `get_passage_body`
accepts a raw non-canonical `book` id from direct callers, and
the generated-timestamps job's upstream-derived worklist is
unfiltered.

## What Changes

- Search responses (paginated and `group_by=book`) never carry
  non-canonical `book_id`s, for every provider.
- Where the provider supports a book filter (DBT `books`), the
  canon restriction is pushed upstream so provider-reported
  `total`/`total_pages` reflect canonical counts — no phantom
  pages and no extra calls.
- A `books` request param intersects with the canon; an empty
  intersection short-circuits to an empty result set without
  an upstream call (coverage-miss semantics, not an error).
- `get_passage_body` rejects non-canonical `book` ids as
  `book_not_in_fileset` (404) before provider dispatch —
  defense in depth; the public `passage=` path cannot produce
  one today.
- The `generate_audio_timestamps` worklist skips non-canonical
  books so a fileset containing apocrypha cannot burn STT
  spend or trip the consecutive-failure abort.
- Zero new upstream calls and no per-request lookups beyond
  O(1) set membership — no latency added.

## Capabilities

### New Capabilities

- `book-canon`: The system exposes only the 66-book Protestant
  canon across every provider surface — passage, search, and
  provider-derived worklists — regardless of what an upstream
  fileset contains.

### Modified Capabilities

None — no existing requirement text changes; `book-canon` owns
the entire contract (provider-routing's dispatch, caching, and
payload-shape requirements are unaffected).

## Impact

- Repo: `bible_research` only. `reactive-bible` needs no
  changes — its book lists and refs are already canonical, and
  unknown `book_id` groups degrade to raw codes it will no
  longer receive.
- Backend code: `bible/utils/bible_books.py`,
  `bible/services/providers.py`,
  `bible/services/search_grouping.py`,
  `bible/management/commands/generate_audio_timestamps.py`.
- API contract: `bible/search/` responses lose non-canonical
  rows on deuterocanonical filesets; `books` param semantics
  narrow to the canon; passage 404s for non-canonical `book`
  ids (previously unreachable via `passage=`).
- No dependencies, no migrations, no config changes.
