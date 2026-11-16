# Design

## Context

~400 user-facing English strings are hardcoded across ~70
components (labels, placeholders, `title` tooltips,
notification bodies). No i18n library exists. The store
persists `bible-storage` at schema v6 with
`migratePersistedState`. TranslationSelector already tabs
`eng`/`lvs`/`rus` for *Bible content* — orthogonal to UI
locale. Default text fileset today: `ENGESV_API`.

## Goals / Non-Goals

**Goals:**

- `en`/`lv`/`ru` UI catalogs with English fallback, CLDR
  plural support, lazy-loaded non-English catalogs.
- Persisted `uiLanguage` (BCP-47) + first-run
  `navigator.language` detection.
- Locale-matched default translation on first run.
- Localized notifications incl. `error_code` -> key mapping.

**Non-Goals:**

- Localized book names per translation — the
  `add-bible-structure` change owns that (its bundled
  `books.json` catalogs live in the same `locales/<lang>/`
  dirs; this change only provides `en`-catalog book names for
  surfaces the structure change doesn't reach).
- Translating stored content (notes, tags) or backend error
  `message` text.
- RTL support, locale-aware date formatting (may come later).

## Decisions

### react-i18next over Lingui/react-intl/custom

CLDR plural rules are the forcing factor (`lv`
zero/one/other, `ru` one/few/many) — hand-rolled dictionaries
get them wrong. react-i18next is the React standard:
`useTranslation`, `<Trans>` for markup-bearing strings,
per-locale JSON catalogs, mature TS typing. Locale catalogs
lazy-load via `import()` — `en` stays in the main chunk.

### BCP-47 internally, DBT iso at the boundary

`uiLanguage` uses `en`/`lv`/`ru`; DBT's `eng`/`lvs`/`rus`
appear only when mapping locale -> `language_iso` for
translation defaults (same pattern as `fileset_id` routing).

### Detection + persistence

Init order: persisted `uiLanguage` -> `navigator.language`
prefix match (`lv-LV` -> `lv`) -> `en`. Persisted choice
lives in `bible-storage` (v8 -> v9 migration adds
`uiLanguage`); detection runs only when the field is absent.

### Curated default-translation map

`DEFAULT_TRANSLATION_BY_LOCALE = {en: 'ENGESV_API',
lv: 'LVSGLU8', ru: 'RUSSYN'}` — each entry verified
full-canon (RUSSYN `text_plain` size `C`; LVSGLU8 Glück
complete canon; ENGESV_API is today's default). Applied only
on first run (no persisted `activeTextFilesetId`); unknown
locales get `en`. Rejected "best-coverage search at runtime":
determinism beats discovery — coverage was verified at
authoring time against live DBT.

### Selector placement

MainMenu next to the Theme toggle (locale isn't
account-scoped — UserMenu is auth-gated). Options labeled
with autonyms: `English`, `Latviešu`, `Русский`.

### Tutorial catalogue localization

`TUTORIAL_VIDEOS` (`src/data/gettingStartedVideos.ts`, added
in #247) keeps `{id, youtubeIds}` where `youtubeIds` is
`{en?, lv?, ru?}` — null/absent = not yet recorded in that
language. `title`/`description` leave the data file for
catalog keys `tutorials.<id>.title` / `.description` (typed
via the `en` resource). The route resolves the embed id as
`youtubeIds[uiLanguage] ?? youtubeIds.en`; renders the card
only when the resolved id exists. `YouTubeEmbed`'s own
strings (play label, "Watch on YouTube") localize like
everything else.

Why per-locale ids instead of one id: narration is recorded
per language (scripts at `docs/video-scripts/getting-started.
<locale>.md` in reactive-bible), so the embed id
itself is locale data — while card visibility keys on the
catalogue entry, not the id.

Three-locale rule for agents: written into
`reactive-bible/AGENTS.md` (tutorials page) and the workspace
`AGENTS.md` (docs/video-scripts convention) — tutorial content
changes ship en+lv+ru in the same PR.

### error_code -> key map

Coded errors render `t('errors.<error_code>')`
(`rate_limited`, `book_not_in_fileset`, `not_found`,
`provider_error`, `internal_error`); uncoded failures keep
existing generic keys. Raw provider `error`/`message` text
never renders for coded errors.

### Typed keys

`react-i18next` TS augmentation over the `en` resource type —
new locales fail typecheck on missing keys; extraction adds
keys to `en` first.

## Risks / Trade-offs

- ~400-string extraction is a wide diff -> single focused PR,
  mechanical `t()` replacement; test hooks stay code-keyed so
  the 953-test suite still finds elements.
- Partial `lv`/`ru` coverage -> English fallback per key is
  acceptable; catalogs are append-only per locale.
- i18next init before first render -> init in `main.tsx`
  synchronously for `en`; non-en locales load via Suspense-
  safe lazy resources.
- Notification strings interpolated with book labels ->
  book labels localize via the structure change's display
  layer, not this catalog.

## Migration Plan

1. Frontend PR: i18n module + en catalog + extraction pass +
   selector + detection/defaults + tests.
2. Rollback: additive — revert PR; persisted `uiLanguage` is
   ignored by older builds.

## Open Questions

- Namespace split (one `common.json` vs per-domain files) —
  start with `common` + `books`; split if a catalog grows
  unwieldy.
