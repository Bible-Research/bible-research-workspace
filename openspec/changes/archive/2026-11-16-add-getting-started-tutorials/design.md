# Design

## Context

`reactive-bible` (React 18 + TS + Vite + SWC, Mantine v6, Zustand,
React Router). The burger in `MyHeader` opens the `MainMenu` drawer
mounted from `App.tsx`; routes live in `src/routes/index.tsx` behind
`lazyWithReload`. Only `/login`, `/register`, `/search` are whitelisted
for `overflow: auto` in `App.tsx` — other routes sit inside a
`100vh` non-scrolling main, so the new document-style page must be
added to that whitelist (or use its own ScrollArea; whitelist is
simpler and matches SearchRoute precedent).

No embed library is installed; Mantine v6 provides `AspectRatio`.
There is no i18n framework — all strings are hardcoded English.
No CSP anywhere, no `frame-src` restrictions; iframes are already
used (BibleMapper in `VerseActionToolbar`). Vercel Analytics counts
the route automatically; no per-page-title convention exists.

## Goals / Non-Goals

**Goals:**
- A document-style page that can host N tutorial videos without
  paying YouTube's ~600 KB player cost per video on load.
- Zero new dependencies; tests run in happy-dom without network.
- Page ships before any video exists and grows by editing one data
  file.

**Non-Goals:**
- Recording/publishing the videos or maintaining their ids in this
  change (the catalogue ships with placeholder entries).
- Native fullscreen video in the Android shell (needs a custom
  `WebChromeClient`; the "Watch on YouTube" link is the v1 answer).
- Any i18n of page strings (app is English-only today).

## Decisions

**D1 — Hand-rolled lite-embed facade, not eager iframes, not a dep.**
~60 lines: `AspectRatio` + `UnstyledButton` over the
`i.ytimg.com/vi/{id}/hqdefault.jpg` thumbnail + a play icon; on click,
swap in the real iframe with `autoplay=1`. Alternatives rejected:
eager `<iframe>` per card (player JS cost × N videos), and
`lite-youtube-embed` (custom element needing JSX type declarations,
a dependency for ~60 lines of local code). Matches the repo's
defer-third-party performance rule and tests cleanly in happy-dom.

**D2 — `youtube-nocookie.com` embed domain.** Privacy-enhanced mode;
no `referrerPolicy` (YouTube's player requires a Referer — stripping
it breaks playback with "video unavailable") and no `sandbox` attr
(the player needs scripts/same-origin). `allow` covers autoplay,
fullscreen, PiP; `title` set for a11y; `loading="lazy"` as belt.

**D3 — Typed catalogue in `src/data/gettingStartedVideos.ts`.**
Closest precedent is the typed const maps in
`src/utils/externalLinks.ts`; `src/config.ts` is env-only so a new
tiny `src/data/` folder marks it as content, not logic. Shape:
`{ id, title, description, youtubeId }` — `youtubeId: null` entries
stay in the file as the recording checklist but are filtered out of
the render. Seeded with the planned tutorial topics
(see `video-scripts/getting-started.en.md` at the workspace root).

**D4 — Menu item: ungated, bottom, above "Report a problem".**
New anonymous users are the audience, so no `isAuthenticated` or
`isNativeApp` gate. Uses the icon + title + dimmed-subtitle row
pattern already in `MainMenu` (e.g. `IconRocket` from
`@tabler/icons-react`, already a dependency), with a `title`
attribute as the test hook per repo convention.

**D5 — Plain public lazy route + App.tsx overflow whitelist.**
`lazyWithReload(() => import('./GettingStartedRoute'))`, no
`ProtectedRoute`, default export required. Pathname added to the
`overflow: "auto"` condition alongside auth/search pages.

## Risks / Trade-offs

- [Android shell can't fullscreen embedded video] → "Watch on
  YouTube" link per card; a native `WebChromeClient` fix is a
  deliberate follow-up, not this change.
- [Thumbnail `i.ytimg.com` fails or video id is wrong] → broken
  image/play surface; accepted — ids are curated manually and the
  "Watch on YouTube" link still works.
- [Catalogue edited with a malformed id] → a light unit test on the
  data file (unique ids, `youtubeId` shape) catches typos.
- [Third-party iframes in the Capacitor WebView] → no CSP or
  allowlist blocks them; embeds are https-only.
