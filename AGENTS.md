# AGENTS.md — Bible Research workspace

This directory is a **workspace containing two independent git
repos** (not a monorepo — each subdir has its own `.git`, remotes,
and deploy pipeline). The root-level `.git` only tracks workspace
config/docs; it does NOT track the app repos.

- `bible_research/` — Django 6.1 + DRF REST API → Google App
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
  `ENGNIV_API`/`ENGNIVC1DA` (API.Bible NIV text/audio),
  DBT ids otherwise. Changes to provider routing, the
  `error`/`error_code` failure contract, or note/tag/comment
  payloads usually touch BOTH repos.
- `/bible/translations/` normalization lives in
  `src/utils/filesetGroups.ts` (frontend) with the backend
  half in `bible/services/fileset_groups.py` (`text_options`/
  `audio_options` fields, 404
  `{"error_code": "book_not_in_fileset"}`) — merged on `main`
  via Bible-Research/bible_research#29. The leftover
  `feat/fileset-groups` branch is an older revision lacking
  API.Bible entries — do not work from it. The frontend still
  falls back to client-side grouping and walks an
  `alternates` chain (codec → other-testament → same-kind →
  any option).
- Note image uploads are the one backend feature the frontend
  does not consume yet (comment images and reading positions
  ARE consumed — see `reactive-bible/AGENTS.md` "Known gaps").

## Deployments

- **Backend: Google App Engine only, never Vercel.** Prod host:
  `bible-research-489314.ey.r.appspot.com`. `gcloud app deploy`
  uploads the *local working directory* — always deploy from a
  clean worktree on `origin/main`, never from the user's
  feature-branch checkout (a WIP branch was briefly deployed
  this way on 2026-10-26).
- **Frontend: Vercel only.** Prod domain:
  **`bibleresearch.vercel.app`**, served by the Vercel project
  `bible-research` (team `industr-analysers-projects`), which
  builds PR previews + main-merge production deploys from the
  fork. Do NOT confuse it with `reactive-bible.vercel.app` —
  that serves the *upstream* `reactive-bible` project the repo
  was forked from; it rebuilds `main` independently, is NOT
  this project's prod, and env vars set in `bible-research`
  don't apply to it. `bible-research.vercel.app` and
  `bibleresearchapi.vercel.app` are likewise stale redirects
  (the latter 302s to `/api/v1/docs/`).
- **Vercel env vars bake in at build time.** `VITE_*` vars are
  compiled into the bundle; changing or adding them requires a
  redeploy to take effect, and they must be scoped to
  Production. Build-time secrets (`SENTRY_AUTH_TOKEN`,
  `SENTRY_ORG`, `SENTRY_PROJECT`) are `process.env`-only and
  never reach the client. `vite.config.ts` logs a
  `[sentry-env-debug]` presence-only summary in every build.
- **Sentry** (org `bible-research` at de.sentry.io): projects
  `bible-research-api` (Django) + `reactive-bible-web` (React);
  DSNs live in GCP Secret Manager (`SENTRY_DSN`) and Vercel env
  (`VITE_SENTRY_DSN`) respectively; per-repo event-stream
  contracts are documented in each repo's `AGENTS.md`.

## Workspace-wide rules (apply everywhere)

1. **79-char max line length in every file type** (py, ts, md).
   Verify: `awk 'length > 79 {print FILENAME":"NR}' <file>`.
   Match the file's style when wrapping (split imports, break
   args/props onto their own lines). Fix violations you
   introduce, plus pre-existing long lines in files you are
   already modifying.
2. **Never run DB migrations** (`makemigrations`, `migrate`,
   `sqlmigrate`), even if a task seems to need them (e.g. tests
   failing on missing tables). Tell the user the needed command
   and let them run it.
3. **Commit format**: `Type: Capitalized message` —
   `Feat:`/`Fix:`/`Docs:`/`Refactor:`/`Test:`/`Chore:`.
4. **Commits/PRs happen inside each subrepo**, on feature
   branches (never `main`). Stage only files you created or
   modified this session — never `git add -A`, `git add .`, or
   `git commit -a`; stage each file by path and confirm the
   staged set with `git status` first. Leave the user's own
   dirty files alone; never commit plan/scratch markdown
   (`*-plan.md`, `PR_DESCRIPTION.md`…).
5. **PR creation**: write the body to a temp file and use
   `gh pr create --body-file` — see `.devin/workflows/create-pr.md`.
6. **Update the repo's `DEVELOPER_GUIDE.md`** after functionality
   changes — it's the project's contract doc for future sessions.
7. No secrets in code: backend uses `config.yaml` (gitignored) /
   GCP Secret Manager; frontend uses `VITE_*` env vars.
8. `bible_research`'s local `config.yaml` points `default` at the
   production Aiven Postgres via the read-only `ai_agent` user —
   use it for read-only investigation; pytest stays on in-memory
   SQLite. Details in `bible_research/AGENTS.md` ("Databases").
9. **Test login**: when the user asks you to log in and run tests
   (UI or API), use the account `Tester5` / `password123`.
10. **Write all plans in `docs/plans/`** — implementation plans,
    investigations, and design docs belong in the workspace-level
    `docs/plans/` directory as `<feature>-plan.md` (multi-file
    plans get a subdirectory, e.g. `rust-rewrite/`), each with a
    status header (implemented / partial / in progress / not
    implemented / reverted). Never put plan files inside the app
    subrepos.
