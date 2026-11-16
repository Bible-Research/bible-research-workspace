# Proposal

## Why

The entire UI is hardcoded English (~400 strings across ~70
components). To serve Latvian- and Russian-speaking users — and
any language added later — the app needs a real i18n layer with
a persisted, auto-detected UI language and per-locale string
catalogs.

## What Changes

- Adopt `react-i18next`/`i18next` for UI strings: semantic
  dotted keys, lazy-loaded per-locale JSON catalogs under
  `src/i18n/locales/<lang>/`, `en` as the source/fallback
  catalog. CLDR plural rules come free (critical for `lv`
  zero/one/other and `ru` one/few/many).
- New persisted `uiLanguage` setting (BCP-47 codes: `en`, `lv`,
  `ru`, extensible) in the Zustand store — schema version bump
  + migration; selector in MainMenu labeled with language
  autonyms (Latviešu, Русский).
- First-run locale detection: `navigator.language` → supported
  locale if matched, else `en`. Detection also selects a
  sensible default Bible translation per locale via a curated
  map (pre-verified for full canon): `en->ENGESV_API`
  (current default), `lv->LVSGLU8` (Glück, complete canon),
  `ru->RUSSYN` (1876 Synodal `text_plain` size `C`, all 66
  books). Detected locale with no shipped UI locale falls back
  to `en` + current English default.
- Independent of Bible translation choice: changing translation
  never changes `uiLanguage`; book names already follow the
  translation's language via the separate `bible-structure`
  change.
- Extract all user-facing strings — labels, placeholders,
  titles, notification bodies (incl. `notifyAudioSubstitution`,
  mismatch hints) — into catalogs; `error_code` values map to
  localized notification keys instead of raw English provider
  messages.
- Localize the `/getting-started` tutorials page (added in
  reactive-bible #247): the `TUTORIAL_VIDEOS` catalogue's
  title/description strings move to catalog keys, and
  `youtubeId` becomes per-locale (`{en, lv, ru}`) with `en`
  fallback — each language gets its own recorded videos
  (scripts live in reactive-bible as
  `docs/video-scripts/getting-started.<locale>.md`).
- Rule for future work: any change to tutorial content or the
  instructions page MUST ship all three locales (en/lv/ru) in
  the same PR — written into the spec and AGENTS.md docs.
- Test hooks (`title="nav-book-{code}"` etc.) stay code-keyed —
  locale-independent.
- Extendability: adding a locale = new catalog dir + one entry
  in the supported-locale list (+ default-translation map).

## Capabilities

### New Capabilities

- `ui-localization`: the UI-language contract — supported
  locales, detection + persisted preference, catalog layout and
  fallback behavior, plural-form requirements, localized
  error/notification mapping, and the per-locale default
  translation map.

### Modified Capabilities

(none — provider routing, error contract and auth are
unchanged; `error_code` remains the stable English-keyed
contract clients branch on.)

## Impact

- **Frontend** (`reactive-bible`): new `src/i18n/` module,
  i18next deps, `main.tsx` init, `setupTests` default-`en`
  init, store schema v7 migration, MainMenu selector,
  mechanical `t()` extraction across components/routes/hooks.
- **Backend** (`bible_research`): none — `error`/`error_code`
  and `passage=`/`book` contracts stay English.
- **Specs**: new `ui-localization` spec; no deltas to existing
  specs.
