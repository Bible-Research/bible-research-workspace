# Tasks

## 1. Backend: model + migration

- [ ] 1.1 backend: Add `Note.source_language` (CharField,
  `max_length=10`, blank) and the `NoteTranslation` model to
  `annotations/models.py` — `id` CharField PK prefixed `NTR`
  (uuid suffix, matching sibling model conventions), `note` FK
  (`related_name='translations'`, CASCADE), `language`, `text`
  (translated TipTap JSON; EMPTY on `source` rows),
  `status` (pending/ready/failed/source),
  `source_hash` (sha256, 64 chars), `engine`, `error`,
  `author_verified`, `verified_at`, timestamps,
  `UniqueConstraint(note, language)` (its btree covers the
  hot `note_id + language` lookups) plus
  `Index(fields=['language', 'status'])` for single-language
  scans (readers typically request one language); verify no
  line exceeds 79 chars and `python manage.py check` passes
- [ ] 1.2 backend: Hand-write the `annotations` migration for
  both model changes modeled on the latest existing migration;
  verify `dependencies` names the real latest migration and
  `python manage.py check` passes (never run
  makemigrations/migrate)

## 2. Backend: translation service + worker pipeline

- [ ] 2.1 backend: Create
  `annotations/services/translation_service.py` with
  `translate_texts(texts, source, targets)` (language-agnostic
  so comments can reuse it): structural TipDoc walk batching
  all `type:'text'` node values in ONE `translate_text` call,
  splice-back in order, plain-text fallback for non-doc
  strings, detection via `translate_text` without
  `source_language_code`, per-note ~10k char cap,
  `@USFM.C[.V[-E]]` plain-text tokens (match via the
  frontend's `SCRIPTURE_TOKEN_REGEX` so hand-typed colon
  variants are covered) masked before the call and restored
  on splice-back; verify with a unit test on a real TipTap
  fixture incl. `scriptureRef`/`mention` atoms and an embedded
  `@USFM…` token that structure and tokens survive round-trip
- [ ] 2.2 backend: Create `annotations/services/tasks.py`:
  `enqueue(note_id, source_hash)` issuing a Cloud Tasks POST
  to `{APP_BASE_URL}/api/v1/tasks/translate-note/` with an OIDC
  token for `TASKS_INVOKER_SA_EMAIL`; when
  `not IS_GCP_ENVIRONMENT`, call the worker function inline;
  verify a local note create translates without any GCP infra
- [ ] 2.3 backend: Add the worker endpoint
  `POST /api/v1/tasks/translate-note/` (`AllowAny` + OIDC
  `id_token.verify_oauth2_token` + token-email check against
  `TASKS_INVOKER_SA_EMAIL`): stale `source_hash` → 204 no-op;
  re-check the hash inside the write transaction under
  `select_for_update` on the note so a run superseded
  mid-translation aborts rather than overwriting fresher
  rows; detect + store `source_language`; ALWAYS upsert a `source`
  row for the detected language with EMPTY `text` — a
  language-slot/metadata anchor, the original is served from
  `note_text` (even when it is not a configured target — a
  `ru` source under `['en','lv']` keeps its uniform row) plus
  one row per configured target (`target == source` reuses
  that row); stamp each with the `engine` actually used; reset
  `author_verified=False`/`verified_at=NULL` on every write;
  failure → row `failed` + non-2xx for retry; verify tests
  cover 401 on bad/missing OIDC and the stale-hash no-op
- [ ] 2.4 backend: Hook `NoteViewSet.perform_create` /
  `perform_update`: inside `transaction.on_commit`, upsert
  `pending` rows for `NOTE_TARGET_LANGUAGES` and enqueue —
  update path only when `note_text` changed. Cover the
  bulk-create path too (`bulk_create_notes` bypasses
  `perform_create` — apply the same per-note upsert+enqueue);
  verify a tag-only PATCH enqueues nothing, a text PATCH does,
  and a bulk create enqueues per note
- [ ] 2.5 backend: Add monthly Translate usage counter via the
  existing `increment_monthly_usage` pattern
  (`fileset_id='note-translate'`) and log billed chars per
  language; verify a unit test asserts the counter increments

## 3. Backend: API contract + verify endpoint

- [ ] 3.1 backend: Extend `NoteSerializer`/`to_representation`
  with `source_language` and `translations` meta (`language`,
  `status`, `verified`, `engine`) via a `translations`
  prefetch; apply `?lang=` swap on detail AND list with
  unchanged-`note_text` fallback when no `ready`/`source` row
  exists; verify tests cover `?lang=` on both endpoints plus
  the mixed-page fallback
- [ ] 3.2 backend: Add owner-only
  `POST|DELETE /api/v1/notes/<id>/translations/<lang>/verify/`:
  non-owner/missing → 404, `pending`/`failed`/`source` → 409
  `translation_not_ready` in the `error`/`error_code` shape,
  idempotent set/clear of `author_verified`/`verified_at`;
  verify tests cover 404s, both 409 cases, set and clear
- [ ] 3.3 backend: Owner-only
  `POST /api/v1/notes/<id>/retranslate/` re-enqueueing the
  task for `failed` rows only (non-owner → 404, other states
  untouched — the frontend retry path); verify it re-marks
  rows `pending` and enqueues
- [ ] 3.4 backend: Update `bible_research/DEVELOPER_GUIDE.md`
  and `AGENTS.md` with the new model, endpoints, `?lang=`
  contract, and settings; verify docs match the code

## 4. Backend: settings, deps, infra

