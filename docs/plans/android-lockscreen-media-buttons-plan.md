**Status:** not implemented

# Fix: Android lock-screen / headphone / car prev-next buttons dead

- **Repo:** `reactive-bible` (frontend only; no backend change)
- **Date:** 2026-10-22 (HEAD `674210b`)
- **Platform:** Capacitor 8 Android APK shell; the PWA in Chrome is
  not affected by the primary root cause but shares the code.

## 1. Symptom

With the screen locked in the Android APK:

- Chapter auto-advance works (chapter N ends → 2 s gap → N+1).
- Headphone (wired/BT), car-stereo and lock-screen-notification
  **previous/next** do nothing. They should jump to the previous/
  next **verse** (`Audio.tsx:214-263`), or to the previous/next
  **chapter** at the first/last verse (`advanceChapter(±1)`,
  `Audio.tsx:233`, `Audio.tsx:262`).
- The user recalls it working (intermittently) ~20-40 commits
  ago. Historical PWA symptoms of earlier bugs: presses while
  locked froze playback, stuck audio, garbled sound on unlock,
  crashes (see `MEDIA_SESSION_BEST_PRACTICES.md` pitfall table).

## 2. Summary / root cause (current best understanding)

There is **no single confirmed root cause yet** — on-device data
is required (Phase 0). Reading the code and the plugin/bridge
sources narrows it to two families, and one popular hypothesis is
weaker than it looks:

