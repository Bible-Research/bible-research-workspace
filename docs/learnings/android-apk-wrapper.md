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
main push, carrying `reactive-bible.apk` (fixed asset name → stable
URL) and `update.json`. The app polls the manifest once per launch
(hourly throttle) and compares `versionCode` to `App.getInfo().build`.

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

### 4. XML comments are not free

A `<!-- comment -->` placed *inside* the `<application ...>`
start-tag of `AndroidManifest.xml` is not well-formed XML — it
broke `processReleaseMainManifest` (`ManifestMerger2`) and killed
every main release build until moved above the element. After
editing manifests, parse-check:
`python3 -c "import xml.etree.ElementTree as ET; ET.parse(...)"`.

### 5. The WebView environment leaks web assumptions

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

### 6. One-time native prompts: don't burn the flag

Battery-optimization exemption shows a system dialog once per
install. Only mark `prompt_shown` in localStorage **after** the
status check succeeded — a failed plugin call must not consume
the single prompt. Same pattern as the update-check throttle.

### 7. State races get louder in a WebView

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

### 8. Security settings that are easy to forget

- `android:allowBackup="false"` — Auto Backup would carry WebView
  localStorage (incl. the auth token) onto other devices.
- Sign with an openssl-generated **PKCS12** keystore — no JKS
  needed; `apksigner` auto-detects the type. Store it base64'd in
  CI secrets; `zipalign` before `apksigner sign`.

### 9. Keep the generated project in git

`android/` is committed (pure codegen output). Nobody needs a
local Android toolchain — CI runs `npx cap sync android` then
gradlew. Add a **cap-sync drift check** job (`git status
--porcelain android/` must be empty after sync) so the committed
project can't silently diverge from `package.json` plugins.

### 10. Debug artifacts for blind iteration

Without a device farm, a `workflow_dispatch` + `android`-branch
push that builds a **debug APK artifact** (30-day retention) is
the fastest iteration loop — install by hand, check the fix,
then let the signed release path ship it.
