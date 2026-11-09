# Learnings — React SPA → Android APK (Capacitor, no Play Store)

Distilled from the commit/PR history that built and then repeatedly
fixed the `reactive-bible` Capacitor shell and its sideload update
flow:

- `d4bd620` — Android APK shell + lock-screen audio (Capacitor 8)
- `0f7fb92` / PR #99 — resize WebView above the keyboard
- `f945c6f` — signed APK on main pushes + in-app update check
- PR #117 — lock-screen auto-advance Howl race
- PR #133 — batch merge that shipped the shell
- PR #141 — hardening: manifest `apkUrl` validation, throttle
  semantics, `allowBackup=false`
- PR #143 — skip Vercel telemetry in the native shell
- PR #173 — `CapacitorHttp` for the manifest (WebView CORS)
- PR #174 — XML comment inside `<application>` broke the release
  build
- CI failures: `setup-android@v3` broken, missing
  `contents: write`, retag push, TS-strict build
- `dbfbf94` / PR #181 — a Capacitor plugin `Proxy` is an
  accidental `thenable`; `await` hangs forever
- `db2bbd9` / `f9850de` — media-debug branch APK; skips the
  automatic update prompt
- PR #204 — Play Protect flags the background-mode plugin's
  manifest profile; strip it via `patch-package`
- PR #205 — "problem parsing the package": publish versioned
  APK assets *before* `update.json`
- PR #206 — emulator install test gates `continuous` publishes
- PR #207 — Sentry in APK builds + updater telemetry
- PR #208 — move the `continuous` tag via the refs API, not
  `git push`
- PR #209 — `v*` tags reach installed apps through
  `continuous/update.json`
- PR #210 — job outputs get secret-scanned: use sentinels or
  artifact files
- PR #212 — the menu APK link resolves the newest *versioned*
  asset via `api.github.com`
- PR #213 — `versionName` `<latest-tag>+<run>` so continuous
  builds don't read as downgrades
- PR #214 — tag releases get versioned APK names too (Downloads
  filename collisions)

## Biggest pitfalls (in rough order of pain)

### 1. WebView fetch is not native fetch — CORS still applies

The update check failed on-device from day one but passed all tests
because `fetch` was mocked. The Capacitor WebView origin is
`https://localhost`; `github.com/.../releases/download/...` sends no
`Access-Control-Allow-Origin`, so every cross-origin read is blocked.

- **Rule:** any HTTP call that must work on-device to a host you
  don't control goes through `CapacitorHttp` (`@capacitor/core`),
  which runs on the native stack with no origin policy. On web it
  degrades to `fetch` (`webFetchExtra` for fetch options).
- **Rule:** mocked `fetch` tests prove nothing about on-device
  networking. At least one real-device pass per network feature.

### 2. A sideload updater is a small product, not a script

What worked: a rolling `continuous` **prerelease** rebuilt on every
main push, carrying a stable-name `reactive-bible.apk` plus a
never-re-uploaded `reactive-bible-<run>.apk` (what `update.json`
points at — see the parsing-package bullet below). The app polls
the manifest once per launch (hourly throttle) and compares
`versionCode` to `App.getInfo().build`.

Details that took iterations to get right:

- `prerelease: true` + `make_latest: false` keeps
  `releases/latest` pointing at real milestones while the fixed
  `continuous` tag gives the updater a stable URL.
- `versionCode` must be monotonic — `github.run_number` is the
  easy choice; drive it via `./gradlew -PversionCode=...`.
- **Verify the built APK's real versionCode** (`aapt dump badging`)
  equals what you baked into `update.json` — otherwise installed
  apps compare against a stale number and never see updates.
- **Validate the manifest on the client**: `apkUrl` must be HTTPS
  on `github.com`/`*.githubusercontent.com`. A compromised
  manifest must not be able to smuggle `javascript:`/`intent:`
  URLs into `window.open`.
- **Stamp the throttle only after a successful check** — a
  transient failure must not suppress retries for a full interval.
- Failed checks resolve to a typed result (`kind: 'error'`),
  never throw; errors only notify on manual checks.
