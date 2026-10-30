# GET /api/v1/bible/copyright/ (CopyrightView) — code review

## Summary

The DBT happy path works and provider failures are classified
correctly without leaking exception text, but the view only routes
two of the app's four providers: ESV and SWORD fileset ids (and the
API.Bible `abbr` `NIV`) fall through to DBT and 404 there — which is
exactly what the production frontend sends for non-DBT
translations. Copyright text is static yet every request makes an
upstream HTTP call with no timeout, no cache, and no `Cache-Control`
header. Verdict: **fair**.

## Endpoint contract

- Route: `bible/urls.py:16-20` → `CopyrightView.get`
  (`bible/views.py:475-533`).
- Param: `bible_id` (required; missing → 400 `{"error"}`,
  views.py:483-487). Naming/semantics differ from siblings' uniform
  `fileset_id`: the frontend passes the translation `abbr`
  (`reactive-bible/src/components/CopyrightNotice.tsx`), the
  API.Bible branch expects synthetic fileset ids (`ENGNIV_API`,
  `ENGNIVC1DA`), and the DBT branch forwards the raw string to
  `GET /bibles/{bible_id}/copyright` (dbt/client.py:308).
- Success: `200 {"data": [{id, type, size, copyright,
  copyright_date, copyright_description}]}` (views.py:499-508,
  513-528). API.Bible branch fabricates the DBT shape.
- Errors: `{"error", "error_code"}` via `provider_error_response`
  (views.py:139-153): `rate_limited`→429, `not_found`→404, else
  `provider_error`→502. The hand-rolled 400 body carries `error`
  only — no `error_code`.

## Findings

### Bugs / correctness

1. **ESV and SWORD ids fall through to DBT** — only
   `is_apibible_fileset` is checked (views.py:490); `is_esv_fileset`
   and `is_sword_fileset` are imported (views.py:19-25) but unused
   here. `bible_id=ENGESV_API`, `LVSGLU8`, `LVSGLU8C1DA`, `GLU8` all
   become `GET b4.dbt.io/api/bibles/{id}/copyright` → upstream 404 →
   `{"error": "Passage not found (HTTP 404)", "error_code":
   "not_found"}`. The registries already carry usable `license`
   strings (esv/registry.py:8, sword/registry.py:19). The frontend
   works around ESV with a hardcoded client-side notice
   (CopyrightNotice.tsx) but SWORD (LVSGLU8 `abbr` GLU8) gets no
   copyright at all.
2. **API.Bible `abbr` is not canonicalized** —
   `canonical_apibible_fileset_id` (apibible/registry.py:30-45)
   matches registry keys and `audio_fileset_id` only, unlike
   `canonical_sword_fileset_id` (sword/registry.py:32-47) which also
   matches `abbr`. The frontend sends `abbr` (`NIV`) as `bible_id`,
   so `bible_id=NIV` routes to DBT and the entire API.Bible branch
   (views.py:490-508) is unreachable from the real client.
3. **Audio fileset returns the text bible's copyright** —
   `bible_id=ENGNIVC1DA` canonicalizes to `ENGNIV_API` and the view
   fetches `meta['bible_id']` (views.py:491-495), then echoes
   `"id": "ENGNIVC1DA"` with `"type": "text_plain"`. The audio is a
   distinct audio-bible (`audio_bible_id`, "The Listener's Bible")
   with its own copyright — the response misattributes text
   copyright to the audio fileset.
4. **`bible_id` is unvalidated and interpolated into a URL path** —
   any string (with `/`, `?`, `..`, arbitrary length) reaches
   `f"{base_url}/bibles/{bible_id}/copyright"` (dbt/client.py:308).
   Django already decodes `%2F`, so `../`/`/` reach DBT verbatim;
   low-impact since the host is fixed, but a charset/length check
   (e.g. `^[A-Za-z0-9_-]{1,32}$`) would turn path garbage into a
   clean 400.
5. **Malformed upstream payload → 502** — `for item in (result or
   [])` (views.py:513-514) assumes a list of dicts; if DBT ever
   returns a dict (`{"data": [...]}`) or list items aren't dicts,
   `item.get` raises `AttributeError` → `provider_error` 502. A
   non-dict `copyright` value breaks `cr.get` the same way
   (views.py:515, 520). Spec is a bare array
   (dbt_openapi.json:1615-1622), so this is defensive, not urgent.
6. **400 body lacks `error_code`** — views.py:484-487 returns
   `{"error"}` only; the frontend branches on `error_code`. Same
   gap as the timestamps endpoint's 400/404s — consistent across
   views but inconsistent with the stated contract.
7. **Misleading `not_found` message** — upstream 404 renders as
   `"Passage not found (HTTP 404)"` (provider_errors.py:57-62); a
   copyright lookup has no passage. Cosmetic but user-visible.
8. **Possibly wrong API.Bible field name** — the view reads
   `copyrightStatement` (views.py:496-498); current API.Bible docs
   also emit a plain `copyright` field. If the live object drops
   `copyrightStatement`, `or ''` silently yields an empty
   copyright instead of falling back — worth verifying against the
   real NIV payload.

### Security & auth

- `AllowAny` (views.py:478) matches sibling read endpoints and the
  guest-fallback model; covered by
  test_public_read_permissions.py:46-48.
