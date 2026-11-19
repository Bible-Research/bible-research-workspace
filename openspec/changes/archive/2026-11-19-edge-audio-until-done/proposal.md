# Proposal

## Why

`generate_edge_audio` already produces per-chapter MP3 + exact
verse-timestamp artifacts from the vendored LVSGLU8 SWORD module via
the Edge read-aloud endpoint, but it is tuned for the Cloud Run Job:
a 60-chapter / 60-minute cap per run, and no documented standalone
path. The user wants to run the whole Latvian Bible on a plain
Ubuntu box with a single command that queues, sources and
synthesizes everything itself. Two frictions block that: the caps
have no "unbounded" mode, and the default resume state file is
shared with the Gemini job, so a local box that has ever run
`generate_gemini_audio` would silently skip its chapters.

## What Changes

- Add `--until-done` to `generate_edge_audio`: removes the
  per-run chapter and wall-clock caps so one invocation drains
  the whole pending worklist. Default caps are unchanged —
  the Cloud Run Job path (`--upload-gcs`) keeps its bounds.
- Give the edge command its own default resume state file
  (`edge-tts-state.json`) instead of inheriting
  `gemini-tts-state.json`, so edge and gemini local runs never
  share completion state.
- Document the standalone Linux/Ubuntu invocation (venv +
  `pip install -r requirements.txt` +
  `python manage.py generate_edge_audio --until-done`),
  including the minimal secrets-file requirement Django
  settings impose on a fresh clone.

## Capabilities

### New Capabilities
- `audio-generation`: offline/scheduled generation of chapter
  audio artifacts (MP3 + verse timestamps) from SWORD text via
  TTS engines — run bounds, resume semantics and engine
  isolation of local state.

### Modified Capabilities
<!-- None — no existing spec covers audio generation. -->

## Impact

- `bible/management/commands/generate_edge_audio.py` — new flag,
  state-file default.
- `bible/tests/test_generate_edge_audio_command.py` — coverage
  for the flag and the new default.
- `bible_research/DEVELOPER_GUIDE.md` (and repo `AGENTS.md` if
  it documents the job) — standalone-run instructions.
- No deploy pipeline change: the Cloud Run Job path is
  untouched; this only widens the local-run mode.