- `@capacitor/browser` wasn't installed — `window.open(url,
  '_system')` hands the APK URL to the OS browser. Verify
  on-device it doesn't open inside the WebView.
- Sideload installs always require user consent — design the UX
  around "download → user confirms install", not silent updates.
- Same-signing-key installs go over the top; tell users never to
  uninstall first — uninstalling wipes WebView localStorage
  (auth token, caches, settings).
- **"There was a problem parsing the package" is a delivery
  bug, not an APK bug.** Three separate root causes produced the
  identical dialog: the fixed-name asset was deleted and
  re-uploaded mid-publish (phones fetched a 404 page or a
  truncated file), the menu download link pointed at that same
  racy name, and unversioned filenames collided in Downloads so
  Android reopened an older partial file. Serve only
  never-re-uploaded versioned names
  (`reactive-bible-<run>.apk`, `reactive-bible-<ver>.apk`),
  publish APK bytes *before* `update.json`, prune old assets
  last, and tell the user the expected file name + size in the
  update modal so they can spot a stale/partial download.
- `update.json` is the commit point — flip it last. Installed
  apps only ever poll `continuous/update.json`, so milestone
  `v*` tags reach phones by pointing that manifest at the tag's
  own asset; there is no second channel.
- `versionName` is human-facing — keep it honest-looking too:
  `0.0.106` offered over an installed `1.0.0` reads as a
  downgrade even though `versionCode` is strictly increasing.
  Continuous builds are named `<latest-tag>+<run>`.
- `api.github.com` is the one GitHub host that sends
  `Access-Control-Allow-Origin: *` — the PWA menu link resolves
  the newest versioned asset from the release asset listing
  with a plain `fetch`. The release *download* endpoints still
  send no ACAO (why the on-device manifest read needs
  `CapacitorHttp`).
- **Tip — the "Check for updates" button:** the main-menu row
  (native builds only, `isNativeApp()`-gated) calls
  `checkForUpdatesManually()` → `checkForUpdates({force:true})`
  — it bypasses the hourly throttle and reports *every*
  outcome: "No update available", "Update check failed", or the
  update modal. Automatic checks stay silent on current/error.
  Use this row as the on-device verification step for any
  update-flow change, and point users at it when a sideload
  "didn't update" — it always re-reads the live manifest.
- Make the funnel observable: `VITE_SENTRY_DSN`,
  `VITE_APP_ENV=android`, `VITE_APP_RELEASE=android-<vc>` must
  reach `npm run build` or the APK reports nothing. `app-update`
  breadcrumbs plus `updateStep: download-opened` / `installed`
  events let you trace the flow without a device.
- Diagnostic/media-debug builds skip the automatic update
  prompt so a test APK doesn't nag about the release channel.

### 3. CI bring-up is its own battle

Failure log of `.github/workflows/android-release.yml`:

- `android-actions/setup-android@v3` is **broken on new
  cmdline-tools** (runs `sdkmanager tools` which no longer
  exists). Hand-roll it: `ubuntu-latest` already ships the SDK —
  find `sdkmanager` under `$ANDROID_HOME/cmdline-tools/*/bin`,
  `yes | sdkmanager --licenses`, install `platforms`/`build-tools`.
- Release upload needs `permissions: contents: write` on the job —
  default `GITHUB_TOKEN` gets a 403 from the releases API.
- Force-pushing the `continuous` tag from the job needs the same
  permission and `contents: write` on the checkout token.
- `npm run build` on CI caught TS-strict errors locally missed —
  the APK job is another typecheck gate; keep it fast by reusing
  `npm ci` cache.
- Vite build needs `NODE_OPTIONS: --max-old-space-size=8192` on
  Actions runners.
- **Job outputs are secret-scanned and silently dropped** — a
  plain asset URL passed via a job output was eaten ("may
  contain secret") and downstream jobs got an empty string.
  Pass sentinel strings resolved inside the called workflow
  (`'continuous'`), or carry values in artifact files
  (`version.txt` holds the built `versionName` for the publish
  jobs).
- Force-moving the `continuous` tag with `git push` fails once
  the pushed commit touches workflow files — `GITHUB_TOKEN`
  lacks the `workflows` permission. Use `gh api -X PATCH
  repos/…/git/refs/tags/continuous` instead, and keep the retag
  step non-fatal: the assets and `update.json` attach to the
  release regardless.
- **An emulator install test is the publish gate.** With no
  local Android SDK, `reactivecircus/android-emulator-runner`
  (API 24/33/35/36) covers fresh install, in-place update over
  the previously advertised APK (`adb install -r`), the
  PackageInstaller UI, and a real Chrome download→install — the
  exact path the updater sends phones down. Ship a negative
  control (truncated APK) so you know the gate can fail.
  Attach milestone/tag APKs only after it passes, and give the
  update leg a genuinely *older* APK: resolve what `continuous`
  currently advertises, not the tag being cut (which doesn't
  exist yet and 404s).

### 4. Play Protect cold-scans every sideloaded build

Every `main` push produces a never-seen APK from a non-Play
signer, and Play Protect scans each against its *current*
rules — a flow that installed fine for months started being
blocked as "harmful" with no change on our side. The trigger
was the manifest profile `@anuradev/capacitor-background-mode`
merges in:

- a `specialUse|microphone` foreground service in an app that
  never records audio
- a special-use justification mentioning "SIP servers"
- `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` plus the system prompt
  that asks for it
- wake-lock / show-when-locked / dismiss-keyguard code paths

Fix: pin the plugin to an exact version and `patch-package` it
down to a plain `mediaPlayback` FGS (the manifest type and the
`startForeground` type must match on Android 14+). Then:

- CI fails the build if red-flag permissions reappear in the
  merged manifest — keep that check green.
- Never re-add a battery-optimization prompt — that permission
  is one of the flags.
- Regenerate the patch whenever the plugin version bumps; a
  stale patch silently stops applying.

### 5. XML comments are not free

A `<!-- comment -->` placed *inside* the `<application ...>`
start-tag of `AndroidManifest.xml` is not well-formed XML — it
broke `processReleaseMainManifest` (`ManifestMerger2`) and killed
every main release build until moved above the element. After
editing manifests, parse-check:
`python3 -c "import xml.etree.ElementTree as ET; ET.parse(...)"`.

### 6. The WebView environment leaks web assumptions

- Vercel `<Analytics />` / `<SpeedInsights />` inject `/_vercel/*`
  scripts that 404 inside the shell — and the global resource-load
  error reporter turned each into a non-dismissible "Unexpected
  error" toast on every launch. Gate web-only integrations on
  `isNativeApp()` (`Capacitor.isNativePlatform()`), and make global
  error reporters ignore optional telemetry URLs.
- `interactive-widget=resizes-content` in the viewport meta works
  in Chrome on Android but is a **no-op in Android WebView** —
  keyboard behavior is governed by `windowSoftInputMode`.
- With targetSdk 36 the app is edge-to-edge and even
  `adjustResize` doesn't shrink the WebView (capacitor#7983,
  capacitor#8055). Fix: `windowSoftInputMode="adjustResize"` +
  `@capacitor/keyboard` with `resizeOnFullScreen: true` + a sticky
  submit bar in the form.
- WebView JS is frozen minutes after screen-off. Lock-screen
  audio needs a **foreground service**
  (`@anuradev/capacitor-background-mode` with
  `disableWebViewOptimization: true`), a **media session**
  (`@capgo/capacitor-media-session`), and
  `FOREGROUND_SERVICE_MEDIA_PLAYBACK` on Android 14+.
- Capacitor plugin objects are `Proxy`s — and accidental
  *thenables*: resolving a promise with the plugin object makes
  `PromiseResolveThenableJob` invoke `.then` on it, which throws
  "not implemented" as an abandoned rejection while the `await`
  hangs forever. Resolve plain data, never the plugin. (#181 —
  surfaced as the keep-alive foreground service silently never
  starting, so audio froze minutes after screen-off.)

### 7. One-time native prompts: don't burn the flag

Battery-optimization exemption shows a system dialog once per
install. Only mark `prompt_shown` in localStorage **after** the
status check succeeded — a failed plugin call must not consume
the single prompt. Same pattern as the update-check throttle.

### 8. State races get louder in a WebView

Lock-screen auto-advance died because a store write preceded
`navigate()`, letting the URL→store sync effect revert the
chapter mid-`play()` — `AbortError` surfaced as "Failed to play
audio". Fixes that generalize:

- One source of truth: navigate, let the route sync the store;
  never write passage state ahead of the URL.
- Detach listeners (`howl.off()`) before `stop()`/`unload()` so a
  torn-down Howl's pending `play()` rejection can't reach the UI.
- Pass Howler `format` from the source URL's extension — `blob:`
  URLs have none and leave the Howl `unloaded` on CORS hosts.

### 9. Security settings that are easy to forget

- `android:allowBackup="false"` — Auto Backup would carry WebView
  localStorage (incl. the auth token) onto other devices.
- Sign with an openssl-generated **PKCS12** keystore — no JKS
  needed; `apksigner` auto-detects the type. Store it base64'd in
  CI secrets; `zipalign` before `apksigner sign`.

### 10. Keep the generated project in git

`android/` is committed (pure codegen output). Nobody needs a
local Android toolchain — CI runs `npx cap sync android` then
gradlew. Add a **cap-sync drift check** job (`git status
--porcelain android/` must be empty after sync) so the committed
project can't silently diverge from `package.json` plugins.

### 11. Debug artifacts for blind iteration

Without a device farm, a `workflow_dispatch` + `android`-branch
push that builds a **debug APK artifact** (30-day retention) is
the fastest iteration loop — install by hand, check the fix,
then let the signed release path ship it.

- Branch builds can also publish `reactive-bible-debug.apk` to a
  rolling `media-debug` prerelease; an `applicationId` suffix
  lets it install *next to* the release app. `VITE_MEDIA_DEBUG=1`
  adds a "Send diagnostic report" menu row and skips the auto
  update prompt.
- `gh workflow run "Android install test" [-f apk_url=<url>]`
  runs the emulator install suite (fresh / update /
  PackageInstaller / Chrome download on API 24/33/35/36)
  against any APK URL — the closest thing to a device farm.

See also: [android-lockscreen-audio.md](android-lockscreen-audio.md).
