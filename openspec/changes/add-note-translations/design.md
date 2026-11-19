# Design

Distilled from
`docs/plans/note-translations-investigation-and-plan.md`
(investigation + confirmed decisions), split so this change
covers TEXT translation only; note TTS audio is the follow-up
change `add-note-tts`.

## Context

See proposal.md — Why. Key facts from investigation:

- `Note` (`annotations/models.py`) stores `note_text` as a
  serialized TipTap doc JSON string (legacy plain text is
  possible). No language field exists anywhere. `Comment`,
  `Image`, `NoteVerse` are all FK children of `Note` — a child
  table is the established shape.
- `NoteSerializer.create()` is a plain `Note.objects.create` +
  `NoteVerse.objects.bulk_create`; `to_representation()`
  enriches verses per provider. `NoteViewSet` has standard CRUD
  + `bulk` + `reorder`; `get_accessible_notes_qs` is the shared
  own ∪ public visibility scope.
- Frontend `note_text` is rendered via `RichTextView`;
  `utils/tiptapContent.ts` has `isTiptapDocString` and
  `toPlainText`. `NoteCard` already carries action icons; the
  reader's UI language lives in the persisted `uiLanguage`
  store setting.
- Infra: App Engine standard F1 (min=max 1 instance), project
  `bible-research-489314`, runtime appspot SA. Enabled APIs
  include `texttospeech`, `storage`, `secretmanager`, `run`,
  `cloudscheduler` — NOT `translate` or `cloudtasks`. Settings
  follow the `_audio_settings_source` env/config.yaml pattern
  used by `GOOGLE_TTS_*`; GCS usage counters
  (`increment_monthly_usage`) already exist as a pattern.
- Cost: Cloud Translation ~$20/1M chars with a 500k chars/month
  free tier (~1,000 typical notes free); Cloud Tasks free tier
  is 1M ops/month.

## Goals / Non-Goals

**Goals:**

- Automatic per-note language detection and machine
  translation into a configurable target set (en + lv first,
  language-agnostic design).
- Note text served in the reader's language on detail AND
  list endpoints via `?lang=`, with per-note fallback.
- Translation lifecycle visible to clients (`pending` /
  `ready` / `failed` / `source`) with idempotent retries.
- Author verification as a per-language trust signal.
- Reader-facing switcher with sensible defaults, "Show
  original" peek, and honest failure states.

**Non-Goals:**

- TTS audio for notes — separate change `add-note-tts`,
  which depends on this one.
- Translating `Comment` rows — the service is designed
  language-agnostic so comments can reuse it later.
- Translating verse text — verses follow the selected Bible
  fileset and are legally fraught (copyrighted translations).
- A per-note "written in" picker — auto-detection is free and
  reliable for paragraph-length notes; may come later.
- Backfilling existing notes by default — optional
  `translate_notes` management command, run only if wanted.

## Decisions

### D1: `NoteTranslation` child table, not a JSONField on Note

`NoteTranslation(id 'NTR'-prefixed CharField PK, note FK
related_name='translations' CASCADE,
language, text, status, source_hash, engine, error,
author_verified, verified_at, created_at, updated_at)` with
`UniqueConstraint(note, language)` — its btree already covers
the hot `note_id + language` lookups (detail `?lang=` get +
list prefetch) — plus a separate `Index(language, status)`
for single-language scans (readers typically request one
language; also serves backfill/diagnostics filters on
`status`), plus
`Note.source_language`. Each note carries at most 2–3 short
rows; every read is one indexed FK lookup (or a
`prefetch_related`) — negligible next to the provider call
`to_representation()` already makes. A JSONField would lose
per-language status/retry tracking, indexing, and partial
updates (a failed `ru` retry would rewrite the whole doc).
Separate `Note` rows per language were rejected too: they
pollute `unique_tag_position_per_tag`, duplicate verse/comment
links, and break "edit once".

### D2: `status='source'` row stores the original text

The detected source language gets a row holding the original
doc with `status='source'`, so every language a reader can
pick has a uniform row — no special-casing in the serializer
or the frontend switcher.

### D3: Eager pipeline via Cloud Tasks + OIDC worker

`perform_create`/`perform_update` (only when `note_text`
changed) upsert `pending` rows inside `transaction.on_commit`
and enqueue `POST {APP_BASE_URL}/api/v1/tasks/translate-note/`
with body `{note_id, source_hash}` and an OIDC token minted
for the `tasks-invoker` SA. The worker endpoint verifies the
`Authorization: Bearer` OIDC token
(`google.oauth2.id_token.verify_oauth2_token`) and checks the
token email against `TASKS_INVOKER_SA_EMAIL`; Cloud Tasks
retries non-2xx (max 5, exponential backoff,
`max_concurrent_dispatches=5` protects the single F1 instance
and Translate quotas). Synchronous translation in the request
was rejected: ~5–8 s per note on one F1 instance.

Local dev (`not IS_GCP_ENVIRONMENT`): the enqueue helper calls
the worker function inline — zero-infra fallback, no emulator.

### D4: `source_hash` idempotency

The task body carries `sha256(note_text)`; the worker compares
it to the current hash and 204-no-ops on mismatch — an
edit-during-flight leaves the newest text's run authoritative
(last writer wins).

### D5: Structural TipDoc translation, one API call

