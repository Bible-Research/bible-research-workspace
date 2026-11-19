# Spec Delta

## Purpose

Defines lazy text-to-speech audio for notes: on-request
synthesis of a note's original or translated text per
language, audio lifecycle metadata in note payloads, the
audio request contract, and the frontend mini-player.

## ADDED Requirements

### Requirement: Per-language audio lifecycle metadata

Each note translation entry SHALL track audio state:
`audio_status` of `none`, `pending`, `ready`, or `failed`, the
stored audio object reference, and clip duration. Note API
payloads SHALL expose per-language `has_audio` in the
translations metadata.

#### Scenario: Audio state listed per language

- **WHEN** a client fetches a note whose translation entries
  have generated audio
- **THEN** each entry's `has_audio` reflects whether a clip
  exists for that language

### Requirement: Lazy audio request endpoint

`POST /api/v1/notes/<id>/audio/` with `{language}` SHALL
return `{audio_url, duration_seconds, expires_in}` for any
caller who can access the note (owner or public note).
Requesting the note's `source_language` uses the original
text; any other language requires a `ready` translation.

#### Scenario: Source-language audio

- **WHEN** an authorized caller requests audio in the note's
  `source_language`
- **THEN** audio is produced from the note's original text

#### Scenario: Translated-language audio

- **WHEN** an authorized caller requests audio in a language
  with a `ready` translation
- **THEN** audio is produced from the translated text, so the
  spoken language matches the displayed one

#### Scenario: Inaccessible note rejected

- **WHEN** a caller requests audio for a note they cannot
  access
- **THEN** the API responds 404 as though the note does not
  exist

### Requirement: Readiness and size failures are honest

Requesting audio for a language whose translation is not
`ready` SHALL fail with 409 and `error_code`
`translation_not_ready` in the standard error shape. Text
exceeding the configured character cap SHALL fail with 413.

#### Scenario: Unfinished translation cannot be spoken

- **WHEN** audio is requested for a `pending` or `failed`
  translation
- **THEN** the API responds 409 with `error_code`
  `translation_not_ready`

#### Scenario: Oversized note refused

- **WHEN** the text to synthesize exceeds the configured
  per-note audio character cap
- **THEN** the API responds 413 instead of truncating

### Requirement: Repeat requests reuse generated audio

A request for a language whose audio already exists SHALL
return a fresh access URL for the existing clip without
re-synthesizing; synthesis SHALL happen at most once per
note-language-voice-text combination — when the note's text
for that language is regenerated, prior audio MUST be
invalidated so the next request synthesizes the new text
rather than serving the old clip.

#### Scenario: Cache hit skips synthesis

- **WHEN** audio is requested twice for the same note and
  language
- **THEN** the second request returns the stored clip's URL
  and no new synthesis occurs

#### Scenario: Re-translation invalidates audio

- **WHEN** a note's text is edited and its translation for a
  played language is regenerated
- **THEN** the stored audio no longer counts as existing and
  the next request synthesizes the new text

### Requirement: Audio storage is bounded and cleaned up

Generated clips SHALL live in a dedicated audio bucket keyed
by note and language. Deleting a note SHALL best-effort
delete its clips, and a bucket lifecycle rule SHALL remove
objects past a retention window as a safety net.

#### Scenario: Note deletion removes clips

- **WHEN** a note with generated audio is deleted
- **THEN** its stored clips are removed or expire under the
  lifecycle rule

### Requirement: Frontend mini-player per language

The note card SHALL offer a lightweight play/pause control
for the currently displayed language that fetches the audio
URL on demand and plays it in an inline mini-player — not the
chapter audio player. Synthesis failures SHALL surface as an
error state with retry, never silently.

#### Scenario: Play in displayed language

- **WHEN** the reader taps play while viewing a translation
- **THEN** audio in that language plays inline on the card

#### Scenario: Failure offers retry

- **WHEN** the audio request fails or synthesis errors
- **THEN** the card shows an error affordance that can be
  retried
