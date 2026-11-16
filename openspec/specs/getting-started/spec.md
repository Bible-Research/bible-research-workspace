# getting-started Specification

## Purpose
Gives new users a self-serve onboarding path: a menu entry opening a
page of short embedded YouTube tutorials that teach the app's
features.

## Requirements

### Requirement: Menu entry for getting started

The main menu SHALL contain a "Getting started" item, placed above
"Report a problem" near the bottom of the drawer, visible to both
anonymous and authenticated users. Activating it SHALL navigate to
the getting-started route and close the menu.

#### Scenario: Anonymous user opens the page
- **WHEN** an anonymous user opens the main menu and activates
  "Getting started"
- **THEN** the app navigates to the getting-started route and the
  menu closes

#### Scenario: Menu placement
- **WHEN** any user opens the main menu
- **THEN** "Getting started" appears above "Report a problem" and is
  not hidden behind any authentication or platform gate

### Requirement: Public getting-started route

The getting-started route SHALL be reachable without
authentication and SHALL NOT redirect anonymous users elsewhere.
The page SHALL scroll when its content exceeds the viewport.

#### Scenario: Anonymous visit
- **WHEN** an anonymous user navigates directly to
  `/getting-started`
- **THEN** the tutorials page renders without a login redirect

### Requirement: Curated video catalogue

The page SHALL render one card per catalogue entry that has a
YouTube video id, in catalogue order, each showing at least the
video title. Entries without a video id SHALL NOT be rendered, so
the page can ship before all videos are recorded.

#### Scenario: Mixed catalogue
- **WHEN** the catalogue contains entries both with and without a
  YouTube video id
- **THEN** only the entries with a video id render as cards, in
  catalogue order

### Requirement: Click-to-play embedded player

Each rendered video SHALL initially show a thumbnail with a
labelled play control instead of loading the YouTube player. The
player iframe SHALL load only after the user activates the
control, using the privacy-enhanced `youtube-nocookie.com` domain
with autoplay. The embed SHALL maintain a 16:9 aspect ratio at any
viewport width.

#### Scenario: No eager player load
- **WHEN** the getting-started page renders
- **THEN** no YouTube player iframe is present in the document

#### Scenario: Activation
- **WHEN** the user activates a video's play control
- **THEN** the thumbnail is replaced by a player iframe whose
  source is a `youtube-nocookie.com` embed URL for that video with
  autoplay enabled

### Requirement: External watch link

Each rendered video card SHALL provide a link that opens the same
video on `youtube.com` in a new browser context, so users can watch
outside the app — including on the Android shell, which cannot
fullscreen embedded video.

#### Scenario: Watch on YouTube
- **WHEN** a user activates a card's "Watch on YouTube" link
- **THEN** the video's `youtube.com` watch URL opens in a new tab
  or the external browser
