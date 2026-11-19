# Proposal

## Why

Scripture-reference previews in notes and comments always render
the ESV text (`ScripturePassage` hardcodes `ENGESV_API`), no
matter which translation the user reads. A Latvian reader sees
English verse text in every `@…` preview, and there is no way to
pin previews to a different translation. The app also has no
Settings surface — the persisted `bookTitleMarquee` preference
already waits for one.

## What Changes

- `ScripturePassage` stops hardcoding `ENGESV_API` and resolves
  the preview fileset per render: an explicit user override
  wins, otherwise the active reader text fileset
  (`activeTextFilesetId`, resolved per-book via
  `resolveTextFileset` so `_ET` testament splits and the offline
  `ENGKJV` bundle keep working). `ENGESV_API` remains only the
  last-resort fallback when nothing resolves.
- New persisted store field `noteRefFilesetId: string | null`
  (`null` = follow the selected translation). Persisted schema
  bumps to v10 with a sanitize-on-migrate rule (non-string
  values drop to `null`).
- New `/settings` route (public — it only writes local
  preferences) with a "Note reference translation" control:
  - "Selected translation" (default) — previews follow
    `activeTextFilesetId`;
  - "Specific translation" — opens the existing
    `TranslationSelector` flow in a pick-only mode that returns
    a text fileset id instead of overwriting the reader's
    active filesets. The chosen translation's name/abbr shows
    next to the control; a "reset to selected" path clears the
    override.
- The burger menu (`MainMenu`) gains a "Settings" entry that
  navigates to `/settings` and closes the drawer.
- `TranslationSelector` gains a picker mode (props-controlled)
  so the settings page can reuse the full
  language → version → text-fileset flow without its Save
  writing `activeTextFilesetId`/`activeAudioFilesetId`.
- i18n keys for the menu entry, the settings page and the new
  control are added to `en` (source catalog) plus `lv`/`ru`.

## Capabilities

### New Capabilities

- `app-settings`: the Settings surface — a `/settings` route
  reachable from the main menu that edits persisted,
  device-local user preferences (first preference: the note
  reference translation override).
- `note-ref-translation`: which translation scripture-reference
  previews fetch passage text in — the
  override → active-selection → fallback resolution order and
  how the override is chosen and cleared.

### Modified Capabilities

<!-- None: provider-routing, error-contract, book-canon and
     ui-localization behavior are unchanged — the preview simply
     consumes the same fileset-id routing contract as the
     reader. -->

## Impact

- `reactive-bible` only — no backend or API-contract changes.
- Code: `src/components/ScripturePassage.tsx`,
  `src/components/TranslationSelector.tsx` (picker mode),
  `src/components/MainMenu.tsx`, `src/App.tsx` (overflow
  whitelist for `/settings`), `src/routes/index.tsx`,
  `src/store.ts` + `src/stores/bibleStore/persistence.ts`
  (v10 migration), new `src/routes/SettingsRoute.tsx`,
  `src/i18n/locales/{en,lv,ru}/common.json`.
- Tests: store persistence migration, `ScripturePassage`
  fileset resolution, `MainMenu` settings entry, settings page
  behavior, `TranslationSelector` picker mode.
- Docs: `reactive-bible/AGENTS.md` + `DEVELOPER_GUIDE.md`
  (settings surface + preview fileset contract).