`note_text` is TipTap JSON, not HTML. The worker walks the doc,
collects `text` values of `type:'text'` nodes, sends them in
ONE `translate_text` call (`contents=[…]` batch), and splices
results back in order — marks and non-text nodes
(`scriptureRef`/`mention` inline atoms) survive untouched.
`@USFM.C[.V[-E]]` tokens stored as PLAIN text node content
(matched via the frontend's `SCRIPTURE_TOKEN_REGEX` so
hand-typed colon variants are covered too) are masked to
do-not-translate placeholders before the call and restored on
splice-back — the API would otherwise mangle them and break
`scriptureLinkify` matching in the translated doc.
Non-doc strings fall back to a single plain-text call. An HTML
round-trip was rejected: it needs a JSON↔HTML converter and
risks schema drift.

### D6: Detection via the same `translate_text` call

Calling `translate_text` without `source_language_code` returns
`detected_language_code`, so detection costs nothing extra; the
result is stored on `Note.source_language`. Detection is
per-note because the corpus is mixed — it MUST NOT be inferred
from user or UI locale.

### D7: `engine` provenance on every row

Each row stamps the engine actually used (`google-nmt` for the
v3 `general/nmt` MVP; `google-llm` or `gemini-<model>` later).
`NOTE_TRANSLATION_ENGINE` is a config flip — an upgrade then
backfills only rows built by the old engine instead of
retranslating everything, and the UI can badge quality tiers.

### D8: `?lang=` on detail AND list with per-note fallback

`GET /api/v1/notes/<id>/?lang=` and `GET /api/v1/notes/?lang=`
both swap
`note_text` when a `ready`/`source` row exists — the list swap
is a page-wide `translations` prefetch (one indexed query) so
the notes view renders uniformly in one request. A note lacking
the row emits `note_text` unchanged with metadata intact —
never a 404, because 404ing the whole note over a missing
translation would hide real content.

### D9: Owner-only verify endpoint, never note PATCH

`POST|DELETE /api/v1/notes/<id>/translations/<lang>/verify/`
sets or clears `author_verified`/`verified_at` on the child
row. Non-owners get 404 (same as private-note access — the
endpoint leaks nothing); `pending`/`failed`/`source` rows get
409 `translation_not_ready` in the `error`/`error_code` shape
(a `source` row is the author's own text — nothing to verify).
Verification never rides on the note PATCH: it is a
per-language action on a child row, and a note PATCH would
wrongly interact with `source_hash` flows. Every fresh
translation write resets `author_verified=False,
verified_at=NULL` — new machine output is unreviewed by
definition. The companion owner-only
`POST /api/v1/notes/<id>/retranslate/` re-enqueues `failed`
entries (non-owner → 404, other states untouched) so a retry
is one cheap call, not an edit dance.

### D10: Frontend defaults and peek semantics

The reader's preferred note language is a persisted Zustand
slice seeded from `uiLanguage`, with `source_language`
fallback when no `ready`/`source` row exists — a mixed corpus
then renders uniformly (card 1 its `source` LV row, card 2 an
`en→lv` translation, identically rendered). "Show original"
is a PEEK: it swaps the rendered body to the already-fetched
original `note_text` with zero extra fetch, and toggles back —
the `SegmentedControl` selection, stored preference, and
audio/verify targets stay on the translation.

### D11: Cost guardrails

Per-note char cap (~10k → mark `failed`/`too_long`), target
list and engine from settings/env, monthly Translate usage
counter in GCS reusing `increment_monthly_usage`
(`fileset_id='note-translate'`), queue concurrency cap.

## Risks / Trade-offs

- [TipTap schema drift: translated docs must round-trip
  `RichTextView`] → serializer/worker test with a real doc
  fixture incl. `scriptureRef`/`mention` atom nodes.
- [MT quality for theological Latvian is imperfect] →
  "Auto-translated" label + "Show original" everywhere +
  optional author verification tier.
- [OIDC verification is the auth boundary for the task
  endpoint — a bug opens an unauthenticated write path] →
  explicit 401 tests; endpoint accepts nothing else.
- [Race: edit while a task is in-flight] → `source_hash` check
  makes it a safe no-op; last writer wins.
- [Queue/APIs don't exist locally] → inline worker fallback
  keeps dev zero-infra; the same code path runs the worker.
- [Hand-written migration drift] → mirror prior migration
  style; user verifies `makemigrations annotations` reports no
  changes.

## Migration Plan

1. Terraform: enable `translate.googleapis.com` +
   `cloudtasks.googleapis.com`, create the queue,
   `tasks-invoker` SA + `serviceAccountTokenCreator`, grant the
   appspot SA `cloudtasks.enqueuer` + `cloudtranslate.user`
   (user runs `terraform apply`).
2. Merge backend PR (model + hand-written migration +
   pipeline + endpoints); user runs
   `python manage.py migrate` and confirms
   `makemigrations annotations` produces nothing new.
3. Merge frontend PR — it only reads the new fields; old
   payloads without `translations`/`source_language` degrade to
   source-text display.
4. Set `NOTE_TARGET_LANGUAGES` in prod config; optionally run
   the `translate_notes` management command (`--user`,
   `--all`) to backfill old notes — default is notes created
   after launch only.

Rollback: revert PRs; the new table/columns are additive and
unused rows are harmless. Disable by emptying
`NOTE_TARGET_LANGUAGES`.

## Open Questions

- `NOTE_TRANSLATION_ENGINE` default is `google-nmt`; whether
  the v3 `translation_llm` model covers en↔lv at acceptable
  latency is a deploy-time config decision, not a code one.
