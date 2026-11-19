# Design

Distilled from
`docs/plans/note-translations-investigation-and-plan.md`,
covering the AUDIO half of the hybrid pipeline. It builds on
`add-note-translations` (`NoteTranslation` rows,
`Note.source_language`, the `?lang=` serving contract) and
MUST land after that change ships.

## Context

See proposal.md — Why. Key facts from investigation:

- The Cloud TTS stack already exists
  (`bible/services/google_tts/`): `client.py` —
  `GoogleTTSClient.synthesize_text(text, language_code,
  voice_name, sample_rate_hertz) -> bytes` with `QuotaExceeded`
  mapping, reusable as-is; `budget.py` — `CharBudget` monthly
  ledger; `synthesizer.py` is chapter-specific (SWORD verses →
  per-verse MP3 concat + timestamps) and NOT reusable — notes
  need a simpler text→mp3 path; `registry.py` is fileset-keyed
  so notes need a per-LANGUAGE registry instead.
- GCS helpers (`bible/services/storage/gcs.py`) and
  `annotations/services/image_storage.py` already implement
  the signed-URL machinery this needs: `gs://` URI stored on
  the row, `signed_*_url()` on read via IAM signBlob, bounded
  LRU cache — `image_storage` is the better template (user
  content, same lifecycle).
- Buckets follow a per-purpose pattern (`…-bible-audio`,
  `…-images-originals`, `…-images-thumbnails`); infra lives in
  terraform; the app runs on App Engine standard F1 with a
  60 s request deadline.
- `get_accessible_notes_qs` is the shared own ∪ public authz
  scope; `translation_not_ready` (409) is introduced by
  `add-note-translations` in the `error`/`error_code` shape.
- Cost: Chirp 3 HD ~$30/1M chars (500-char note ≈ $0.015);
  TTS free tier 1M chars/month; lazy generation means only
  played notes cost.

## Goals / Non-Goals

**Goals:**

- One button per displayed language → spoken note audio in
  that language, synthesized on first request only.
- Audio state visible per language in note payloads.
- Reuse of the existing TTS client and signed-URL machinery;
  no new external services.
- Bounded storage: dedicated bucket, cleanup on note delete
  plus a lifecycle safety net.

**Non-Goals:**

- Eager/batch audio generation — lazy only; only played notes
  cost.
- Streaming, waveform UI, or the chapter `AudioPlayer` —
  note clips are 10–60 s; a mini play/pause control suffices.
- Verse text audio — chapter audio already covers that via
  the fileset pipeline.
- Audio for text the reader cannot see — only source and
  `ready` languages are speakable.

## Decisions

### D1: Audio fields live on `NoteTranslation`

`audio_status` (`none`/`pending`/`ready`/`failed`),
`audio_gcs_uri`, `audio_duration_seconds` are added to
`NoteTranslation` in a second migration — text and audio
lifecycles stay independent (text eager, audio lazy) while
one row still represents "this note in this language". The
`status='source'` row is where source-language audio hangs.
`has_audio` in the serializer's `translations` meta lets the
frontend show the play state without extra fetches.

### D2: Lazy synchronous synthesis inside the request

`POST /api/v1/notes/<id>/audio/` `{language}`:

1. Authz via `get_accessible_notes_qs` — 404 for notes the
   caller cannot access.
2. Resolve the text row: `language == note.source_language` →
   source row/original; else require `status='ready'` (409
   `translation_not_ready` on `pending`/`failed`/missing).
3. `audio_status='ready'` + blob exists → return a fresh
   signed URL (cache hit — no re-synthesis). The cache is
   keyed to the ROW's current text, not just language+voice:
   whenever the translation pipeline rewrites a row's text
   (edit → re-translation), it also clears the row's
   `audio_status`/`audio_gcs_uri`/`audio_duration_seconds`,
   so a re-translated note re-synthesizes on next play
   instead of serving the OLD text's clip.
4. Miss → synthesize in-request (~1–2 s for typical notes,
   well under the 60 s GAE deadline), upload, record
   `audio_gcs_uri`/`audio_duration_seconds`, return
   `{audio_url, duration_seconds, expires_in}`.

Eager synthesis was rejected: it would burn TTS quota on
notes nobody plays — lazy means only played notes cost. A
Cloud Tasks hop was rejected too: the synthesis is fast
enough to feel instant inline, and a queue would add latency
plus a second OIDC surface for no reliability win.

