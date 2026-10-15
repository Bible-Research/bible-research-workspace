# Porting This Workspace's Agent Configuration to Another Codebase

**Status: not implemented (hand-off plan for an agent in another
repository).**

- **Date:** 2026-10-15
- **Purpose:** instructions for an agent working in a DIFFERENT
  codebase. That agent should reproduce this project's
  agent-instruction setup — `AGENTS.md` rules, `SKILL.md` skills,
  PR workflow, plan-file conventions — adapted to the target
  repo's stack. Everything below is the distilled, portable set
  of user-specific demands extracted from:
  - workspace `AGENTS.md`
    (`Bible Research/AGENTS.md`)
  - `bible_research/AGENTS.md`, `reactive-bible/AGENTS.md`
  - `.devin/skills/{fix-good-first-issue-backend,
    fix-good-first-issue-frontend,implement-plan}/SKILL.md`
  - `.devin/workflows/create-pr.md`
  - `.devin/config.local.json`,
    `reactive-bible/.claude/settings.local.json`
  - user globals: `~/.config/devin/AGENTS.md`,
    `~/.codeium/windsurf/memories/global_rules.md`

Placeholders below use `<angle brackets>` — the implementing
agent fills them from the target codebase.

## Step 0 — Survey the target codebase first

Do NOT copy this project's business rules. Before writing
anything, determine and record:

1. Single repo or multi-repo workspace? (This project is a
   workspace of two independent git repos; if the target is one
   repo, produce ONE `AGENTS.md` at its root.)
2. Languages, frameworks, package managers, deploy targets.
3. Test/lint/build commands actually used (read package.json,
   pyproject.toml, CI yaml — do not guess).
4. Default branch name and PR/CI flow (`gh repo view`, CI files).
5. Dangerous operations that must be forbidden for agents
   (this project forbids DB migrations; the target may forbid
   e.g. `terraform apply`, `--force` deploys, schema changes).
6. Whether `origin` is the user's fork of an upstream repo.

## Step 1 — Write `AGENTS.md` at the repo root

Use this skeleton (the shape matters as much as the content —
every section exists because agents repeatedly needed it):

```markdown
# AGENTS.md — <repo-name> (<one-line stack summary>)

Guidance for AI agents working in this repository. Read this
before editing; trust the code over any doc that disagrees
with it.

## What this is          # stack, deploy target, sibling repos
## Layout                # directory map of what lives where
## <Core domain contract> # the 1-2 things that break if wrong
## Commands              # dev / build / lint / test one-liners
## Conventions / rules   # the universal rules — see Step 2
## Known gaps (do NOT "fix" without asking)
## Testing               # scoped test commands, known failures
```

Section requirements learned from this project:

- **"Trust the code over docs" preamble** — doc drift is real;
  name the stale docs explicitly (here: `DEVELOPER_GUIDE.md`
  predates several features).
- **Layout** — actual structure, not aspirational; list the
  files agents always need (`api.tsx`, `config.ts`,
  `cacheManager.ts` equivalents: the client entrypoint, config
  module, caching layer, auth module).
- **Core domain contract** — the shared keys/error shapes both
  sides of an API depend on (here: `fileset_id` routing and the
  `error`/`error_code` contract). In the target repo, identify
  the equivalent: what must stay stable because something else
  branches on it.
- **Known gaps section** — list things that LOOK broken but are
  intentional (here: commented-out Terraform scheduler, KJV
  audio fetched outside the API). Explicitly forbid "fixing"
  them without asking.
- **Testing** — the scoped command, not the whole suite; list
  known pre-existing failures so agents don't chase them.
- **Databases/auth quirks** — if local config points at a live
  or shared resource, say so loudly, state what's read-only,
  and forbid writes/DDL.
- **Commands** — verbatim runnable; note expected duration if a
  suite is slow (agents otherwise kill or avoid it).

## Step 2 — Universal rules to embed verbatim

These are the user's cross-project demands. Put them in the
`Conventions / rules` section of every repo's `AGENTS.md`
(adapt only the language-specific parts):

1. **Max 79 characters per line, every file type** (source,
   markdown, configs). Verify:
   `awk 'length > 79 {print FILENAME":"NR}' <file>`.
   Fix violations you introduce plus pre-existing long lines in
   files you are already modifying. For Python specifically:
   never write lines > 79 chars.
2. **Commit format:** `Type: Capitalized message` —
   `Feat:` / `Fix:` / `Docs:` / `Refactor:` / `Test:` /
   `Chore:`.
