# Proposal

## Why

Localized book names are long (`Jāņa evaņģēlijs`,
`1. Mozus grāmata`, and longer in other languages). The book
picker renders single-line rows that truncate/overflow, and
the bottom-bar passage title (`{book} {chapter}`) would push
the chapter out of view — losing the one piece of context the
reader always needs.

## What Changes

- PassagePicker book rows wrap long names across lines
  instead of truncating (row height becomes content-driven,
  `nav-book-{code}` test hooks unchanged).
- BottomNav passage title splits into two zones: the book
  name occupies the flexible width and, when it overflows,
  scrolls horizontally on a slow continuous loop (whole name
  revealed, brief hold, repeat); the chapter number stays
  pinned at a fixed position on the right — always visible,
  never scrolling.
- Non-overflowing names render statically (no motion).
- `prefers-reduced-motion` and a new persisted preference
  (`bookTitleMarquee`, default on) disable the animation;
  disabled state falls back to a clipped name — the settings
  toggle UI is deliberately deferred, only the store seam
  ships now.
- Frontend only; depends on `add-bible-structure`'s
  localized display layer.

## Capabilities

### New Capabilities

(none)

### Modified Capabilities

- `localized-book-display`: adds long-name presentation
  requirements — picker wrapping, bottom-bar marquee with a
  pinned chapter, motion preferences, disable seam.

## Impact

- **Frontend** (`reactive-bible`): `PassagePicker` row
  styles, `BottomNav` title markup + marquee component, a
  persisted pref in the store (schema bump), tests.
- **Backend** (`bible_research`): none.
- **Specs**: delta on `localized-book-display`.
