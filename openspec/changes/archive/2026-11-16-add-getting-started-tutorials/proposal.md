# Proposal

## Why

New users have no in-app guidance: the app has many non-obvious
features (auto-hiding chrome, verse selection toolbar, per-fileset
audio options, notes/tags/comments behind auth) and nothing that
teaches them. A "Getting started" page of short embedded YouTube
tutorials gives users a self-serve onboarding path directly from the
main menu.

## What Changes

- Add a public `/getting-started` route rendering a list of video
  tutorial cards (title, description, embedded player).
- Add a "Getting started" item to the main menu drawer, placed near
  the bottom above "Report a problem", visible to anonymous users.
- Add a click-to-play YouTube embed (thumbnail facade that swaps in
  the iframe on interaction) so the page does not eagerly load the
  YouTube player for every listed video.
- The video catalogue is a static, typed in-repo list; entries
  without a YouTube id are not rendered, so the page ships before
  any video is recorded and grows as ids are added.
- Each card offers a "Watch on YouTube" external link (also the
  escape hatch on the Android shell, which cannot fullscreen
  embedded video).

## Capabilities

### New Capabilities

- `getting-started`: The getting-started tutorials surface — its
  menu entry, the public route, the static video catalogue, and the
  click-to-play YouTube embed behavior.

### Modified Capabilities

(none — no existing spec-level behavior changes)

## Impact

- **Repo:** `reactive-bible` only (frontend). No backend or
  cross-repo contract changes.
- New files: a route component, a YouTube embed component, a typed
  video-catalogue data module, and their tests.
- Modified: `src/routes/index.tsx`, `src/components/MainMenu.tsx`,
  `src/App.tsx` (scrollable-route whitelist), `MainMenu.test.tsx`,
  `DEVELOPER_GUIDE.md`, `README.md`, repo `AGENTS.md`.
- Third-party surface: `youtube-nocookie.com` iframes and
  `i.ytimg.com` thumbnails, loaded only on user interaction; no new
  npm dependencies.
- Capacitor/Android: inline playback works; fullscreen is a known
  no-op in the stock shell — mitigated by the "Watch on YouTube"
  link, not fixed in this change.