3. **Branching/staging:** commits go on feature branches, never
   the default branch. Stage ONLY files created or modified
   this session — never `git add -A`, `git add .`, or
   `git commit -a`; confirm the staged set with `git status`
   first. Leave the user's own dirty files alone. Never commit
   plan/scratch markdown (`*-plan.md`, `PR_DESCRIPTION.md`).
4. **Forbidden operations:** never run the repo's dangerous
   commands (here: `makemigrations`/`migrate`/`sqlmigrate`).
   If a task needs one, write the code change and tell the user
   the command to run — do not run it.
5. **Scoped tests:** test only the files/apps you changed —
   do not run the whole suite when it's slow (the user's global
   Ruby rule: never bare `bundle exec rspec`). List the scoped
   command in AGENTS.md.
6. **PR creation:** write the body to a temp file and use
   `gh pr create --body-file` — never inline `--body` (shell
   quoting breaks on multi-line bodies). Delete the temp file
   after.
7. **PR body must include a UI verification / QC section:**
   concrete manual steps (which pages, which clicks, expected
   result) exercising the change end-to-end through the UI.
   Only omit when genuinely unreachable from UI — then document
   an equivalent (curl command, log line to check).
8. **Fork rule:** if `origin` is the user's fork (an `upstream`
   remote exists or it's a GitHub fork), ALL GitHub operations
   target the fork — issues, PRs, comments, releases. Never
   create things on upstream unless asked.
9. **No secrets in code.** Name the repo's secret mechanism
   (here: `config.yaml` gitignored / GCP Secret Manager /
   `VITE_*` env vars).
10. **Update the repo's developer-guide doc** after
    functionality changes (here: `DEVELOPER_GUIDE.md`; use
    whatever the target repo calls it — create it if absent).
11. **Plans live in `docs/plans/`** — implementation plans,
    investigations, design docs as `<feature>-plan.md`
    (multi-file plans get a subdirectory). Each carries a
    status header: implemented / partial / in progress /
    not implemented / reverted. Never put plan files inside
    app subrepos of a multi-repo workspace.
12. **Test login:** if the project has a test account, document
    it (here: `Tester5` / `password123`); if not, ask the user
    for one and record it.
13. **Read `AGENTS.md` before working** — if the target is
    multi-repo, the root file says which subrepo file to read.
14. **AGENTS.md is self-updating — record environment/setup
    learnings.** When the agent hits local-setup,
    infrastructure, or missing-tooling friction — a CLI that
    isn't installed, a non-obvious way to run the project
    locally, a required tool version or local-only path — it
    must not grind through many failing alternatives. Missing
    tool → tell the user what to install instead of working
    around it repeatedly. Working flow or local quirk
    discovered → write it into the relevant `AGENTS.md` so the
    next agent uses it directly, and commit that doc update
    alongside the work that discovered it. Embed this directive
    in the new AGENTS.md verbatim so the behavior propagates.

## Step 3 — Create the skills

This project keeps skills as `SKILL.md` files (YAML frontmatter
`name` + `description`, then numbered steps). NOTE: the
workspace AGENTS.md says `.agents/skills/` but the actual
workflow skills live in `.devin/skills/` — put skills in
whichever directory the target agent's tooling reads (Devin:
`.devin/skills/` or `.agents/skills/`; check first), and keep
the AGENTS.md pointer accurate.

### 3a. `fix-good-first-issue` (one per repo, or one total)

Generalized template — replace `<REPO>` / `<ORG/REPO>` /
stack-specific bits:

```markdown
---
name: fix-good-first-issue-<repo>
description: Pick an open GitHub issue labeled "good first
  issue" in <ORG/REPO>, fix it on a new branch cut from
  <default>, open a draft PR, wait for checks, fix failures,
  mark ready. Invoke when the user asks to fix a
  good-first-issue / pick up an issue.
---

# Fix a "Good First Issue" in <repo>

## 1. Pick an issue
- gh issue list --repo <ORG/REPO> --label "good first issue"
  --state open
- If user gave a number, use it; else pick top candidate.
- ALWAYS verify no PR already covers it:
  gh pr list --repo <ORG/REPO> --state all
    --search "#<N> in:title,body"
  Skip if referenced; if user insisted, name the covering PR
  and ask before proceeding.
- gh issue view <N> --repo <ORG/REPO> --comments

## 2. Decide which repo the fix belongs in
(only if multiple repos / a client-server pair — list the
shared contract areas that usually touch both; search the
sibling repo for companion issues; if unsure, ask the user)

## 3. Branch from default
git -C <repo> switch <default> && git -C <repo> pull --ff-only
git -C <repo> switch -c fix/issue-<N>-<slug>
If uncommitted changes block the switch, stop and ask.

## 4. Fix it
- Read <repo>/AGENTS.md first. Lines <= 79 chars everywhere.
- Honor the repo's forbidden commands.
- Verify with the scoped test/lint/build commands from AGENTS.md.
- Update the developer-guide doc if behavior changed.

## 5. Commit, push, draft PR
- Stage only edited files. `Type: Capitalized message`.
- gh pr create --draft --title "<Type: ...>" --base <default>
  --body-file pr-body.md   (write body to file; rm after)
- Body links the issue: `Closes #<N>`, includes the UI
  verification section.

