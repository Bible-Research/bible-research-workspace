# 00 — Base architecture: bible_research Django → Rust

Status: BASELINE. All per-endpoint plans in
`.research/rust-rewrite/endpoints/` build on this file. Cited paths
are relative to the Django repo root `bible_research/`. Analysis is
against `main`; see §4.8 for two unmerged feature branches that
change the contract.

## 1. Executive summary

Rewrite the Django 5.2 + DRF API as an **axum 0.8** service on tokio,
with **sqlx 0.8** against the existing PostgreSQL schema (no ORM;
the schema stays Django-owned — we snapshot the DDL once as a
read-only baseline), **serde** for DRF-style serialization,
**reqwest** for the DBT/ESV/Google-TTS HTTP clients,
**utoipa + utoipa-swagger-ui + utoipa-redoc** to reproduce the
`/api/v1/schema|docs|redoc/` surfaces, and the official
**google-cloud-storage / google-cloud-auth** crates for GCS access
and V4 signed URLs via IAM `signBlob` (no JSON keys, matching the
current ADC approach). SWORD support goes behind a provider trait:
evaluate the young `rsword_chirho` crate; fall back to a ~400-line
port of pysword's ztext reader (the vendored module is a plain
`ztext` zip). Deploy as a Docker image to **Cloud Run** (a Cloud Run
job binary replaces the `generate_chapter_audio` management
command); App Engine standard cannot run Rust and is retired.

## 2. Recommended stack

Format: concern — Django component → Rust approach.

- Web framework — Django + DRF → axum 0.8, tokio 1, tower-http 0.6
- Serializers — DRF serializers → serde 1 DTOs (manual validation
  where exact DRF error bodies are required)
- DB access — Django ORM → sqlx 0.8, `PgPool`, runtime `query_as`
- Auth — Token/Bearer/Session + middleware → axum
  `FromRequestParts` extractor (`core::auth`, §4.2)
- Passwords — `check_password` PBKDF2 → `djangohashers` 1.x
- Config — `config.yaml` + env + Secret Manager (`gcp.py`) →
  `config` crate: optional yaml + env layer; secrets as env vars
- HTTP clients — `requests` + generated `openapi_client` →
  reqwest 0.12 (one shared `Client` per provider, keep-alive)
- Error contract — `provider_errors.py` → thiserror 2 +
  `IntoResponse` (`core::errors`, §4.4)
- OpenAPI — drf-spectacular → utoipa 5 + utoipa-axum 0.3
- Swagger/Redoc — Spectacular views → utoipa-swagger-ui 9,
  utoipa-redoc 6
- Logging — `logging` + `print()` → tracing 0.1,
  tracing-subscriber 0.3, `TraceLayer`
- CORS — django-cors-headers (allow-all) → `CorsLayer::permissive()`
- Prefixed PKs — `uuid4` helpers → uuid 1 (`new_v4().simple()`) + hex
- Book lookup — `bible_books.py` (difflib fuzzy) → `difflib` crate
  (same `SequenceMatcher.ratio` semantics)
- Multipart upload — DRF `MultiPartParser` → axum `Multipart`
- Image validation — Pillow `verify()` → `image` 0.25 decode check
- MP3 duration — mutagen → `lofty` (`TaggedFile` duration)
- GCS — google-cloud-storage (Py) → `google-cloud-storage` +
  `google-cloud-auth` `Signer` (official Google Rust SDK)
- Secret Manager — `gcp.py get_secret` → env vars via Cloud Run
  `secret_key_ref` (drop the runtime client)
- Cloud TTS — google-cloud-texttospeech (gRPC) → reqwest POST to
  `texttospeech.googleapis.com/v1/text:synthesize` + OAuth token
  from `google-cloud-auth`
- SWORD — pysword (ztext zip) → `rsword_chirho` 0.3, else port
  pysword (§4.10)
- Tests — pytest + DRF client → cargo test, `axum-test`,
  `wiremock` for provider HTTP
- Deploy — App Engine std + gunicorn → Cloud Run service +
  Cloud Run job (one Docker image, two binaries)

### Framework choice

- **axum (chosen).** Dominant, Tokio-native, stable 0.8 API. DRF
  concepts map cleanly: serializer → serde DTO + extractor, viewset
  action → handler fn, permission class → `FromRequestParts` impl,
  `query_params` → `Query<T>`. Middleware is tower/tower-http —
  CORS, tracing, timeouts off the shelf. Best docs and examples for
  a Django dev new to Rust.
- **actix-web.** Mature and fast, but its own middleware/extractor
  conventions and a parallel ecosystem to tower; nothing it offers
  that axum lacks for this API.
- **loco.** Rails-like full-stack framework: generators, SeaORM,
  workers, mailers. Opinionated about owning schema + migrations —
  conflicts with our Django-owned schema and the never-migrate
  constraint. Overkill for an API-only rewrite.