### D3: Plain-text extraction via a backend `toPlainText` port

New `annotations/services/tiptap_text.py` — a ~30-line port
of the frontend `utils/tiptapContent.ts` `toPlainText`: walk
the doc, collect text nodes, newline per block type, and
mirror the frontend's bounded-walk abuse caps
(`MAX_WALK_DEPTH`/`MAX_WALK_NODES`) — note JSON is
user-controlled. Synthesizing the TipDoc JSON verbatim would
read markup aloud; an HTML path would need a converter.
Non-doc strings pass through as-is.

### D4: Per-language voice registry in settings

New `google_tts/note_voices.py` reads `NOTE_TTS_VOICES` —
`{'lv': 'lv-LV-Chirp3-HD-Sadachbia', 'en':
'en-US-Chirp3-HD-*'}` — rather than the fileset-keyed chapter
registry. Chirp 3 HD covers en-US and lv-LV; future languages
(e.g. ru-RU) verify Chirp support first, else fall back to
`Neural2`/`Standard` voices — config, not code. The voice
name is part of the blob key (`{language}_{voice}.mp3`) so a
voice change naturally re-synthesizes.

### D5: Dedicated bucket + `image_storage` signed-URL pattern

Clips go to `note-audio/{note_id}/{language}_{voice}.mp3` in
a NEW `{project}-note-audio` bucket (per-purpose precedent).
Signed URLs follow `image_storage`: parse the stored `gs://`
URI, IAM signBlob, bounded LRU cache — no SA key material.
mutagen reads duration post-upload (or pre-upload from the
bytes) into `audio_duration_seconds`.

### D6: Cleanup on delete + lifecycle safety net

`perform_destroy` best-effort deletes `note-audio/<id>/*`
(tolerating 404, like `delete_original`); a 30-day bucket
lifecycle rule catches orphans (e.g. hard-deleted rows,
failed cleanups).

### D7: Frontend mini-player, not the chapter player

A per-language play/pause `ActionIcon` on `NoteCard` calls
`requestNoteAudio(noteId, lang)` → signed URL → Howler.
Note clips are short; hijacking the chapter `AudioPlayer`
(chapters, keep-alive, Media Session) would be wrong weight.
The played language always matches the displayed one —
`?lang=` text and `language` audio use the same selection.

## Risks / Trade-offs

- [Synchronous synthesis latency on a cold cache] → ~1–2 s
  typical; the client shows a loading state; 60 s GAE
  deadline is ample. Notes near `NOTE_TTS_MAX_CHARS` take
  longer — still one TTS call.
- [Voice availability per locale is uncertain for future
  languages] → voices live in `NOTE_TTS_VOICES` settings, not
  code; adding a language is a config entry.
- [Quota/billing abuse on the lazy endpoint] → authz scoped
  to accessible notes, per-note char cap (413), monthly usage
  counters in the existing pattern, `CharBudget`-style ledger
  available if needed. Anonymous viewers of public notes CAN
  trigger first-time synthesis — accepted for MVP because the
  cost is one bounded clip per note-language and counters
  surface anomalies; if abuse appears, gate first-time
  synthesis to authenticated callers (anonymous plays then
  return only already-generated audio).
- [Race: two concurrent first-plays synthesize twice] →
  `if_generation_match`/update-or-create on the row bounds it
  to a duplicate blob, self-healing on next request.
- [Note deletion orphaned blobs] → best-effort delete plus
  the 30-day lifecycle rule.
- [Frontend `toPlainText` and backend port drift] → port is
  ~30 lines; a fixture test compares the port against a known
  doc's expected plain text.

## Migration Plan

1. Terraform: `google_storage_bucket.note_audio` +
   `objectAdmin` for the appspot SA + 30-day lifecycle (user
   runs `apply`).
2. Merge backend PR (fields + second hand-written migration +
   services + endpoint); user runs `python manage.py migrate`
   and confirms `makemigrations` is clean.
3. Merge frontend PR — `has_audio` is additive; old payloads
   degrade to "no play button".

Rollback: revert PRs; the fields are additive, blobs expire
under the lifecycle rule, and the endpoint can be disabled by
emptying `NOTE_TTS_VOICES`.

## Open Questions

- Exact Chirp 3 HD voice names for `en`/`lv` are config
  values to confirm at deploy time against the current
  `voices.list` output; future languages (ru-RU) pick
  Chirp vs Neural2/Standard per availability.
