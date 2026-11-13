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
- **Never deploy preview/staging/ephemeral resources** — no
  PR-preview or staging Cloud Run services, no extra GAE
  versions left serving, no second deployments of anything.
  Idle resources are what blew the Oct 2026 bill: every serving
  GAE version runs its own `min_instances` instance 24/7, and
  `-staging`/`-pr-*` Cloud Run leftovers add up. Manual
  `gcloud app deploy` MUST pass `--stop-previous-version` and
  be followed by
  `bible_research/scripts/cleanup_app_engine_versions.sh`
  (CI already does both). Anything that must exist long-term
  has to scale to zero.
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

Universal rules — follow exactly:

1. **Max 79 chars per line, every file type.** Verify:
   `awk 'length > 79 {print FILENAME":"NR}' <file>`. Fix
   violations you introduce plus long lines in files already
   being modified. Exempt: formatter output (e.g. Prettier at
   printWidth 80 may emit 80-char lines) and vendored upstream
   skill files under `.agents/skills/` and `.devin/skills/`
   (`openspec-*`, `vercel-react-best-practices/`) — refresh
   tools own them.
2. **Commit format** `Type: Capitalized message` —
   `Feat:`/`Fix:`/`Docs:`/`Refactor:`/`Test:`/`Chore:`.
3. **Commit on feature branches only, never `main`.**
   Name branches `<type>/<slug>` with the commit types
   (`feat/`, `fix/`, `docs/`, `refactor/`, `test/`, `chore/`).
   Stage only files changed this session, by path — never
   `git add -A`, `git add .` or `git commit -a`; confirm the
   staged set with `git status` first; leave the user's own
   dirty files alone; never commit plan/scratch markdown.
4. **Dangerous commands — never run** the destructive/deploy
   commands listed under "Repo-specific rules" below; write
   the change and tell the user the command to run instead.
5. **Scoped verification** — run only the tests/checks for
   what you changed, not the whole suite.
6. **PR bodies go to a temp file** + `gh pr create
   --body-file`, never inline `--body`; delete the temp file
   after. See `.devin/skills/create-pr/`.
7. **Every PR body includes a UI verification / QC section** —
   concrete steps (pages, clicks, expected result) exercising
   the change end-to-end through the UI. Omit only when
   genuinely unreachable from the UI — then give an
   equivalent (curl command, log line, file to inspect).
8. **GitHub remote targeting**: when `origin` is the user's
   fork (an `upstream` remote exists), ALL GitHub operations
   — issues, PRs, comments, releases — target the fork via
   `--repo <owner>/<fork>`; never upstream unless asked.
   Otherwise `origin` is the target.
9. **No secrets in code.** Secrets live in gitignored local
   files / env vars / GCP Secret Manager. Never create new
   Secret Manager secrets — append KEY=VALUE lines to the
   shared `env-file` secret (each extra secret adds cost).
