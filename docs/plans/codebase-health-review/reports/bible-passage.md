# GET /api/v1/bible/ (BiblePassageView) — code review

## Summary

The endpoint parses a tolerant `passage` grammar, routes on
`fileset_id` to one of four providers, and honors the
`error`/`error_code` contract without ever leaking `str(exc)` —
with strong tests for exactly that. But the API.Bible branch
silently ignores verse selectors, the verse-range parser expands
unbounded ranges into a `set` (unauthenticated memory DoS),
DBT calls have no timeout, and behavior for the same
`fileset_id` × `response_format` combination diverges per
provider. Verdict: **fair**.

## Endpoint contract

- Route: `bible/urls.py:5` → `BiblePassageView.get`
  (`bible/views.py:178`), `permission_classes = [AllowAny]`
  (views.py:176). Anonymous: `DeviceAndCountryMiddleware` treats
  `/api/v1/bible` as a public-read prefix
  (`bible_research/middleware.py:16-20,31-39`).
- Params:
  - `passage` (required): `"<Book name> <chapter[:v[-e][,...]]>"`
    parsed at views.py:209-252 by `passage.split()` +
    `_parse_chapter_part` (views.py:58-94); book names resolve via
    `get_dbt_book_id` — exact → alias → fuzzy 0.8
    (`bible/utils/bible_books.py:195-217`).
  - `fileset_id` (default `ENGESV`, views.py:183): `ENGESV_API` →
    ESV; `LVSGLU8`/`GLU8`/`LVSGLU8C{1,2}DA` → SWORD (+GCS audio);
    `ENGNIV_API`/`ENGNIVC1DA` → API.Bible; anything else → DBT
    (serializers.py:123,191,245,291,337). Unvalidated before
    routing.
  - `response_format`: `text` (default) or `audio`, else 400
    (views.py:200-207).
- Success `200`: `{book, book_name, chapter, format}` plus
  `verses:[{verse,text}]` (+`headings` ESV/API.Bible only) for
  text; `audio_url` (+`duration_seconds`/`file_size_bytes`
  SWORD/DBT only, +`meta` FUMS API.Bible only) for audio. Audio
  is chapter-level — verse selectors are ignored on
  `response_format=audio` everywhere.
- Errors: `{"error","error_code"}` — `rate_limited`→429,
  `not_found`/`book_not_in_fileset`→404, else
  `provider_error`→502 (views.py:49-53,139-153,293-300;
  `bible/utils/provider_errors.py:36-74`). Client-input 400s use
  `{"error": str(e)}` with no `error_code`
  (views.py:192-207,342-345).

## Findings

### Bugs / correctness

1. **API.Bible ignores the verse selector entirely** —
   `to_representation` returns every verse of the chapter and
   never applies `_select_verses` or the heading filter on the
   `is_apibible_fileset` branch (serializers.py:312-335), unlike
   ESV (serializers.py:148-172), SWORD (serializers.py:268-277)
   and DBT (serializers.py:387-397). `?passage=John 3:16&
   fileset_id=ENGNIV_API` returns John 3 whole; the
   `verse_start`/`verse_end` hints computed at
   serializers.py:112-113 are dead for this provider.
2. **Unbounded verse-range expansion — unauthenticated memory
   DoS** — `_parse_chapter_part` materializes every range into a
   `set` of ints (views.py:83-94): `John 3:1-999999999`
   allocates ~10^9 ints (tens of GB) on an `AllowAny` endpoint;
   there is also no cap on segment count or chapter/verse
   magnitude. Rejects nothing until the allocation happens.
3. **Same-meaning misses map to different `error_code`s** — the
   frontend branches on `book_not_in_fileset` for fileset
   fallback, but a coverage miss on API.Bible yields `not_found`
   instead: upstream 404 → `provider_error_fields` (serializers.
   py:410-417, provider_errors.py:57-62) and the client's own
   `PassageNotFoundError` (apibible/client.py:105-109,143-146)
   both classify as `not_found`, while ESV/DBT/SWORD map the
   equivalent case to `book_not_in_fileset` (serializers.py:
   181-189,258-267,342-355). Relatedly, an out-of-range verse
   selection is `book_not_in_fileset` for SWORD (client's
   `PassageNotFoundError` is a `ValueError`, caught at
   serializers.py:258) but `not_found` for ESV/DBT.
4. **`format` is provider-dependent for mismatched
   fileset/format pairs** — a `text` request to a DBT `*DA`
   audio fileset returns `format:'audio'` + `audio_url`
   (serializers.py:356-374), while `LVSGLU8C1DA` and
   `ENGNIVC1DA` + `text` return verse text (serializers.py:245+,
   312+). Conversely `audio` on a text fileset: ESV/SWORD/
   API.Bible serve audio (serializers.py:125-136,191-243,
   294-311) but a DBT text fileset → 404 `book_not_in_fileset`
   (serializers.py:375-380).
