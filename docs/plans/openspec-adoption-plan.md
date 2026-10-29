# OpenSpec Adoption — status: implemented

Goal: adopt OpenSpec (spec-driven development, npm
`@fission-ai/openspec`) for this workspace. Decided 2026-10-29
after surveying OpenSpec docs (setup, spec-driven schema, stores
beta) and the workspace layout.

## Decision: umbrella at the workspace root, not per-repo

Run `openspec init` once at the workspace root and commit
`openspec/` to the root `.git` — the same repo that already
versions `AGENTS.md`, `docs/plans/`, and `.devin/`. Do NOT init
inside `bible_research/` or `reactive-bible/`.

Why umbrella wins here:

1. **Features are cross-repo by default.** Workspace AGENTS.md
   states provider routing, the `error`/`error_code` contract,
   and note/tag/comment/image payloads "usually touch BOTH
   repos". One change folder with one proposal + tasks covering
   both codebases matches reality; per-repo OpenSpec would force
   splitting or duplicating every shared change.
2. **Planning is already centralized.** `docs/plans/` lives at
   the root precisely because plans span repos. OpenSpec
   `changes/` is the same idea with structure and lifecycle
   (propose → apply → archive).
3. **The root git repo is a free "store".** OpenSpec's
   multi-repo answer is a beta "store" — a standalone planning
   repo plus per-machine registration and worksets. The root
   repo already plays that role; a third repo adds sync burden
   for no benefit.
4. **Specs belong at the contract level.** The valuable specs
   here (provider routing by `fileset_id`, the error contract,
   auth quirks, translations normalization) are system
   contracts both repos implement — neither repo owns them
   alone.

What per-repo init would buy, and why it's not worth it now:
specs sit next to code and travel with repo checkouts/PRs —
but these repos are always worked from this workspace, and the
shared-contract specs are the ones that matter. Revisit only if
a repo starts being cloned/edited standalone.

## Prerequisites

- `npm i -g @fission-ai/openspec` (v1.14.x; node/npm live in
  `/opt/homebrew/bin` — prefix `export PATH=...` per rule 12).
- No DB, no secrets, no deploy impact — planning docs only.

## Steps

1. **Install + init at root.** `openspec init` in the workspace
   root (use non-interactive flags — `--tools` or equivalent —
   picking whichever option writes shared `.agents/skills/`
   workflow skills). Then symlink each `openspec-*` skill into
   `.devin/skills/` per the Decisions section. Leave commits to
   the user — report the file list.
2. **Write `openspec/config.yaml` context.** Record the
   two-repo layout, that specs describe system-level contracts,
   that change tasks must name the target repo
   (`backend:`/`frontend:`/`both:` prefixes), 79-char limit,
   `Type: Capitalized` commit format, and the no-migrations
   rule.
3. **Seed `openspec/specs/` capabilities** — backfill from the
   two AGENTS.md files and DEVELOPER_GUIDEs. Suggested set:
   - `provider-routing` — `fileset_id` dispatch (ESV →
     API.Bible → SWORD → DBT), caching wrapper, FUMS `meta`,
     `*DA` audio ids.
   - `error-contract` — `error`/`error_code` taxonomy, HTTP
     mapping, `book_not_in_fileset`, frontend error classes.
   - `auth` — token/`Bearer` auth, no auto-provisioning,
     `guest` read-only, 401 semantics, `publicFetch` vs
     `authenticatedFetch`.
   - `annotations` — notes/tags/comments/images/reading
     positions: ownership scoping, ordering/indexes, public
     counts.
   - `translations-listing` — `/bible/translations/`
     normalization + frontend `filesetGroups.ts` fallback.
   - `audio-tts` — SWORD/GCS TTS modes, KJV wordpocket audio,
     media session.
   - `search`, `reading-sync` as needed.
   This is the bulk of the work — do 2–3 core specs first
   (`provider-routing`, `error-contract`, `auth`), grow the
   rest as real changes touch them.
4. **Decide the docs/plans relationship.** New work goes
   through `openspec changes`; keep existing `docs/plans/*`
   as-is (they are history), keep rule 10 for non-OpenSpec
   docs (investigations, reports). Update workspace AGENTS.md
   to say so.
5. **Smoke-test the CLI** from the workspace root:
   `openspec list`, `openspec list --specs`,
   `openspec validate` — all must pass. Do not run a real
   change through the pipeline yet.
6. **Update workspace AGENTS.md** with an OpenSpec section:
   where it lives, the umbrella rationale, task repo-prefix
   convention, and that `docs/plans` is frozen for new work.

## Decisions on setup details

- **Skill install location**: `.devin/skills/` at the workspace
  root — the confirmed discovery point at this level
  (`.agents/skills/` is only proven inside a subrepo). Run
  `openspec init` with whatever tool option it offers, then
  symlink each `openspec-*` skill dir into `.devin/skills/`;
  fall back to copying if discovery doesn't follow symlinks.
  Verify each SKILL.md frontmatter is plain `name`/
  `description` — extra fields silently break discovery.
- **CLI invocation**: always run `openspec` commands from the
  workspace root — the user always opens this directory, so
  upward root-resolution is irrelevant; a fixed convention is
  more deterministic than depending on it.
- **Archive**: `openspec/changes/archive/` is committed to
  root `.git` like all other planning history.
