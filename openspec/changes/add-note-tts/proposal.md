# Proposal

## Why

With `add-note-translations`, every note gains per-language
text a reader can view. The natural next step is hearing it:
users already listen to chapter audio, and notes — especially
shared/public ones — deserve the same treatment in whichever
language the card is showing. Audio must be generated lazily
so only notes that are actually played cost anything.

## What Changes

**Depends on the sibling change `add-note-translations`**:
this change reuses the `NoteTranslation`
rows, `Note.source_language`, and the `?lang=` serving
contract — this change adds the audio half of the hybrid
pipeline (text eager, audio lazy) and MUST land after
translations ship.

- `NoteTranslation` gains three fields in a second migration:
  `audio_status` (`none` | `pending` | `ready` | `failed`),
  `audio_gcs_uri`, `audio_duration_seconds`. The serializer's
  `translations` metadata gains `has_audio`.
- New lazy endpoint `POST /api/v1/notes/<id>/audio/` with body
  `{language}`: authorization reuses
  `get_accessible_notes_qs` (owner or public note);
  `language == source_language` synthesizes the original text,
  any other language requires a `status='ready'` translation
  row (409 `translation_not_ready` otherwise). A cached blob
  returns its signed URL; a miss synthesizes synchronously.
- Synthesis path: plain-text extraction via a new
  `annotations/services/tiptap_text.py` (~30-line port of the
  frontend `toPlainText`) → `NOTE_TTS_MAX_CHARS` (~4000) cap →
  413 over → `GoogleTTSClient.synthesize_text` (existing
  service, Chirp 3 HD voices, per-language `NOTE_TTS_VOICES`
  registry) → upload `note-audio/{note_id}/{language}_
  {voice}.mp3` to a NEW `{project}-note-audio` bucket →
  mutagen duration → signed URL via the `image_storage`
  pattern (IAM signBlob, bounded LRU cache).
- Cleanup: note deletion best-effort deletes
  `note-audio/<note_id>/*` blobs; a 30-day bucket lifecycle
  rule is the safety net.
- Terraform: `google_storage_bucket.note_audio` +
  `objectAdmin` for the appspot SA + the lifecycle rule.
- Frontend: a per-language mini play/pause `ActionIcon` on
  `NoteCard` feeding the signed URL into Howler — a dedicated
  lightweight player, NOT the chapter `AudioPlayer`.
- Cost: Chirp 3 HD ~$30/1M chars; lazy generation means only
  played notes cost; the 1M chars/month free tier covers most
  usage.

## Capabilities

### New Capabilities

- `note-tts`: per-language spoken audio for notes — lazy
  on-request synthesis of the text a reader is viewing
  (original or translated), audio lifecycle metadata served
  in note payloads, the audio request contract including its
  readiness and size-limit failures, storage and cleanup of
  generated clips, and the frontend mini-player affordance.

### Modified Capabilities

- None. Audio fields extend the `NoteTranslation` model
  introduced by `add-note-translations`, but that capability's
  spec is unaffected: all observable audio behavior is owned
  by the new `note-tts` capability. The reuse of
  `translation_not_ready` follows the existing
  `error`/`error_code` shape without changing
  `error-contract`.

## Impact

- Backend (`bible_research/`): `annotations/models.py` (three
  audio fields on `NoteTranslation`), a second hand-written
  migration, `annotations/serializers.py` (`has_audio` in
  translations meta), `annotations/views.py` (audio endpoint,
  blob cleanup on delete), `annotations/urls.py`, new
  `annotations/services/tiptap_text.py` +
  `annotations/services/note_tts.py`,
  `bible/services/google_tts/note_voices.py` (per-language
  voice registry), `bible_research/settings.py`
  (`NOTE_TTS_VOICES`, `NOTE_TTS_MAX_CHARS`,
  `NOTE_AUDIO_BUCKET_NAME`), `terraform/` (bucket + IAM +
  lifecycle), `requirements.txt` (`mutagen` if absent), tests,
  `DEVELOPER_GUIDE.md`, `AGENTS.md`.
- Frontend (`reactive-bible/`): `src/types.ts` (`has_audio`
  on `NoteTranslationMeta`), `src/api/notes.ts`
  (`requestNoteAudio`), `NoteCard` mini play/pause button +
  Howler wiring, MSW handlers, tests, `DEVELOPER_GUIDE.md`.
- Sequencing: implement AFTER `add-note-translations` ships —
  the migration, serializer meta, and endpoint all build on
  its model and `?lang=` contract.
- Migrations/terraform: agents never run `makemigrations`,
  `migrate`, or `terraform apply`; the change ships the
  migration file plus terraform edits and the user runs the
  commands.