- **poem / rocket / ntex.** Workable, smaller ecosystems, weaker
  OpenAPI integrations; no compensating advantage here.

### DB choice

- **sqlx (chosen).** Async, `PgPool`/`SqlitePool`, plain SQL with
  `query_as`. We must preserve exact SQL semantics the ORM produces
  today — partial unique index on `(tag, tag_position)`, `FILTER`
  counts, `FOR UPDATE` locks, `ON CONFLICT` upserts — so
  hand-written SQL is the honest port and sqlx stays out of the way.
  Prefer runtime `query_as` over compile-checked `query!` so builds
  never need a live DB.
- **sea-orm.** Wants its own entity/migration model; fixed schema
  with prefixed string PKs maps awkwardly. Wrong tool.
- **diesel.** Sync-first; `diesel-async` less mature; DSL friction
  where we need literal SQL control.

## 3. Cargo workspace layout

One workspace, five crates — mirrors the Django apps so each
endpoint agent owns a predictable module, and the Cloud Run job
reuses the same domain code:

```
bible_research_rs/            (new repo)
├── Cargo.toml                # workspace members = crates/*
├── crates/
│   ├── core/                 # shared foundations (bible_research/)
│   │   └── src/
│   │       ├── config.rs     #  ← settings.py, config.yaml, gcp.py
│   │       ├── db.rs         #  PgPool, tx helpers
│   │       ├── auth.rs       #  Principal ← authentication.py
│   │       ├── provision.rs  #  device/lang ← middleware.py
│   │       ├── ids.rs        #  TAG/NOT/CMNT_/IMG_ PK generation
│   │       ├── errors.rs     #  ApiError, ProviderError contract
│   │       ├── pagination.rs #  DRF PageNumberPagination shape
│   │       └── books.rs      #  ← bible/utils/bible_books.py
│   ├── providers/            # ← bible/services/*
│   │   └── src/
│   │       ├── mod.rs        #  PassageProvider trait + routing
│   │       ├── dbt.rs        #  ← services/dbt/client.py
│   │       ├── esv.rs        #  ← services/esv/{client,registry}
│   │       ├── sword.rs      #  ← services/sword/{client,registry}
│   │       ├── tts.rs        #  ← services/google_tts/*
│   │       └── grouping.rs   #  ← services/search_grouping.py
│   ├── storage/              # ← services/storage/gcs.py +
│   │   └── src/…             #   annotations image_storage.py
│   ├── server/               # the API binary (axum)
│   │   └── src/
│   │       ├── main.rs       #  state, router, layers, serve
│   │       ├── state.rs      #  AppState { pool, config, … }
│   │       ├── openapi.rs    #  utoipa registration
│   │       └── routes/
│   │           ├── bible/    #    ← bible/views.py (5 handlers)
│   │           ├── annot/    #    ← annotations/views.py
│   │           └── users/    #    ← users/views.py + /api/token
│   └── audio_job/            # binary ← generate_chapter_audio.py
│       └── src/main.rs
└── tests/
    ├── fixtures/schema.sql   # pg_dump --schema-only baseline
    └── parity/               # contract tests vs the Django API
```

Rationale: a single `server` crate keeps route registration
centralized (the `urls.py` equivalent lives in one place = fewer
merge conflicts for parallel agents); `core`/`providers`/`storage`
give agents stable shared APIs. `audio_job` is a second binary so
the Cloud Run job and the API share one image.

## 4. Cross-cutting design

### 4.1 Routing mount points

Reproduce `bible_research/urls.py` exactly:

- `GET /` → 302 redirect to `/api/v1/docs/`
- `/api/v1/bible/…` → `routes::bible` (bible/urls.py)
- `/api/v1/{tags,notes,reading-positions,comments,images}…` →
  `routes::annot` (annotations/urls.py, DRF router)
- `/api/v1/users/…` → `routes::users` (users/urls.py)
- `POST /api/token/` → `obtain_auth_token` equivalent (outside v1)
- `GET /api/v1/schema/` → OpenAPI JSON (public)
- `GET /api/v1/docs/` → Swagger UI (public)
- `GET /api/v1/redoc/` → Redoc (public)
- `GET /healthz` → Cloud Run health (new; no Django equivalent)

NOT ported: `/admin/` (§4.12), `/static/`, `/favicon.ico` (GAE
handlers), `django_session` handling.

Router subtleties:

- DRF `DefaultRouter` registers `notes/bulk/`, `notes/linked/`,
  `notes/reorder/` (detail=False actions) BEFORE `notes/{pk}/`.
  axum prefers static segments over `{param}` anyway — register all
  three explicitly.
- DRF path converters are `[^/.]+` — IDs never contain dots. No
  `.json` format-suffix support needed.
- `CommentViewSet.http_method_names` excludes PUT — expose only
  GET/POST/PATCH/DELETE under `notes/{note_pk}/comments/{pk}`.

