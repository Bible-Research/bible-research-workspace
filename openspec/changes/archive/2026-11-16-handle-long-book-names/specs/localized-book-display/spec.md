# Spec Delta

## ADDED Requirements

### Requirement: Book picker wraps long names

Book rows in the picker SHALL wrap names that exceed the row
width across multiple lines — names SHALL NOT be truncated or
clipped in the picker. Row height SHALL be content-driven.

#### Scenario: Long Latvian name wraps

- **WHEN** the picker renders `Jāņa evaņģēlijs` wider than
  the column
- **THEN** the name wraps to a second line and remains fully
  readable

### Requirement: Bottom bar pins the chapter, scrolls the name

The bottom-bar passage title SHALL keep the chapter number at
a fixed position on the right — always visible and never
moving. When the book name overflows its container it SHALL
scroll horizontally on a slow, continuous loop: scroll until
the whole name is revealed, hold briefly, then repeat.

#### Scenario: Overflowing name marquees

- **WHEN** the active passage is `Jāņa evaņģēlijs 1` and the
  name exceeds the available width
- **THEN** the name scrolls left until fully revealed, holds,
  and restarts, while `1` stays fixed at the right edge

#### Scenario: Short name stays still

- **WHEN** the book name fits its container
- **THEN** no scroll animation runs

### Requirement: Scroll is opt-out and motion-aware

A persisted preference SHALL control the marquee (default
on); `prefers-reduced-motion` SHALL also disable it. When
disabled, the name SHALL render clipped (chapter still
pinned) rather than animating. The preference SHALL be wired
to the store so a future settings toggle only needs UI.

#### Scenario: Reduced motion honored

- **WHEN** the OS/browser reports `prefers-reduced-motion`
- **THEN** the title renders statically regardless of the
  stored preference

#### Scenario: Preference disables scrolling

- **WHEN** the persisted marquee preference is off
- **THEN** the overflowing name renders clipped with no
  animation; `1` stays pinned
