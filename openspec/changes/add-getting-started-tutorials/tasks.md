# Tasks

## 1. Catalogue and embed component

- [x] 1.1 frontend: Create `src/data/gettingStartedVideos.ts`
  exporting `TutorialVideo` (`{id, title, description, youtubeId}`)
  and `TUTORIAL_VIDEOS` seeded with the planned tutorial topics
  (`youtubeId: null` placeholders); verify a new unit test asserts
  unique ids and the `youtubeId` shape when non-null.
- [x] 1.2 frontend: Create `src/components/YouTubeEmbed.tsx` — a
  click-to-play facade (`AspectRatio` 16:9, `UnstyledButton`,
  `i.ytimg.com` thumbnail, play icon) that swaps in a
  `youtube-nocookie.com` iframe (`autoplay=1`, `allow`,
  `allowFullScreen`, `title`) on activation, plus a "Watch on
  YouTube" external link; verify `YouTubeEmbed.test.tsx` shows no
  iframe before activation and the nocookie+autoplay iframe after.

## 2. Route and menu wiring

- [x] 2.1 frontend: Create `src/routes/GettingStartedRoute.tsx`
  (default export) rendering the heading, intro text, and one card
  per catalogue entry that has a `youtubeId`, in order; verify
  `GettingStartedRoute.test.tsx` renders only id'd entries in
  catalogue order.
- [x] 2.2 frontend: Register `/getting-started` as a lazy route via
  `lazyWithReload` in `src/routes/index.tsx` (no `ProtectedRoute`)
  and add the pathname to the `overflow: "auto"` whitelist in
  `src/App.tsx`; verify a route test mounts it anonymously at
  `/getting-started` without redirect.
- [x] 2.3 frontend: Add an ungated "Getting started" item to
  `MainMenu.tsx` above "Report a problem" using the icon +
  title + dimmed-subtitle row pattern and a `title` test hook;
  verify `MainMenu.test.tsx` shows it to anonymous users and that
  clicking it navigates to `/getting-started` and closes the menu.

## 3. Docs and scoped verification

- [x] 3.1 frontend: Update `DEVELOPER_GUIDE.md` (routes table,
  route-components list) and `README.md` (features bullet), and add
  `src/data/` to the Layout section of the repo `AGENTS.md`; verify
  the new route is documented.
- [ ] 3.2 frontend: Run scoped checks — `npx vitest run` on the new
  and touched test files, `npm run lint` (must stay 0 errors / 0
  warnings), `npm run build`, and the 79-char `awk` check on every
  file this change created or modified; verify all pass.

## Workflow follow-up

- Archive the change in the implementing PR (`openspec archive`),
  then run `openspec validate --all --strict` from the workspace
  root.
