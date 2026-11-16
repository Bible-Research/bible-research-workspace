# Tasks

## 1. Backend: structure endpoint

- [x] 1.1 backend: extend the provider surface with a
  `get_structure()` uniform method (DBT via
  `v4_bible_books(verify_content=true)`, SWORD via pysword
  `get_structure().get_books()`, bundled/registry for
  `ENGKJV`) and verify each provider returns
  `{book_id, name, name_short, testament, chapters}` in unit
  tests with mocked clients
- [x] 1.2 backend: add a canonical-bible-per-language registry
  (`{'eng': 'ENGESV', 'lvs': 'LAVNLI', 'rus': 'RUSSYN'}`,
  default `ENGESV`) wired into the structure fallback, and
  verify a fileset whose provider cannot answer resolves the
  canonical source for its `language_iso`
- [x] 1.3 backend: add `GET /api/v1/bible/structure/`
  (`fileset_id` required, charset-validated via
  `request_params`) with `VercelEdgeCacheMixin` +
  module-level cache; verify 400 `missing_parameter` without
  `fileset_id`, `{books:[...]}` on success, and `error`/
  `error_code` mapping on provider failure in tests
- [x] 1.4 backend: filter response book ids to the recognized
  USFM canon set (drop `C01`-style and deuterocanon entries)
  and verify a mocked payload containing them emits only
  known ids
- [x] 1.5 backend: update `DEVELOPER_GUIDE.md` endpoint table
  and `bible_research/AGENTS.md` ("Endpoints",
  provider-routing notes) and verify docs list
  `/bible/structure/`

## 2. Frontend: display-name layer

- [x] 2.1 frontend: add `localizedBookName(code)` /
  `useBookName(code)` display lookup backed by the active
  structure, keeping `toBookName` English for `passage=`,
  stored `book` fields and external links; verify tests pin
  the contract calls still emitting English
- [x] 2.2 frontend: generate bundled `en`/`lv`/`ru` book-name
  catalogs (script hitting DBT canonical bibles
  `ENGESV`/`LAVNLI`/`RUSSYN`, output committed under
  `src/assets/`); verify `GEN` -> `1. Mozus grāmata` /
  `Бытие` / `Genesis` in the generated files

## 3. Frontend: structure fetching + picker

- [x] 3.1 frontend: add `api/structure` endpoint call +
  `structureByBible` Zustand slice + cacheManager tier keyed
  on bible abbr, resolving SWR (cache -> bundled catalog ->
  previous -> fetch); verify a translation switch leaves the
  old names rendered until the fetch resolves, then swaps
  atomically
- [x] 3.2 frontend: PassagePicker renders the active
  structure's names + coverage, greys uncovered books
  (dimmed, clickable — not `disabled`), click -> notification
  `"{translation} doesn't include {book}"` without
  navigating; verify component test asserts no navigation and
  the notification fires
- [x] 3.3 frontend: migrate display call-sites to
  `localizedBookName` (PassageView header, Audio title,
  playlists, VerseActionToolbar, `noteHeading` mapping stored
  English `book` -> USFM -> localized); verify rendered
  labels switch language on translation change in tests
- [x] 3.4 frontend: mention parser + `parseBibleRef` match
  localized names additively (`Jāņa 3` -> `JHN` under lv);
  verify English names and USFM codes still resolve

## 4. Integration

- [x] 4.1 both: run scoped verification — backend
  `pytest bible/` structure tests, frontend
  `vitest` on touched files + `npm run lint` (0/0) + `tsc`;
  verify all green
- [x] 4.2 frontend: update `reactive-bible/AGENTS.md` (data
  model contracts + layout) and `DEVELOPER_GUIDE.md` for the
  structure slice and display/contract split; verify the docs
  name the new endpoint
