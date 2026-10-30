# GET /api/v1/bible/timestamps/ (AudioTimestampView) — code review

## Summary

The endpoint routes correctly across three providers (SWORD/GCS,
API.Bible, DBT), never leaks `str(exc)` into response bodies, and has
solid happy-path and credential-leak test coverage. However, its
hand-rolled 404 responses omit `error_code` — breaking the contract
the frontend branches on — DBT `"data": null` responses escalate to
a 502 instead of an empty/404 result, and the DBT HTTP call has no
timeout. Verdict: **fair**.

## Endpoint contract

- Route: `bible/urls.py:12-14` → `AudioTimestampView.get`
  (`bible/views.py:353`).
- Params (all required, else 400 `{"error"}` at views.py:358-366):
  - `fileset_id` — routing key. SWORD ids/abbr (`LVSGLU8C1DA`,
    `LVSGLU8C2DA`, `LVSGLU8`, `GLU8`) → GCS JSON; API.Bible ids
    (`ENGNIVC1DA`, `ENGNIV_API`) → API.Bible audio-bibles endpoint;
    anything else → DBT `GET /timestamps/{fileset}/{book}/{chapter}`.
    Not validated before routing.
  - `book` — book *name* (e.g. `John`), resolved via
    `get_dbt_book_id` → 400 `Unknown book` on miss (views.py:384-389).
  - `chapter` — must parse to int >= 1, else 400 (views.py:371-379).
- Success: `200 {"data": [{"verse_start": int, "timestamp": float}]}`
  (views.py:422, 445-450, 467).
- Errors: `{"error", "error_code"}` via `provider_error_fields`
  (`bible/utils/provider_errors.py:36-74`): `rate_limited`→429,
  `not_found`→404, else `provider_error`→502
  (`PROVIDER_ERROR_STATUSES`, views.py:49-53). Hand-rolled 400/404
  bodies in the view carry `error` only — no `error_code`.

## Findings

### Bugs / correctness

1. **404 bodies lack `error_code`** — both not-found paths return
   `{"error": "Timestamps not yet generated..."}` with no
   `error_code` (views.py:404-412 SWORD `NotFound`, views.py:435-444
   API.Bible empty `timecodes`). The passage view uses
   `error_code: "book_not_in_fileset"`/`"not_found"` for the same
   situation (views.py:319, provider_errors.py:57-62), so the
   timestamps contract is inconsistent and the frontend cannot branch
   on it.
2. **`"data": null` → 502** — `result.get("data", [])` at
   views.py:465 returns `None` when DBT answers `{"data": null}`;
   iterating it raises `TypeError`, caught by the outer `except` →
   `provider_error` 502 for what is an empty/not-found result.
   Should be `result.get("data") or []`.
3. **GCS non-404 errors misclassify** — `upstream_status_code`
   (provider_errors.py:23-33) reads `exc.status` /
   `exc.response.status_code`, but `google.api_core` exceptions
   expose the HTTP code as `exc.code`. A GCS 429/403 surfaces as
   `provider_error` 502 rather than `rate_limited` 429
   (views.py:413-421). `NotFound` is handled earlier, so this only
   affects non-404 GCS failures. Shared-helper gap.
4. **Raw book IDs are rejected inconsistently** — `get_dbt_book_id`
   takes names only (bible_books.py:195-217). `book=GEN` 400s
   (fuzzy ratio ~0.6 < 0.8 cutoff) while `book=JHN` accidentally
   fuzzy-matches `john` (~0.857). If the frontend ever sends USFM
   ids, behavior depends on typo distance.
5. **`get_tts_config` KeyError → 502** — views.py:393 indexes
   `get_tts_config(canon)["voice_name"]`; a future SWORD module
   without TTS config raises `KeyError` → `provider_error` 502,
   misrepresenting a config gap as an upstream failure. Latent
   today (only LVSGLU8 registered).
6. **Unvalidated `fileset_id`** — garbage or text-only ids (e.g.
   `ENGESV_API`, which `is_esv_fileset` would match) fall through to
   the DBT branch and produce an upstream error instead of a clean
   400/404 (views.py:452-455).
7. **Minor**: `int(chapter)` is computed twice (views.py:372 and
   400); `chapter_num` is available. API.Bible timecodes that fail
   parsing are emitted as `{"verse_start": null, "timestamp": null}`
   rather than filtered (views.py:97-136, 445-450).

### Security & auth

- `AllowAny` (views.py:351) with the guest fallback is consistent
  with sibling read endpoints; covered by
  `test_public_read_permissions.py:42-44`.
- Positive: `provider_error_response` never embeds `str(exc)`;
  credential-leak regression tests exist
  (test_timestamp_view.py:120-143,
  test_audio_timestamps_endpoint.py:46-64).
- `book` is reflected verbatim in the 400 `error` (views.py:387) —
  low risk in a JSON body but inconsistent with the "don't echo raw
  input/exception text" posture elsewhere.