### 4.2 Auth extractor (Token/Bearer/Session + middleware)

Django truth — prod vs tests differ:

- `bible_research/authentication.py`: `TokenAuthentication`
  (`Authorization: Token <key>`), `BearerTokenAuthentication`
  (`Bearer <key>`), `CSRFExemptSessionAuthentication`.
- `DeviceAndCountryMiddleware` (`bible_research/middleware.py`) runs
  BEFORE DRF auth. `auto_authenticate` runs iff
  `!request.user.is_authenticated` OR path is NOT under `/admin` or
  `/api/v1/bible`. It derives username `"{primary_language}
  {device}"` (truncate 150 chars), get-or-creates the user with
  `email="{username}@example.com"` + unusable password, then
  `login()`s so session auth picks it up. On ANY exception it
  falls back to get-or-create user `guest`.
- `device`: first `(...)` group of `User-Agent`; split on `;` →
  `"p0; p1"` if ≥2 parts else `p0`; fallback `"Unknown Device"`.
- `primary_language`: parse `Accept-Language` q-values, sort desc,
  first code not in `{en, en-US, en-GB}`; else the highest-q code;
  if the header is absent the Python value is `None` → the username
  literally becomes `"None <device>"`. REPRODUCE VERBATIM —
  usernames are DB identity.
- Prod effect: every request resolves to a user; DRF token auth
  overrides when credentials are present. Truly-anonymous requests
  never reach views in prod.
- Test effect: `conftest.py` configures settings WITHOUT the
  middleware and WITHOUT `REST_FRAMEWORK` (DRF defaults = Session
  +Basic auth, `AllowAny`) — so tests DO hit the anonymous branches.

Rust design (`core::auth`):

```rust
pub enum AuthVia { Token, Device, Guest, Anonymous }
pub struct Principal { pub user: Option<User>, pub via: AuthVia }
// is_authenticated() = user.is_some() — Django semantics:
// a provisioned device/guest user IS authenticated.
```

`impl FromRequestParts for Principal`:

1. `Authorization` present:
   - scheme `Token`/`Bearer` →
     `SELECT user_id FROM authtoken_token WHERE key = $1` → hit:
     token user; miss: **401** `{"detail": "Invalid token."}` with
     `WWW-Authenticate: Token`. Do NOT fall through to provisioning
     (DRF raises `AuthenticationFailed`). `Token`/blank or malformed
     key → same 401.
   - other scheme → treat as absent (step 2).
2. No usable `Authorization`:
   - `auto_provision` enabled (prod): derive username per the rules
     above; SELECT-or-INSERT user (`ON CONFLICT (username) DO
     NOTHING` then re-SELECT, in a txn); any error → get-or-create
     `guest` identically.
   - disabled (tests): `Principal` is Anonymous.
3. `django_session` is NOT ported — every session user was a
   provisioned device user anyway; observably equivalent. Accepted
   deviation, documented.

Unusable password marker: Django stores `!…`; Rust writes
`format!("!{}", rand_string())`.

Endpoint agents: use `Principal` for AllowAny-ish endpoints and
`AuthUser` (thin wrapper rejecting `Anonymous` with 401
`{"detail": "Authentication credentials were not provided."}`) for
`IsAuthenticated` ones. Prod's default permission IS
`IsAuthenticated` (`settings.REST_FRAMEWORK`), but provisioning
makes it pass — do not hand-roll per-endpoint checks.

`POST /api/token/` uses `CaseInsensitiveAuthBackend`
(`users/backends.py`): `WHERE lower(username) = lower($1)` +
`djangohashers::check_password`; on unknown user hash the password
anyway (timing blunting, Django does the same). Token issue =
`get_or_create` on `authtoken_token`; new key = 40 lowercase hex
chars of 20 random bytes. Tokens stored PLAINTEXT (Django default)
— keep for parity; hashing at rest is follow-up hardening.

### 4.3 DB access & prefixed PKs

- `sqlx::PgPool` in `AppState`. Tests use a scratch Postgres seeded
  from `tests/fixtures/schema.sql` (committed `pg_dump
  --schema-only`). No sqlx migrations — schema is a snapshot; the
  Django app keeps owning it during the transition.
- Tables (Django defaults + one override): `auth_user`,
  `authtoken_token`, `bible_verse`, `annotations_tag`,
  `annotations_note`, `annotations_noteverse`, `annotations_comment`,
  `annotations_image`, `reading_positions` (`db_table` override).
- PKs: `core::ids` — `TAG`,`NOT`,`VER`,`NVE`,`RDP` + first 15 chars
  of `Uuid::new_v4().simple().to_uppercase()`; `CMNT_`,`IMG_` + 13.
  All varchar(18) PKs.
- Transactions: `pool.begin()` for reorder two-phase update, note
  create + NoteVerse links, image row-lock + insert, position
  upsert. `SELECT … FOR UPDATE` = `select_for_update()`.
