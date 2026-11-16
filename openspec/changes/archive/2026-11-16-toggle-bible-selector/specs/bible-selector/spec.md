# Spec Delta

## Purpose

Defines how the Bible selector panel (the book/chapter/verse
picker in the Bible view) opens and closes, and guarantees that
dismissing it without a selection never changes the passage on
screen.

## ADDED Requirements

### Requirement: Passage button toggles selector visibility

The bottom-bar current-passage button (`{book} {chapter}`) SHALL
act as a toggle for the Bible selector: it opens the panel when
the selector is closed and closes the panel when the selector
is already open.

#### Scenario: Open from closed state

- **WHEN** the user presses the current-passage button while
  the selector is closed
- **THEN** the selector opens on the current book and chapter

#### Scenario: Close from open state

- **WHEN** the user presses the current-passage button while
  the selector is open
- **THEN** the selector closes

### Requirement: Toggle-close preserves the current passage

Closing the selector via the current-passage button SHALL NOT
change the displayed passage: the active book, chapter, and any
selected verses remain exactly as they were before the press.

#### Scenario: No navigation on close

- **WHEN** the user is viewing `/bible/JHN.3` and presses the
  current-passage button to close the open selector
- **THEN** the view still shows John chapter 3 with the same
  URL and verse selection

### Requirement: Selection still navigates and closes

Choosing a chapter or verse inside the selector SHALL keep its
existing behavior: navigate to that passage and close the
panel; the toggle affordance does not replace or alter it.

#### Scenario: Chapter pick navigates

- **WHEN** the user picks a different chapter in the open
  selector
- **THEN** the app navigates to that chapter and the selector
  closes
