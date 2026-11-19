# Spec Delta

## Purpose

Gives users a Settings surface — reachable from the main menu —
where device-local preferences are viewed and changed. Settings
persist across reloads and need no account.

## ADDED Requirements

### Requirement: Settings entry point

The main menu SHALL contain a "Settings" entry that navigates
to a dedicated settings page. The entry SHALL be available to
anonymous and authenticated users alike.

#### Scenario: Menu entry opens the settings page

- **WHEN** the user opens the main menu and taps "Settings"
- **THEN** the settings page renders and the menu closes

#### Scenario: Anonymous visitor can open settings

- **WHEN** a visitor who is not logged in opens the main menu
- **THEN** the "Settings" entry is present and opens the page

#### Scenario: Entry sits with the navigation items

- **WHEN** the main menu renders
- **THEN** "Settings" appears among the navigation entries,
  before the theme/language block

### Requirement: Preferences persist locally

Every preference edited on the settings page SHALL persist
across reloads on the same device without requiring an account.
A preference never edited SHALL keep its documented default.

#### Scenario: Choice survives reload

- **WHEN** a user changes a setting and reloads the app
- **THEN** the settings page and the affected behavior reflect
  the stored choice

#### Scenario: Untouched setting keeps its default

- **WHEN** a user opens the settings page having never changed
  a given preference
- **THEN** the control shows the documented default value

### Requirement: Settings are device-local

Settings stored by this page SHALL NOT sync to the backend or
across devices, and changing them SHALL NOT require
authentication.

#### Scenario: Settings work fully offline of an account

- **WHEN** an anonymous user sets a preference
- **THEN** no authenticated request is needed for the choice
  to take effect and persist
