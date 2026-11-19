# Tasks

All work is in `reactive-bible/` (frontend only). Repo rules:
worktree off `origin/main`, PR `--base staging`, ≤79 chars/line,
scoped `npm test`/`npm run lint` verification only.

## 1. Store + resolution plumbing

- [x] 1.1 Add `noteRefFilesetId: string | null` +
  `setNoteRefFilesetId` to `BibleState`/`initialState` in
  `src/store.ts`; include it in `partializeState`
  (`src/stores/bibleStore/persistence.ts`), bump persist
  `version` 9 → 10, and add a `version < 10` sanitize branch
  (non-string → `null`). Update the existing persisted-key
  assertion and `src/__snapshots__/store.test.tsx.snap`, plus
  the `{ version: 9 }` seed in `src/i18n/index.test.ts` —
  verify with a persistence test that a v9 blob migrates and
  `partializeState` round-trips the field
- [x] 1.2 Add `useNoteRefFilesetId()` hook
  (`src/hooks/useNoteRefFilesetId.ts`) returning
  `noteRefFilesetId ?? activeTextFilesetId` — verify a unit
  test covers both branches (override set vs. follow selection)

## 2. Preview + note surfaces use the resolver

- [x] 2.1 Replace `ScripturePassage`'s hardcoded `FILESET_ID`
  with the hook + `resolveTextFileset(id,
  toUsfmCode(reference.book) ?? reference.book, translations)`
  — `reference.book` is a display name; the raw name breaks
  testament-split resolution — `'ENGESV_API'` last-resort
  fallback, and the resolved id added to the fetch-effect deps.
  Also update the now-stale "preview is only ever ESV" comment
  in `ScripturePassagePopover` — verify the existing
  `ScripturePassage` tests pass and a new test asserts
  `getVersesInChapter` receives the active/pinned id
- [x] 2.2 Pass the hook's value into `useVerseTexts` in
  `EditNoteModal` and `AddTagNoteModal` — verify their tests
  still pass and linked-verse text honors the pin

## 3. TranslationSelector picker mode

- [x] 3.1 Add controlled-picker props (`opened`, `onClose`,
  `seedFilesetId`, `onPickText`, pick-text-only mode): hide
  BOTH audio surfaces (radio group + Advanced raw-fileset
  radios), swap the seed id at every active-fileset read site
  (`openModal` owner lookup, `seedLanguageTab`, the
  `applyVersionSelection` audio-only fallback, the mirror
  effect, audio auto-pick), move the `openModal` transient
  reset into an `[opened]`-keyed effect, and make Save call
  `onPickText(selectedTextId)` without writing the active
  filesets — verify new tests: pick mode calls back with the
  text id, seeds the pinned language even when the reader's
  audio owner differs, leaves
  `activeTextFilesetId`/`activeAudioFilesetId` untouched, and
  default reader mode is unchanged

## 4. Settings page + menu entry

- [x] 4.1 Create `src/routes/SettingsRoute.tsx` (lazy, public)
  with the note-reference-translation control: `selected` →
  `setNoteRefFilesetId(null)`, `pinned` → opens the picker;
  shows the pinned translation name/abbr with change/reset —
  verify a route test covers select, pick, reset
- [x] 4.2 Register `/settings` in `src/routes/index.tsx` and
  whitelist the path for `overflow: auto` in `App.tsx` —
  verify `/settings` renders and scrolls like
  `/getting-started`
- [x] 4.3 Add the "Settings" entry (icon + `t('menu.settings')`)
  to `MainMenu`, navigating to `/settings` and closing the
  drawer — verify a `MainMenu` test for anonymous and
  authenticated renders

## 5. i18n + docs

- [x] 5.1 Add `menu.settings` and `settings.*` keys to
  `en/common.json` (source) plus `lv`/`ru` — verify `npm test`
  i18n specs pass and no key renders raw
- [x] 5.2 Update `reactive-bible/AGENTS.md` +
  `DEVELOPER_GUIDE.md` (settings surface, preview fileset
  contract, `useNoteRefFilesetId`) — verify doc text matches
  the shipped behavior
- [x] 5.3 Run `npm run lint` (0 errors/0 warnings) and
  `awk 'length > 79'` on every touched file — verify clean

## Workflow follow-up

- Archive the change once the staging PR merges cleanly.
