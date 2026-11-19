# Tasks

All work is in `reactive-bible/` (frontend only). Repo rules:
worktree off `origin/main`, PR `--base staging`, ≤79 chars/line,
scoped `npm test`/`npm run lint` verification only.

## 1. Resolver hardening

- [ ] 1.1 In `src/utils/filesetGroups.ts`, make
  `resolveTextFileset` return `null` when the stored id
  contains `:` and `findTextOption` misses all lookups; keep
  pass-through for non-product ids — verify
  `filesetGroups.test.ts` gains cases: orphaned product id →
  null, unknown raw id → passthrough

## 2. Reader self-heal

- [ ] 2.1 Add a `useBibleStore.subscribe` guard that, once
  `translations` is non-empty, resets a product-shaped
  unresolvable `activeTextFilesetId` to
  `resolveInitialTextFileset()` and fires one notification
  (new `translations.*`/`errors.*` i18n key in en/lv/ru) —
  verify a store test covers reset-on-load and no-refire

## 3. Note-ref chain + settings page

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
