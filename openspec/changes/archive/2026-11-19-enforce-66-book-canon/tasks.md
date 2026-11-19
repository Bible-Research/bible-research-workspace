# Tasks

## 1. Canon constant

- [x] 1.1 backend: Export `CANONICAL_BOOK_IDS` (frozenset of the
      66 USFM codes derived from `_BIBLE_BOOKS`) in
      `bible/utils/bible_books.py` and verify
      `len(CANONICAL_BOOK_IDS) == 66` in a unit test in
      `bible/tests/test_bible_books.py`

## 2. Search canon enforcement

- [x] 2.1 backend: In `BibleSearchView.get`, after
      `is_valid_book_id_list`, `strip().upper()`-normalize each
      `books` token and intersect with `CANONICAL_BOOK_IDS`,
      serializing survivors as a sorted comma-separated string.
      On an empty intersection return the provider-shaped
      empty payload with no provider call —
      `{groups: [], meta: {total: 0, truncated: false}}` in
      grouped mode immediately; in paginated mode AFTER
      limit/page validation (so `page=abc` still 400s):
      `{verses: [], meta: {pagination: {total: 0, count: 0,
      per_page: <limit>, current_page: <page>,
      total_pages: 0}, truncated: false}}`. Verify with tests
      that `books=TOB` returns the empty payload without a
      provider call and `books=jhn` behaves as `books=JHN`
- [x] 2.2 backend: In `DbtProvider.search` and
      `fetch_dbt_matches`, always send the `books` string
      upstream — the view-intersected list, or the full
      sorted 66-id string when no `books` param was given —
      verifying a test asserts the outgoing DBT `search` call
      carries only canonical ids
- [x] 2.3 backend: Post-filter normalized search rows to
      canonical `book_id`s (`normalize_dbt_verse`,
      `normalize_apibible_verse`, or the provider aggregation
      seam); filtered rows must be REMOVED from the emitted
      `verses` list, not nulled (the paginated path emits
      rows verbatim). Verify a test with a mocked `TOB` verse
      shows it dropped from both paginated and grouped
      payloads while canonical rows survive
- [x] 2.4 backend: Update `bible_research/AGENTS.md` (repo
      root — there is no `bible/AGENTS.md`), the workspace
      `AGENTS.md` cross-repo notes, and `DEVELOPER_GUIDE.md`
      with the canon-filtering contract, the API.Bible totals
      limitation, and the `--book TOB` → empty pending list
      behavior — verify docs render within 79 chars

## 3. Passage + worklist guards

- [x] 3.1 backend: In `get_passage_body`, return
      `_book_not_in_fileset_body` when `book.upper()` is not in
      `CANONICAL_BOOK_IDS`, before provider dispatch — verify
      a test calling `get_passage_body` with `book='TOB'`
      gets `error_code='book_not_in_fileset'` and no provider
      call
- [x] 3.2 backend: In `generate_audio_timestamps._worklist`,
      skip rows whose `book_id` is non-canonical — verify a
      test with a `TOB` file row shows it excluded from the
      worklist

## 4. Verification

- [x] 4.1 backend: Run the scoped test files
      (`bible/tests/test_bible_books.py`,
      `test_passage_providers.py`, search/timestamp view
      tests, plus any new test module) and confirm all pass;
      run `awk 'length > 79'` check on edited files

## Workflow follow-up

- Open PR to `Bible-Research/bible_research` `main` per
  `.devin/skills/create-pr/` (body via `--body-file`, UI
  verification section: search on a deuterocanonical DBT
  fileset such as a Catholic translation shows only canonical
  groups)
- After merge, archive this change per
  `openspec-archive-change`
