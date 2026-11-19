# Proposal

## Why

The note corpus is mixed-language: some users write in English,
others in Latvian, some in both. A `public` note is only useful
to readers who happen to read its source language, so notes
cannot be shared across the EN/LV audience. There is no
machine-translation pipeline, no per-note language metadata, and
no way for a reader to view a note in their own language.

## What Changes

- `Note` gains `source_language` — a per-note detected ISO 639-1
  code. Detection runs per note on the note's own text; it is
  never inferred from the user profile or the UI locale.
- New `NoteTranslation` model: `note` FK (related_name
  `translations`, CASCADE), `language`, `text` (translated TipTap
  doc JSON), `status` (`pending` | `ready` | `failed` | `source`),
  `source_hash` (sha256 of `note_text` — makes the async task
  idempotent), `engine` (provenance: `google-nmt` / `google-llm` /
  `gemini-…`), `author_verified` + `verified_at` (reset on every
  fresh translation write), `error`, timestamps, and a
  `UniqueConstraint(note, language)`. Audio fields are explicitly
  out of scope — they land in the `add-note-tts` change.
- Eager translation pipeline: `perform_create`/`perform_update`
  (only when `note_text` changed) → `transaction.on_commit` →
  Cloud Tasks enqueue → OIDC-verified worker endpoint → Cloud
  Translation `translate_text` with structural TipDoc translation
  (all `type:'text'` node values batched in ONE call, spliced
  back, marks survive; plain-text fallback for non-doc strings)
  → upserted rows. Language detection reuses `translate_text`
  without `source_language_code`. Local dev runs the worker
  inline — zero-infra fallback. Cloud Tasks retries max 5 with
  exponential backoff.
- API contract: the note serializer gains `source_language` and
  a `translations` metadata list (`language`, `status`,
  `verified`, `engine`). `?lang=` swaps `note_text` on BOTH the
  detail endpoint (`GET /api/v1/notes/<id>/?lang=`) and the LIST
  endpoint (`GET /api/v1/notes/?lang=` — page-wide prefetch
  swap). Notes lacking a `ready`/`source` row emit `note_text`
  unchanged — never a 404.
- Owner-only verification: `POST|DELETE
  /api/v1/notes/<id>/translations/<lang>/verify/` — 404 for
  non-owners and missing rows, 409 `translation_not_ready` on
  `pending`/`failed`/`source` rows (same `error`/`error_code`
  shape as the error-contract spec). Owner-only
  `POST /api/v1/notes/<id>/retranslate/` re-runs `failed`
  entries (non-owner → 404).
- Frontend: a `SegmentedControl` language switcher on `NoteCard`;
  the default is the reader's persisted preferred note language
  (seeded from `uiLanguage`) with `source_language` fallback; a
  "Show original" peek button swaps in the original `note_text`
  inline (no fetch, selection unchanged); author-only "Mark as
  verified" / "Verified by author" affordance; `pending` →
  "Translating…" and `failed` → retry states; a "translations
  updating" hint after edits.
- Infra (terraform): enable `translate.googleapis.com` and
  `cloudtasks.googleapis.com`; a `google_cloud_tasks_queue`
  (max_attempts 5, `max_concurrent_dispatches` 5); a
  `tasks-invoker` service account + `serviceAccountTokenCreator`;
  the appspot SA gets `cloudtasks.enqueuer` and
  `cloudtranslate.user`. Settings: `NOTE_TARGET_LANGUAGES`
  (en + lv first; the design stays language-agnostic),
  `NOTE_TRANSLATION_ENGINE`, `TASKS_INVOKER_SA_EMAIL`,
  `TASK_QUEUE_ID`, `APP_BASE_URL`. Guardrails: a ~10k char
  per-note cap and a monthly Translate usage counter in GCS.
- Cost sketch: Cloud Translation ~$20/1M chars; the 500k
  chars/month free tier covers ~1,000 typical notes.

## Capabilities

### New Capabilities

- `note-translations`: per-note source-language detection,
  machine-translated copies of note text with status/engine/
  verification metadata, the `?lang=` serving contract on note
  detail and list endpoints, the async translation lifecycle
  (pending/ready/failed, retries, idempotent re-runs), the
  author-verification contract, and the frontend language-
  switching UX including default-language rules, the "Show
  original" peek, and verification trust signals.

### Modified Capabilities

- None. The new `translation_not_ready` error code follows the
  existing `error`/`error_code` payload shape owned by
  `error-contract` without altering that spec's existing code
  mapping; it is declared inside `note-translations` itself.
  Note-verse enrichment, auth, and provider routing are
  untouched.

## Impact

- Backend (`bible_research/`): `annotations/models.py`
  (`Note.source_language`, new `NoteTranslation`),
  `annotations/serializers.py` (translations meta, `?lang=` on
  detail and list), `annotations/views.py` (`perform_create`/
  `perform_update` hooks, verify endpoint, worker task view),
  `annotations/urls.py`, new
  `annotations/services/translation_service.py` +
  `annotations/services/tasks.py` (enqueue + OIDC verify),
  `bible_research/settings.py`, `requirements.txt`
  (`google-cloud-translate`, `google-cloud-tasks`),
  `terraform/` (APIs, tasks queue, service accounts, IAM),
  tests in `annotations/tests.py` (or a new test module), a
  hand-written migration, `DEVELOPER_GUIDE.md`, `AGENTS.md`.
- Frontend (`reactive-bible/`): `src/types.ts`
  (`NoteTranslationMeta`, `Note.source_language`,
  `Note.translations`), `src/api/notes.ts` (`lang` params,
  verify calls), `NoteCard` + language-switcher component,
  persisted preferred-note-language store slice seeded from
  `uiLanguage`, MSW handlers, tests, `DEVELOPER_GUIDE.md`.
- Coordination: `add-note-tts` builds on this change — it
  reuses `NoteTranslation` rows, `source_language`, and the
  `?lang=` serving contract, and must land after this ships.
- Migrations/terraform: agents never run `makemigrations`,
  `migrate`, or `terraform apply`; the change ships a
  hand-written migration plus terraform edits, and the user
  runs the commands.
