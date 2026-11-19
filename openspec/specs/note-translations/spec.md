# note-translations Specification

## Purpose
Defines machine translation of user notes: per-note source
language detection, translated copies with status and
verification metadata, the `?lang=` serving contract on note
endpoints, and the reader-facing language-switching UX.

## Requirements

### Requirement: Per-note source language

Each note SHALL carry a `source_language` ISO 639-1 code
detected from that note's own text. Detection MUST run per
note and MUST NOT be inferred from the note's owner, the
user's profile, or the UI locale.

#### Scenario: Detection runs per note

- **WHEN** a note is created or its text is edited and its
  translation run completes
- **THEN** `source_language` reflects the language detected
  in that note's text, independent of any other note; before
  the run completes the field MAY be empty

#### Scenario: Mixed-language corpus

- **WHEN** one user's notes are written in different source
  languages
- **THEN** each note reports its own `source_language`

### Requirement: One translation row per note-language

A note SHALL have at most one translation entry per target
language, each carrying translated text, a `status` of
`pending`, `ready`, `failed`, or `source`, the engine that
produced it, and an error code when failed. The detected
source language SHALL also get a `source` entry — a language
slot whose text field stays empty, because the original is
served from the note's own `note_text` rather than duplicated.

#### Scenario: Translation targets created on write

- **WHEN** a note is created and its translation run completes
- **THEN** one translation entry exists per configured target
  language, and a single entry for the detected source
  language — coinciding with the target entry when the source
  is itself a configured target — is marked `source`

#### Scenario: Failed translation keeps status and error

- **WHEN** translation of one language fails
- **THEN** that entry reports `failed` with an error code and
  other languages are unaffected

### Requirement: Eager asynchronous translation on note writes

Creating a note SHALL enqueue asynchronous translation into
every configured target language; when asynchronous execution
is configured (the production path) the write response SHALL
return before translation completes with `pending` metadata,
while a synchronous local-dev fallback MAY complete inside the
request. Editing SHALL re-enqueue only when `note_text`
changed.

#### Scenario: Create returns pending translations

- **WHEN** a note is created under asynchronous execution
- **THEN** the response already lists the target languages as
  `pending` so clients can render a translating state

#### Scenario: Non-text edits skip translation

- **WHEN** a note is updated without changing `note_text`
- **THEN** no new translation run is started

#### Scenario: Text edits refresh translations

- **WHEN** a note's `note_text` is edited
- **THEN** its translation entries are regenerated for the new
  text

### Requirement: Stale translation runs are harmless

An asynchronous translation run SHALL be keyed to the exact
note text it was enqueued for; a run arriving after a newer
edit MUST be a no-op so the freshest text wins. Failed runs
SHALL be retried with backoff up to a bounded attempt count.

#### Scenario: Superseded run no-ops

- **WHEN** a queued translation run executes after the note
  was edited again
- **THEN** it writes nothing and the newer text's translations
  are used

#### Scenario: Overlapping runs keep the newest text

- **WHEN** two translation runs for different text versions
  overlap and the older-text run finishes last
- **THEN** the stored entries still reflect the newest note
  text — write-order never determines the outcome

#### Scenario: Transient failure retries

- **WHEN** a translation run fails transiently
- **THEN** it is retried with backoff and gives up after the
  configured maximum, leaving the entry `failed`

### Requirement: Translation preserves note text structure

Translated note text SHALL keep the rich-text document
structure and inline formatting of the original — marks such
as bold and links, non-text `scriptureRef`/`mention` atoms,
and plain-text scripture-reference tokens (masked through
translation). Non-structured plain text SHALL be translated
as plain text.

#### Scenario: Formatted note keeps marks

- **WHEN** a structured note containing formatted spans is
  translated
- **THEN** the translated document renders with the same
  structure and formatting in the target language

#### Scenario: Oversized note is marked, not truncated

- **WHEN** a note's text exceeds the configured per-note
  character cap
- **THEN** its translation entries are marked `failed` rather
  than silently truncated

### Requirement: Translation metadata in note payloads

Every note API representation — detail, list, and create
responses — SHALL include `source_language` and a
`translations` list with each entry's `language`, `status`,
`verified` flag, and `engine`.

#### Scenario: Note payload lists languages

- **WHEN** a client fetches a note that has translation
  entries
- **THEN** the payload contains `source_language` and one
  metadata entry per language

### Requirement: Translated serving on note detail

`GET /api/v1/notes/<id>/?lang=<code>` SHALL return `note_text`
in the requested language when a `ready` entry exists, or the
original `note_text` when the language matches the note's
`source_language` (its `source` entry carries no text of its
own). Otherwise `note_text` MUST be returned unchanged —
the request MUST NOT fail.

#### Scenario: Ready translation served

- **WHEN** a caller requests a note with `?lang=` matching a
  `ready` translation
- **THEN** `note_text` contains the translated document

#### Scenario: Missing translation falls back to original

- **WHEN** a caller requests a note with `?lang=` that has no
  `ready` or `source` entry
- **THEN** the response succeeds with the original `note_text`
  and the translation metadata intact

### Requirement: Translated serving on note list

`GET /api/v1/notes/?lang=<code>` SHALL apply the same swap to
every note in the page in one request, with the same
unchanged-text fallback per note.

#### Scenario: Page rendered in reader language

- **WHEN** a caller lists notes with `?lang=`
- **THEN** each note with a `ready` entry for that language
  returns the translation's text as `note_text`, each note
  whose `source_language` matches returns its original text,
  and the rest return their original text

#### Scenario: Anonymous reader sees translated public notes

