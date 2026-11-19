# Tasks

## 1. Backend: audio fields + migration

- [ ] 1.1 backend: Add `audio_status` (none/pending/ready/
  failed, default `none`), `audio_gcs_uri`, and
  `audio_duration_seconds` to `NoteTranslation` in
  `annotations/models.py`; verify no line exceeds 79 chars
  and `python manage.py check` passes
- [ ] 1.2 backend: Hand-write the second `annotations`
  migration (three `AddField`s on `NoteTranslation`,
  depending on the translations migration from
  `add-note-translations`); verify `dependencies` names the
  real prior migration (never run makemigrations/migrate)

## 2. Backend: synthesis service + endpoint

- [ ] 2.1 backend: Create `annotations/services/
  tiptap_text.py` — a port of the frontend `toPlainText`
  (walk doc, collect text nodes, newline per block type,
  pass non-doc strings through); verify a fixture test
  produces the expected plain text incl. multi-block docs
- [ ] 2.2 backend: Add `bible/services/google_tts/
  note_voices.py` reading `NOTE_TTS_VOICES` from settings
  (`{'lv': 'lv-LV-Chirp3-HD-…', 'en': 'en-US-Chirp3-HD-…'}`);
  verify an unknown language returns a clear config error
- [ ] 2.3 backend: Create `annotations/services/note_tts.py`:
  resolve text row (source → original; else `status='ready'`
  or raise `translation_not_ready`), extract plain text,
  enforce `NOTE_TTS_MAX_CHARS` (~4000 → 413), call
  `GoogleTTSClient.synthesize_text`, upload to
  `note-audio/{note_id}/{language}_{voice}.mp3` in
  `NOTE_AUDIO_BUCKET_NAME`, read duration via mutagen, store
  `audio_gcs_uri`/`audio_duration_seconds`/`audio_status`;
  verify tests cover cache-hit (no re-synthesis), the 409,
  and the 413
- [ ] 2.4 backend: Add `POST /api/v1/notes/<id>/audio/`
  `{language}` → `{audio_url, duration_seconds, expires_in}`
  using `get_accessible_notes_qs` authz and the
  `image_storage` signed-URL pattern (IAM signBlob, bounded
  LRU cache); verify tests cover 404 on inaccessible notes
  and signed-URL shape on success
- [ ] 2.5 backend: Best-effort delete `note-audio/<id>/*`
  blobs in `perform_destroy` (tolerate 404 like
  `delete_original`); verify deleting a note issues the blob
  cleanup
- [ ] 2.6 backend: Surface `has_audio` in the serializer's
  `translations` metadata; verify the note payload reflects
  generated clips per language

## 3. Backend: settings, deps, infra, docs

- [ ] 3.1 backend: Add settings `NOTE_TTS_VOICES`,
  `NOTE_TTS_MAX_CHARS`, `NOTE_AUDIO_BUCKET_NAME` via the
  `_audio_settings_source` pattern; verify config.yaml and
  env-var overrides both work
- [ ] 3.2 backend: Ensure `mutagen` is in `requirements.txt`
  (pin ≥7-day-old release if added); verify the app imports
  cleanly
- [ ] 3.3 backend: Terraform: `google_storage_bucket.
  note_audio` (`{project}-note-audio`) + `objectAdmin` for
  the appspot SA + 30-day lifecycle rule; verify `terraform
  plan` (user runs `apply`)
- [ ] 3.4 backend: Update `bible_research/DEVELOPER_GUIDE.md`
  and `AGENTS.md` with the audio endpoint, fields, voices
  registry, and bucket; verify docs match the code

## 4. Frontend: types, API, mini-player

- [ ] 4.1 frontend: Add `has_audio` to `NoteTranslationMeta`
  in `src/types.ts` and `requestNoteAudio(noteId, lang)` to
  `src/api/notes.ts` returning `{audio_url,
  duration_seconds}`; verify `tsc` and `npm run lint` pass
- [ ] 4.2 frontend: Add a per-language mini play/pause
  `ActionIcon` on `NoteCard` that requests the signed URL for
  the currently displayed language and plays it via Howler
  with a thin inline progress state — NOT the chapter
  `AudioPlayer`; verify component tests cover
  request→play→pause
- [ ] 4.3 frontend: Surface audio failure states (loading
  while synthesizing, error with retry on 409/413/5xx);
  verify tests cover the retry path
- [ ] 4.4 frontend: Update MSW handlers/fixtures for the
  audio endpoint and `has_audio`; verify `npm test -- --run`
  stays green and no line exceeds 79 chars
- [ ] 4.5 frontend: Update `DEVELOPER_GUIDE.md` with the
  mini-player and per-language audio behavior; verify docs
  match shipped UI

## Workflow follow-up

- User runs `python manage.py makemigrations annotations` to
  confirm the hand-written migration matches, `python
  manage.py migrate` on environments that need it, and
  `terraform apply` for the bucket.
- Land AFTER `add-note-translations` ships; deploy order:
  terraform → backend → frontend.
- Archive the change after PRs merge (`openspec archive`);
  verify `openspec validate --all --strict` after archiving.
