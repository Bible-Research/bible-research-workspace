# Tasks

## 1. Persisted per-language memory

- [x] 1.1 frontend: Add `lastSelectionByLanguage` (Record of
  `language_iso` → `{ abbr, textId, audioId }`) to BibleState
  plus a setter in `src/store.ts`
- [x] 1.2 frontend: Persist the field in `partializeState`, bump
  the store version 9 → 10, and add a `version < 10` migration
  sanitizing entries in
  `src/stores/bibleStore/persistence.ts`

## 2. Selector record + restore

- [x] 2.1 frontend: In `TranslationSelector.tsx` `handleSave`,
  record the saved `{ abbr, textId, audioId }` under the saved
  translation's `language_iso`
- [x] 2.2 frontend: On language switch, after the language's
  list is merged and no active-owner mirror applied, stage the
  recorded entry when its `abbr` still exists in `translations`
  — guarded by `userInteractedRef` so a fresh user pick wins

## 3. Tests and verification

- [x] 3.1 frontend: Extend `TranslationSelector.test.tsx` (and
  persistence tests if present) — record-on-save, restore on
  tab switch, no overwrite on cancel, stale-`abbr` fallback,
  v9 → v10 migration
- [x] 3.2 frontend: Run the touched Vitest files,
  `npm run lint` (0 errors/0 warnings), `tsc` via
  `npm run build`; verify lines ≤ 79 chars
- [x] 3.3 frontend: Update `AGENTS.md`/`DEVELOPER_GUIDE.md`
  notes that mention the persisted store shape (schema
  version) if they now drift
