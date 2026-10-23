# Learnings — Lock-screen audio + media buttons in a Capacitor APK

How `reactive-bible` (React + Howler.js) keeps playing Bible chapters
with the Android screen locked, and why hardware / Bluetooth / car
prev/next buttons are harder than playback itself. The APK shell,
CI, and sideload updater are covered in
[android-apk-wrapper.md](android-apk-wrapper.md); this doc covers
only the audio path. Fix plan for the open button bug:
[media buttons plan][plan].

[plan]: ../plans/android-lockscreen-media-buttons-plan.md

Paths below are relative to `reactive-bible/` unless noted.

## TL;DR

- A PWA can't do this reliably. Chrome freezes a background tab's
  JS soon after screen-off. A chain driven by `ended` events
  survives for a while, but creating new audio, seeking, or calling
  `play()` from a frozen page gives stuck audio, garbled sound on
  unlock, and crashes.
- The APK adds two native layers: a **foreground service** that
  keeps the WebView's JS running
  (`@anuradev/capacitor-background-mode`,
  `disableWebViewOptimization: true`), and a **native
  MediaSessionCompat** with a media notification
  (`@capgo/capacitor-media-session`, `mediaPlayback` FGS type).
- **Auto-advance works while locked.** The next chapter is
  preloaded into a blob, and its Howl is built ahead of time and
  adopted inside the `ended` event chain. No OS→JS hop is needed.