- Unique violations: sqlx `is_unique_violation()` + `constraint()`
  — `unique_tag_position_per_tag` (partial, `WHERE tag_id IS NOT
  NULL`) drives the reorder 409 contract. `reading_positions`
  upsert: `ON CONFLICT (user_id, book) DO UPDATE`.
- Timestamps `timestamptz` → serialize RFC 3339 with `Z`
  (`chrono`, `SecondsFormat::Micros` — DRF emits
  `"2025-12-28T03:51:55.123456Z"`).
- `tag_position`/`position` are `float8` → `f64`; reorder writes
  negative temporaries then final values (`annotations/views.py`
  `reorder`).

### 4.4 Error contract — EXACT (frontend branches on it)

`bible/utils/provider_errors.py` is the source of truth. Rust:
`core::errors::ProviderError { upstream_status: Option<u16>,
kind }` → `error_fields()` producing `{"error": …,
"error_code": …}`:

- `error_code` ∈ `rate_limited` | `not_found` | `provider_error`.
- upstream HTTP 429 → `rate_limited`, status 429, error
  `"Bible provider rate limit exceeded (HTTP 429)"`.
- `not_found`, status 404: `PassageNotFoundError` equivalent
  (empty provider `data`, missing SWORD chapter) OR upstream 404.
  error `"Passage not found"` or `"Passage not found (HTTP 404)"`.
- else `provider_error`, status 502: error
  `"Bible provider error (HTTP {status})"` or
  `"Bible provider request failed ({ErrorKind})"` where
  `{ErrorKind}` is a Rust enum name — NEVER `err.to_string()`:
  reqwest errors embed request URLs carrying `?key=<DBT_KEY>`.
  Never leak upstream URLs/bodies into `error`.
- View-level: `provider_error_response()` wraps fields with the
  mapped status (default 502).
- `BiblePassageSerializer.to_representation` embeds the SAME fields
  in a 200-shaped body `{book, book_name, chapter, error,
  error_code}`; the VIEW then upgrades HTTP status. Both layers.
- `NoteSerializer` degradation: `verses` kept with empty `text`,
  `headings: []`, `error`/`error_code` merged — HTTP stays 200.
- SWORD audio not generated → 200-shaped `{…, "format": "audio",
  "audio_url": null, "message": "Audio not yet generated for this
  chapter"}`; the VIEW turns it into 404.

DRF standard bodies to reproduce:

- 401 no creds: `{"detail": "Authentication credentials were not
  provided."}`; bad token: `{"detail": "Invalid token."}`
- 403 `PermissionDenied`: `{"detail": "<msg>"}`
- 404 `get_object_or_404`: `{"detail": "Not found."}`
- 405: `{"detail": "Method \"GET\" not allowed."}`
- serializer `ValidationError`: 400 field→`["msg"]` map (bare
  `{"detail":…}` for non-field errors) — check each endpoint's
  Django tests for exactness (`annotations/tests.py`,
  `bible/tests/`).
- hand-rolled 400s: `{"error": "<msg>"}` (bible views) or
  `{"error": "validation_error", "message": "<msg>"}` (notes
  reorder); reorder 409 adds `"error": "position_conflict"` +
  `"constraint": "unique_tag_position_per_tag"`.

### 4.5 Pagination

- Notes list (`annotations/pagination.py`): DRF
  `PageNumberPagination`; params `page` (1), `page_size` (25,
  max 100). Body `{"count", "next"|null, "previous"|null,
  "results":[…]}`; `next`/`previous` are absolute URLs built from
  scheme + Host (honor `X-Forwarded-Proto`; prod is https).
- Bible search: provider-shaped `{"data":{"verses":[…],
  "meta":{"pagination":{"total","count","per_page",
  "current_page","total_pages"}}}}`; `group_by=book` instead returns
  `{"data":{"groups":[{book_id,count,verses}],"meta":{"total",
  "truncated"}}}`.
- `linked`/`counts`/`bulk`/image lists/comment trees: custom shapes
  — per-endpoint plans.

### 4.6 Config (replaces settings.py + config.yaml + gcp.py)

`core::config::Config` via the `config` crate, layered: optional
`config.yaml` (dev) → env vars (accept legacy names verbatim for
terraform parity). Secrets (`DATABASE_URL`, `DBT_KEY`, `ESV_KEY`)
arrive as env vars via Cloud Run `secret_key_ref` — DROP the
runtime Secret Manager client; one terraform edit needed.

