# Design

## Context

`PassagePicker` book rows are single-line (`height: rem(30)`,
`lineHeight: rem(30)`). `BottomNav` renders one Mantine
`Title` — `{activeBookName} {activeChapter}` — centered
between prev/next arrows. Localized names (`Jāņa evaņģēlijs`,
longer in other languages) overflow both. `activeBookName`
already comes from the localized display layer
(`selectActiveBookName`).

## Goals / Non-Goals

**Goals:**

- Picker rows wrap; bottom-bar chapter pinned right; slow
  looping marquee for overflowing names; persisted opt-out +
  `prefers-reduced-motion` seam.

**Non-Goals:**

- The settings toggle UI itself (a follow-up adds the button
  — the pref is wired and honored now).
- Marquee in other surfaces (audio title, notes headings) —
  those wrap or truncate by existing styles; only the
  bottom bar has the always-visible-chapter constraint.

## Decisions

### BottomNav title: two-zone split

Replace the single `Title` text with a flex row: an
overflow-hidden name container (flex 1, min-width 0) + a
fixed chapter span (flex none, right-aligned). The chapter
never moves regardless of name length or scroll position.

### Marquee via CSS animation, not JS timers

Measure `scrollWidth > clientWidth` to decide whether to
animate; set the keyframe duration proportional to the
overflow (`~40-50px/s` — slow, sub-attention) and the travel
distance via a CSS custom property. Re-measure on
name/container resize (`ResizeObserver` or the store-driven
name change + Mantine `useResizeObserver`). CSS `transform`
keeps it off the main thread; `translateX(-overflow)` →
hold (keyframe pause) → snap back and repeat.

Rejected JS `requestAnimationFrame`/interval loops:
unnecessary main-thread cost, harder to honor
`prefers-reduced-motion`; rejected marquee-by-duplication
(two copies sliding) — the spec wants reveal-then-restart,
not a seamless belt.

### Off states collapse to clip

Pref off / reduced-motion → `text-overflow: ellipsis` on the
name container, chapter still pinned. `title` attr keeps the
full name on hover/tooltips.

### Persisted pref seam

`bookTitleMarquee: boolean` (default `true`) in
`bible-storage` — schema v7 bump + migration (v7 if the
i18n change lands first uses its numbering; coordinate at
implementation). A future settings switch only writes this
field.

## Risks / Trade-offs

- Animation in the always-visible chrome risks distraction
  -> slow rate + hold pause + reduced-motion/pref off-switches.
- Measured width races during font load -> re-measure on
  `document.fonts.ready` / container resize; worst case the
  first animation cycle is slightly mis-sized.
- RTL locales later -> travel direction follows
  `direction`; note as future consideration, no work now.

## Migration Plan

1. Frontend PR only: picker row styles + BottomNav marquee +
   pref + tests.
2. Rollback: additive styles/logic — revert PR.

## Open Questions

- Exact speed/hold timing — tune at implementation
  (`~40-50px/s`, ~1s hold) per the spec's "slow" wording.
