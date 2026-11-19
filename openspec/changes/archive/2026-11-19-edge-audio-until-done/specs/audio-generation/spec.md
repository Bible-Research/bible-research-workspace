# Spec Delta

## Purpose

Defines how chapter audio artifacts (MP3 + verse timestamps)
are generated offline from SWORD text via TTS engines: run
bounds, resume semantics, and isolation between engines'
local state.

## ADDED Requirements

### Requirement: Unbounded local generation run
`generate_edge_audio` SHALL accept an `--until-done` flag that
removes the per-run chapter and wall-clock caps so a single
invocation drains the entire pending worklist. Without the
flag, the existing capped defaults SHALL be unchanged.

#### Scenario: Single command drains the worklist
- **WHEN** the operator runs `generate_edge_audio
  --until-done` in local (non-GCS) mode
- **THEN** generation continues until every pending chapter is
  synthesized or the process is interrupted, without hitting
  a chapter or time cap

#### Scenario: Defaults unchanged for scheduled runs
- **WHEN** the command runs without `--until-done`
- **THEN** it still stops at the configured `--max-chapters`
  and `--max-minutes` bounds

### Requirement: Engine-isolated resume state
Each audio generation command SHALL default to its own resume
state file, named for its engine, so completion recorded by
one engine never causes another engine to skip chapters it
has not generated.

#### Scenario: Edge run ignores Gemini state
- **WHEN** `generate_edge_audio` runs on a checkout where
  `generate_gemini_audio` previously marked chapters complete
  in its own state file
- **THEN** the edge command treats all chapters lacking edge
  artifacts as pending and synthesizes them

#### Scenario: Explicit state file still honored
- **WHEN** `--state-file` is passed explicitly
- **THEN** the command reads and writes that file instead of
  its engine default

### Requirement: Documented standalone invocation
The repository SHALL document a single-command recipe that
runs the full-Bible edge-audio generation on a fresh Linux
checkout, including any minimal configuration the Django
settings layer requires.

#### Scenario: Fresh clone instructions exist
- **WHEN** an operator follows the documented steps on a
  machine with Python and no project secrets
- **THEN** they reach a working
  `python manage.py generate_edge_audio --until-done`
  invocation without contacting cloud services
