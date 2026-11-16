# Proposal

## Why

In the Bible view the bottom bar's current-passage button
(`{book} {chapter}`) opens the Bible selector, but pressing it
again does nothing — it calls `setBibleSelectorOpened(true)`
unconditionally. Once the selector is open there is no way to
dismiss it without picking a different chapter/verse or leaving
the view, so a user who opened it by accident (or changed their
mind) is stuck. The button should act as a toggle: a second
press closes the selector and leaves the current passage
untouched.

## What Changes

- The bottom-bar current-passage button toggles the Bible
  selector: it opens the panel when closed and closes it when
  already open, without navigating or altering the current
  book/chapter/verse view.
- No other open/close affordances change — picking a chapter or
  verse still closes the selector and navigates as today.

## Capabilities

### New Capabilities

- `bible-selector`: visibility behavior of the Bible selector
  panel in the Bible view — which affordances open it, which
  close it, and the guarantee that closing via the toggle
  leaves the current passage unchanged.

### Modified Capabilities

(none)

## Impact

- **Frontend** (`reactive-bible`): `App.tsx` (pass selector
  open state or a toggle to `BottomNav`), `BottomNav.tsx`
  (toggle on press), `BottomNav.test.tsx` (toggle coverage).
- **Backend** (`bible_research`): none.
- **Specs**: new capability spec `bible-selector`.