Fields: `DATABASE_URL`, `SECRET_KEY` (kept, unused), `DBT_KEY`,
`ESV_KEY`, `DEBUG`, `GOOGLE_CLOUD_PROJECT`,
`GOOGLE_TTS_LANGUAGE_CODE` (`lv-LV`), `GOOGLE_TTS_VOICE_NAME`
(`lv-LV-Chirp3-HD-Sadachbia`), `GOOGLE_TTS_SAMPLE_RATE_HERTZ`
(24000), `AUDIO_BUCKET_NAME` (`{project}-bible-audio`),
`AUDIO_SIGNED_URL_TTL_SECONDS` (3600), `MONTHLY_TTS_CHAR_LIMIT`
(100000), `LOCK_STALE_HOURS` (24), `IMAGE_BUCKET_ORIGINALS` /
`IMAGE_BUCKET_THUMBNAILS` (`{project}-images-originals` /
`-thumbnails`), `IMAGE_MAX_BYTES` (10 MiB),
`IMAGE_SIGNED_URL_TTL_SECONDS` (600), `IMAGE_MAX_PER_NOTE` (10),
`IMAGE_MAX_PER_COMMENT` (5), plus NEW `AUTO_PROVISION_USERS`
(prod true, tests false — §4.2).

### 4.7 Provider routing (trait replacing translation_service)

```rust
#[async_trait]
pub trait PassageProvider: Send + Sync {
    async fn chapter_text(&self, book: &str, chapter: i32,
        fileset_id: &str) -> Result<TextChapter, ProviderError>;
    async fn chapter_audio(&self, book: &str, chapter: i32,
        fileset_id: &str) -> Result<AudioChapter, ProviderError>;
    async fn timestamps(&self, fileset_id: &str, book: &str,
        chapter: i32) -> Result<Vec<Timestamp>, ProviderError>;
    async fn copyright(&self, bible_id: &str)
        -> Result<Vec<Copyright>, ProviderError>;
    async fn search(&self, q: &SearchQuery)
        -> Result<SearchPage, ProviderError>;
    async fn search_all(&self, q: &SearchQuery)
        -> Result<SearchAll, ProviderError>; // group_by=book
    fn listing(&self, language_iso: Option<&str>)
        -> Vec<TranslationListing>;
}
```

Routing order (preserve): `is_esv_fileset` → ESV;
`canonical_sword_fileset_id` → SWORD (its audio arm consults GCS);
else DBT catch-all — note `ENGESV` (no `_API` suffix) is a DBT id.
`AppState.providers` resolves `fileset_id → ProviderKind` and hands
out `Arc`s.

Parity details:

- `TranslationListView` SWALLOWS DBT errors (logs, `processed=[]`)
  then still appends SWORD+ESV registry entries — DBT outage ⇒ 200
  with partial data, not 502.
- DBT "audio" detection: `data[0]` has key `path`.
- DBT empty `data` → `not_found`, not 200.
- ESV text: `include-headings=true`; parse `[N]` verse markers +
  indented-block heuristics (`_parse_passage`) — port verbatim with
  the `regex` crate.
- ESV audio: capture the `Location` header of the 3xx from
  `/v3/passage/audio/` (`redirect::Policy::none()`), return it.
- SWORD audio = GCS artifacts from the job:
  `audio/{fileset}/{voice}/{BOOK}/{CH}_audio.mp3` +
  `_timestamps.json`; read path = exists check → timestamps JSON →
  signed URL; `file_size_bytes` embedded in JSON (legacy-blob
  fallback HEAD in the Django code — decide in passage plan).
- Note verse enrichment: ONE provider call per note — DBT
  `verse_start..verse_end` on first verse's `dbt_book_id`+chapter,
  or ESV chapter+headings filtered to the note's verse numbers;
  `include_verse_text=False` (notes/linked) skips it. This is N
  upstream calls per list page — keep parity; caching later.

### 4.8 fileset_id registry + in-flight branches

Static maps replacing `esv/registry.py` + `sword/registry.py`:

- ESV: `ENGESV_API` → `{name, abbr:"ESV", language:"English",
  iso:"eng"}`, listing filesets `[{id,"text_plain","C"},
  {id,"audio","C"}]`.
- SWORD: `LVSGLU8` → `{module:"LvGluck8", zip:"LvGluck8.zip",
  abbr:"GLU8", iso:"lvs", audio_fileset_id:"LVSGLU8C1DA"}`;
  `canonical_sword_fileset_id` accepts canonical id, abbr, or audio
  id, case-insensitive.
- `get_tts_config` → per-canonical-id TTS config from `Config`
  (only `LVSGLU8` today).
- DBT = default; `bible_id`/`fileset_id` passed through verbatim.

IN-FLIGHT WORK — check before writing plans: unmerged branch
`feat/fileset-groups` adds `bible/services/fileset_groups.py`
(normalized `text_options`/`audio_options` with `by_testament`,
`coverage_label`, `partial` on the translations response) plus a
fourth `error_code` value `book_not_in_fileset` (404, with an
`alternates` chain the frontend walks). `feat/apibible-niv` adds an
API.Bible provider (`ENGNIV_API`, `ENGNIVC1DA`; its `apibible/`
source is absent on main — only `__pycache__` remains). If either
has merged by the time the translations/passage plans are written,
that agent must incorporate them.

