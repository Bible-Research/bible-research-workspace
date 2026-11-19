# Design

## Context

`ScripturePassage` (the `@…` popover body in `NoteCard`,
`CommentNode` and the note editor preview) hardcodes
`FILESET_ID = 'ENGESV_API'`. The reader's translation lives in
the persisted Zustand store as `activeTextFilesetId` — possibly
a product id (`{abbr}:text:{n}`) that `resolveTextFileset(id,
bookId, translations)` maps to the concrete per-testament
member, with `ENGKJV` resolving offline. Note dialogs
(`EditNoteModal`, `AddTagNoteModal`) already fetch linked-verse
text via `useVerseTexts(refs, activeTextFilesetId,
translations)` — they follow the selection but would also
ignore a pin. `TranslationSelector` owns the full
language → version → text/audio flow and writes the active
filesets on Save. There is no `/settings` route; `MainMenu` is
a right-side `Drawer` whose entries navigate or open modals.

## Goals / Non-Goals

**Goals:**

- One resolution rule for every note-reference text surface:
  pinned override → selected translation → ESV fallback.
- A Settings page that hosts this preference now and future
  ones (e.g. the deferred `bookTitleMarquee` toggle) later.
- Reuse the existing translation-selection UX for picking the
  pin — no second language/version browsing implementation.

**Non-Goals:**

- No backend changes; the setting is device-local only.
- No audio pick in the pin flow — previews render text only.
- The search route and the `/bible` reader keep resolving
  `activeTextFilesetId` directly — the override applies to
  note-reference surfaces only.
- The popover's "view in Bible" navigation and verse-narrowing
  behavior are unchanged.

## Decisions

### D1 — Persisted field, not a separate store

`useBibleStore` gains `noteRefFilesetId: string | null` +
`setNoteRefFilesetId`; `null` means "follow the selected
translation". It joins `partializeState`, `version` bumps
9 → 10, and `migratePersistedState` sanitizes a persisted value
to `string | null`. Persisting the picker's returned id
verbatim (usually a product id) lets `resolveTextFileset` keep
working across testament splits and member-list drift.
Alternative considered — storing the owning translation `abbr`:
loses which text product the user picked when a translation
carries several, so the raw id wins.

### D2 — Shared resolver hook

A `useNoteRefFilesetId()` hook (next to
`src/hooks/useBookName.ts`) returns
`noteRefFilesetId ?? activeTextFilesetId`; call sites then
resolve per book with `resolveTextFileset` exactly as
`PassageView`/`useVerseTexts` already do, falling back to
`'ENGESV_API'` only when resolution yields `null`. Consumers:

- `ScripturePassage` subscribes via the hook and puts the
  resolved id in its fetch-effect deps — a translation or
  override change refetches the open preview. Critical:
  `ScriptureRef.book` is a display name, so the resolver call
  must use the same `toUsfmCode(book) ?? book` conversion the
  fetch already does — otherwise `getTestament` returns null
  and a testament-split product id collapses to its OT member
  (book_not_in_fileset on every NT ref).
- `useVerseTexts` call sites in `EditNoteModal` and
  `AddTagNoteModal` swap `activeTextFilesetId` for the hook's
  value so note-linked verse text honors the pin too.

### D3 — `/settings` as a real route, not a drawer panel

A lazy `SettingsRoute` (pattern: `GettingStartedRoute`) keeps
settings deep-linkable and leaves the `MainMenu` drawer a pure
menu. `App.tsx` whitelists `/settings` for `overflow: auto`
like `/getting-started` — it is a document-style page. The
route is public: the preferences are localStorage-backed, so
gating it on auth would strand anonymous users' UI.

### D4 — Picker mode on `TranslationSelector`

Rather than duplicating the
language-tab → version → fileset flow, `TranslationSelector`
gains an optional controlled mode:

- `opened`/`onClose` props make the modal external-controlled
  (the settings page renders no trigger button; the component's
  own button stays for the header usage). The transient-state
  reset in `openModal` (selected ids, Other-tab fields,
  fetch/failure markers, `userInteractedRef`) moves into an
  `[opened]`-keyed effect so a controlled open also starts
  clean — reopening on the same language must not show stale
  selections.
- `pickTextOnly` (or `mode: 'pick'`): hides BOTH audio surfaces
  (the audio radio group AND the Advanced `rawAudioFilesets`
  radios), seeds from a `seedFilesetId` prop (the current pin,
  else the active text fileset), and Save calls
  `onPickText(selectedTextId)` instead of writing
  `setActiveTextFilesetId`/`setActiveAudioFilesetId`. The KJV
  lazy-download path in `handleTextChange` is unchanged.
- Pick mode must swap the seed id at every site that reads the
  active filesets — the audio-owner preference would otherwise
  seed the wrong language for a text-only pin (pin=NIV while
  audio=ESV opens on `eng`):
  - `openModal`'s synchronous owner lookup;
  - the `seedLanguageTab` effect (skip the audio-owner
    preference entirely — a pick mode has no audio selection);
  - `applyVersionSelection`'s audio-only branch, including its
    `handleTextChange(activeTextFilesetId)` fallback — in pick
    mode a version with no text must not silently pin the
    reader's translation;
  - the "mirror the active filesets" effect (audio lookup
    dropped in pick mode);
  - the audio-restore/`grouped.audio[0]` auto-pick — inert in
    pick mode but kept consistent for readability.

Alternative considered — a stripped-down pin picker listing
only `store.translations`: rejected, because that list only
holds languages already fetched; pinning a never-browsed
language would be impossible.

### D5 — Settings page composition

`SettingsRoute` renders a narrow `Container` with the page
title and a labeled control (Mantine `Select` or
`Radio.Group`): value `selected` writes
`setNoteRefFilesetId(null)`; value `pinned` opens the picker
modal, seeded by the current pin or (no pin yet) the active
text fileset. When a pin exists the page shows the resolved
`Translation` name + abbr (`findTextOption`/`findTranslationBy
FilesetId`, falling back to the raw id when the merged
`translations` list lacks it) with "change"/"reset" actions.
The `MainMenu` entry sits with the navigation entries, above
the theme/language block. Strings go into `settings.*`/
`menu.settings` i18n keys across `en`, `lv`, `ru`.

## Risks / Trade-offs

- Pinned translation delisted upstream → `resolveTextFileset`
  passes the stale id through and the API returns
  book-not-in-fileset/provider errors — already localized in
  the popover; the settings page's reset path is the
  escape hatch.
- `translations` lacking the pinned id's owner → the settings
  page can only show the raw fileset id until that language's
  list is fetched, and an unresolvable `{abbr}:text:{n}` id
  passes through to the API (400 invalid_parameter /
  book_not_in_fileset) rather than hitting the ESV fallback —
  the same behavior the reader has today for drifted ids, so
  parity is acceptable.
- Stored `note.verses` text (backend-enriched at note creation)
  stays in its creation-time translation — the note corpus and
  `@…` previews will intentionally follow different rules.
- Preview now defaults to the reader's translation instead of
  ESV — intended, but it means ESV-only heading payloads
  (e.g. Psalm superscriptions) vary by translation; the
  popover already renders whatever `headings` the fileset
  returns.
- Picker mode inside an already large component adds props —
  mitigated by keeping the mode strictly additive (default
  reader behavior untouched) and covered by tests.

## Migration Plan

Frontend-only, staged via the `staging`-first flow. The v10
persisted migration writes no data — absent
`noteRefFilesetId` simply behaves as `null` (follow the
selection). Rollback = revert; a stale persisted override is
inert once the resolution code is gone.
