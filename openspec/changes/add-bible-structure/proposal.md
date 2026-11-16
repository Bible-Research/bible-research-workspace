# Proposal

## Why

The reading UI assumes every translation has the same 66-book
English structure. Book names are hardcoded English in three
places (`bibleStructure.json`, `BIBLE_BOOKS` in `bibleUtils.ts`,
`_BIBLE_BOOKS` in `bible_books.py`), and the picker offers all
66 books for every translation — including partial-canon ones
(e.g. `LAVLVR` has 28 books), so users can navigate into books
that 404 with `book_not_in_fileset`. To localize book names per
translation language and fix partial-canon UX, the app needs a
provider-aware structure contract.

## What Changes

- New backend endpoint `GET /api/v1/bible/structure/?fileset_id=`
  returning the selected translation's real coverage:
  `{books: [{book_id, name, name_short, testament, chapters}]}`
  where `chapters` is a chapter-number list and `name` is
  localized to the fileset's language.
- Provider-aware sourcing through the existing facade pattern:
  DBT-backed filesets read their own bible's
  `/bibles/{bible_id}/book?verify_content=true`; SWORD filesets
  read the local module's structure; `ENGKJV` answers from the
  bundled/registry English structure; anything unresolvable
  falls back to a curated canonical bible for the fileset's
  `language_iso` (`eng->ENGESV`, `lvs->LAVNLI`, `rus->RUSSYN`).
- Book names in the response use a canonical per-language source
  rather than per-bible orthography where the provider cannot
  supply names (SWORD, bundled, API.Bible), so display naming is
  consistent within a language.
- Endpoint is anonymous-identical catalog data:
  `VercelEdgeCacheMixin` headers + a process-wide LRU, mirroring
  `bible/translations/` caching posture.
- Frontend consumes structure per active translation, cached in
  localStorage keyed by bible abbreviation; PassagePicker greys
  (but keeps clickable) books the translation lacks — a click
  fires a notification instead of navigating. Book display names
  come from the structure's language-matched names.
- A display-name layer is split from the contract layer:
  `toBookName()` keeps returning English names for the `passage=`
  param, stored `book` fields and external links; a new
  locale-aware lookup serves all rendering call-sites.
- Stale-while-revalidate: on translation switch the picker keeps
  showing the previous structure until the new one arrives;
  bundled `en`/`lv`/`ru` name catalogs cover the shipped locales
  cold. Names and coverage swap atomically.
- Out-of-scope books (e.g. deuterocanon ids outside the 66-book
  table) are filtered from the response in v1.

## Capabilities

### New Capabilities

- `bible-structure`: the `/bible/structure/` endpoint contract —
  provider-aware coverage (books + chapters) and
  language-consistent book naming for a `fileset_id`, including
  sourcing precedence, fallbacks, filtering and caching posture.
- `localized-book-display`: how the frontend resolves and
  renders book names — language-keyed display names as a
  display-only layer (English names stay the API/storage
  identity), stale-while-revalidate loading, bundled-locale
  fallback, and uncovered-book presentation in the picker.

### Modified Capabilities

- `provider-routing`: adds a requirement that structure
  resolution reuses the same provider dispatch order and facade
  (`provider_for`) as passage/audio requests.

## Impact

- **Backend** (`bible_research`): new route + view under
  `bible/`, provider surface extension (structure/books per
  provider), canonical-bible registry, caching reuse. No
  migrations; no changes to `passage=` resolution or stored
  `book` fields — they stay English.
- **Frontend** (`reactive-bible`): PassagePicker, PassageView,
  Audio title/playlist, VerseActionToolbar, noteHeading,
  scriptureMention/scriptureRef (additive localized-name
  matching), store (per-bible structure cache), cacheManager
  (structure cache tier).
- **Specs**: new `bible-structure` + `localized-book-display`
  specs; `provider-routing` delta. `error-contract` unchanged.