### 4.9 GCS + signed URLs (storage crate)

Primary: official `google-cloud-storage` + `google-cloud-auth`
`Signer` for V4 signed URLs via IAM `signBlob` using the runtime
service account — exactly what `generate_signed_url(
service_account_email=…, access_token=…)` does today. Fallback:
hand-rolled GCS JSON API via reqwest + `signBlob` REST (~10 call
sites total).

Port verbatim:

- `chapter_object_paths`: `audio/{fileset}/{voice}/{BOOK}/{CH}`
  `_audio.mp3` / `_timestamps.json`.
- Signed-URL caches: audio keyed `(fileset, voice, book, chapter)`,
  expiry = TTL − 60 s margin; images LRU cap 2048 keyed by
  `storage_url`.
- State objects `state/{usage|lock}/{fileset}/{YYYY-MM}.json`,
  `ifGenerationMatch` read-modify-write, exponential backoff
  (0.1 s base, 8 retries → `UsageIncrementConflict`), run-lock
  reasons `acquired`/`stale_overridden`/`already_completed`/
  `active_run`, idempotent `mark_run_complete`; naive-UTC
  timestamps inside the JSON.
- `upload_chapter_artifacts` order: JSON first, then MP3.
- Images: `originals/{image_id}/source{ext}`; allowlist
  jpeg→.jpg / png→.png / webp→.webp / gif→.gif; else 415
  `UnsupportedMediaType` (`{"detail": …}`); >`IMAGE_MAX_BYTES` →
  413 `{"detail": "File exceeds maximum allowed size."}`;
  validate magic bytes by `image`-crate decode (Pillow `verify()`
  equivalent); delete = best-effort (swallow errors).

### 4.10 SWORD modules

`bible/sword_modules/LvGluck8.zip` is a standard rawzip module:
`mods.d/lvgluck8.conf` + `modules/texts/ztext/lvgluck8/{ot,nt}.
{bzs,bzv,bzz}` (zlib-compressed). pysword lazily reads
verse→block (`.bzv` index) → decompress block (`.bzz`) → slice.

Rust plan (behind `PassageProvider`, evaluate in order):

1. `rsword_chirho` 0.3 — pure-Rust SWORD reader incl. zText. It
   loads from a module DIRECTORY, so extract the zip at startup
   (`zip` crate → temp dir) or ship it extracted in the image.
   VERIFY output vs pysword for every LvGluck8 book before
   adopting (versification + `clean` markup stripping must match).
2. Fallback: port pysword's ztext reader (~400 lines: zip central
   directory, `.conf` parse, testament index, `flate2` decompress,
   verse slice, OSIS/GBF markup strip) inside `providers/sword.rs`.
3. Do NOT shell out to `diatheke`/libsword — native dep +
   subprocess-per-verse is worse than either option.

Required surface: `get_chapter_verses` (iterate verses 1.., stop
on empty/miss, hard cap 200), `list_chapters` from the module's
OWN versification (never assume KJV counts),
`get_translation_listing`. The client is stateful — `OnceCell` +
`Mutex` inside `Arc`, mirroring `sword/client.py`'s singleton+lock.

### 4.11 TTS job (binary `audio_job`)

Port `bible/management/commands/generate_chapter_audio.py`:
worklist = `sword.list_chapters`; `acquire_run_lock`; read monthly
usage → `CharBudget`; per chapter: per-verse `synthesize_text`
(reqwest POST `…/v1/text:synthesize` + OAuth token from
`google-cloud-auth`; the gRPC crate is optional — REST is one call),
MP3 duration via `lofty` for timestamps, budget consumed per verse,
persist `increment_monthly_usage` in a finally-equivalent, upload
artifacts, `mark_run_complete`. HTTP 429/quota → `QuotaExceeded` →
stop run cleanly. Deploys as the existing `audio-generator` Cloud
Run job with the `audio_job` binary as its command. The Cloud
Scheduler trigger stays DISABLED — `terraform/scheduler.tf` is
intentionally commented out; do not enable it.

### 4.12 Django admin, static, misc

- **Django admin: DROP in v1** (`annotations/admin.py` is a debug /
  audit convenience only). Keep the legacy Django deploy reachable
  for admin during the transition — it shares the same Postgres —
  then use SQL or a tiny admin tool if operators actually miss it.
- `STATIC_*`/`MEDIA_*` only served admin + browsable-API assets —
  nothing to port.
- CORS `CORS_ALLOW_ALL_ORIGINS=True` → `CorsLayer::permissive()`;
  the Vercel-preview regex is moot under allow-all.
- `print()` calls in middleware → `tracing::debug!`.
- `bible/services/apibible/` is dead code on main — do not port
  (but see §4.8: it lives on `feat/apibible-niv`).

### 4.13 OpenAPI