11. **Record environment/setup learnings in `AGENTS.md`.** When
    you hit local-setup, infrastructure, or missing-tooling
    friction — a CLI that isn't installed on this machine, a
    non-obvious way to spin the project up locally, a required
    tool version or local-only path — don't burn time trying
    many failing alternatives. If a needed tool is missing,
    tell the user what to install rather than working around it
    repeatedly. And whenever you do figure out the working flow
    or a local quirk, write it into the relevant `AGENTS.md`
    (this file for workspace-level, the subrepo's for
    repo-specific) so the next agent uses it directly — commit
    that doc update alongside the work that discovered it.
12. **`gcloud`, `gh`, `node`/`npm` are installed but not on
    agent PATH.** They live under `/opt/homebrew/bin` (Homebrew);
    agent shells only get
    `/usr/bin:/bin:/usr/sbin:/sbin`, so prefix commands with
    `export PATH="/opt/homebrew/bin:$PATH"` on every exec call —
    env does not persist between calls. Authenticated account:
    `tedis.rozenfelds@gmail.com`; open billing account
    `016064-8B1D40-715CAA`. Projects incl.
    `bible-research-489314`, `django-apps-7345`,
    `gen-lang-client-0833674612` (Industry Analyser).
13. **Merging PRs on user request**: when the user asks you to
    merge a pull request, do it — `gh pr merge <N> --squash
    --admin` is allowed even when a required review is unmet;
    don't block on branch-protection review requirements. The
    only reason to hold back is if the merge could introduce
    significant ongoing costs (e.g. Terraform changes that
    scale up infra, code that keeps services running
    constantly) — flag those to the user instead of merging.

## Agent config layout

Instructions live in `AGENTS.md` files only (this file plus the
per-repo `AGENTS.md`) — the cross-tool standard every agent
reads. Workspace-level skills live in `.devin/skills/` (the
location discovered at this root); repo-scoped skills live in
`<repo>/.agents/skills/` (`SKILL.md` format). Skill frontmatter
must be plain `name`/`description` — extra fields like
`triggers` or `argument-hint` silently break discovery.

- `.devin/skills/fix-good-first-issue-{frontend,backend}/` —
  issue-fixing workflows for `reactive-bible` and
  `bible_research`; both include a step to check whether the fix
  belongs in the other repo.
- `.devin/skills/implement-plan/` — execute a `docs/plans/*`
  file with subagents.
- `.devin/skills/openspec-*/` — OpenSpec workflow skills
  (propose, apply, update, sync, archive, explore); copies of
  the CLI-installed `.agents/skills/openspec-*/` with
  frontmatter stripped to `name`/`description` (see OpenSpec
  section below).
- `reactive-bible/.agents/skills/vercel-react-best-practices/` —
  React perf rules.

Other `.devin/` entries:

- `.devin/workflows/create-pr.md` — PR body via `--body-file`.
- `.devin/config.local.json` — permission allowlist (untracked).

## OpenSpec

`openspec/` at the workspace root is the spec-driven planning
layer (npm `@fission-ai/openspec`, v1.14.x). It is an umbrella
over BOTH repos by design: the contracts that matter (provider
routing, `error`/`error_code`, auth) are cross-repo, so specs
live at the system level — never run `openspec init` inside
`bible_research/` or `reactive-bible/`.

- **Run `openspec` commands only from the workspace root** —
  a fixed convention, not upward root-resolution.
- `openspec/specs/` holds capability specs (current behavior as
  source of truth): `provider-routing`, `error-contract`,
  `auth` — grow as real changes touch new areas.
- `openspec/config.yaml` carries project context; change
  `tasks.md` items MUST be prefixed `backend:` /
  `frontend:` / `both:` to name the target repo.
- Workflow: `openspec-propose` → implement via
  `openspec-apply-change` → `openspec-archive-change`. The
  skills live in `.devin/skills/openspec-*` (copies — the
  upstream frontmatter in `.agents/skills/openspec-*/` has
  extra fields that break Devin discovery, so keep the
  `.devin` copies stripped if `openspec update` refreshes the
  originals).
- `openspec/changes/archive/` is committed to root `.git`
  like all other planning history.
- **`docs/plans/` is frozen for new feature plans** — new
  feature work goes through `openspec changes`. Existing
  `docs/plans/*` stay as history, and investigation/report
  docs (non-feature) still belong there per rule 10.

## Documentation map (and staleness)

- `bible_research/DEVELOPER_GUIDE.md` — API reference; predates
  comments/images/reading-positions, SWORD+ESV providers, and the
  Django 6.1 upgrade.
- `reactive-bible/DEVELOPER_GUIDE.md` — architecture; predates
  auth, comments, mentions, Android.
- `reactive-bible/docs/` — test documentation
  (`TAG_REFRESH_TESTS.md`).
- `docs/plans/` — historical implementation
  plans/investigations, incl. `docs/plans/rust-rewrite/`
  (multi-file). Each doc carries a status header: implemented /
  partial / in progress / not implemented / reverted. Frozen
  for new feature plans — see the OpenSpec section.
- `openspec/` — spec-driven planning layer: `specs/`
  (capability contracts), `changes/` + `changes/archive/`
  (change lifecycle), `config.yaml` (project context).
- `bible_research/docs/` — GCP / Artifact Registry cleanup
  runbooks.
