**Status:** not implemented — PR #195 closed unmerged. The
DSN-in-APK half shipped separately (reactive-bible#207:
`VITE_SENTRY_DSN` Actions secret, `VITE_APP_ENV=android`);
the media-debug tracer itself lives only on
`feat/media-debug-sentry-apk`.

# Media-debug APK with Sentry diagnostics

- **Repo:** `reactive-bible` only.
- **Goal:** a side-installable debug APK that records a detailed
  trace of media-session / audio lifecycle while the screen is
  locked and ships it to Sentry, so the lock-screen prev/next bug
  (`android-lockscreen-media-buttons-plan.md`, Phase 0) can be
  diagnosed remotely. Lives on a PR branch; never needs `main`.

## Current state (verified)

- Sentry (`@sentry/react` 11.9, #192) inits in `src/main.tsx`
  only when `VITE_SENTRY_DSN` is set.
- `.github/workflows/android-release.yml` runs `npm run build`
  with **no** `VITE_SENTRY_DSN` → Sentry is a no-op in every APK.
  `VITE_APP_ENV` comes from `VERCEL_ENV`, unset in Actions →
  APK events would be tagged `production` like the web app.
- Debug APK (`assembleDebug`) is built for non-`main` refs and
  uploaded only as an Actions artifact (zip, login required).
- `android/app/build.gradle` has no `debug` buildType →
  debug-signed APK cannot install over the release-signed app.
- No GitHub secret for the DSN exists yet (user adds
  `VITE_SENTRY_DSN`).
- `@capacitor/app` ^8.1.1 is installed (appStateChange events).

## Changes

### 1. Workflow (`android-release.yml`)

- `npm run build` env:
  - `VITE_SENTRY_DSN: ${{ secrets.VITE_SENTRY_DSN }}` (all
    builds; missing secret = no-op, as today).
  - `VITE_APP_ENV`: `android` on `main`/tags, `android-debug`
    otherwise.
  - `VITE_MEDIA_DEBUG`: `'1'` only for non-`main`, non-tag refs.
  - `VITE_APP_RELEASE: ${{ github.sha }}`.
- Non-`main`, non-tag runs: after building the debug APK, publish
  it (renamed `reactive-bible-debug.apk`) to a rolling
  `media-debug` prerelease (`prerelease: true`,
  `make_latest: false`, force-move the `media-debug` tag to the
  built sha, like `continuous`). Must not touch `continuous`,
  `update.json`, or `latest`. Keep the artifact upload.

### 2. Android (`android/app/build.gradle`)

- `buildTypes.debug { applicationIdSuffix ".debug";
  versionNameSuffix "-debug" }` and a distinct launcher label
  (e.g. `resValue "string", "app_name", "Bible Research Debug"`
  if `strings.xml` app_name can be overridden without a duplicate
  resource error — otherwise skip the label change).
- Check `capacitor.config.ts` / plugin manifests don't hardcode
  the package name in a way the suffix breaks.

### 3. Sentry init (`src/main.tsx`)

- `release: import.meta.env.VITE_APP_RELEASE` when set.
- Tag `platform` = `native` | `web` via the existing platform
  util (`src/utils/platform*`).
- When media debug is on: `maxBreadcrumbs: 300`,
  `transport: makeBrowserOfflineTransport(makeFetchTransport)`
  (queue events in IndexedDB while offline/locked).

### 4. Debug logger (`src/utils/mediaDebug.ts`, new)

- `isMediaDebugEnabled()`: `VITE_MEDIA_DEBUG === '1'` or
  `localStorage['mediaDebug'] === '1'`.
- `trace(category, message, data?)`: no-op when disabled; else
  `Sentry.addBreadcrumb({category: 'media.'+category, ...})` and
  append `{t: Date.now(), vis: document.visibilityState, ...}` to
  a localStorage ring buffer (500 entries, key `mediaDebugLog`,
  guard quota errors).
- `sendMediaDebugReport(reason)`: `captureMessage('media-debug
  snapshot', {level: 'info', tags: {reason}})` with the ring
  buffer attached as `media-debug.json` (scope attachment), then
  `Sentry.flush()`; clears the buffer after sending.
- Auto-report: on `visibilitychange` → visible (and
  `App.addListener('appStateChange')` isActive true on native)
  after being hidden ≥ 30 s, send report `reason: 'unlock'`.
- Startup trace: `'mediaSession' in navigator`,
  `Capacitor.isNativePlatform()`, userAgent (contains WebView
  version).
- Pure enough to unit-test; keep sync and never throw (media
  handlers must stay fire-and-forget — ANR rule).
- Release builds: everything gated, zero behavior change.

### 5. Trace points (one-line calls, no logic change)

- `src/hooks/useMediaSession.ts`: handler register/unregister
  (web vs native, per effect run); in `dispatch`, action +
  source (`web`/`native` — pass source from the two wrappers);
  playbackState pushes; metadata clears.
- `src/components/Audio.tsx` `nextTrack`/`previousTrack`: pos,
  `audioRef.current?.state()`, timestamps length, branch taken
  (seek / advanceChapter / no-op null pos / ±10 s).
- `src/hooks/useChapterAudio.ts`: `onChapterEnd` adopted vs
  fallback, `setAudio(null)` sites, reset effect dispose,
  `scheduleChapterPlay` fire, `safeSeek` bail vs seek.
- `src/hooks/useAudioKeepAlive.ts` / `src/utils/nativeAudio.ts`:
  enable/disable, failures.

### 6. Manual trigger

- When media debug is on, show a small "Send diagnostic report"
  item in the existing main menu (`MainMenu`) calling
  `sendMediaDebugReport('manual')`.
- Skip the in-app update prompt in media-debug builds (it would
  offer the release APK).

### 7. Tests & docs

- Unit tests for `mediaDebug` (disabled no-op, ring buffer cap,
  report attaches + clears) following existing vitest patterns.
- Existing suites must pass; `npm run lint` 0/0; `npm run build`;
  79-char lines.
- `DEVELOPER_GUIDE.md`: "Media debug APK" section;
  `AGENTS.md`: how to get/install the debug APK and query Sentry.

## User steps (after PR is up)

1. Add repo secret `VITE_SENTRY_DSN` (reactive-bible-web DSN).
2. `gh workflow run android-release.yml --ref <branch>
   --repo Bible-Research/reactive-bible`.
3. On the phone open the `media-debug` release page, install
   `reactive-bible-debug.apk` (installs next to the release app),
   log in, play, lock, press headphone next ×3 / prev ×1, let a
   chapter end, unlock → report sent automatically.
4. Query: Sentry issues with `message:"media-debug snapshot"`,
   environment `android-debug`.

## Out of scope

- `@sentry/capacitor` native crash reporting (Capacitor 8
  support unverified).
- Any actual fix for the media buttons.
- Removal: once fixed, delete `mediaDebug.ts` + trace calls or
  keep gated.