- Positive: `provider_error_response` never embeds `str(exc)`;
  credential-leak regression test exists
  (test_copyright_view.py:133-154).
- DBT key still leaks into *logs*: `logger.error(f"DBT copyright
  request error: {e}")` (dbt/client.py:316-319) logs the requests
  exception, whose message contains the request URL with
  `?key=DBT_KEY` — same exposure the provider-errors docstring
  warns about for response bodies.
- No DRF throttling (`settings.py:368-382` defines none):
  anonymous callers can burn DBT/API.Bible quota by varying
  `bible_id`; caching would mitigate.
- `bible_id` is echoed in the API.Bible-branch response `id`
  (views.py:502); benign in JSON but inconsistent — the DBT branch
  echoes upstream ids.

### Performance

- **No caching anywhere** — copyright text is effectively static
  yet each request costs a warm ~300 ms DBT round-trip (or an
  API.Bible call). No `CACHES` override exists (default locmem is
  usable), no `Cache-Control`/`ETag` on the response, and no
  `cache_page`/manual `cache.get_or_set`. The frontend caches in
  localStorage (cacheManager.ts:683+), but backend-side caching is
  what protects provider quota.
- **No timeout on DBT calls** — `session.get` at
  dbt/client.py:311-314 (and every raw DBT call) can hang a worker
  indefinitely; API.Bible uses `timeout=10` throughout
  (apibible/client.py:193-196).
- Good: shared `DBTClient` keeps the connection warm
  (dbt/client.py:384-397).

### Code quality & consistency

- No `@extend_schema` on the view — unlike `BibleSearchView`
  (views.py:536-605), so `bible_id` is undocumented in the OpenAPI
  schema.
- Param name `bible_id` vs the `fileset_id` convention everywhere
  else; worse, it carries *two* semantics (DBT bible abbr vs
  synthetic fileset id) with no docstring noting which providers
  are supported (views.py:476).
- `DBTClient.get_copyright` is annotated `-> Dict[str, Any]`
  (dbt/client.py:285-289) but returns a list — the docstring
  ("List of fileset copyright objects") is right, the type hint
  is wrong.
- Provider dispatch is inlined per-view; a shared
  `route_fileset(id)` helper would have prevented finding 1.
- Response shaping is hand-built in the view; fine for size, but
  the API.Bible fabrication duplicates the DBT field map.

### Test coverage

Covered (`bible/tests/test_copyright_view.py`): DBT success
(52-80), missing `bible_id` →400 (83-91), generic exception →502
without text leak (94-113), upstream 429 →429 (116-130), and
credential no-leak (133-154).

Missing: the entire API.Bible branch (success, missing
`API_BIBLE_KEY`, upstream 404 →`not_found`); ESV/SWORD/abbr ids
falling through to DBT (would pin findings 1-2); upstream 404 →404
on the DBT path; `result=None` or non-list/non-dict items; and
`bible_id` format validation if added.

## Refactoring recommendations

1. **P0 — Route non-DBT ids before the DBT call** (views.py:489+):
   answer ESV/SWORD filesets from their registry `license` fields,
   and canonicalize `abbr` in `canonical_apibible_fileset_id` like
   the SWORD registry does. Why: today every non-DBT translation
   the app serves returns 404/incorrect copyright; the frontend
   already hardcodes ESV client-side to compensate. Effort: M.
2. **P0 — Add `timeout` to the DBT `session.get` calls**
   (dbt/client.py:311 and siblings; match API.Bible's 10 s). Why:
   a stalled upstream socket pins the GAE worker forever.
   Effort: S.
3. **P1 — Cache copyright responses** per normalized
   `(provider, bible_id)` with a multi-hour TTL (Django cache or a
   bounded in-process dict) and set `Cache-Control: public,
   max-age=…` on the 200. Why: data is static; removes an upstream
   round-trip per request and protects provider quota. Effort: M.
4. **P1 — Validate `bible_id`** before routing: trim, length cap,
   charset whitelist; return 400 (+`error_code`, per finding 6) on
   garbage instead of an upstream call. Effort: S.
5. **P1 — Fix the API.Bible audio-fileset branch** (views.py:
   490-508): for `*DA` ids use `audio_bible_id` (or emit
   `type: "audio"`) so audio filesets don't carry the text
   copyright; add a `copyright` fallback to `copyrightStatement`.
   Effort: S.
6. **P2 — Attach `error_code` to the 400 body** and use a shared
   `bad_request(msg)` helper (same fix as the timestamps report)
   so every error response satisfies `{"error","error_code"}`.
   Effort: S.
7. **P2 — `isinstance` guards on the DBT payload** (views.py:
   513-527): skip/coerce non-dict items instead of 502-ing on a
   shape change; fix the `-> Dict` annotation in
   `get_copyright`. Effort: S.
8. **P2 — Add `@extend_schema` param docs** mirroring
   `BibleSearchView`, documenting accepted ids per provider and
   the `bible_id`-vs-`fileset_id` naming quirk. Effort: S.
9. **P2 — Extend tests**: API.Bible success/404/missing-key,
   registry-id fall-through cases, upstream 404 →404, malformed
   payload items, and (once added) validation + cache-hit paths.
   Effort: M.
