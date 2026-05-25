# Audio Verse Highlighting — PR Review Fixes

> **Audience:** Windsurf AI agent (Cascade).
> Address each item in order. After each fix, run the specified tests. Do not
> skip ahead. Mark each step done before moving on.
>
> **PRs under review:**
> - Backend: https://github.com/Bible-Research/bible-research/pull/11
>   (branch `feature/audio-timestamps`)
> - Frontend: https://github.com/Bible-Research/reactive-bible/pull/37
>   (branch `feature/audio-highlighting`)

---

## Step 1 — Backend: Remove global SSL-verification disable (BLOCKER)

**File:** `bible_research/bible/services/dbt/client.py`

The PR added:
```python
config.verify_ssl = False
```
inside `DBTClient.__init__`, right after the `Configuration(...)` call.

This disables TLS verification for **every** DBT API call made by this client
(bibles, search, annotations, timestamps, etc.). It is a security regression,
not required by the plan, and must be removed.

### Action
Delete the `config.verify_ssl = False` line.

If the author ran into a local cert issue, the correct fix is environment
setup (e.g. `certifi`, `REQUESTS_CA_BUNDLE`), not disabling verification in
shared production code. Do **not** add a feature flag for it.

### Verify
```bash
cd bible_research
python -m pytest bible/tests/test_dbt_client.py bible/tests/test_timestamp_view.py -v
```
All existing tests must still pass.

---

## Step 2 — Backend: Remove unused `AudioTimingApi` wiring

**File:** `bible_research/bible/services/dbt/client.py`

`get_timestamps` was implemented using raw `requests.get` (to work around a
deserialization bug in the generated `V4AudioTimestampsData` model). That is
fine, but the PR still adds an unused attribute:

```python
from openapi_client.api.audio_timing_api import AudioTimingApi
...
self.audio_timing_api = AudioTimingApi(self.api_client)
```

Neither the import nor the attribute is referenced anywhere.

### Action
Choose **one** of:

- **Preferred:** Remove both the import and the `self.audio_timing_api = ...`
  line to keep the client free of dead code.
- **Alternative:** Keep them but add a `# TODO:` comment above
  `get_timestamps` explaining why the generated client isn't used, and link
  the upstream issue (if known). Do **not** leave it undocumented.

### Verify
```bash
cd bible_research
python -m pytest bible/tests/test_dbt_client.py -v
```

---

## Step 3 — Frontend: Do not persist `audioActiveVerse`

**File:** `reactive-bible/src/store.tsx`

In the zustand `partialize` config, the PR added:
```ts
audioActiveVerse: state.audioActiveVerse,
```

`audioActiveVerse` is transient audio-session state driven by the highlighter
hook. Persisting it to localStorage causes a stale highlight to appear on
page reload before audio plays, and it has no meaning outside of an active
playback session. The plan does **not** require persistence.

### Action
Remove the `audioActiveVerse: state.audioActiveVerse,` line from the
`partialize` return object. Leave the `initialState` entry
(`audioActiveVerse: null as number | null`) and the reset logic in
`setActiveBook` / `setActiveChapter` unchanged.

### Verify
```bash
cd reactive-bible
npx vitest run src/__tests__/store.test.ts
```
Also manually: reload the page after pausing playback — no stale blue
highlight should appear until audio plays again.

---

## Step 4 — Frontend: Tighten the `v=4` test assertion (minor)

**File:** `bible_research/bible/tests/test_dbt_client.py`

`test_get_timestamps_calls_api` contains:
```python
assert 'v=4' in call_args[0] or call_kwargs.get(
    'params', {}
).get('v') == 4
```

The first branch is never true — `call_args[0]` is the URL without the
querystring; `v` is always passed via `params=`. The `or` branch masks
potential regressions.

### Action
Replace the assertion with:
```python
assert call_kwargs.get('params', {}).get('v') == 4
```

### Verify
```bash
cd bible_research
python -m pytest bible/tests/test_dbt_client.py -v
```

---

## Out of scope (do NOT change in this pass)

The following were noted in review but are **not** required fixes. Leave as
is unless explicitly asked:

- Client-side codec-suffix stripping in `Audio.tsx`
  (`activeAudioFilesetId.split('-')[0]`).
- Fetching timestamps on chapter/book change even when audio isn't playing
  (cache absorbs repeats).
- `AudioTimestampView` echoing raw exception messages to the client.
- Unused `clearTimestampCache` export.

---

## Final checklist

- [ ] Step 1 — `config.verify_ssl = False` removed.
- [ ] Step 2 — unused `AudioTimingApi` import/attribute removed (or TODO'd).
- [ ] Step 3 — `audioActiveVerse` removed from `partialize`.
- [ ] Step 4 — `v=4` test assertion tightened.
- [ ] Backend tests pass: `pytest bible/tests/test_dbt_client.py bible/tests/test_timestamp_view.py -v`
- [ ] Frontend tests pass: `npx vitest run src/__tests__/store.test.ts src/hooks/__tests__/useVerseHighlighter.test.ts src/utils/__tests__/cacheManager.timestamp.test.ts`
- [ ] Manual smoke test: play a chapter — highlight follows audio; reload page — no stale highlight.
