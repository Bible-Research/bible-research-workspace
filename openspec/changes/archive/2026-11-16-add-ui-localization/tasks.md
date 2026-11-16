# Tasks

## 1. i18n foundation

- [x] 1.1 frontend: add `i18next` + `react-i18next` deps
  (pinned, >=7 days old) and create `src/i18n/` —
  `index.ts` init + `locales/{en,lv,ru}/common.json` +
  `books.json` stubs; verify `npm run build` compiles and
  lazy locale chunks emit
- [x] 1.2 frontend: wire init in `main.tsx` (sync `en`,
  Suspense-safe lazy locales) and `setupTests.ts`
  (default `en`); verify existing test suite still finds
  elements via title hooks and text queries
- [x] 1.3 frontend: type-augment resources from the `en`
  catalog so missing keys fail `tsc`; verify `tsc` errors on
  a deliberately missing `lv` key in a scratch check

## 2. Language state

- [x] 2.1 frontend: add persisted `uiLanguage` (BCP-47) to
  the store — `bible-storage` v6 -> v7 migration; verify
  migration test covers missing field -> `en` and existing
  fields preserved
- [x] 2.2 frontend: detection — persisted choice >
  `navigator.language` prefix match > `en`; verify unit test
  covers `lv-LV` -> `lv`, `de-DE` -> `en`, persisted `ru`
  wins over navigator
- [x] 2.3 frontend: `DEFAULT_TRANSLATION_BY_LOCALE`
  (`en`->`ENGESV_API`, `lv`->`LVSGLU8`, `ru`->`RUSSYN`)
  applied only when no persisted `activeTextFilesetId`;
  verify first-run `ru` resolves `RUSSYN` and a persisted
  selection is never overridden
- [x] 2.4 frontend: language selector in MainMenu (autonyms
  `English`/`Latviešu`/`Русский`) writing `uiLanguage`;
  verify switching swaps rendered chrome strings in a
  component test

## 3. String extraction

- [x] 3.1 frontend: localize the `/getting-started` tutorials
  page — `TUTORIAL_VIDEOS` switches to per-locale
  `youtubeIds` + `t('tutorials.<id>.title'/'.description')`
  keys; route resolves `youtubeIds[uiLanguage] ?? .en` and
  hides cards with no id anywhere; `YouTubeEmbed` strings
  localized; verify tests assert lv-id preferred, en
  fallback, and hidden unrecorded cards
- [x] 3.2 frontend: extract component/route strings
  (labels, placeholders, titles, alerts) to `common.json`
  keys for `en`; verify no literal user-facing strings
  remain in touched files (`rg` sweep) and snapshots/tests
  pass under `en`
- [x] 3.3 frontend: map `error_code` ->
  `errors.<error_code>` keys for coded errors
  (`rate_limited`, `book_not_in_fileset`, `not_found`,
  `provider_error`, `internal_error`) incl.
  `notifyAudioSubstitution` + mismatch hints; verify a `lv`
  UI renders the Latvian notification for a mocked
  `rate_limited` body
- [x] 3.4 frontend: count-bearing strings use i18next plural
  forms; verify a `ru` count renders `few`/`many` forms in a
  unit test
- [x] 3.5 frontend: author `lv` + `ru` catalogs for all `en`
  keys; verify a completeness check (en-key diff vs each
  locale) reports zero missing keys

## 4. Integration

- [x] 4.1 frontend: scoped verification — `vitest` on touched
  files, `npm run lint` (0/0), `npm run build`; all green
- [x] 4.2 frontend: update `reactive-bible/AGENTS.md`
  (layout, conventions, i18n layer, locale list, how to add a
  language) + the three-locale rule for tutorial content;
  update workspace `AGENTS.md` with the
  `reactive-bible/docs/video-scripts/getting-started.<locale>.md`
  convention;
  update `DEVELOPER_GUIDE.md`; verify docs name `src/i18n/`
  and the tutorials parity rule
