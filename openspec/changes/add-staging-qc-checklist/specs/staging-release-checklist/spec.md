# Spec Delta

## Purpose

Gives the release tester an in-app list of pull requests merged
into `staging` but not yet promoted to `main`, each with the
PR's own UI-verification steps rendered as a checklist — the
"what still needs clicking through before release" view.

## ADDED Requirements

### Requirement: Preview-only entry point
The app SHALL show a "What's on staging" menu entry in the main
menu only on Vercel preview deploys (`VITE_APP_ENV` =
`'preview'`). Production and Android builds MUST NOT render it.

#### Scenario: Staging preview shows the entry
- **WHEN** user opens the main menu on a deploy whose build
  environment is a Vercel preview
- **THEN** the menu shows the "What's on staging" entry

#### Scenario: Production hides the entry
- **WHEN** user opens the main menu on the production deploy or
  inside the Android app
- **THEN** the menu does not show the entry

### Requirement: Pending-promotion list
Opening the entry SHALL display the set of pull requests merged
into `staging` that have not been promoted to `main`: the
latest merged `staging` PR per head branch, excluding branches
with a merged `main` PR at the same or a later point. Each item
shows the PR title and number and links to the PR on GitHub.

#### Scenario: Un-promoted PR is listed
- **WHEN** a PR was squash-merged into `staging` and no `main`
  PR from its head branch has merged since
- **THEN** the checklist contains that PR's title and number

#### Scenario: Promoted PR is excluded
- **WHEN** a `main` PR from the PR's head branch has merged
  after the `staging` merge
- **THEN** the checklist does not contain that PR

#### Scenario: Empty after a clean release
- **WHEN** every staged branch has been promoted (staging
  matches main)
- **THEN** the view shows an explicit "nothing pending" state,
  not an error

### Requirement: Verification checklist per PR
For each listed PR the view SHALL render the steps from the PR
body's `## UI verification / QC` section as a human-readable,
individually checkable list. Checkbox state MAY be
session-local and MUST NOT affect other users.

#### Scenario: Steps render as checkboxes
- **WHEN** a listed PR's body contains a `## UI verification /
  QC` section
- **THEN** its steps are shown as tickable checklist items
  under the PR title

#### Scenario: PR without a verification section
- **WHEN** a listed PR's body has no `## UI verification / QC`
  section
- **THEN** the item shows a "see PR body" affordance linking to
  the PR instead of an empty checklist

### Requirement: Failure and fallback
On GitHub API failure (network error or rate limit) the view
SHALL show an error state with a link to the GitHub
`main...staging` compare view of the repository.

#### Scenario: Rate-limited fetch
- **WHEN** the GitHub API returns an error or is unreachable
- **THEN** the view reports it could not load the list and
  offers the compare-view link as a manual alternative