10. **Update docs after functionality changes** — this file
    and the repo's other contract docs (README /
    DEVELOPER_GUIDE / docs/*) — in the same change.
11. **`docs/plans/` is frozen** — existing files stay as
    history with their status headers (implemented / partial /
    in progress / not implemented / reverted); nothing new is
    added there or anywhere else — all new planning goes
    through OpenSpec (`openspec-propose` for changes,
    `openspec-explore`/change artifacts for investigations).
12. **This file is self-updating.** Missing tool → tell the
    user what to install instead of grinding through failing
    alternatives. Discovered flow/local quirk → write it into
    the relevant AGENTS.md and commit that doc update
    alongside the work that discovered it.

### Repo-specific rules

- **Dangerous commands — never run**: `makemigrations`,
  `migrate`, `sqlmigrate` (even if a task seems to need them,
  e.g. tests failing on missing tables — tell the user the
  needed command); `gcloud app deploy` and other mutating
  `gcloud` calls; `terraform apply`; creating any
  preview/staging/ephemeral resources (see Deployments).
- **Commits/PRs happen inside each subrepo** on feature
  branches — the root repo only tracks workspace config/docs.
- `bible_research`'s local `config.yaml` points `default` at
  the production Aiven Postgres via the read-only `ai_agent`
  user — use it for read-only investigation; pytest stays on
  in-memory SQLite. Details in `bible_research/AGENTS.md`
  ("Databases").
- **Test login**: `Tester5` / `password123` — use it when
  asked to log in and run tests (UI or API).
- **Merging PRs on user request**: `gh pr merge <N> --squash
  --admin` is allowed even when a required review is unmet —
  don't block on branch-protection review requirements. Hold
  back only if the merge could introduce significant ongoing
  costs (e.g. Terraform that scales up infra, code that keeps
  services running constantly) — flag those to the user
  instead of merging.
- **Worktrees are mandatory for all implementation work** —
  never edit/commit in any repo's main checkout: it may hold
  the user's uncommitted work or another agent's session. For
  every task: `git -C <repo> fetch origin`, then
  `git -C <repo> worktree add worktrees/<slug>
  -b <type>/<slug> origin/main` and do all work — commits,
  push, PR — inside it; follow-up requests get a FRESH
  worktree, never a reused one. Finishing a task means a
  green, reviewed PR — never ask the user whether to open
  one. After pushing: open the PR ready for review (NEVER a
  draft) per `.devin/skills/create-pr/`, then run a review
  loop — spawn a FRESH reviewer subagent on the diff, fix
  everything useful it reports, push, and repeat with a new
  reviewer until a round finds nothing meaningful. Once the
  loop is clean and the PR's checks are green, clean up
  IMMEDIATELY — don't wait for the merge (the PR lives on
  the remote branch; removing the local worktree/branch does
  not touch it, and the remote branch is deleted at merge
  time). From the main
  checkout: `git -C <repo> worktree remove
  worktrees/<slug>` + `git -C <repo> branch -D
  <type>/<slug>` (`-D`, not `-d` — squash-merges leave the
  branch "unmerged"); abandoned/rejected work gets the same
  cleanup. Goal state: `git -C <repo> worktree list` shows
  only the main checkout when the task ends.
  `git -C <repo> worktree add <path>` resolves `<path>`
  relative to the REPO root, not the caller's cwd —
  `worktrees/x` lands in `<repo>/worktrees/x` (each repo
  gitignores `worktrees/`). Nested worktrees work fine:
  `.eslintrc.cjs` has `root: true` on main (merged via #215)
  so `npm run lint` no longer cascades into the parent
  checkout's config. Backend agents: each worktree needs its
  own venv (`python3 -m venv venv`) and a `config.yaml`
  symlink to the main checkout's (gitignored — worktrees
  don't get one); reactive-bible worktrees need their own
  `npm install`.
- Never put plan files inside the app subrepos — planning
  lives at the workspace root only (rule 11 + OpenSpec below).

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
- `.devin/skills/implement-plan/` — legacy: executes a
  pre-existing `docs/plans/*` file with subagents (new planning
  uses OpenSpec — see below).
- `.devin/skills/openspec-*/` — OpenSpec workflow skills
  (propose, apply, update, sync, archive, explore); copies of
  the CLI-installed `.agents/skills/openspec-*/` with
  frontmatter stripped to `name`/`description` (see OpenSpec
  section below).
- `.devin/skills/create-pr/` — PR body via `--body-file`.
- `reactive-bible/.agents/skills/vercel-react-best-practices/` —
  React perf rules.

Other `.devin/` entries:

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
- **Always run `openspec validate --all --strict` after
  touching `openspec/`** — proposing, editing specs, syncing
  or archiving. It must pass before committing; strict mode
  catches >500-char requirement bodies and spec/delta drift.
- The CLI is a global npm install — it lives in
  `/opt/homebrew/bin` next to `node`/`npm`, which is not on
  agent-shell PATH (see the machine-environment notes in
  `~/.config/devin/AGENTS.md`).
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
- `openspec/changes/` has never held a proposal — two
  post-adoption features were planned via `docs/plans/`
  before the convention landed. No retroactive migration;
  `changes/archive/` only gains entries for work proposed
  through `openspec changes`. The `docs/plans/` freeze is
  now total (rule 11).
- **When to use it**: features, breaking changes, contract
  changes, anything a reviewer would ask "why?" about.
  **Skip the ceremony** for bug fixes restoring spec'd
  behavior, typos, config/dep bumps, and tests — fix those
  directly. Pure refactors may propose with
  `skip_specs: true` in the change's `.openspec.yaml`.
- **Specs describe observable behavior** — error codes,
  HTTP statuses, payload shapes, routing outcomes. Prefer
  behavior-level wording, but naming the code symbol
  (function, file, registry key) is acceptable when it
  anchors the contract; implementation steps and library
  choices still belong in `design.md`/`tasks.md`.
- **Don't backfill specs** — write a spec only when a change
  touches that capability; `specs/` grows via archived
  deltas, not upfront documentation.
- **Archive timing**: archive the change in the
  implementing PR when feasible — specs, code and history
  land atomically. Otherwise archive immediately after the
  merge in a small follow-up commit on `main`. Never leave
  a merged change unarchived: unsynced deltas leave
  `specs/` stale.
- **Update specs in the same PR as the code.** Any change
  that alters behavior covered by `openspec/specs/` MUST
  update the affected spec in the same PR — whether it came
  through `openspec changes` (that update is the archive) or
  not (edit the spec directly). A spec that no longer
  matches shipped behavior is a bug, not documentation
  drift.
- **`docs/plans/` is fully frozen** — new feature work goes
  through `openspec changes`, and investigations/reports go
  through `openspec-explore` and change artifacts. Nothing
  new is added to `docs/plans/` (rule 11); existing files
  stay as history.

## Documentation map (and staleness)

- `bible_research/DEVELOPER_GUIDE.md` — API reference; predates
  comments/images/reading-positions, SWORD+ESV providers, and the
  Django 6.1 upgrade.
- `reactive-bible/DEVELOPER_GUIDE.md` — architecture; predates
  auth, comments, mentions, Android.
- `reactive-bible/docs/` — test documentation
  (`TAG_REFRESH_TESTS.md`).
- `docs/plans/` — frozen archive of historical implementation
  plans/investigations, incl. `docs/plans/rust-rewrite/`
  (multi-file). Each doc carries a status header: implemented /
  partial / in progress / not implemented / reverted. Nothing
  new is added — see rule 11 and the OpenSpec section.
- `openspec/` — spec-driven planning layer: `specs/`
  (capability contracts), `changes/` + `changes/archive/`
  (change lifecycle), `config.yaml` (project context).
- `docs/research/` — committed reference/source documents
  (e.g. `ChronologyOfTheFourGospels.pdf`). New
  research/audit reports belong here going forward;
  `docs/plans/codebase-health-review/` predates this dir but
  stays put under the freeze.
- `bible_research/docs/` — GCP / Artifact Registry cleanup
  runbooks.