`#[utoipa::path]` per handler + `ToSchema` DTOs, assembled in
`server::openapi`. Serve `GET /api/v1/schema/` (JSON),
`/api/v1/docs/` (utoipa-swagger-ui), `/api/v1/redoc/`
(utoipa-redoc) — all public (`AllowAny` parity), plus the `Token`/
`Bearer` security scheme. Schema parity is best-effort: the React
app consumes endpoints, not the schema; keep summaries and
parameter docs reasonably faithful.

### 4.14 Logging / tracing

`tracing` + `tracing-subscriber` fmt; `TraceLayer` request spans;
JSON encoder behind `LOG_JSON` for Cloud Run parsing. Replace
`logger.*`/`print` with `info!/debug!` on module targets — the
upstream-call logging is this app's only observability, keep it
generous.

### 4.15 Testing

- Unit beside each module (books parsing, ESV `_parse_passage`,
  error mapping, PK gen, username derivation).
- HTTP-level: `axum-test` (or `tower::ServiceExt::oneshot`) on the
  router against scratch Postgres seeded from `schema.sql`; seed
  users/tokens in SQL. `AUTO_PROVISION_USERS=false` reproduces the
  conftest anonymous world.
- Providers: `wiremock` HTTP mocks for DBT/ESV/TTS; fixture SWORD
  module or `FakeProvider` impl otherwise.
- Parity: `tests/parity/` — a request corpus replayed against
  `DJANGO_BASE_URL` and `RUST_BASE_URL`, diffed after normalizing
  volatile fields (`created_at`, `*_url`, `next`/`previous` hosts,
  `date_joined`).

## 5. Migration / rewrite strategy

1. **Phase 0 (base agent)**: workspace skeleton, config, pool,
   `Principal`/`AuthUser` extractors, error contract, ids, books,
   router skeleton, `/healthz`, OpenAPI plumbing.
2. **Phase 1 (no providers)**: `users/register`, `users/me`,
   `/api/token/`, `tags` CRUD, `reading-positions`. Validates
   auth + DB + serialization end to end.
3. **Phase 2 (annotations DB)**: notes CRUD + `linked` + `bulk` +
   `reorder`; comments tree + counts. Verse enrichment can stub
   the provider until Phase 3's `dbt.rs` lands.
4. **Phase 3 (providers)**: `dbt.rs` → passage text/audio,
   copyright, timestamps; then `esv.rs`; then `sword.rs`; then the
   GCS audio path; search + grouped search last (all providers).
5. **Phase 4**: images (multipart, GCS, signed URLs, caps).
6. **Phase 5**: `audio_job`, docs endpoints, parity harness, Cloud
   Run deploy config.

Side-by-side: Django stays on App Engine; Rust deploys to a Cloud
Run staging service on the SAME Postgres (data-compatible by
construction) with isolated GCS env where safe. Run the parity
corpus per endpoint; flip the frontend base URL only when green.

Verification rule: every endpoint plan lists the Django tests
covering it and adds a parity-corpus entry. Pre-existing Django
test failures (`test_note_serializer`,
`test_single_query_assertion`, `test_position_already_occupied`)
are NOT blockers — match implemented behavior, flag mismatches.

## 6. Agent onboarding guide

You are rewriting the Django `bible_research` backend to Rust. Read
this file (`00-base.md`) first, then your endpoint's Django code
(§7). The Rust base lives in a NEW repo `bible_research_rs/`
(workspace layout §3). Do NOT modify the Django repo.

Conventions:

- **Files**: one module per endpoint group under
  `crates/server/src/routes/{bible,annot,users}/`. DTOs live beside
  their handler (`XxxRequest`/`XxxResponse`, shared `*Dto` structs);
  shared helpers in `core`.
- **Naming**: serde fields are already snake_case — no
  `rename_all`. Keep Django's exact key spelling (`fileset_id`,
  `error_code`, `tag_position`). Float fields = `f64`.
- **Handler shape**:
  `async fn list_notes(State(s): State<AppState>, p: Principal,
  Query(q): Query<NotesQuery>) -> ApiResult<Json<Value>>`.
  `ApiResult<T> = Result<T, ApiError>`; `ApiError` maps to §4.4
  bodies (`Validation(fields)`, `NotFound`, `Forbidden(detail)`,
  `Provider(ProviderError)`). Never `unwrap` in handlers.
- **Auth**: `Principal` for AllowAny-ish endpoints, `AuthUser`
  (rejects Anonymous with the DRF 401 body) for `IsAuthenticated`.
  Owner checks are manual (`row.user_id == user.id`), mirroring
  `perform_update`/`perform_destroy`.
- **DB**: `sqlx::query_as::<_, RowStruct>(…)` with explicit column
  lists; `state.pool.begin()` for transactions. Match Django's
  queries 1:1 — including the odd ones; they ARE the contract.
  `select_related`/`prefetch_related` → JOIN or a second query with
  the same count discipline (comment tree = 1 query + in-memory
  assembly).