1. **Web-handler gate (`99cccdf`, #141).** Since #141 the hook
   registers `navigator.mediaSession` handlers only when
   `isSupported() && !native` (`useMediaSession.ts:164`, cleanup
   `:190`). On native, only the Capgo plugin's handlers exist
   (`:180-187`), yet the hook still pushes `playbackState`,
   `metadata` and `setPositionState` into `navigator.mediaSession`
   (`:120-130`, `:207-237`, `:259-265`). If the WebView surfaces
   its own OS session and that session wins media-button routing,
   presses land in a session with no JS handlers → dead buttons.
   **Caveat:** Android System WebView historically has *no*
   platform integration for the Media Session API (w3c/
   mediasession#337, react-native-webview#2886; in some WebView
   builds `'mediaSession' in navigator` is even `false`). If that
   holds on the test device, the `!native` gate is a no-op on
   device and #141 is *not* the regression — Phase 0 step 0.3
   decides this in one console line.
2. **Native session churn / lifecycle (plugin side).** Every time
   chapter `audio` becomes `null` (non-preloaded auto-advance
   `useChapterAudio.ts:497-501`; **every** manual chapter jump via
   the reset effect `:720-731`), `mediaActive` (`Audio.tsx:141`)
   flips false and:
   - the hook sends `playbackState: 'none'`
     (`useMediaSession.ts:258`) → plugin `setPlaybackState`
     calls `stopMediaService()` (`MediaSessionPlugin.java:
     228-229`) → `unbindService` → `onUnbind` → `destroy()`
     (`MediaSessionService.java:86-90,211-214`): notification and
     foreground service go away; the `MediaSessionCompat` is
     **never `release()`d** (zombie session, no `release()` call
     anywhere in the service);
   - all 8 native handlers are nulled (`useMediaSession.ts:
     193-199`);
   - `useAudioKeepAlive` calls `disableAudioKeepAlive()`
     (`useAudioKeepAlive.ts:35-38`), dropping the
     `BackgroundMode` foreground service that keeps the WebView
     un-frozen.
   - When audio returns, `'playing'` → `startMediaService()`
     (`MediaSessionPlugin.java:97-106`) calls
     `ContextCompat.startForegroundService` **from the
     background** (screen locked) — on Android 12+ that throws
     `ForegroundServiceStartNotAllowedException` unless the app is
     battery-optimization exempt (the app asks only once,
     `nativeAudio.ts:43-58`). Result: after the first chapter
     boundary while locked there may be **no native session at
     all**, while audio keeps playing — exactly "auto-advance works
     but buttons do nothing".
   - The *boundary* buttons themselves trigger this churn: last-
     verse `next` → `advanceChapter(1)` → reset effect disposes
     the Howl, `setAudio(null)` and also **discards the preloaded
     next-chapter Howl** (`useChapterAudio.ts:741`) → fresh fetch
     + brand-new `<audio>` element — the very path the
     `onChapterEnd` comment says breaks on a locked screen
     (`:436-440`).

Contrast with auto-advance: it is driven by the html5 `ended`
event inside the WebView plus a JS `setTimeout`
(`useChapterAudio.ts:241-255`) and normally adopts the preloaded
Howl (`:463-490`) without `audio` ever going `null`, so it never
touches the OS media-session hop. Prev/next requires
OS → MediaSession → plugin → Capacitor bridge → JS, which is where
it breaks.

The plan therefore: **diagnose first** (Phase 0, ~1 h with a
device), then ship the cheap defensive web-handler restore with a
de-dup guard (Phase 1) **and** the session-stability fixes
(Phase 2) — Phase 2 is likely the real fix if Phase 0 shows the
WebView has no OS session.

## 3. Verified code facts (file:line)

### `src/hooks/useMediaSession.ts`

- `isSupported` `:59-60`; `isNative` `:62-68`; `ALL_ACTIONS`
  `:84-93`.
- `updatePositionState` `:116-138` pushes to web **and** native.
- Registration effect `:141-202`, deps `[active,
  updatePositionState]` → registers once per active session.
- Shared `dispatch` `:149-158` → `dispatchMediaAction`.
- Web handlers gated `isSupported() && !native` `:160-178`.
- Native handlers `:180-187`; cleanup sends `null` `:193-199`.
- Metadata effect `:205-254`; on deactivate writes an empty native
  metadata (`:216-223`, added in #141).
- Playback state `:257-271`: `!active ? 'none' : ...` (`:258`).
- Position poll 1 s while playing `:275-282`.

### `src/utils/mediaSession.ts`

- `dispatchMediaAction` `:70-119`. `previoustrack`/`nexttrack`
  call `updatePosition()` **only** in the seek fallback
  (`:87-100`); real track controls (chapter verse jumps!) skip it.
- No de-duplication; no notion of action source.

### `src/components/Audio.tsx`

- `mediaActive = isPlaylistMode ? playlist.isActive : audio !==
  null` `:141`; `useAudioKeepAlive(mediaActive, ...)` `:146`.
- Chapter `nextTrack` `:214-234`, `previousTrack` `:235-263`:
  `getPlayPosition(audioRef.current)` → `null` → silent return
  (`:216`, `:237`); KJV (no timestamps) → ±10 s (`:217-221`,
  `:238-242`); boundary → `advanceChapter(±1)` (`:233`, `:262`).
- `getMediaPosition` `:284-293`; hook call `:295-301`.

### `src/hooks/useChapterAudio.ts`

- `goToChapter` `:267-281` (navigate on `/bible`, store write
  elsewhere — PR #117 rule: URL is the source of truth).
- `advanceChapter` `:285-305` (adjacency from
  `audioChapterKeyRef`, falls back to store).
- `preloadNextChapter` `:309-381` (blob fetch + `createChapterHowl`
  with `preload: true`), triggered from `handleChapterPlay`
  `:398`.
- `onChapterEnd` `:441-514`: adoption path `:463-490` keeps
  `audio` non-null; fallback path `:497-501` sets
  `setAudio(null)`.
- Reset effect `:682-753`: key mismatch → dispose + `setAudio(null)`
  `:720-731`; then `discardPreloaded()` `:741` even when the
  preload *is* the chapter we are moving to.
- `safeSeek` `:805-819` bails unless `state() === 'loaded'`
  (`:807`) — correct per rule 4.
- Load effect `:867-1118` adopts a matching preload `:965-985`.
- `handleClose` `:1120-1137`.

### Other app files

- `src/utils/chapterHowl.ts:38-47` — every chapter Howl is
  `html5: true, pool: 1`.
- `src/utils/audioUtils.ts:13-17` — `getPlayPosition` returns
  `null` unless `'loaded'`.
- `src/utils/verseSeek.ts:72-83` — `seekHowlWhenLoaded` (queue a
  seek until `'load'`), reusable for Phase 3.
- `src/hooks/useAudioKeepAlive.ts:35-38` — `!active` → disable the
  BackgroundMode foreground service immediately.
- `src/utils/nativeAudio.ts:66-88` — enable keep-alive with
  `disableWebViewOptimization: true`; battery prompt once `:43-58`.
- `capacitor.config.ts` — no `plugins.MediaSession` config, so the
  plugin runs with `startServiceOnlyDuringPlayback = true`.
- `android/app/src/main/AndroidManifest.xml` — declares
  `FOREGROUND_SERVICE_MEDIA_PLAYBACK`; the plugin's own manifest
  declares `MediaSessionService` + `androidx.media
  .session.MediaButtonReceiver` (both `MEDIA_BUTTON`). No other
  plugin declares a `MEDIA_BUTTON` service (BackgroundMode's
  service does not), so `MediaButtonReceiver` resolution is OK.

### `@capgo/capacitor-media-session` 8.0.31 (Java)

- `MediaSessionPlugin.java`
  - `load()` `:75-86`: `foregroundService: 'always'` config would
    start the service at plugin load and never stop it.
  - `handleOnStop()` `:88-95`: stops the service on activity stop
    when state is not playing/paused.
  - `setPlaybackState` `:221-235`: `'none'` → `stopMediaService`;
    `playing|paused` with no service → `startMediaService`.
  - `setActionHandler` `:247-259`: `setKeepAlive(true)`,
    `actionHandlers.put(action, call)` (replaces, never releases
    the previous saved call — small leak).
  - `hasActionHandler` `:261-264` / `actionCallback` `:270-278`:
    a call whose callback id is `CALLBACK_ID_DANGLING` (`"-1"`)
    counts as "no handler" and logs
    `D/CapgoMediaSession: No handler for action <a>`.
- `MediaSessionService.java`
  - `connectAndInitialize` `:92-209` creates a **new**
    `MediaSessionCompat` on every bind; `destroy()` `:211-214`
    never calls `mediaSession.release()`.
  - `onStartCommand` `:216-222`: `MediaButtonReceiver.handleIntent`
    only if `mediaSession != null`; a service instance started by
    the receiver (not bound) has no session and never calls
    `startForeground` → possible
    `ForegroundServiceDidNotStartInTimeException` crash on a BT
    press when no session is bound (matches "crashes" history).
  - `update()` `:285-360`: playback-state actions and notification
    buttons are derived from `plugin.hasActionHandler(...)`
    (`:297`); `play` only shown when PAUSED, `pause` only when
    PLAYING.
- `MediaSessionCallback.java:42-49`: `onSkipToPrevious/Next` →
  `actionCallback("previoustrack"/"nexttrack")`.

### Capacitor bridge semantics (verified)

- `@capacitor/core/dist/index.js:111-129`: every plugin method
  wrapper returns a **Promise** (so `.catch(ignoreRejection)` on
  the callback-style `setActionHandler` is safe).
- `@capacitor/android/.../native-bridge.js:891-913`: `toNative`
  stores a callback and assigns a real id only when
  `typeof callback === 'function'`; a `null` handler is sent with
  `callbackId: '-1'` = `PluginCall.CALLBACK_ID_DANGLING`.
  ⇒ **Hypothesis H2 below is refuted by source:**
  `setActionHandler({action}, null)` *does* make
  `hasActionHandler` false; nulled buttons are hidden, not
  "advertised-but-dead". The only side effect is a leaked
  keep-alive `PluginCall` per re-registration.

### Git history relevant to "it used to work"

- `d4bd620` (2026-09-23) added the plugin + keep-alive; effect
  registered **both** web (`if (isSupported())`) and native.
- `78afcbf` (2026-10-05) verse-aware next/previous.
- `99cccdf` (#141, 2026-10-20) added the `!native` gate **and**
  put an XML comment inside `<application>`, which broke every
  release APK build until `8969fab` (#174). So the last APK the
  user had before the gate was built from ≤ `d9bc223`; the first
  APK with the gate came from `8969fab` onward.
- `3515eff` (#154) introduced the lazy BackgroundMode import whose
  proxy-in-promise hang (`enableAudioKeepAlive` never settled)
  was fixed only in `dbfbf94` (#181) — keep-alive was silently
  off for APKs built between those commits. Any device test of
  "does it work" from that window is unreliable.
- `e91c01b`, `30db367`, `d484722` kept the gate.

## 4. Hypotheses (re-ranked after reading plugin/bridge source)

- **H3 — prior: high.** Native session / keep-alive torn down on
  `audio === null` gaps; the restart from the background fails or
  lags. Evidence: logcat FGS exceptions; `dumpsys media_session`
  before vs. after a chapter boundary.
- **H1 — prior: medium (low if `navigator.mediaSession` is
  absent).** A WebView OS session owns media-button routing and
  has had no JS handlers since #141. Evidence: `'mediaSession' in
  navigator`; dumpsys session list.
- **H5 — prior: medium.** The press reaches JS but
  `getPlayPosition` is `null` (Howl not `'loaded'`) or
  `timestamps` is empty → silent no-op. Evidence: JS breadcrumb in
  `nextTrack`.
- **H7 — prior: medium.** Zombie `MediaSessionCompat`s, or a
  receiver-started service without a session. Evidence: dumpsys
  shows several `CapgoMediaSession` records.
- **H4 — prior: low** (the auto-advance timer runs while locked).
  Renderer frozen, bridge result not delivered. Evidence: native
  `actionCallback` reached, JS breadcrumb absent.
- **H2 — refuted by source (§3).** A `null` handler leaves dead
  advertised buttons. Evidence would be `No handler for action`
  while the buttons are visible.

## 5. Phase 0 — on-device diagnosis (do this first)

Goal: identify which hop drops the press. Needs a USB-connected
Android device, `adb`, desktop Chrome. Nothing here is committed.

### 0.1 Build and install a debuggable APK

- Push the HEAD commit to the `android` branch (the
  `android-release.yml` workflow builds `assembleDebug` on
  `android` pushes and on `workflow_dispatch`) and download the
  debug APK artifact, or build locally:

  ```bash
  export PATH="/opt/homebrew/bin:$PATH"
  cd reactive-bible && npm run build && npx cap sync android
  cd android && ./gradlew assembleDebug
  adb install -r app/build/outputs/apk/debug/app-debug.apk
  ```

- Confirm the installed build:
  `adb shell dumpsys package com.bibleresearch.app | grep -E
  'versionCode|versionName'`.
- Debug builds enable WebView remote debugging by default
  (Capacitor `webContentsDebuggingEnabled` follows
  `isDebuggable`).

### 0.2 Temporary breadcrumbs (uncommitted, one local patch)

Add a tiny ring buffer so traces survive the lock screen and can
be read after unlock (and appear in logcat via
`Capacitor/Console`):

```ts
// src/utils/msDebug.ts (TEMPORARY — do not commit)
const KEY = 'ms_debug_log';
export const msLog = (...a: unknown[]): void => {
  const line = `${new Date().toISOString()} ${a
    .map((x) => JSON.stringify(x))
    .join(' ')}`;
  console.info('[MS]', line);
  try {
    const arr = JSON.parse(localStorage.getItem(KEY) ?? '[]');
    arr.push(line);
    localStorage.setItem(KEY, JSON.stringify(arr.slice(-300)));
  } catch { /* ignore */ }
};
```

Call sites:

- `useMediaSession.ts` `dispatch` (`:149`): `msLog('dispatch',
  action, source)` — add a `source` arg (`'web' | 'native'`) to
  the two wrappers.
- Registration effect start/cleanup (`:141`, `:189`):
  `msLog('register', native, isSupported())` / `'unregister'`.
- Playback-state effect (`:257`): `msLog('state', state)`.
- `Audio.tsx` `nextTrack`/`previousTrack` (`:214`, `:235`):
  `msLog('next', pos, audioRef.current?.state(),
  timestamps.length)` and on each early return.
- `useChapterAudio.ts` `safeSeek` (`:805`): target, state.
- `useChapterAudio.ts` `setAudio(null)` sites (`:501`, `:730`):
  `msLog('audio-null', reason)`.
- Once at start-up: `msLog('caps', 'mediaSession' in navigator,
  typeof MediaMetadata)`.

Read after unlock in chrome://inspect console:
`JSON.parse(localStorage.ms_debug_log).join('\n')`.

### 0.3 Steps and commands

1. `chrome://inspect/#devices` → inspect the app's WebView. In
   the console run `'mediaSession' in navigator` and
   `navigator.userAgent` (record WebView version).
2. Start logcat in a terminal:

   ```bash
   adb logcat -c
   adb logcat -v time Capacitor/Console:V CapgoMediaSession:V \
     MediaSessionService:V MediaSessionCompat:V \
     MediaButtonReceiver:V ActivityManager:W chromium:W \
     AndroidRuntime:E '*:S'
   ```

3. Play a chapter with timestamps (e.g. ESV/NIV, JHN 3), screen
   **on**. Snapshot sessions:

   ```bash
   adb shell dumpsys media_session > ms_on.txt
   ```

   Look for: "Media button session is …", every session record
   with `package=com.bibleresearch.app` (tag
   `CapgoMediaSession`, and any chromium-tagged one),
   `active=true`, `state=PlaybackState {state=3 …
   actions=…}`. `actions` must include `SKIP_TO_NEXT` (32) and
   `SKIP_TO_PREVIOUS` (16).
4. Inject presses (screen on, then locked):

   ```bash
   adb shell cmd media_session dispatch next
   adb shell cmd media_session dispatch previous
   # older Android: adb shell media dispatch next
   adb shell input keyevent KEYCODE_MEDIA_NEXT      # 87
   adb shell input keyevent KEYCODE_MEDIA_PREVIOUS  # 88
   ```

   Lock with `adb shell input keyevent KEYCODE_POWER` (or the
   button), repeat, then tap prev/next on the lock-screen
   notification and on real BT headphones / car.
5. Let a chapter boundary pass while locked (both preload-adopted
   and — by disabling network after preload is skipped — fresh-
   load); re-run step 3 into `ms_after_gap.txt` and step 4.
6. While locked, press `next` on the **last verse** (boundary) and
   re-snapshot `ms_after_boundary.txt`.
7. Grep logcat for: `No handler for action`,
   `ForegroundServiceStartNotAllowedException`,
   `did not then call Service.startForeground`,
   `ForegroundServiceDidNotStartInTimeException`, `ANR in`.
8. Bisect if still unclear: build debug APKs at `d9bc223`
   (pre-#141 gate) and at `99cccdf` with the manifest comment
   moved (cherry-pick `8969fab`), repeat steps 3-4.

### 0.4 Interpreting results

- `'mediaSession' in navigator === false`, or dumpsys shows no
  chromium session for the app → **H1 refuted**; Phase 1 is
  defensive only; prioritize Phase 2.
- The media-button session is a chromium/WebView session and
  `[MS] dispatch` is absent → **H1 confirmed**; Phase 1 is the
  primary fix.
- `[MS] dispatch nexttrack native` logged, then `next null …` →
  **H5** → §8.3 (Phase 3).
- `[MS] dispatch` + `safeSeek` logged but audio does not move →
  seek mechanics on the html5 node while locked — new
  investigation.
- Works before a boundary, dead after; an FGS exception, or no
  `CapgoMediaSession` in `ms_after_gap.txt` → **H3** → Phase 2
  (§7.1-7.2).
- Several `CapgoMediaSession` records and the button session is a
  stale one → **H7** → §7.4.
- `No handler for action nexttrack` while the notification shows
  next → handler-window race (**H3**) → §7.1/§7.3.
- Native `actionCallback` reached (no "No handler" log) but no
  `[MS] dispatch` → **H4** bridge delivery → check keep-alive /
  `disableWebViewOptimization`.

Record the outcome in this plan (update status to "in progress")
before coding.

## 6. Phase 1 — restore web handlers on native, with de-dup

### 6.1 Why #141's double-fire concern is (mostly) theoretical

- Android routes one hardware key event to **exactly one**
  session: `MediaSessionService` picks the media-button session
  (global-priority session, else the most recent active session
  of the app currently playing). A lock-screen/notification
  button is a `PendingIntent` bound to **one** session. So one
  physical press cannot reach both the WebView session and the
  Capgo session through the OS.
- With the app in the foreground, a key first goes to the focused
  window (WebView/Blink `keydown`) and only falls back to
  `MediaSessionManager` if unconsumed — again single delivery.
- The #141 commit cites no reproduction; the gate was a
  precaution. Its cost was that, *if* the WebView session wins
  routing, presses are swallowed by a session the app keeps
  feeding state to but gave no handlers.
- Real double delivery is still conceivable on exotic OEM
  builds (e.g. a car head unit sending both AVRCP passthrough and
  a key event, or two sessions after a zombie bind, H7). A cheap
  source-aware de-dup makes restoring web handlers safe either
  way.

### 6.2 Change: `src/utils/mediaSession.ts`

Add a pure, testable de-duper (keep `dispatchMediaAction` pure):

```ts
export type MediaActionSource = 'web' | 'native';

/** Window in which the same press from the OTHER session is
 * treated as a duplicate delivery of one physical press. */
export const MEDIA_ACTION_DEDUPE_MS = 300;

// play/pause/stop dedupe as one group: two sessions that
// disagree about state could deliver 'play' and 'pause' for
// the same press, which must not toggle twice.
const dedupeGroup = (a: MediaSessionAction): string =>
  a === 'play' || a === 'pause' || a === 'stop'
    ? 'transport'
    : a;

export const createActionDeduper = (
  windowMs = MEDIA_ACTION_DEDUPE_MS,
  now: () => number = () => Date.now(),
) => {
  let last: { group: string; source: MediaActionSource;
    at: number } | null = null;
  return (
    action: MediaSessionAction,
    source: MediaActionSource,
  ): boolean => {
    const t = now();
    const group = dedupeGroup(action);
    if (
      last &&
      last.group === group &&
      last.source !== source &&
      t - last.at < windowMs
    ) {
      return false; // duplicate from the other session
    }
    last = { group, source, at: t };
    return true;
  };
};
```

Rules: same-source repeats are **never** dropped (rapid
deliberate double-next in a car must skip two verses); only a
cross-source repeat inside the window is dropped. `seekto`
dedupes by action only (both sessions would carry the same
`seekTime`). Must stay sync and never throw.

### 6.3 Change: `src/hooks/useMediaSession.ts`

- Registration effect (`:141-202`):
  - Create `const shouldRun = createActionDeduper();` per effect
    run.
  - `dispatch(action, details, source)` → `if (!shouldRun(action,
    source)) return;` then `dispatchMediaAction(...)`.
  - Revert the gate: `if (isSupported())` at `:164` and `:190`;
    web wrapper passes `'web'`, native wrapper `'native'`.
  - Replace the comment at `:160-163` with the reasoning from
    §6.1 (single delivery per press; de-dup guards the rest).
- Update the file header comment (`:44-48`).

### 6.4 Alternative considered: one authoritative session

- **A (HEAD):** native handlers only, state pushed to both. The
  worst combination if a WebView OS session exists: it is
  advertised (state/metadata) but handler-less.
- **B (chosen):** handlers + state on both, source-aware de-dup.
  Works whichever session wins routing; tiny code.
- **C:** on native write **nothing** to `navigator.mediaSession`
  (gate metadata/state/position/handlers on `!native`). Clean
  ownership, but it cannot stop Chromium from creating a session
  for a playing `<audio>`; without handlers that session would
  default-handle only play/pause and still swallow prev/next.
- **D:** play audio natively (e.g. an ExoPlayer/Media3 plugin)
  and drop html5. Most robust; a large rewrite; out of scope.

Pick **B**. Revisit C only if Phase 0 proves the WebView never
creates an OS session on all supported WebView versions — then
C is equivalent and simply reduces bridge traffic.

## 7. Phase 2 — session stability across chapter gaps

### 7.1 Decouple "session active" from `audio !== null`

`src/components/Audio.tsx:141` → chapter mode becomes:

```ts
// The session survives transient chapter gaps: a fresh load
// after auto-advance or a boundary jump briefly has no Howl,
// but the user still intends to play.
const chapterSessionActive = audio !== null || isPlaying;
const mediaActive = isPlaylistMode
  ? playlist.isActive
  : chapterSessionActive;
```

`isPlaying` already goes false on close (`handleClose`
`useChapterAudio.ts:1120-1123`), on load/play errors
(`:412-414`, `:423-425`, `:574`), on "no next chapter" (`:510`)
and on the in-flight cancel (`Audio.tsx:324-327`), so a real stop
still ends the session.

### 7.2 Report `'paused'`, not `'none'`, during the gap

Pass `isPlaying: mediaIsPlaying && (isPlaylistMode ||
audio !== null)` to `useMediaSession` (`Audio.tsx:295-301`) so the
gap shows `'paused'`. The plugin then keeps the service bound
(`MediaSessionPlugin.java:221-235`: `paused` counts as active) and
never re-enters `startForegroundService` from the background.

- Keep `useAudioKeepAlive(mediaActive, mediaIsPlaying)`
  (`Audio.tsx:146`) on the **raw** `isPlaying` so the
  BackgroundMode service is not released during the gap
  (`useAudioKeepAlive.ts:35-45`).
- `getMediaPosition` already returns `null` without a Howl
  (`Audio.tsx:286-287`) → no invalid `setPositionState`.
- Metadata (`Audio.tsx:158`) returns `null` without a Howl, which
  would blank the notification (`useMediaSession.ts:205-224`).
  Keep the last chapter title during the gap: build metadata from
  `activeBookId`/`activeChapter` when `chapterSessionActive`
  (title then already shows the *new* chapter — desired).

### 7.3 Handler cleanup semantics

- Verified (§3): `setActionHandler({action}, null)` → dangling id
  → `hasActionHandler` false. Keep the null cleanup; with 7.1 it
  only runs on real stops.
- Do **not** re-register on every activation edge; the effect deps
  stay `[active, updatePositionState]`.
- Optional: on native, defer deactivation cleanup by ~1.5 s and
  cancel if `active` returns (belt and braces for any remaining
  flip, e.g. playlist item swaps). Implement inside the hook with
  a ref timer; must not delay `'none'` on explicit close more than
  that.

### 7.4 Plugin-side hardening (patch-package, optional)

Only if Phase 0 shows H7 or FGS crashes. Add `patch-package`
(dev dep + `postinstall`) and patch
`node_modules/@capgo/capacitor-media-session/android/.../
MediaSessionService.java`:

- `destroy()`: `if (mediaSession != null) { mediaSession
  .setActive(false); mediaSession.release(); mediaSession =
  null; }` before `stopForeground`.
- `onStartCommand`: if `mediaSession == null`, call
  `startForeground` with a minimal notification then `stopSelf()`
  (avoids `ForegroundServiceDidNotStartInTimeException` when a BT
  press starts an unbound service).
- File an upstream issue/PR at Cap-go/capacitor-media-session and
  drop the patch when released.
- Alternative without patching: `plugins: { MediaSession: {
  foregroundService: 'always' } }` in `capacitor.config.ts`
  (`MediaSessionPlugin.java:78-85`) — rejected by default: a
  permanent media notification from app launch.

## 8. Phase 3 — robustness of the actions themselves

### 8.1 Boundary `next` reuses the preloaded Howl

`useChapterAudio.ts:741` discards the preload even when it is the
chapter being entered. Change the reset effect tail to:

```ts
if (preloadedRef.current?.key !== key) discardPreloaded();
```

The load effect then adopts it (`:965-985`) — no fetch, no new
`<audio>` element, same path auto-advance relies on. The new
chapter starts immediately (no `pendingGapRef`), which is the
expected behavior for a manual skip.

### 8.2 Boundary `previous`

No previous-chapter preload exists. Options (pick after Phase 0):

- a. Accept a fresh load (works when keep-alive holds, 7.2).
- b. Lazily preload the previous chapter when the playhead is in
  verse 1 (`idx <= 0` in `previousTrack`, `Audio.tsx:246-249`):
  generalize `preloadedRef` into `{ next, prev }` slots in
  `useChapterAudio.ts:114-118` with separate tokens; dispose both
  in `discardPreloaded`. Memory: one extra blob (~2-8 MB).
- Default: **a**, add **b** only if Phase 5 shows locked
  previous-chapter failures.

### 8.3 Howl not `'loaded'` (H5)

In `Audio.tsx` `nextTrack`/`previousTrack`, when
`getPlayPosition` is `null` but `audioRef.current` exists and is
`'loading'`, queue a step instead of dropping it:

- New ref in `Audio.tsx` (or `useChapterAudio`):
  `pendingTrackStepRef = { howl, delta }` (accumulate ±1).
- `howl.once('load', …)` recomputes the target from the
  then-current position and timestamps and calls `safeSeek` —
  reuse the `seekHowlWhenLoaded` pattern (`verseSeek.ts:72-83`).
- Clear on Howl change (`audio` effect) so a step never lands on
  the next chapter's Howl.
- Never call `seek()` on a non-loaded Howl (rule 4).
- When there is **no** Howl but the session is active (gap, 7.1):
  `nexttrack` → `advanceChapter(1)`, `previoustrack` →
  `advanceChapter(-1)` — lets a driver skip chapters while one is
  loading. `advanceChapter` falls back to the store when
  `audioChapterKeyRef` is `null` (`useChapterAudio.ts:292-298`);
  the store may lag the URL by a commit — acceptable, document it.

### 8.4 Position after track actions

`mediaSession.ts:87-100`: call `updatePosition()` after
`controls.nextTrack()` / `previousTrack()` too, so the lock-screen
scrubber jumps with the verse. `safeSeek` is synchronous on html5,
so the next `getPosition` read is already updated; after an
`advanceChapter` the read is the old Howl or `null` (validated by
`isValidPosition`, harmless; the 1 s poll corrects it).

## 9. Phase 4 — tests (vitest, existing patterns)

Follow `src/hooks/useMediaSession.test.tsx` (renderHook +
`stubMediaSession` from `src/__tests__/mocks/mediaSession.ts`) and
`src/components/Audio.mediaControls.test.tsx` (shared `FakeHowl`
from `src/__tests__/mocks/FakeHowl.ts`, `FakeHowl.instances`,
`vi.mock('howler', …)`, `vi.mock('../api', …)`). Mock Capacitor
like `src/utils/appUpdate.test.ts:20-27`.

New shared mock `src/__tests__/mocks/nativeMediaSession.ts`:

```ts
// handlers: action -> handler | null; calls recorded in order
export const nativeMs = vi.hoisted(() => ({
  handlers: new Map<string, ((d: object) => void) | null>(),
  states: [] as string[],
  metadata: [] as object[],
  positions: [] as object[],
}));
vi.mock('@capgo/capacitor-media-session', () => ({
  MediaSession: {
    setActionHandler: vi.fn(async ({ action }, h) => {
      nativeMs.handlers.set(action, h);
    }),
    setPlaybackState: vi.fn(async ({ playbackState }) => {
      nativeMs.states.push(playbackState);
    }),
    setMetadata: vi.fn(async (m) => { nativeMs.metadata.push(m); }),
    setPositionState: vi.fn(async (p) => {
      nativeMs.positions.push(p);
    }),
  },
}));
```

(`vi.mock` must live in each test file; the helper exports the
hoisted store and a `mockNativeMediaSession()` factory.)

Tests to add:

1. `src/utils/mediaSession.test.ts` — `createActionDeduper`:
   cross-source within window dropped; same-source repeats pass;
   after window passes; `play` web + `pause` native dropped
   (transport group); different actions pass. Inject `now`.
2. `src/hooks/useMediaSession.native.test.tsx` (stub
   `Capacitor.isNativePlatform` → `true`):
   - registers all 8 native handlers **and** all 8 web handlers
     (regression test for the #141 gate);
   - one press delivered by both web and native within 300 ms
     calls `controls.nextTrack` once; two native presses call it
     twice;
   - deactivate → every native handler set to `null`, state
     `'none'`, empty metadata;
   - `previoustrack`/`nexttrack` with track controls push a
     position (8.4).
3. `src/components/Audio.mediaControls.native.test.tsx` (FakeHowl):
   - fallback auto-advance (no preload) never emits native
     `'none'` and never nulls handlers between chapters (7.1/7.2);
   - last-verse `nexttrack` adopts the preloaded FakeHowl: no new
     `FakeHowl.instances` entry for the next chapter (8.1);
   - `nexttrack` while the Howl is `'loading'` seeks to the next
     verse after `emit('load')`, and never calls `seek(v)` before
     (8.3);
   - `handleClose` ends the session (`'none'`).
4. `src/hooks/useAudioKeepAlive` — new `useAudioKeepAlive.test.ts`
   (mock `../utils/nativeAudio`, `../utils/platform`): no
   `disableAudioKeepAlive` call while `isPlaying` stays true across
   an `audio` gap.
5. Existing suites must stay green unchanged (web path):
   `useMediaSession.test.tsx`, `mediaSession.test.ts`,
   `Audio.mediaControls.test.tsx`, `Audio.autoAdvance.test.tsx`.

Commands:

```bash
export PATH="/opt/homebrew/bin:$PATH"
cd reactive-bible
npx vitest run src/utils/mediaSession.test.ts \
  src/hooks/useMediaSession.test.tsx \
  src/hooks/useMediaSession.native.test.tsx \
  src/components/Audio.mediaControls.test.tsx \
  src/components/Audio.mediaControls.native.test.tsx \
  src/components/Audio.autoAdvance.test.tsx
npm run lint && npm run build && npm test -- --run
awk 'length > 79 {print FILENAME":"FNR}' <changed files>
```

## 10. Phase 5 — manual QC matrix and docs

### 10.1 Matrix (debug APK, then signed release)

Run each cell for **prev** and **next**; mark ✓/✗ and note the
logcat line if ✗. Translation with timestamps (ESV or NIV) unless
stated.

Format: input · screen · position → expected.

1. Lock-screen notification · locked · mid-chapter → ±1 verse,
   scrubber moves.
2. Wired headset (buttons or 2×/3× click) · locked · mid-chapter
   → ±1 verse.
3. BT headphones · locked · mid-chapter → ±1 verse.
4. BT car stereo · locked · mid-chapter → ±1 verse, title shows
   the right chapter.
5. Any input · screen on, app foreground · mid-chapter → ±1 verse,
   highlight follows.
6. Any input · screen on, another app foreground · mid-chapter →
   ±1 verse.
7. Next · locked · last verse → next chapter starts immediately;
   no fetch spinner on unlock.
8. Prev · locked · >2 s into verse 1 → restart verse 1.
9. Prev · locked · <2 s into verse 1 → previous chapter.
10. Next ×3 fast · locked · mid-chapter → 3 verses forward, no
    freeze.
11. Any · locked · during the 2 s auto-advance gap and the first
    5 s of the new chapter → works; notification never
    disappears.
12. KJV (no timestamps) · locked · mid-chapter → ±10 s.
13. Playlist mode (notes "Play all") · locked · mid-item →
    previous/next item.
14. Pause from notification, wait 1 min, then next · locked →
    seeks, stays paused.
15. Unlock after 10 min of locked playback → no garbled audio,
    UI in sync.
16. Close the player · screen on → notification gone
    (`'none'`).
17. BT press with the app closed / no session → no crash
    (`adb logcat` clean).

Also re-run the `MEDIA_SESSION_BEST_PRACTICES.md` testing
checklist and the PWA in Chrome Android (rows 1-5, 12, 13) to
confirm the web path is unchanged.

### 10.2 Docs updates (same PR)

- `reactive-bible/MEDIA_SESSION_BEST_PRACTICES.md`: architecture
  bullet "both sessions registered on native, source-aware
  de-dup"; session lifetime spans chapter gaps (`'paused'` not
  `'none'`); WebView Media Session caveat; new pitfall rows
  ("native session torn down on chapter gap → dead buttons",
  "manual boundary jump discarded preload"); update the
  "Cleanup on active → false" bullet; add matrix rows 7-11, 17.
- `reactive-bible/DEVELOPER_GUIDE.md` (Media Session sections
  ~`:1545-1590`, `:3359-3385`): replace the stale inline
  `navigator.mediaSession` example with the hook contract.
- `reactive-bible/AGENTS.md` "Behavior notes": one line — media
  controls go through `useMediaSession` only; on native both
  sessions are wired with de-dup; how to read `dumpsys
  media_session`.
- `docs/learnings/android-apk-wrapper.md` §5: add the Phase 0
  finding on whether the WebView exposes an OS media session, and
  "the native plugin service must not be stopped between
  chapters; background FGS restarts can fail on Android 12+".
- Update this plan's status header at each milestone.

## 11. Files and functions to change (checklist)

All paths relative to `reactive-bible/`.

- [ ] `src/utils/mediaSession.ts` (Phases 1, 3): new
  `createActionDeduper`, `MediaActionSource`,
  `MEDIA_ACTION_DEDUPE_MS`; `dispatchMediaAction` pushes position
  after track actions (`:87-100`).
- [ ] `src/hooks/useMediaSession.ts` (Phases 1, 2): registration
  effect `:141-202` (gate revert, `source` arg, de-dup, optional
  deferred native cleanup); header comment `:24-48`.
- [ ] `src/components/Audio.tsx` (Phases 2, 3): `mediaActive`
  `:141`; `isPlaying`/metadata passed to the hook `:148-173`,
  `:295-301`; `nextTrack`/`previousTrack` `:214-263` (pending
  step, no-Howl gap behavior).
- [ ] `src/hooks/useChapterAudio.ts` (Phase 3): reset effect
  `:736-743` keeps a matching preload; optional prev-preload slot
  `:114-118`, `:309-381`.
- [ ] `src/hooks/useAudioKeepAlive.ts` (Phase 2): no code change;
  fed the stabilized `active` and the raw `isPlaying`.
- [ ] `patches/@capgo+capacitor-media-session+8.0.31.patch`
  (Phase 2, optional): `destroy()` release, `onStartCommand`
  guard; `package.json` `postinstall`.
- [ ] Tests listed in §9 (Phase 4).
- [ ] Docs listed in §10.2 (Phase 5).

Commit plan (feature branch in `reactive-bible`, never `main`;
stage files by path):

1. `Test: Cover native media-session branch` (failing tests for
   gate + de-dup, marked as expected-fail only if needed).
2. `Fix: Register web media handlers on native with de-dup`.
3. `Fix: Keep media session alive across chapter gaps`.
4. `Fix: Reuse preloaded chapter on boundary skips`.
5. `Fix: Queue verse skips until the Howl loads`.
6. `Docs: Media session native lifetime and de-dup`.

## 12. Risks

- **Double action after restoring web handlers** — mitigated by
  the de-dup; residual risk is a genuine double press from two
  *sources* within 300 ms being collapsed (very unlikely: a
  human cannot press via notification and headset that fast).
- **Session stuck "paused" after a silent failure** — if a load
  path ever leaves `isPlaying` true with no Howl and no error,
  the notification lingers. Every failure path today sets
  `isPlaying(false)` (§7.1); add a test per path.
- **Keep-alive held longer** — battery impact limited to the
  load time of one chapter.
- **Preload reuse on manual skip** — a preload built for a
  different fileset/codec is excluded by the key check
  (`chapterKey(filesetId, …)`); fallback walk still armed
  (`:977-985`).
- **Queued verse step lands on the wrong Howl** — cleared on
  Howl change; tested.
- **Patching a third-party plugin** — must be re-validated on
  every plugin bump; prefer upstreaming.
- **WebView variance** — behavior of `navigator.mediaSession` in
  WebView differs by version/OEM; the chosen option B is correct
  in both worlds.

## 13. Rollback

- Each phase is an independent commit; revert in reverse order.
- Phase 1 alone: re-add `&& !native` at `useMediaSession.ts:164`
  and `:190` (the de-dup helper can stay; it is inert with one
  source).
- Phase 2: restore `mediaActive = audio !== null`
  (`Audio.tsx:141`) and the raw `isPlaying` prop.
- Phase 3.1: restore unconditional `discardPreloaded()`
  (`useChapterAudio.ts:741`).
- Plugin patch: delete the patch file and `postinstall`, run
  `npm ci && npx cap sync android`.
- Ship via the `continuous` APK prerelease; users on a bad build
  update over the top (same signing key), never uninstall.

## 14. Open questions for the user

- Exact device / Android version / WebView version and whether
  battery optimization was disabled for the app (affects H3).
- Which inputs fail: all three (notification, headset, car) or a
  subset? Does it also fail with the screen **on**?
- Does it fail from the very first press after starting playback,
  or only after a chapter boundary / some minutes locked?
