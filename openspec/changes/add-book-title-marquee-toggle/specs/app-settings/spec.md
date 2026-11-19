# Spec Delta

## ADDED Requirements

### Requirement: Book-title marquee preference

The settings page SHALL expose a toggle for the bottom-bar
book-title marquee. The preference persists (default on);
turning it off collapses the overflowing book name to a
clipped label, matching the `prefers-reduced-motion`
behavior.

#### Scenario: Toggle off clips the book name

- **WHEN** the user turns the marquee toggle off and views a
  chapter whose book name overflows the bottom bar
- **THEN** the name renders clipped — no scrolling marquee

#### Scenario: Toggle persists

- **WHEN** the user changes the toggle and reloads the app
- **THEN** the bottom bar honors the stored choice

#### Scenario: Reduced motion always wins

- **WHEN** `prefers-reduced-motion` is set and the toggle is
  on
- **THEN** the book name still renders clipped