- **Errors**: provider failures only via `ProviderError` +
  `error_fields()` (§4.4). Validation → field-map or `{"error":
  …}` exactly as the Django view emits. Re-read §4.4 first.
- **Routes**: register in `crates/server/src/routes/mod.rs`
  (`Router::new().route("/notes/{id}", get(..).put(..)…)` nested
  under `/api/v1`) AND add `#[utoipa::path]` + the `openapi.rs`
  entry.
- **Tests**: mirror the Django test file for your endpoint
  (`annotations/tests.py` classes ↔ `tests/annot_*.rs`); add one
  parity-corpus entry. Use `AUTO_PROVISION_USERS=false` + explicit
  tokens/guest fixture like conftest.
- **Line length**: the 79-char rule is for Python/markdown files;
  Rust uses rustfmt defaults.

Steps per endpoint:

1. Read the Django view + serializer + model + tests (paths §7).
2. Write `endpoints/NN-<name>.md`: request/response shapes, auth
   rule, DB queries, provider calls, EVERY edge case (defaults,
   400 bodies, ordering, 404 vs 403 vs 409).
3. Implement DTOs + handler + SQL; register route + OpenAPI.
4. Tests: unit for pure logic, axum-test for the handler, parity
   entry.
5. Report deliberate contract deviations in the plan.

## 7. Endpoint inventory

Legend — Auth: `tok` = Authorization token else provisioned/anon;
`auth!` = additionally IsAuthenticated/owner-checked. Deps:
`pg`=Postgres, `dbt`/`esv`/`sw`=providers, `gcs`=storage.
Sz: S/M/L. All paths under `/api/v1/` except #19.

```
#  Methods            Path                              View/impl
1  GET                bible/                            BiblePassageView
   auth:tok  sz:M  deps:dbt,esv,sw,gcs
2  GET                bible/translations/               TranslationListView
   auth:tok  sz:M  deps:dbt + registries
3  GET                bible/timestamps/                 AudioTimestampView
   auth:tok  sz:S  deps:dbt,gcs
4  GET                bible/copyright/                  CopyrightView
   auth:tok  sz:S  deps:dbt
5  GET                bible/search/                     BibleSearchView
   auth:tok  sz:L  deps:dbt,esv,sw
6  GET,POST           tags/                             TagViewSet
   GET,PUT,PATCH,DELETE tags/{id}
   auth:tok  sz:S  deps:pg
7  GET,POST           notes/                            NoteViewSet
   GET,PUT,PATCH,DELETE notes/{id}
   auth:tok  sz:L  deps:pg,dbt,esv (verse enrichment)
8  POST               notes/reorder/                    NoteViewSet.reorder
   auth:auth!  sz:M  deps:pg (txn, 409 contract)
9  POST               notes/bulk/                       NoteViewSet
   .bulk_create_notes  auth:tok  sz:M  deps:pg (txn)
10 POST               notes/linked/                     NoteViewSet.linked
   auth:auth!  sz:M  deps:pg
11 GET,POST           reading-positions/                ReadingPositionViewSet
   GET,PUT,PATCH,DELETE reading-positions/{id}
   POST               reading-positions/bulk/
   auth:auth!  sz:M  deps:pg (upsert)
12 GET,POST           notes/{note_pk}/comments/         CommentViewSet
   GET,PATCH,DELETE   notes/{note_pk}/comments/{pk}/    (NO PUT)
   auth:tok (write auth!)  sz:M  deps:pg (tree build)
13 GET,POST           notes/{note_pk}/images/           NoteImageViewSet
   auth:read=tok, write=auth!+owner  sz:M  deps:pg,gcs
14 GET,POST           notes/{note_pk}/comments/         CommentImageViewSet
                      {comment_pk}/images/
   auth:read=tok, write=auth!+author  sz:M  deps:pg,gcs
15 GET                comments/counts/                  CommentCountView
   auth:tok  sz:S  deps:pg
16 DELETE             images/{image_pk}/                ImageDestroyView
   auth:auth!  sz:S  deps:pg,gcs
17 POST               users/register/                   register_user
   auth:any  sz:S  deps:pg (token issue)
18 GET                users/me/                         current_user
   auth:auth!  sz:S  deps:pg
19 POST               /api/token/                       obtain_auth_token
   auth:any  sz:S  deps:pg (pwd verify)
20 GET                schema/ docs/ redoc/ ; GET /      spectacular +
   auth:any  sz:S  deps:-                                RedirectView
21 -                  admin/ static/ favicon            DROPPED (§4.12)
```

Suggested agent split (≈11 plans): users+token [17,18,19]; tags
[6]; notes CRUD [7]; note actions [8,9,10]; reading-positions [11];
comments+counts [12,15]; images [13,14,16]; passage [1];
translations+timestamps+copyright [2,3,4]; search [5]; docs +
base plumbing [20] (part of the base agent's Phase 0).
