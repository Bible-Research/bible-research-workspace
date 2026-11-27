# Tasks

## 1. Command changes (bible_research)

- [x] 1.1 In `bible/management/commands/generate_edge_audio.py`,
      add `--until-done` (store_true). When set, skip both the
      chapter cap and the time cap in the pending loop (treat as
      unbounded); keep `--max-chapters`/`--max-minutes` working
      as today when the flag is absent. Update the option help
      text.
- [x] 1.2 In the same command, default the local-mode `RunState`
      to `<repo>/edge-tts-state.json` (mirroring
      `gemini_tts.state.DEFAULT_STATE_PATH` resolution) instead
      of `None`; `--state-file` still overrides.
- [x] 1.3 Verify `.gitignore` covers `edge-tts-state.json` and
      `tts-out/`; add patterns only if missing.

## 2. Tests

- [x] 2.1 In `bible/tests/test_generate_edge_audio_command.py`:
      `--until-done` generates past the default 60-chapter cap
      (mock synthesizer/SWORD worklist; assert >60 processed).
- [x] 2.2 Same file: without `--until-done`, the run still stops
      at `--max-chapters`/`--max-minutes`.
- [x] 2.3 Same file: default state path is
      `edge-tts-state.json`, not `gemini-tts-state.json`; an
      existing gemini state file does not mark chapters
      complete. `--state-file` override still honored.

## 3. Docs

- [x] 3.1 `DEVELOPER_GUIDE.md`: document the standalone Linux
      recipe — `git clone`, `python3 -m venv venv`,
      `pip install -r requirements.txt`,
      `cp config_template.yaml config.yaml`, then
      `python manage.py generate_edge_audio --until-done`
      (output under `tts-out/LVSGLU8/<BOOK>/<chapter>.mp3` +
      `_timestamps.json`; optional `while true` restart loop).
- [x] 3.2 Repo `AGENTS.md`: one-line note that the edge job has
      an unbounded local mode (only where the job is already
      documented — no new section).

## 4. Verification

- [x] 4.1 `python manage.py generate_edge_audio --until-done
      --dry-run` reports the full pending worklist.
- [x] 4.2 `pytest bible/tests/test_generate_edge_audio_command.py
      bible/tests/test_edge_tts_client.py` green.
- [x] 4.3 `awk 'length > 79 {print FILENAME":"NR}'` clean on
      every touched file.