5. **Dead chapter-positivity check** — `_parse_chapter_part`
   already raises for `chapter <= 0` (views.py:77-79), so the
   guard at views.py:254-264 is unreachable; its comment refers
   to a removed `int()` try/except.
6. **`getattr(settings, 'DBT_KEY')` has no default** —
   `dbt/client.py:42` raises `AttributeError` when the setting is
   absent, making the friendly `raise ValueError` at
   client.py:44-45 dead code (ESV/API.Bible use
   `getattr(..., None)` correctly at esv/client.py:22,
   apibible/client.py:51).
7. **Minor**:
   - SWORD audio sniffs audio format from `rows[0]` only
     (serializers.py:356) and returns only `rows[0]`'s URL —
     multi-file chapters drop siblings.
   - SWORD per-verse loop `break`s on first empty/failed verse
     (sword/client.py:129-143): a mid-chapter gap truncates the
     rest of the chapter rather than skipping it.
   - `"1John 3"` (missing space) fuzzy-matches `john` → JHN, not
     1JN (bible_books.py:165-192); common abbreviations `Ps`,
     `Gen`, `Matt` all 400 — no alias table for them.
   - A blank `fileset_id=` 400s via `serializer.errors` — a
     third, differently-shaped 400 body (views.py:325-331).

### Security & auth

- `AllowAny` + public-read middleware → truly anonymous reads;
  consistent with sibling endpoints. `CSRFExemptSessionAuthenti
  cation` is irrelevant for GET. Clean.
- **No throttling** — `REST_FRAMEWORK` has no throttle classes
  (settings.py:368-382); anonymous callers can spend ESV/
  API.Bible/DBT quota (paid upstreams) and, per finding 2,
  exhaust worker memory.
- **Response bodies never leak `str(exc)`** — enforced by
  `provider_error_fields` (provider_errors.py:36-74) and locked
  by tests (test_provider_errors.py:102-124,260-284). Good.
- **…but logs do leak the DBT key** — `key` travels as a query
  param (dbt/client.py:106-108,268-270,304-306,351-355), so
  `HTTPError`/`MaxRetryError` text embeds the full URL; that
  text is logged by `logger.exception(f"Error fetching Bible
  passage: {e}")` (serializers.py:411) and the client's own
  `logger.error(f"...: {e}")` calls (dbt/client.py:87,92,120,
  282,317,374). Response-scrubbed, log-unscrubbed.
- **Unvalidated `fileset_id` flows into upstream URL paths** —
  the generated call is safe, but sibling raw f-string calls
  interpolate it unquoted (dbt/client.py:272-275,308,365); only
  reachable for non-passage calls today, still worth quoting.
- Raw `passage`/`book_name` echoed in INFO logs and the
  `Unknown book` 400 (views.py:186,236) — log-forging-grade
  only; low risk.

### Performance

- **No timeout on any DBT call** — raw `session.get` calls pass
  no `timeout` (dbt/client.py:114-116,278,311-313,368-370) and
  the generated client defaults `_request_timeout=None`
  (openapi_client/api_client.py:126): a stalled b4.dbt.io socket
  pins a worker indefinitely on the hottest endpoint. ESV and
  API.Bible use `timeout=10` throughout (esv/client.py:56,96,
  180; apibible/client.py:73,136,186,195).
- **No response caching anywhere** — each request is a live
  upstream call; chapter text is effectively immutable but
  `CACHES` is unconfigured (settings.py) and nothing memoizes
  passage bodies. Only signed audio URLs are cached
  (gcs.py:174-237).
- **SWORD audio = 2-3 GCS round-trips/request** —
  `chapter_audio_exists` HEAD (gcs.py:78-88) + immutable
  `read_timestamps_json` download (gcs.py:162-171) + a
  `get_blob` fallback for legacy artifacts
  (serializers.py:218-227); the same timestamps JSON is then
  re-downloaded by the timestamps endpoint.
- `signed_audio_url` re-runs `google.auth.default()` +
  `credentials.refresh` on every cache miss (gcs.py:217-231) —
  credentials could be fetched once.
- DBT verse selection post-filters locally rather than
  narrowing upstream (serializers.py:381-397) — deliberate for
  audio filesets (documented comment) but wasteful for text.
- SWORD text does one `bible.get()` per verse, up to 200
  (sword/client.py:129-143) — local zip reads, acceptable.
- `print()` runs 3x per request in middleware
  (middleware.py:44,71,117) — trivial cost, wrong channel.

### Code quality & consistency

- Provider routing, upstream I/O, filtering and error mapping
  all live inside `BiblePassageSerializer.to_representation` —
  a ~315-line `except Exception`-wrapped method
  (serializers.py:102-417). A serializer that fetches is the
  codebase's own pattern violation vs. its AGENTS.md guidance
  ("Validate input in serializer validate_* methods").
- `serializer.to_representation(data)` is called on the raw
  input dict, not `serializer.validated_data` (views.py:281-287)
  — validation is decorative; `response_format` isn't even a
  declared field so `is_valid()` ignores it.