## 6. gh pr checks <N> --watch; fix failures until green.

## 7. gh pr ready <N>; report issue, PR URL, check status.
```

### 3b. `implement-plan` (multi-agent plan executor)

Port the orchestration pattern, not the repo names:

- Phase 0: orchestrator reads the plan file; extracts a branch
  slug, which repos it touches, and the cross-repo contract.
- Phase 1: one background `subagent_general` per repo the plan
  touches. Prompt template: plan path (absolute), repo dir,
  branch `feat/<slug>` cut from updated default, which
  conventions file to read first, 79-char rule, forbidden
  commands, "implement ONLY this repo's part against the plan's
  contract — the other repo changes in parallel", scoped
  verify commands, staging rules, draft PR via `--body-file`
  with UI-verification section, `gh pr checks --watch`, return
  PR URL + files changed + test results.
- Phase 2: one FRESH reviewer subagent per PR (never reuse
  implementers): `gh pr diff`, check correctness vs plan, edge
  cases, cross-repo contract match, tests cover the change,
  79-char limit, no plan/scratch files in diff, no forbidden
  artifacts (migrations). Return numbered findings with
  severity blocker|should-fix|nit and file:line; "NO ISSUES"
  if clean.
- Phase 3: fix loop — one fixer per affected PR on its branch,
  findings verbatim, then a NEW review round; repeat until all
  clean or 3 rounds, then stop and report.
- Guardrails: agents share the working tree — never two agents
  writing the same repo concurrently; fixers for the same PR
  run sequentially; background agents can't gain permissions —
  resume blocked ones in foreground; if a plan is
  unimplementable as written, stop and ask the user.

### 3c. Stack skill (optional)

`vercel-react-best-practices` is a vendored upstream skill
(Vercel's 45 React perf rules, `rules/*.md` + compiled doc).
Port only if the target repo is React/Next.js — copy it
verbatim; otherwise substitute the equivalent vendored
guidelines for the target stack, or skip.

## Step 4 — Workflow + permissions files

- `.devin/workflows/create-pr.md` (or the agent tool's
  equivalent): the `--body-file` rule — write PR body to
  `pr-body.md`, `gh pr create ... --body-file pr-body.md`,
  `rm pr-body.md`. Never inline `--body`.
- Permission allowlist for routine commands so agents stop
  prompting — Devin `.devin/config.local.json`:
  `{"permissions": {"allow": ["Exec(gh pr)"]}}`;
  Claude Code `.claude/settings.local.json`:
  `{"permissions": {"allow": ["Bash(npm test:*)",
  "Bash(git add:*)", "Bash(git commit:*)",
  "Bash(gh pr view:*)"]}}`.
  Use whichever the target agent reads; keep it untracked if
  it's user-local.
- If the user's environment has `rtk` (token-saving proxy),
  note: quote paths with spaces, never backslash-escape.

## Step 5 — `docs/plans/` seed

Create `docs/plans/` at the workspace/repo root if absent.
Record in AGENTS.md that all plans go there with status
headers. Move any stray `*-plan.md` files into it.

## Step 6 — Verify

- `awk 'length > 79 {print FILENAME":"NR}' <each new file>`
  returns nothing — the config files themselves obey the rule.
- `git status` — confirm nothing unrelated got staged; commit
  on a feature branch as `Docs: Add agent instructions and
  skills` (or equivalent type).
- Re-read the new AGENTS.md as a fresh agent would: can it
  build, test, and PR without asking? Fill what's missing.

## What NOT to port

Project-specific, do not copy: `fileset_id` provider routing,
`ENGKJV`/`ENGESV_API`/`LVSGLU8` ids, the `error_code` values,
USFM book codes, GCP/App Engine/Vercel specifics, the
production-read-only Aiven DB details, `Tester5` credentials
(ask for the target's own test account), the commented-out
Terraform scheduler note. These are examples of WHAT KIND of
thing belongs in "Core domain contract" / "Known gaps" — find
the target's equivalents.
