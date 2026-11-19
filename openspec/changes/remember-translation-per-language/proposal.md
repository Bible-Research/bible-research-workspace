# Proposal

## Why

The translation selector forgets the user's per-language choices
the moment they switch language tabs. A user who reads ESV in
English and GLÜK in Latvian must re-pick the version from scratch
every time they browse back to a previously used language — the
picker only mirrors the single globally active fileset pair and
has no memory of what was used in each language.

## What Changes

- The client persists the last saved translation selection per
  language (keyed by the translation's `language_iso`), alongside
  the existing active fileset ids.
- When the user switches language inside the translation selector
  modal (a fixed tab or an Other-language pick), the version
  picker pre-selects the remembered translation for that language
  and stages its text/audio fileset ids — applied only on Save,
  matching today's staged-selection semantics.
- A language with no remembered selection keeps today's behavior
  (empty picker / mirror of the active selection when it belongs
  to that language); a remembered translation that vanished from
  the fetched list falls back the same way.
- Persisted store schema bumps (v9 → v10) with a migration that
  sanitizes the new per-language map.

## Capabilities

### New Capabilities

- `translation-selector`: How the translation selector modal
  stages a version choice per language and restores the last used
  translation when the user returns to a language — covering
  persistence of the per-language memory, restore-on-switch
  behavior, and fallback when nothing is remembered or the
  remembered version is gone.

### Modified Capabilities

## Impact

- **Repo**: `reactive-bible/` (frontend) only — no backend or
  cross-repo contract changes.
- `src/components/TranslationSelector.tsx` — restore remembered
  selection on language change; record on Save.
- `src/store.ts` + `src/stores/bibleStore/persistence.ts` — new
  persisted field, version bump, migration.
- Tests around the selector's staged-selection effects.