- No DRF throttling configured anywhere (`settings.py:368-382` has
  no throttle classes): anonymous callers can force GCS downloads
  and upstream DBT/API.Bible calls, spending provider quota.

### Performance

- **No timeout on DBT calls** — `self.session.get(url, params=...)`
  in `get_timestamps` (client.py:278) and every other raw DBT call
  has no `timeout`; a hung connection pins a Gunicorn/App Engine
  worker indefinitely. API.Bible uses `timeout=10` throughout
  (apibible/client.py:73,136).
- **Timestamps JSON re-downloaded per request** —
  `read_timestamps_json` (gcs.py:162-171) does a fresh
  `blob.download_as_bytes()` on every hit; artifacts are immutable
  per (fileset, voice, book, chapter, mode). Contrast with
  `signed_audio_url`, which caches per key (gcs.py:174-237).
- **Double GCS read per chapter play** — the passage audio path
  already downloads the same timestamps JSON for
  `duration_seconds`/`file_size_bytes` (serializers.py:209-212), then
  the frontend calls this endpoint which downloads it again.
- Good: shared DBT client with warm keep-alive
  (client.py:380-397); fileset-size payloads are small.

### Code quality & consistency

- No `@extend_schema` on this view while `BibleSearchView` declares
  its query params (views.py:536+) — the endpoint is undocumented in
  the OpenAPI schema despite drf-spectacular being configured.
- The "Timestamps not yet generated for this chapter." literal is
  duplicated at views.py:406-409 and 438-441; the two branches also
  duplicate the whole response construction.
- Inconsistent error-body construction overall: some paths use
  `provider_error_response`, others hand-roll `{"error"}` — a small
  `bad_request()/not_found()` helper that always attaches
  `error_code` would close finding 1 structurally.
- Clear comments explain the mode split (views.py:394-397) and why
  non-NotFound GCS errors → 502 (views.py:413-416). Good.

### Test coverage

Covered (test_timestamp_view.py, test_audio_timestamps_endpoint.py,
test_public_read_permissions.py): DBT success, missing params →400,
generic exception →502 with no text leak, upstream 429 →429,
unknown book →400, API.Bible ranged `verseId` shapes, SWORD GCS
serve, SWORD GCS failure →502, C1DA-vs-C2DA `mode=` selection,
anonymous access.

Missing: non-numeric/zero/negative chapter →400; GCS `NotFound`
→404 (would pin the missing `error_code`); API.Bible empty
`timecodes` →404 and `PassageNotFoundError` →`not_found`; DBT
`data: null`; upstream 404 →404; text-only fileset id behavior.

## Refactoring recommendations

1. **P0 — Add `error_code: "not_found"` to both 404 responses**
   (views.py:404-412, 435-444). Why: contract compliance; the
   frontend branches on `error_code` and currently gets `undefined`.
   Effort: S.
2. **P0 — Use `result.get("data") or []`** at views.py:465 so a
   DBT `"data": null` yields `{"data": []}` (or map to `not_found`)
   instead of 502. Effort: S.
3. **P1 — Add a `timeout` to DBT `session.get` calls**
   (client.py:278 and siblings; match API.Bible's 10s). Why: a
   stalled upstream socket currently hangs the worker forever.
   Effort: S.
4. **P1 — Teach `upstream_status_code` about `exc.code`**
   (provider_errors.py:23-33) so `google.api_core` exceptions map
   their HTTP code — GCS 429 then yields `rate_limited` 429 per the
   contract. Effort: S.
5. **P1 — Cache `read_timestamps_json` results** keyed by
   (fileset, voice, book, chapter, mode), e.g. a bounded
   in-process dict like `_signed_url_cache` or Django cache with a
   multi-minute TTL. Why: artifacts are immutable; removes a GCS
   round-trip per request and the double-read with the passage
   endpoint. Effort: M.
6. **P2 — Validate `fileset_id` before routing**: require a known
   audio-capable id (registry lookup or `*DA` suffix) and return 400
   for text-only/unknown ids; optionally accept USFM book ids
   explicitly in `get_dbt_book_id` callers. Effort: M.
7. **P2 — Add `@extend_schema` query-param docs** mirroring
   `BibleSearchView` so `/api/v1/docs/` describes the endpoint.
   Effort: S.
8. **P2 — Extract `not_found(msg)` / `bad_request(msg)` helpers**
   that always emit `error` + `error_code`, and drop the duplicated
   "not yet generated" literal. Effort: S.
9. **P2 — Filter or reject unparseable API.Bible timecodes**
   instead of emitting `null` fields (views.py:445-450). Effort: S.
10. **P2 — Consider embedding timestamps in the passage audio
    response** (serializers.py:228-243 already has the payload in
    hand) so clients need one GCS fetch instead of two; keep this
    endpoint for backwards compat. Effort: M.
