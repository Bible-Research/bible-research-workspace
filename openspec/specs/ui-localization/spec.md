# ui-localization Specification

## Purpose
Defines how the app localizes its user interface: which
locales ship, how the UI language is detected, chosen and
persisted, how strings resolve and fall back, and which
default Bible translation each locale starts on.

## Requirements

### Requirement: Supported locales and fallback

The UI SHALL ship locale catalogs for `en`, `lv` and `ru`
(BCP-47 codes), keyed by semantic dotted keys, with `en` as
the source and fallback catalog. A missing key in a
non-English catalog SHALL render the English string.

#### Scenario: Missing Latvian key falls back

- **WHEN** a `lv` catalog entry is absent for a key present in
  `en`
- **THEN** the English string renders — never the raw key

### Requirement: UI language detection and persistence

On first run the UI language SHALL resolve from
`navigator.language` to a shipped locale when one matches,
else `en`; once the user picks a language the choice SHALL be
persisted and SHALL win over detection on later runs.

#### Scenario: Latvian browser auto-detects

- **WHEN** a fresh install runs with `navigator.language`
  `lv-LV`
- **THEN** the UI renders the `lv` catalog without the user
  touching the selector

#### Scenario: Unsupported browser locale

- **WHEN** a fresh install runs with `navigator.language`
  `de-DE`
- **THEN** the UI renders the `en` catalog

#### Scenario: Explicit choice persists

- **WHEN** a user sets the UI language to Russian, then
  restarts the app
- **THEN** the UI renders `ru` regardless of
  `navigator.language`

### Requirement: UI language is independent of translation

Selecting a Bible translation SHALL NOT change `uiLanguage`,
and changing `uiLanguage` SHALL NOT change the selected
translation; the two settings are independent.

#### Scenario: Foreign text under English UI

- **WHEN** a user with `uiLanguage` `en` selects the Latvian
  Glück translation
- **THEN** menus and chrome stay English while book names
  follow the translation's language

### Requirement: Locale-matched default translation

A first-run detected locale SHALL also select the curated
default Bible translation for that locale: `en`->`ENGESV_API`,
`lv`->`LVSGLU8`, `ru`->`RUSSYN`. Each mapped entry SHALL be a
full-canon trusted translation verified at authoring time;
unsupported locales use the `en` default.

#### Scenario: Russian first run

- **WHEN** a fresh install detects a Russian locale
- **THEN** the default text fileset resolves to `RUSSYN`
  (1876 Synodal, full canon)

### Requirement: Plural-aware rendering

Strings carrying counts SHALL use the i18n library's CLDR
plural categories per locale (`lv`: zero/one/other; `ru`:
one/few/many) — never string concatenation of count + noun.

#### Scenario: Russian plural forms

- **WHEN** a count of 2 renders under `ru`
- **THEN** the `few` form is used, and a count of 5 uses
  `many`

### Requirement: Tutorial catalogue is locale-aware

The getting-started tutorial catalogue SHALL localize card
titles and descriptions through catalog keys, and SHALL carry
a video id per shipped locale (`en`/`lv`/`ru`) — each locale
gets its own recorded videos. A card SHALL play the active
`uiLanguage` video when present, else the `en` video; a card
with no video id in any locale SHALL NOT render.

#### Scenario: Latvian UI prefers Latvian video

- **WHEN** a tutorial has `youtubeIds` for `lv` and `en` and
  `uiLanguage` is `lv`
- **THEN** the card plays the `lv` video with the `lv`
  title/description

#### Scenario: Missing locale video falls back to English

- **WHEN** a tutorial has an `en` video but no `ru` video and
  `uiLanguage` is `ru`
- **THEN** the card renders `ru` text but embeds the `en`
  video

#### Scenario: Unrecorded tutorial stays hidden

- **WHEN** a tutorial has no video id in any locale
- **THEN** its card does not render on the page

### Requirement: Tutorial content ships all shipped locales

Changes to tutorial content — catalogue entries, card
strings, or the narration scripts — SHALL include all shipped
UI locales in the same change; an update that only adds one
locale's text is incomplete. Agent-facing docs SHALL state
this rule.

#### Scenario: New tutorial entry

- **WHEN** a tutorial entry is added or its strings change
- **THEN** `en`, `lv` and `ru` catalog entries are added in
  the same change, and script updates produce
  `getting-started.<locale>.md` for all three locales

### Requirement: Localized error and notification text

User-visible error and notification text SHALL come from
catalog keys; backend `error_code` values SHALL map to
localized message keys — raw provider `error`/`message` text
SHALL NOT be shown for coded errors.

#### Scenario: Rate limit under Latvian UI

- **WHEN** the API returns `error_code` `rate_limited` under a
  `lv` UI
- **THEN** the notification shows the Latvian catalog message
  for `rate_limited`