- **WHEN** an anonymous caller requests an accessible public
  note or note list with `?lang=`
- **THEN** translated text and `translations` metadata are
  returned without authentication, so the language switcher is
  usable by anonymous viewers of public notes

### Requirement: Author verification of translations

The note's owner SHALL be able to mark a `ready` translation
as author-verified and to revoke that mark, via
`POST`/`DELETE` on
`/api/v1/notes/<id>/translations/<lang>/verify/`. Non-owners
and missing entries SHALL get 404; `pending`, `failed`, or
`source` entries SHALL get 409 with `error_code`
`translation_not_ready` — a `source` entry is the author's own
text, so there is nothing to verify. Fresh translation output
MUST reset verification.

#### Scenario: Owner verifies a ready translation

- **WHEN** the owner POSTs verify for a `ready` translation
- **THEN** the entry reports `verified` and a verification
  timestamp

#### Scenario: Non-owner gets 404

- **WHEN** a non-owner POSTs verify on someone else's note
- **THEN** the API responds 404 as though it does not exist

#### Scenario: Unfinished translation cannot be verified

- **WHEN** the owner POSTs verify for a `pending`, `failed`,
  or `source` translation
- **THEN** the API responds 409 with `error_code`
  `translation_not_ready` — a `source` entry is the author's
  own text, so there is nothing to verify

#### Scenario: Re-translation clears verification

- **WHEN** a verified translation is regenerated after an edit
- **THEN** the new output is unverified until the author
  reviews it again

### Requirement: Owner retry of failed translations

The note's owner SHALL be able to re-run translation for
`failed` entries via
`POST /api/v1/notes/<id>/retranslate/`, which re-marks them
`pending` and enqueues a new run. Non-owners SHALL get 404;
entries not in `failed` state SHALL be left untouched.

#### Scenario: Owner retries a failed translation

- **WHEN** the owner POSTs retranslate for a note with a
  `failed` entry
- **THEN** that entry reports `pending` and a new translation
  run is enqueued

#### Scenario: Non-owner retry gets 404

- **WHEN** a non-owner POSTs retranslate on someone else's
  note
- **THEN** the API responds 404 as though it does not exist

### Requirement: Monthly translation coverage audit

A scheduled job SHALL run at the end of each month and audit
every note for a `ready` or `source` entry — per configured
target language — whose text matches the note's current text.
Any note missing or behind on a translation SHALL be reported
as a Sentry alert with per-language/per-status counts.

#### Scenario: Audit flags uncovered notes

- **WHEN** the monthly audit finds notes lacking a current
  `ready`/`source` entry for a configured target
- **THEN** a Sentry alert is emitted listing how many notes
  are uncovered per language

#### Scenario: Full coverage stays quiet

- **WHEN** the monthly audit finds every note covered
- **THEN** no Sentry alert is emitted

### Requirement: Reader language selection defaults

The frontend SHALL default each note's displayed language to
the reader's persisted preferred note language (seeded from
the UI language), falling back to the note's
`source_language` when no usable translation exists. Choosing
a language SHALL update the persisted preference.

#### Scenario: Preferred language applied across notes

- **WHEN** a reader whose preferred note language differs from
  a note's source language views the notes list
- **THEN** notes with a usable translation display in the
  preferred language and the rest display their originals

### Requirement: Auto-translate display toggle

The reader SHALL be able to disable translated display via a
persisted app-menu setting (default ON). When disabled, every
note renders its original `note_text` and all translation
controls (switcher, peek, verify) are hidden. The toggle is
display-level only — it MUST NOT stop the backend pipeline;
other readers still get translations.

#### Scenario: Toggle off shows originals only

- **WHEN** a reader disables auto-translate and views notes
- **THEN** every card shows the original text with no
  translation controls, regardless of any `ready` entries

#### Scenario: Toggle on restores translated display

- **WHEN** the reader re-enables auto-translate
- **THEN** the preferred-language rendering rules apply again

### Requirement: Show-original peek

While viewing a translation, the reader SHALL be able to peek
at the original text inline without a fetch and without
changing the selected language, and toggle back.

#### Scenario: Peek leaves selection untouched

- **WHEN** the reader invokes "Show original" on a translated
  note
- **THEN** the original text displays inline while the
  language selection and stored preference remain on the
  translation

### Requirement: Translation states and trust signals surfaced

Translated notes SHALL always be labeled as machine output,
with an explicit retry affordance on `failed` (never silent
fallback to source text) and a "Translating…" state on
`pending`. Author-verified translations SHALL display a
distinct "Verified by author" indicator visible to readers;
only the author sees the verify action. Verification is a
trust label only — it MUST NOT gate serving or display: a
`ready` translation is shown to every reader whether or not
the author has verified it.

#### Scenario: Failed state offers retry

- **WHEN** a note's translation entry is `failed`
- **THEN** the UI surfaces the failure with a retry control
  instead of silently showing the original as if translated

#### Scenario: Language fetch failure keeps last-good content

- **WHEN** a `?lang=` request fails (network error or 5xx)
- **THEN** the card keeps rendering the last successfully
  fetched text and surfaces a retryable error notification —
  it MUST NOT blank the note or present the original text as
  if it were the requested translation

#### Scenario: Author marks translation verified

- **WHEN** the author uses the verify action on their own
  ready translation
- **THEN** a "Verified by author" indicator replaces the plain
  auto-translated label for all viewers

#### Scenario: Unverified translation still displays

- **WHEN** a note has a `ready` translation its author has
  not verified
- **THEN** readers see the translated text with the plain
  "Auto-translated" label — it is not hidden pending
  verification
