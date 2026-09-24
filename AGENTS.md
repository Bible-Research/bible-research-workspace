# AGENTS.md — Bible Research workspace

This directory is a **workspace containing two independent git
repos** (not a monorepo — each subdir has its own `.git`, remotes,
and deploy pipeline). The root-level `.git` only tracks workspace
config/docs; it does NOT track the app repos.

- `bible_research/` — Django 5.2 + DRF REST API → Google App
  Engine (`bible-research-489314.ey.r.appspot.com`); infra in
  `terraform/`.
- `reactive-bible/` — React 18 + TS + Vite SPA → Vercel; also a
  Capacitor Android app (`android/`).

**Read the repo-specific `AGENTS.md` before working in either
repo** — they contain the business-logic details (provider routing,
fileset model, auth quirks, caching, error contracts).

## How the pieces connect

- Frontend → backend REST API under `/api/v1/`
  (`bible_research`); auth via `Authorization: Token …`.
- `fileset_id` is the shared routing key: `ENGKJV` (local bundle),
  `ENGESV_API` (api.esv.org), `LVSGLU8` (SWORD + GCS TTS audio),
  DBT ids otherwise. Changes to provider routing, the
  `error`/`error_code` failure contract, or note/tag/comment
  payloads usually touch BOTH repos.
- Backend features the frontend does not consume yet: image
  uploads, `reading-positions`.

## Workspace-wide rules (apply everywhere)

1. **79-char max line length in every file type** (py, ts, md).
   Verify: `awk 'length > 79 {print FILENAME":"NR}' <file>`.
2. **Never run DB migrations** (`makemigrations`, `migrate`,
   `sqlmigrate`). Tell the user the needed command instead.
3. **Commit format**: `Type: Capitalized message` —
   `Feat:`/`Fix:`/`Docs:`/`Refactor:`/`Test:`/`Chore:`.
4. **Commits/PRs happen inside each subrepo**, on feature
   branches (never `main`). Stage only files you edited; never
   `git add -A`; never commit plan/scratch markdown
   (`*-plan.md`, `PR_DESCRIPTION.md`…).
5. **PR creation**: write the body to a temp file and use
   `gh pr create --body-file` — see `.devin/workflows/create-pr.md`.
6. **Update the repo's `DEVELOPER_GUIDE.md`** after functionality
   changes — it's the project's contract doc for future sessions.
7. No secrets in code: backend uses `config.yaml` (gitignored) /
   GCP Secret Manager; frontend uses `VITE_*` env vars.

## Existing agent config (don't duplicate it)

- `.devin/rules/` — no-migrations, line-length,
  commit-only-edited-files (auto-loaded).
- `.devin/workflows/create-pr.md` — PR body via `--body-file`.
- `.devin/skills/fix-good-first-issue/` — issue-fixing workflow.
- `reactive-bible/.devin/skills/vercel-react-best-practices/` —
  React perf rules (full doc at
  `reactive-bible/.agents/skills/vercel-react-best-practices/AGENTS.md`).
- `.windsurfrules` — workspace overview (Windsurf format).
- `bible_research/WINDSURF_AI_GUIDELINES.md` — backend style rules.
- `reactive-bible/CLAUDE.md` + `reactive-bible/.devin/rules.md` —
  older frontend guides.

## Documentation map (and staleness)

- `bible_research/DEVELOPER_GUIDE.md` — API reference; predates
  comments/images/reading-positions, SWORD+ESV providers, and the
  Django 5.2 upgrade.
- `bible_research/WINDSURF_AI_GUIDELINES.md` — style/architecture
  patterns, mostly accurate.
- `reactive-bible/DEVELOPER_GUIDE.md` — architecture; predates
  auth, comments, mentions, Android.
- `reactive-bible/CLAUDE.md` — quick reference; component list is
  stale.
- `reactive-bible/docs/*.md` — feature plans (Android APK, audio,
  USFM URLs).
- Root `*-plan.md` files — historical implementation plans,
  reference only.
- `bible_research/docs/` — GCP / Artifact Registry cleanup
  runbooks.

Root-level plan files (`image-attachments-plan.md`,
`inline-comment-images-plan.md`,
`note-translation-reference-plan.md`) describe features that have
since landed (comments, images) or are in progress.