- `print()` instead of `logger` in middleware on every request
  (middleware.py:44,71,117,150,158,161,174,182).
- `sys.path.append` at import time for the generated client
  (dbt/client.py:16-17); mid-file third-party imports after it
  (client.py:18-24).
- Mixed f-string vs lazy `%s` logging between views.py and
  translation_service.py — cosmetic.
- `ESVClient.get_chapter_verses` (esv/client.py:61-68) is only
  referenced by tests — dead code on the request path.
- `get_audio_bible_id` docstring says default `'1DA'` but the
  signature is `'2DA'` (bible_books.py:236,245).
- `bible/utils/` is a namespace package (no `__init__.py`) —
  works, but inconsistent with the rest of the tree.

### Test coverage

Covered: 400/404/429/502 contract incl. credential-leak
regressions (test_provider_errors.py, test_book_not_in_fileset.
py); verse selectors for DBT/ESV/SWORD incl. narrowing and
out-of-range → `not_found` (test_verse_selection.py); view-level
validation of missing/malformed passages (test_passage_view.py);
real-module SWORD reads (test_sword.py); API.Bible parse shapes,
FUMS pass-through, audio URL, error mapping
(test_apibible_client.py).

Missing: **any API.Bible verse-selector test** (would pin
finding 1); oversized verse ranges / huge chapter numbers;
API.Bible `not_found` vs `book_not_in_fileset` semantics
(finding 3); `text`-on-audio-fileset divergence (finding 4);
ESV-audio and API.Bible-audio success bodies through the view;
provider timeout behavior.

## Refactoring recommendations

1. **P0 — Apply `_select_verses` (+ heading filter) on the
   API.Bible branch** (serializers.py:312-335), mirroring the
   ESV code path at serializers.py:148-172; optionally pass
   `verse_start`/`verse_end` upstream later. Why: `John 3:16`
   on NIV silently returns the whole chapter — a contract bug on
   the second-most-likely text provider. Effort: S.
2. **P0 — Bound the verse grammar** in `_parse_chapter_part`
   (views.py:83-94): cap `end` (e.g. 200), cap total selected
   verses and segments, cap `passage` length at the view.
   Why: unauthenticated single-request memory exhaustion.
   Effort: S.
3. **P1 — Add `timeout=` to all DBT HTTP calls**
   (dbt/client.py:114-116,278,311-313,368-370; pass
   `_request_timeout` on the generated path, client.py:237-243).
   Why: hangs pin workers on the hottest endpoint. Effort: S.
4. **P1 — Normalize the coverage-miss `error_code`** so every
   provider emits `book_not_in_fileset` for "fileset lacks this
   book/chapter": catch API.Bible upstream-404/`PassageNotFound
   Error` in the serializer branch (serializers.py:291-311) and
   map SWORD out-of-range consistently. Why: the frontend's
   fileset-fallback logic keys on this code. Effort: M.
5. **P1 — Scrub credentials from DBT logging**: log
   `type(exc).__name__`/status instead of `str(exc)` at
   serializers.py:411 and dbt/client.py:87,92,120,282,317,374,
   or move `key` to a header/param constant scrubbed by a
   logging filter. Why: API key currently lands in GCP logs on
   every upstream failure. Effort: S.
6. **P1 — Extract the provider dispatch out of
   `to_representation`** into a `passage_service.fetch(data)`
   returning the body; keep the serializer for input validation
   only (and pass `validated_data`). Why: 315-line
   representation method doing I/O is the root of findings
   1-4. Effort: M.
7. **P2 — Add an explicit `text`-vs-`audio` capability check**
   so a `text` request to a `*DA` id 400/404s predictably
   instead of returning `format:'audio'` (serializers.py:356-374)
   — or document the matrix; also decide whether ESV/API.Bible
   should expose duration/size fields to match SWORD/DBT audio.
   Effort: M.
8. **P2 — Configure DRF throttling** (anon rate) on this view or
   globally; even a loose ceiling blunts finding 2 and quota
   burn. Effort: S.
9. **P2 — Cache immutable reads**: `read_timestamps_json` keyed
   by (fileset, voice, book, chapter, mode) like
   `_signed_url_cache` (gcs.py:162-171,174-237), plus a
   short-TTL passage-body cache keyed by the full request.
   Effort: M.
10. **P2 — Delete dead code**: unreachable chapter check
    (views.py:254-264), `ESVClient.get_chapter_verses`
    (esv/client.py:61-68), `middleware.py` `print()`s →
    `logger.debug`. Effort: S.
11. **P2 — Add book abbreviations to `get_dbt_book_id`**
    (`Ps`, `Gen`, `Matt`, OSIS ids like `JHN`) and a max input
    length; fixes `1John`/`Ps 23` usability and bounds the
    fuzzy-matcher CPU. Effort: S.
12. **P2 — Unify the 400 shape**: always emit
    `{"error","error_code"}` (e.g. `invalid_request`) so client
    errors share the contract — currently three different 400
    bodies (views.py:192-207,325-331,342-345). Effort: S.