- [ ] 4.1 backend: Add settings `NOTE_TARGET_LANGUAGES`
  (default `['en','lv']`), `NOTE_TRANSLATION_ENGINE`
  (`'google-nmt'`), `TASKS_INVOKER_SA_EMAIL`, `TASK_QUEUE_ID`,
  `APP_BASE_URL` via the `_audio_settings_source`
  env/config.yaml pattern; verify config.yaml and env-var
  overrides both work
- [ ] 4.2 backend: Add `google-cloud-translate` and
  `google-cloud-tasks` to `requirements.txt` pinned to
  releases ≥7 days old; verify the app imports cleanly
- [ ] 4.3 backend: Terraform: enable `translate.googleapis.com`
  + `cloudtasks.googleapis.com` in `locals.apis`; add
  `google_cloud_tasks_queue` (`max_attempts=5`, exponential
  backoff, `max_concurrent_dispatches=5`),
  `google_service_account.tasks_invoker` +
  `serviceAccountTokenCreator` for the Cloud Tasks service
  agent, `roles/cloudtasks.enqueuer` and
  `roles/cloudtranslate.user` on the appspot SA; verify
  `terraform plan` (user runs `apply`)
- [ ] 4.4 backend: Optional `translate_notes` management
  command (`--user`, `--all`) to backfill existing notes —
  reuse the worker's translate path, skip notes already
  `ready` for all targets; verify `--user` scopes to that
  user's notes only
- [ ] 4.5 backend: Add `check_note_translations` management
  command auditing every note for a `ready`/`source` row per
  configured target with a matching `source_hash`; nonzero
  misses → `sentry_sdk.capture_message` at ERROR with
  per-language/status counts, zero → INFO log; register it in
  the `run_all_jobs` JOBS table as
  `last_day_of_month`; verify a seeded-miss unit test emits
  the Sentry call and a clean run logs only
- [ ] 4.6 backend: Terraform: `google_cloud_scheduler_job`
  `note_translations_audit` (`schedule "0 3 28-31 * *"`, no
  `--only` override so the JOBS schedule predicates gate it)
  invoking `all-jobs` via the existing `all_jobs_scheduler`
  SA pattern in `scheduler.tf`; verify `terraform plan`
  (user runs `apply`)

## 5. Frontend: types, API, store

- [ ] 5.1 frontend: Add `NoteTranslationMeta`,
  `Note.source_language`, `Note.translations` to
  `src/types.ts`; extend `src/api/notes.ts` with `lang` params
  on detail+list fetches and verify/unverify calls; verify
  `tsc` and `npm run lint` pass
- [ ] 5.2 frontend: Add a persisted `preferredNoteLanguage`
  Zustand slice seeded from `uiLanguage`; verify the preference
  persists across reload and seeds correctly on first run
- [ ] 5.3 frontend: Update MSW handlers/fixtures with
  `source_language`, `translations` meta, `?lang=` swap, and
  the verify endpoint; verify existing tests still pass
  (`npm test -- --run`)

## 6. Frontend: NoteCard language UX

- [ ] 6.1 frontend: Add a `SegmentedControl` language switcher
  to `NoteCard` (when `translations.length > 1`) defaulting to
  `preferredNoteLanguage` with `source_language` fallback;
  switching fetches `?lang=` text cached in an in-memory Map
  AND persists the choice back to `preferredNoteLanguage`;
  verify a mixed-language list renders each card in the
  reader's language and a switch survives reload
- [ ] 6.2 frontend: Add "Auto-translated" caption +
  source-language chip, `<Box lang={code}>` on the text
  container, and the "Show original" peek button (inline swap
  to the original `note_text`, no fetch, selection unchanged,
  labeled "Original (<source_language>)"); verify tests cover
  peek toggle and unchanged selection
- [ ] 6.3 frontend: Add author-only "Mark as verified" /
  un-verify actions and a "Verified by author" chip visible to
  all viewers (distinct from the auto-translated gray); verify
  the action is hidden for `status='source'` and non-owners
- [ ] 6.4 frontend: Surface `pending` ("Translating…"
  skeleton on the switcher), `failed` (tooltip + retry icon
  hitting the retranslate endpoint), and a "translations
  updating" hint after `editNote`; verify tests cover all
  three states and no silent source-text fallback on `failed`
- [ ] 6.5 frontend: Handle `?lang=` fetch failure (network/
  5xx): keep the last successfully rendered text, show a
  retryable localized error notification (existing
  `errorText`/notification pattern) — never blank the card or
  render the original as if it were the requested language;
  verify tests cover the failure path and retry
- [ ] 6.6 frontend: Schedule a one-shot delayed refetch of the
  note (or notes list) after `addTagNote`/`editNote` returns
  `pending` translations so `Translating…` flips to `ready`
  without a page reload; verify the refetch fires once and
  stops after `ready`/`failed`
- [ ] 6.7 frontend: Add an "Auto-translate notes" toggle to
  `MainMenu` bound to a persisted `noteAutoTranslate` setting
  (default ON, i18n'd label): when OFF, `?lang=` is omitted
  from note list/detail fetches and `NoteCard` renders
  `note_text` with switcher/peek/verify hidden — display-only,
  backend translation keeps running; verify the preference
  persists across reload and the toggle hides all translation
  UI
- [ ] 6.8 frontend: Update `DEVELOPER_GUIDE.md` with the
  switcher, default-language rules, peek, auto-translate
  toggle, and verification UX; verify docs match shipped UI

## Workflow follow-up

- User runs `python manage.py makemigrations annotations` to
  confirm the hand-written migration matches, `python
  manage.py migrate` on environments that need it, and
  `terraform apply` for the new APIs/queue/SAs.
- Deploy order: terraform → backend → frontend; then set
  `NOTE_TARGET_LANGUAGES` in prod config.
- Archive the change after PRs merge (`openspec archive`);
  verify `openspec validate --all --strict` after archiving.
