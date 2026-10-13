# Rust rewrite research — bible_research → Rust

Status: PLANNING COMPLETE. This directory contains the full analysis
for rewriting the Django 5.2 + DRF backend (`bible_research/`) in
Rust. Nothing here is implementation; each file is a plan an agent
(or human) can execute from.

## Reading order

1. `00-base.md` — READ FIRST. Stack decision, Cargo workspace layout,
   cross-cutting design (auth extractor, error contract, provider
   trait, GCS, SWORD, config, pagination), rewrite strategy, endpoint
   inventory, and §6 "Agent onboarding" — the conventions every
   implementer must follow.
2. `endpoints/*.md` — per-endpoint plans. Each is self-contained
   given `00-base.md` + the cited Django code.
3. This file's "Cross-plan issues" section — corrections and shared
   needs the endpoint analyses surfaced AFTER the base doc was
   written. Implementers of `core` should apply these to the base
   design.

## Stack (decided in 00-base.md)

axum 0.8 · sqlx 0.8 (PgPool, runtime `query_as`) · serde · reqwest ·
utoipa + utoipa-swagger-ui/redoc · google-cloud-storage /
google-cloud-auth (IAM signBlob signed URLs) · djangohashers ·
difflib (book-name fuzzy parity) · rsword_chirho or a pysword port ·
Deploy: Cloud Run service + Cloud Run job (one Docker image).
Django admin: dropped in v1.

## Plan index

| File                       | Covers                                   | Sz |
|----------------------------|------------------------------------------|----|
| endpoints/01-users-auth.md | register, me, /api/token/                | S  |
| endpoints/02-tags.md       | TagViewSet CRUD                          | S  |
| endpoints/03-notes.md      | NoteViewSet CRUD + verse enrichment      | L  |
| endpoints/04-note-actions  | reorder, bulk, linked                   | M  |
| endpoints/05-reading-pos   | positions CRUD + bulk upsert            | M  |
| endpoints/06-comments.md   | comment tree + comments/counts          | M  |
| endpoints/07-images.md     | note/comment images, image delete (GCS) | M  |
| endpoints/08-bible-passage | passage text/audio, provider routing    | M  |
| endpoints/09-bible-discov  | translations, timestamps, copyright     | M  |
| endpoints/10-bible-search  | search + group_by=book                   | L  |

Schema/docs/redoc surfaces are covered by `00-base.md` (utoipa,
Phase 0). Django `admin/` is intentionally not ported.

Suggested implementation order (base §5): Phase 0 = core + plumbing
(auth extractor, errors, config, routing skeleton, OpenAPI) →
users-auth → tags → notes + note-actions → reading-positions →
comments → images → bible-passage → bible-discovery → bible-search
→ audio_job binary.

## Cross-plan issues (apply to 00-base.md design)

These were found by endpoint agents after the base doc landed;
treat as errata for `core` implementation:

- **`ApiError::ValidationList`** — comments/counts, note-actions,
  and notes all emit bare-array 400 bodies (`["msg"]`). Add a
  `ValidationList(Vec<String>)` variant to `core::errors`
  (requested by 03, 04, 06).
- **`PassageProvider::chapter_text` signature** — needs
  `verse_start`/`verse_end` params; the DBT arm forwards them
  upstream and they affect the empty-data → not_found contract
  (03-notes).
- **PBKDF2 iterations = 1,200,000** — deployed Django is 6.1.1
  (requirements.txt), not the stale venv's 4.2.6; confirm
  `djangohashers` lets us set iteration count (01-users-auth).
- **DRF datetime format** — `SecondsFormat::AutoSi`, not `Micros`;
  add shared `core::fmt::drf_dt()` helper (01).
- **413 body correction to §4.9** — actual message is
  `"File too large ({n} bytes). Maximum allowed: {m} bytes."` (07).
- **405 fallback** — comments exclude PUT; image detail routes 405
  every method. Add `method_not_allowed_fallback` emitting the DRF
  405 body (06, 07).
- **Trailing-slash policy** — DRF APPEND_SLASH redirects vs axum's
  strict matching: decide once, router-wide (05).
- **`ImageDto`/`signed_image_url` are shared** between comments and
  images plans; `signed_url: null` on signing failure is legal
  (stub signer is viable early) (06, 07).
- **`preserve_order`/indexmap** needed for reading-positions bulk
  response keyed in request order (05).
- **Anon-write status codes** — prod Django gives 401, test env
  gives 403 (SessionAuthentication has no authenticate_header);
  Rust `AuthUser` returns 401 always — deliberate deviation (07).

## Latent Django bugs found (do NOT port blindly)

Documented in the plans; flag for user decision:

- **`feat/fileset-groups` branch deletes `provider_errors.py`** —
  regresses the error contract: `{"error": str(e)}` reintroduces
  the `?key=` credential leak, and its generic fallback returns
  empty 200s on rate limits. Implement main's contract + only the
  branch's additive fields (08, 09).
- **Anonymous notes list exposure** — anon `GET /notes/` without
  `?public=true` returns ALL private notes (inverted-logic bug;
  unreachable in prod due to provisioning middleware) (03).
- **Comments have no note-accessibility check** — any authenticated
  user can read/comment on any private note (06).
- **Tag DELETE cascades to all notes under the tag** — destructive;
  confirm the frontend expects this (02).
- **Orphan note on failed verse resolution** — note persists after
  a 400 on `verse_references` (no transaction) (03).
- **`test_position_already_occupied` expectedFailure** — code emits
  `position_conflict`, test asserts `conflict`; possible contract
  bug (04).

## For implementation agents

Start at `00-base.md` §6. Each endpoint plan follows the same
5-section shape: contract → Django trace → Rust plan → tests →
risks. New Rust code goes in a new `bible_research_rs/` repo —
never modify the Django repo. 79-char lines in markdown;
rustfmt defaults in Rust.
