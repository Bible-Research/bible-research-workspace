# OpenSpec Adoption Playbook — reusable across projects

Distilled from adopting OpenSpec in the bible-research-workspace
(2026-10-29, PR #2); gotchas updated after nibrastube (PR #59)
and django-apps (PR #51) adoptions the same day. Written to be
handed to an AI agent in ANY repo as its implementation spec.
Repo-specific callouts at the end cover Next.js and Django+React
targets.

## What OpenSpec is

Spec-driven development tool (npm `@fission-ai/openspec`). It
adds an `openspec/` dir with `specs/` (source of truth — how the
system behaves NOW, organized by capability) and `changes/`
(one folder per proposed change: `proposal.md`, `design.md`,
`tasks.md`, delta `specs/` that merge into `specs/` on archive).
It also installs AI-agent workflow skills (propose / apply /
update / sync-specs / archive / explore).

## Decision: where does `openspec/` live?

- **Single repo (Next.js app, Django+React monolith, one
  deployable)** → `openspec init` at THAT repo's root, commit
  it there. This is the default and the right answer ~always.
- **Multi-repo workspace where features routinely span repos**
  → one umbrella `openspec/` in the shared planning/parent
  repo. Do NOT duplicate per repo — cross-repo changes would
  need splitting. (This workspace's case: Django API + React
  frontend sharing contracts like `fileset_id` routing and the
  `error_code` taxonomy.)
- **Multi-repo with NO shared parent repo** → OpenSpec's beta
  "store" (separate planning repo, `openspec store setup`).
  Only reach for this when options 1–2 are impossible; it adds
  per-machine registration + a third repo to sync.

## Implementation steps (validated end-to-end)

1. **Install with age pinning.** `npm view @fission-ai/openspec
   time` — install the newest version published >7 days ago
   (supply-chain hygiene; we took 1.14.3 over same-week 1.15.0):
   `npm i -g @fission-ai/openspec@<version>`.
2. **Init non-interactively.** `openspec init --tools agents`
   installs the shared-format workflow skills into
   `.agents/skills/openspec-*/` and creates `openspec/`. Check
   `openspec init --help` first — flags/option names evolve.
3. **Wire skills into YOUR agent's discovery dir.** OpenSpec
   only knows generic tool targets; verify where your agent
   actually reads skills from and bridge the gap:
   - **Devin**: reads `<root>/.devin/skills/` — and requires
     frontmatter to be ONLY `name`/`description` (extra fields
     like `allowed-tools`/`license` silently break discovery).
     Upstream OpenSpec SKILL.md files carry extra frontmatter,
     so COPY each skill into `.devin/skills/` and strip
     frontmatter — symlinks won't fix frontmatter. Keep the
     `.agents/` originals verbatim so `openspec update` can
     still refresh them; re-copy+re-strip after each update.
   - **Claude Code**: init's `--tools claude` writes
     `.claude/skills/` + commands natively — no bridging.
   - Other tools: check their skill dir; same strip-frontmatter
     rule of thumb applies wherever plain name/description is
     required.
4. **Fill `openspec/config.yaml` context.** Record what an
   agent needs to draft artifacts: repo layout, that specs
   describe CURRENT behavior (not aspirational), project
   conventions (line-length, commit format, forbidden
   operations like migrations), and — for multi-repo setups —
   a task-tagging convention (we use `backend:`/`frontend:`/
   `both:` prefixes in `tasks.md`; single repos don't need it).
   Stick to schema-documented fields (`context`, `rules.<id>`,
   `operations.<op>`); unknown fields are dropped.
5. **Seed `specs/` from authoritative docs — this is the real
   work.** Mine `AGENTS.md`/`CLAUDE.md`/`DEVELOPER_GUIDE.md`/
   READMEs for system CONTRACTS, then write one spec per
   capability domain using OpenSpec's format (`## Requirements`
   → `### Requirement:` SHALL/MUST → `#### Scenario:` WHEN/
   THEN). Don't seed exhaustively: 2–4 core contracts first,
   grow the rest as real changes touch them. Verify claims
   against the CODE, not just docs — docs drift (our docs said
   Django 5.2; code was on 6.1). Typical capability sets:
   - API backend: auth/sessions, error contract (codes→HTTP),
     core domain resources, external-provider integrations,
     caching behavior.
   - Frontend: routing/data model contracts, API-client error
     handling, auth/token lifecycle, offline/caching.
6. **Update the repo's agent-instructions file** (AGENTS.md/
   CLAUDE.md) with a short OpenSpec section: where specs live,
   CLI conventions (e.g. "run openspec from repo root"), any
   task-prefix rule, and where planning docs used to live
   (freeze the old convention — we froze `docs/plans/` for new
   feature plans).
7. **Smoke-test, don't dry-run a fake change.** `openspec
   list`, `openspec list --specs`, `openspec validate --all
   --strict` — all must pass. The first real change should be
   a real (small) feature.
8. **Ship as a PR like any other change** — `openspec/`,
   `.agents/skills/`, agent-skill copies, instruction-file
   updates, all committed. Have a fresh reviewer check spec
   claims against code (ours caught a missing `internal_error`
   → 500 mapping).

## Gotchas learned the hard way

- **CLI PATH**: a global npm install lands wherever npm's bin
  dir is (`/opt/homebrew/bin` on this machine) which may not be
  on agent shells' PATH — document it in the instructions file.
- **`.gitignore` shadowing**: a bare `config.yaml` ignore rule
  (common where `config.yaml` holds secrets) also matches
  `openspec/config.yaml`. After init run
  `git check-ignore -v openspec/config.yaml` and add
  `!openspec/config.yaml` to `.gitignore` if it's caught.
  (Hit in django-apps.)
- **Vendored skills vs line-length limits**: openspec 1.14.x
  generates >79-char prose in SKILL.md bodies. In repos with a
  line-length rule, don't rewrap upstream files — add an
  explicit exemption for `.agents/skills/` AND the `.devin/`
  copies to the instructions file, or a later agent will "fix"
  them and break `openspec update` refreshes. (Hit in both
  nibrastube and django-apps; both exempted it in AGENTS.md.)
- **`gh pr checks --watch` right after push** can report "no
  checks reported" — GitHub hasn't registered the run yet.
  Retry after ~15s instead of assuming checks are absent.
  (Hit on both repos.)
- **Strict mode's 500-char cap** counts requirement bodies
  including newlines — split detail into scenarios early; both
  repos needed a trim pass after review found borderline
  bodies.
- **Expect 2–3 review rounds on spec wording.** Reviewers
  consistently found real spec-vs-code divergences the
  implementer missed (e.g. which auth branch carries `next`
  vs `session['oauth_redirect_url']`; a JS-predicate clone of
  an SQL fragment). Fresh-eyes re-review after fixes catches
  follow-ons; budget the loop, don't one-shot it.
- **`openspec/specs/.gitkeep`** from init is safe to delete
  once real specs exist.
- **Generated skill files are upstream artifacts** — don't
  reformat them (e.g. rewrap their long frontmatter lines);
  `openspec update` owns them. Wrap only your own copies.
- **Root-resolution**: run `openspec` commands from the repo
  root where `openspec/` lives; don't rely on cwd inference.
- **Specs age like docs** — seed from code-verified contracts
  and treat "docs disagree with code → code wins" as a spec-
  authoring rule.

## Repo-type notes

**Next.js app (single repo)**: init at repo root; no umbrella
question. Seed specs around: API-route/server-action contracts,
auth/session flow, data fetching + caching strategy, error/
loading boundaries, env-var contract. `config.yaml` context:
App Router vs Pages Router, which data layer, deploy target.

**Django + React in one repo**: still a single repo →
`openspec/` at repo root, but it's effectively TWO layers, so
(a) keep per-layer task prefixes (`backend:`/`frontend:`) in
`tasks.md` anyway — changes often touch both; (b) seed at
least one contract spec that spans the seam (API request/
response shapes, error envelope, auth header format) — that's
where drift bites. Django-specific seeds: auth/permissions,
ORM/model invariants, serializer contracts, error responses,
background tasks/CELERY if present.

## Copy-paste prompt for the implementing agent

```
Adopt OpenSpec in this repo following the playbook at
docs/learnings/openspec-adoption-playbook.md (or paste the
playbook here). Steps: install @fission-ai/openspec (newest
version >7 days old), `openspec init --tools agents` at repo
root, bridge the workflow skills into my agent's skill dir
(strip frontmatter to name/description if required), fill
openspec/config.yaml context from this repo's conventions,
seed 2-4 capability specs in openspec/specs/ written from the
repo's AGENTS.md/README and verified against the code, add an
OpenSpec section to the agent-instructions file, run
`openspec validate --all --strict`, and put it all on a
feature-branch PR. Don't run a real change through the
pipeline; smoke-test only. Report the PR URL and validation
output.
```