- **Prev/next buttons are broken in the APK today (since
  `99cccdf`).** There are **two** media sessions: Chromium's
  (the WebView's html5 `<audio>`) and the plugin's native one.
  The OS routes keys to whichever session it picks, which is often
  the WebView's. Handlers are registered only on the native one.
  Rule: register on **every** session that might get the key, and
  dedupe in one dispatcher.

## Architecture

```
+---------------- WebView (Android System WebView) ----------------+
| React app                                                         |
|  Audio.tsx -- useChapterAudio -- createChapterHowl()              |
|     |          (html5:true, pool:1, one Howl per chapter,         |
|     |           next chapter preloaded as a blob: Howl)           |
|     +-- useAudioKeepAlive --> nativeAudio.ts ---------+           |
|     +-- useMediaSession --> dispatchMediaAction <--+  |           |
|           navigator.mediaSession (web handlers)    |  |           |
|           NativeMediaSession (plugin handlers)     |  |           |
|  Howler html5 <audio> element                      |  |           |
+--------|-------------------------------------------|--|-----------+
         |                                   JS cb   |  |
         v                                           |  v
  Chromium MediaSession                              | BackgroundMode
  (auto-created for a playing                        | Service (FGS:
  <audio>; runs navigator.                           | keeps JS awake,
  mediaSession handlers)                             | re-sends VISIBLE)
         |                                           |
         |               MediaSessionService --------+
         |               (FGS mediaPlayback, MediaSessionCompat
         |                "CapgoMediaSession", media notification)
         |                          |
         v                          v
  +------------- Android media_session system service ------------+
  | Routes lock screen / notification / BT-AVRCP / headset / car  |
  | keys to ONE "media button session" (usually the most recently |
  | active or playing one) -- often Chromium's, not Capgo's       |
  +---------------------------------------------------------------+
```

Key files:

| Layer | File |
|-------|------|
| Howl factory | `src/utils/chapterHowl.ts` |
| Teardown | `src/utils/howlUtils.ts` (`disposeHowl`) |
| Safe position read | `src/utils/audioUtils.ts` (`getPlayPosition`) |
| Chapter player / preload | `src/hooks/useChapterAudio.ts` |
| Controls wiring | `src/components/Audio.tsx` (`mediaControls`) |
| Session owner | `src/hooks/useMediaSession.ts` |
| Dispatch + validation | `src/utils/mediaSession.ts` |
| Keep-alive | `src/hooks/useAudioKeepAlive.ts`, `src/utils/nativeAudio.ts` |
| FGS permission | `android/app/src/main/AndroidManifest.xml` |
| Architecture doc | `MEDIA_SESSION_BEST_PRACTICES.md` |

## What works and why (auto-advance)

1. **Keep JS alive.** `useAudioKeepAlive(active, isPlaying)`
   (`src/hooks/useAudioKeepAlive.ts`) calls `enableAudioKeepAlive`
   on play. It starts the background-mode FGS with
   `disableWebViewOptimization: true` (the plugin re-dispatches
   `View.VISIBLE` to the WebView so Chromium doesn't treat it as
   hidden). It also requests notification permission, plus a
   one-time battery-optimization exemption. While paused it keeps
   the service for 5 min (`PAUSE_KEEPALIVE_MS`) and then releases
   it.
2. **One html5 `<audio>` per chapter.** `createChapterHowl`
   (`src/utils/chapterHowl.ts`) always uses `html5: true, pool: 1`.
   Media events (`ended`, `timeupdate`) on a dedicated element
   still arrive on a throttled page. WebAudio decoding and a
   shared pool don't hold up.
3. **Preload the next chapter's bytes.** `handleChapterPlay` →
   `preloadNextChapter` (`useChapterAudio.ts` ~L309-381) fetches
   the next chapter's audio (crossing book/testament boundaries),
   wraps it in a `blob:` URL, and builds a `preload: true` Howl.
   It passes `format` from the source URL's extension, because a
   blob URL has none and the Howl would stay `unloaded`. "Next" is
   computed from `audioChapterKeyRef` (the chapter that is
   actually playing), not from the store, which may lag a commit.
4. **Adopt inside the `ended` chain.** `onChapterEnd` (~L441-514)
   checks that the preloaded key matches the next chapter and that
   the Howl isn't `unloaded`. It then disposes the finished Howl,
   wires listeners onto the preloaded one, swaps `audioRef`, sets
   `pendingGapRef`, and calls `goToChapter(next)`. No fetch and no
   new element, so it works while locked. `scheduleChapterPlay`
   (~L241) plays whatever Howl owns `audioRef` after
   `INTER_CHAPTER_GAP_MS` (2 s). That `setTimeout` only fires
   reliably because the FGS keeps timers running.
5. **URL is the source of truth** (PR #117, `3bcbbfd`).
   `goToChapter` navigates on `/bible*` (`replace: true`) and lets
   BibleRoute's URL→store sync write the store. Writing the store
   first let a commit see the old URL with the new chapter and
   revert the advance, tearing down the adopted Howl mid-`play()`
   (`AbortError`, "Failed to play audio"). Off the reader it
   writes the store directly.

Auto-advance never needs the OS to call into JS. That's why it
keeps working even when media-button routing is broken.

## Media buttons while locked

### How a key reaches JS (native path)

Verified in
`node_modules/@capgo/capacitor-media-session/android/src/main/java/`
`com/capgo/mediasession/` (v8.0.31):

- `MediaSessionCallback.onSkipToNext/onSkipToPrevious` →
  `plugin.actionCallback("nexttrack" / "previoustrack")`. This
  resolves the kept-alive `PluginCall` registered by JS
  `setActionHandler`, which runs the JS callback, which calls
  `dispatchMediaAction`.
- `MediaSessionService.update()` advertises an action (in both
  `PlaybackStateCompat` actions and notification buttons) **only
  if `plugin.hasActionHandler(name)`**. `play` is shown only when
  PAUSED, `pause` only when PLAYING.
- `MediaSessionPlugin.setPlaybackState`: with the default config
  (`startServiceOnlyDuringPlayback = true`), the service starts
  only when the state is `playing`/`paused`, and any other state
  (`none`) **stops** it. `handleOnStop` (activity stopped, e.g.
  screen lock) also stops it unless playing/paused. Stopping
  unbinds, and `onUnbind` → `destroy()` → `stopForeground` +
  `stopSelf`. The next start runs `connectAndInitialize`, which
  builds a **new** `MediaSessionCompat`. Note that `destroy()`
  never calls `mediaSession.release()`, so an old session may
  linger until GC or process death (check with `dumpsys`).
- Manifest: plugin service `foregroundServiceType=
  "mediaPlayback"` with a `MEDIA_BUTTON` intent filter, plus
  `androidx.media.session.MediaButtonReceiver`. `onStartCommand`
  forwards intents via `MediaButtonReceiver.handleIntent`, but
  only while the service (and `mediaSession`) exists. The app
  manifest adds `FOREGROUND_SERVICE_MEDIA_PLAYBACK` (Android 14+).

**`setActionHandler(action, null)` in this version:** a `null`
callback makes Capacitor's bridge send callback id `"-1"`
(`native-bridge.js` `toNative`). The plugin stores that call, and
`hasActionHandler` treats `CALLBACK_ID_DANGLING` as "no handler".
The action is therefore **un-advertised** on the next
`updatePossibleActions()`. So in 8.0.31, `null` does *not* leave
advertised-but-dead buttons on the native session. Two caveats:
(a) the old JS callback stays in Capacitor's callback map (a
small leak); (b) if the service is down when `null` arrives,
the stale state is only rebuilt on the next start. Re-check this
whenever the plugin is upgraded.

### The two-session problem

> **Unverified hypothesis.** Android System WebView may expose no
> OS-level media session at all (known Chromium gap: w3c/
> mediasession#337, react-native-webview#2886). If so, the
> `!native` gate is harmless and session churn across chapter
> gaps (below) is the real cause. Check on device with
> `'mediaSession' in navigator` (chrome://inspect) and
> `adb shell dumpsys media_session` — see Phase 0 of
> `docs/plans/android-lockscreen-media-buttons-plan.md`.

When an html5 `<audio>` plays inside Android System WebView,
Chromium creates **its own** platform MediaSession for the page,
and `navigator.mediaSession` handlers attach to it. The plugin
adds a second one (`CapgoMediaSession`). Android sends hardware,
BT, and car keys to a single "media button session", usually the
most recently active or playing one, and that is often
Chromium's.

| Commit | Web handlers | Native handlers | Result in APK |
|--------|--------------|-----------------|---------------|
| `d4bd620` | yes | yes | buttons work (intermittently) |
| `99cccdf` (#141) | **gated `!native`** | yes | prev/next dead |

`99cccdf` added `if (isSupported() && !native)` around the web
handlers (`src/hooks/useMediaSession.ts` ~L160-178) to stop
`seekBy`/`nexttrack` double-firing. The effect: when the OS
routes to Chromium's session, nothing in JS handles the key.
Auto-advance kept working, which hid the regression.

A second destabilizer: `mediaActive` in `Audio.tsx` is
`audio !== null` in chapter mode. On the **non-preloaded**
advance path, `onChapterEnd` calls `setAudio(null)`. That makes
`useMediaSession` send `playbackState: 'none'`, which stops the
plugin service and destroys the native session, and also unsets
all handlers and clears metadata. `useAudioKeepAlive` likewise
disables the keep-alive FGS for the gap. When the next Howl
appears, everything is rebuilt as a new session, and the OS may
now prefer Chromium's.

### Recommended pattern

- Register handlers on **both** `navigator.mediaSession` and the
  native plugin, every time, in the APK too.
- Dedupe in the **shared dispatcher**: drop a repeat of the same
  action within a short window (~300-500 ms). Don't remove one
  session's handlers to prevent a double fire.
- Keep the session **stable across chapter gaps**. Derive
  `active` from "a chapter session is in progress", not
  `audio !== null`. Never send `'none'` or tear down handlers for
  a transiently null Howl.
- Keep handlers sync and fire-and-forget (see rules below).

Sketch, based on the real `useMediaSession` effect:

```ts
// src/hooks/useMediaSession.ts (sketch, not shipped)
const DEDUPE_MS = 400;
const lastFired = new Map<MediaSessionAction, number>();

const dispatch = (
  action: MediaSessionAction,
  details: MediaSessionDetails,
): void => {
  const now = Date.now();
  // Seeks carry distinct payloads; only dedupe discrete keys.
  if (action !== 'seekto') {
    const prev = lastFired.get(action) ?? 0;
    if (now - prev < DEDUPE_MS) return;
    lastFired.set(action, now);
  }
  dispatchMediaAction(
    action,
    details,
    controlsRef.current,
    updatePositionState,
  );
};

// Web: register even when native — the OS may route keys
// to Chromium's session instead of CapgoMediaSession.
if (isSupported()) {
  ALL_ACTIONS.forEach((a) =>
    safeSetHandler(a, (d) => dispatch(a, d)),
  );
}
if (native) {
  ALL_ACTIONS.forEach((a) => {
    void NativeMediaSession.setActionHandler(
      { action: a as NativeAction },
      (d) => dispatch(a, d),
    ).catch(ignoreRejection);
  });
}
```

Plus, in `Audio.tsx`, an `active` flag that stays true through the
2 s inter-chapter gap and the non-preloaded reload, e.g. "chapter
session started and not explicitly stopped".

## Rules / pitfalls checklist

- [ ] **Foreground service on play** with
      `disableWebViewOptimization: true`; notification permission
      (Android 13+); battery exemption asked once, with the flag
      stamped only after a successful check (`nativeAudio.ts`).
- [ ] **`FOREGROUND_SERVICE_MEDIA_PLAYBACK`** in the app manifest
      (`d4bd620`).
- [ ] **`html5: true, pool: 1`**, one Howl per chapter, built only
      through `createChapterHowl` (`d484722`).
- [ ] **Preload + adopt in `ended`.** Never fetch or create audio
      from a frozen page (`3bcbbfd`/#117, `99cccdf`).
- [ ] **Blob Howls need `format`** from the source URL extension.
- [ ] **URL is the single source of truth** for chapter moves
      (#117).
- [ ] **Never `seek()` a non-loaded Howl.** Read positions through
      `getPlayPosition` (returns null unless `state()==='loaded'`
      and the value is finite). Mid-load `seek()` returns the Howl
      itself and corrupts `<audio>` ("broken noise").
- [ ] **`disposeHowl` order: stop → off → volume(0) → unload**
      (`e91c01b`). `stop` lets 'stop' subscribers clean up, `off`
      before `unload` keeps a rejected pending `play()` from
      surfacing, and `volume(0)` silences drained buffers.
- [ ] **Never `mute(true)`.** It sets `node.muted`, which the html5
      pool never resets, so the next chapter is silent.
- [ ] **Handlers are sync and fire-and-forget**, and never throw. An
      awaited handler on a frozen page means ANR (`e5d746f`).
- [ ] **Wrap every MediaSession call in try/catch.** Some WebViews
      throw `NotSupportedError` per action.
- [ ] **Validate `setPositionState`** with `isValidPosition`
      (duration > 0, 0 ≤ position ≤ duration).
- [ ] **One owner of the media session**, `useMediaSession`
      (`e5d746f`). Two owners desynced playlist buttons.
- [ ] **Register on every session that may get the key; dedupe in
      the dispatcher.** Don't gate on `!native` (`99cccdf`
      regression).
- [ ] **Keep the session alive across chapter gaps.** `'none'` stops
      the plugin service and destroys the native session.
- [ ] **Never resolve a promise with a Capacitor plugin proxy**
      (`dbfbf94`/#181). The proxy's get-trap returns a method for
      `then`, so `return m.BackgroundMode` from an async function
      calls `BackgroundMode.then()`. That rejects with "not
      implemented on android" (unhandled) and the await hangs
      forever. Return the module namespace and destructure.
- [ ] **Plugin upgrade?** Re-verify `hasActionHandler`,
      `setActionHandler(null)`, and service lifecycle in the Java
      source.

## How to debug on device

1. **Remote-debug the WebView.** Debug builds have WebView
   debugging on. With USB debugging enabled, open
   `chrome://inspect/#devices` on the desktop and inspect
   `com.bibleresearch.app`. The console shows
   `MediaSession handler error`, `Audio play error`, and
   keep-alive warnings.
2. **Logcat:**

   ```sh
   adb logcat -v time | grep -E \
     'CapgoMediaSession|MediaSessionService|chromium|BackgroundMode'
   ```

   `No handler for action nexttrack` means the native session got
   the key but no JS handler was registered.
3. **Which sessions exist / which gets buttons:**

   ```sh
   adb shell dumpsys media_session
   ```

   Look for `CapgoMediaSession` and a Chromium/WebView session
   under the app's package. Check `active=`, `state=`, and the
   `actions=` bitmask, and see which one the "Media button
   session" line names. Repeat after a chapter boundary to catch
   session churn.
4. **Simulate buttons with the screen locked:**

   ```sh
   adb shell cmd media_session dispatch next
   adb shell cmd media_session dispatch previous
   adb shell cmd media_session dispatch play-pause
   # older Android:
   adb shell media dispatch next
   ```

   Compare against what `dumpsys` says is the button session.
5. Always test with the **screen locked for several minutes**, across
   at least one chapter boundary, both with and without a
   successful preload (e.g. a host without CORS → no blob).

## History timeline

- `1a7ff5f` (2025-12-19) — first hardware media-control fix
  (`Audio.tsx`).
- `ac1d62a` (2025-12-27) — car stereo support for audio controls.
- `e5d746f` (2026-06-30) — `useMediaSession` becomes the single
  owner; sync fire-and-forget handlers (ANR fix).
- `d4bd620` (2026-09-23) — APK shell: background mode + native
  media session; handlers registered on web **and** native →
  buttons work (intermittently).
- `78afcbf` (2026-10-05) — verse-aware next/previous (chapter
  change at the edges).
- `3bcbbfd` (#117) — URL as source of truth for auto-advance.
- `99cccdf` (2026-10-20, #141) — web handlers gated `!native` →
  APK prev/next dead; preload adoption + gap fallback hardening.
- `e91c01b` (2026-10-21, #156) — extract `disposeHowl`,
  `mediaSession.ts`, `activeVerseAtTime`.
- `30db367` (2026-10-21, #165) — split `Audio.tsx` into
  `useChapterAudio` + container.
- `d484722` (2026-10-21, #168) — `createChapterHowl` factory,
  `attachVerseBounds`.
- `dbfbf94` (2026-10-22, #181) — keep the BackgroundMode proxy out
  of promise resolution.

## Open questions

- Which session does Android actually pick as the media-button
  session on the test devices (Samsung vs Pixel, BT vs wired vs
  car)? Confirm with `dumpsys media_session` before and after the
  fix.
- Does Android System WebView always surface a Chromium platform
  MediaSession for html5 audio, or only on some versions or when
  `navigator.mediaSession` metadata is set? Could we drop it
  (e.g. not setting web metadata) so only the native session
  remains?
- Is the dedupe window safe for real double-presses (e.g. "next"
  twice to skip two verses)? It may need ~300 ms, or dedupe only
  across the *other* session's source.
- Do lingering, unreleased `MediaSessionCompat` instances (no
  `release()` in `destroy()`) steal routing after restarts? If so,
  keep the service up (`foregroundService: 'always'` config) or
  patch the plugin.
- Should `useAudioKeepAlive` also stay enabled through the
  non-preloaded chapter gap (it currently disables when `audio`
  is null)?
