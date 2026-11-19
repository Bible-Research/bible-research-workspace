# Design

## Context

`generate_edge_audio` (bible/management/commands/) already does
the hard parts: it enumerates the full chapter worklist from the
vendored SWORD module (`bible/sword_modules/LvGluck8.zip`),
synthesizes each verse through `EdgeTTSClient` (retry with
exponential backoff + jitter), concatenates to a chapter MP3 with
exact verse timestamps via `EdgeSynthesizer`, and resumes across
runs — GCS listing in `--upload-gcs` mode, JSON `RunState` +
on-disk artifact check in local mode.

Two properties fight the "one long-running local command" goal:

- `DEFAULT_MAX_CHAPTERS = 60` / `DEFAULT_MAX_MINUTES = 60.0` are
  sized for the `all-jobs` Cloud Run Job timeout, not a local
  box. Neither option accepts 0 as "unlimited" (`0` short-
  circuits the loop immediately).
- `RunState` defaults to `DEFAULT_STATE_PATH`
  (`gemini-tts-state.json`). Both `generate_gemini_audio` and
  `generate_edge_audio` pass `state_file=None` today, so a
  second local engine would read the first engine's completion
  ledger. `_pending()` checks `state.is_completed()` BEFORE the
  on-disk artifact check, so stale shared state silently skips
  real work.

Django settings also require a `.env` or `config.yaml` to exist
on a fresh clone (FileNotFoundError otherwise, pytest excepted),
even though the edge path needs no secrets — a docs item, not a
code change.

## Goals / Non-Goals

**Goals:**
- One documented command generates the whole Bible locally:
  `python manage.py generate_edge_audio --until-done`.
- Local edge runs resume correctly even on a checkout where
  Gemini audio was previously generated.
- Zero behavior change for the Cloud Run Job (`--upload-gcs`)
  path.

**Non-Goals:**
- Uploading locally generated artifacts to GCS (out of scope —
  `--upload-gcs` already exists for that).
- Parallelism/concurrency options (serial pacing is deliberate —
  the endpoint throttles bursts).
- Changing `generate_gemini_audio` defaults (it keeps
  `gemini-tts-state.json`; only the edge command stops sharing
  it).

## Decisions

1. **`--until-done` flag over a 0/negative sentinel.** A named
   flag self-documents the intent (`--until-done` reads
   naturally next to `--dry-run`), and keeps `--max-chapters` /
   `--max-minutes` typed `int`/`float` with no magic values.
   Implementation: when set, both caps are treated as infinity;
   explicit `--max-*` values are ignored (documented in help).
2. **Edge gets `edge-tts-state.json` as its default state
   file.** `RunState` already accepts an explicit path — the
   command just stops passing `None`. Path lives at repo root
   beside `gemini-tts-state.json` (already gitignored — verify
   `*.json`/`tts-out/` ignore coverage; add `edge-tts-state.json`
   to `.gitignore` only if the existing pattern misses it).
3. **Docs-only for the fresh-clone settings requirement.**
   `settings.py` intentionally hard-fails without `.env` /
   `config.yaml`; the fix is documenting
   `cp config_template.yaml config.yaml` (placeholder values —
   the edge path touches no DB or cloud service) rather than
   weakening the guard.

## Risks / Trade-offs

- **Endpoint flakiness over a multi-day run**: already handled
  in-code (per-verse retry/backoff, per-chapter failure logged
  and skipped, resume on restart). An operator may want a
  `while true; do ...; sleep 60; done` shell loop for
  belt-and-suspenders — document it as optional, not required.
- **Stale shared state on existing checkouts**: operators who
  already ran the edge command with the old shared default keep
  their progress in `gemini-tts-state.json`; the new default
  would restart them from scratch (harmless — artifacts on disk
  are re-checked via `chapter_files_exist`, so real MP3s are not
  re-synthesized).
- **`--until-done` on `--upload-gcs`**: allowed but pointless;
  the day lock and GCS listing already bound cloud runs. No
  special-casing needed — document that it is meant for local
  mode.
