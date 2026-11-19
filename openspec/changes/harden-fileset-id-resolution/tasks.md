# Tasks

All work is in `reactive-bible/` (frontend only). Repo rules:
worktree off `origin/main`, PR `--base staging`, ≤79 chars/line,
scoped `npm test`/`npm run lint` verification only.
Section 3 depends on `add-note-ref-translation-setting` (needs
`useNoteRefFilesetId` and `SettingsRoute`) — sections 1–2 are
independent and can land first; implement the sibling change
before section 3 if it isn't already merged.

## 1. Resolver hardening

- [ ] 1.1 In `src/utils/filesetGroups.ts`, make
  `resolveTextFileset` return `null` when the stored id
  contains `:` and `findTextOption` misses all lookups; keep
  pass-through for non-product ids — verify
  `filesetGroups.test.ts` gains cases: orphaned product id →
  null, unknown raw id → passthrough
- [ ] 1.2 Apply the same rule to `resolveAudioFileset`
  (`findAudioOption` miss + `:` → unresolvable result instead
  of passthrough) — verify a test covers an orphaned
  `{abbr}:audio:{n}` id producing no audio request

## 2. Reader self-heal + re-leak call sites

- [ ] 2.1 Add a `useBibleStore.subscribe` guard that, once
  `translations` is non-empty, resets a product-shaped
  unresolvable `activeTextFilesetId` to the locale default
  (`DEFAULT_TRANSLATION_BY_LOCALE[resolveUiLanguage()]` — NOT
  `resolveInitialTextFileset()`, which returns the persisted
  broken value) and fires one notification (new i18n key in
  en/lv/ru); kick the same check once after store creation —
  the vanilla subscriber misses hydration — verify a store
  test covers reset-on-load and no-refire
- [ ] 2.2 Plug the two `?? storedId`-style re-leaks:
  `resolveNotesFilesetId` (`src/stores/bibleStore/
  notesSlice.ts`) returns `undefined` on null resolution, and
  `useVerseTexts` skips the fetch on null — verify tests show
  no `fileset_id=` request carrying a `:`-id from either path

## 3. Note-ref chain + settings page (sibling-dependent)

- [ ] 3.1 In `useNoteRefFilesetId` (from
  add-note-ref-translation-setting), skip ids that resolve to
  `null` so an unresolvable pin falls through to the
  selection and an unresolvable selection falls to ESV —
  verify hook tests cover both skips
- [ ] 3.2 On `SettingsRoute`, render a
  "pinned translation unavailable" state with a clear action
  when the stored pin is product-shaped and unresolvable —
  verify a settings test shows the state and the reset
- [ ] 3.3 Run `npm run lint` (0/0) and the 79-char check on
  touched files — verify clean

## Workflow follow-up

- Archive the change once the staging PR merges cleanly.
