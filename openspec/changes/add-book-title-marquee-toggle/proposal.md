# Proposal

## Why

`bookTitleMarquee` is a persisted preference (default on) whose
settings UI was explicitly deferred — the store comment says
"a future settings toggle only writes that one field". Once
`add-note-ref-translation-setting` delivers the `/settings`
page, the toggle finally has a home.

## What Changes

- The settings page gains a "Book title marquee" toggle bound
  to the existing persisted `bookTitleMarquee` field. OFF
  collapses the BottomNav book name to a clipped label
  (existing behavior — `prefers-reduced-motion` already
  produces it); ON restores the reveal-then-restart marquee.
- Depends on `add-note-ref-translation-setting` landing first
  (it creates the `/settings` page and menu entry).

## Capabilities

### New Capabilities

<!-- None — extends the app-settings capability. -->

### Modified Capabilities

- `app-settings`: adds the requirement that the settings page
  exposes the book-title marquee preference.

## Impact

- `reactive-bible` only: `src/routes/SettingsRoute.tsx` (new
  control), i18n keys. No store/schema changes — the field is
  already persisted; no backend changes.
