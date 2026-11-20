# Proposal

## Why

Features accumulate on the `staging` branch (staging-first
workflow) until a manual `Promote to main` run releases them.
Before promoting, the tester needs to know which PRs are on
staging but not yet in production and how to exercise each one
through the UI. Today that requires manually cross-referencing
the merged-`staging` and merged-`main` PR lists on GitHub — the
repo even has a `dry_run` input on `promote-to-main.yml` whose
only output is an Actions-step summary nobody browses casually.

## What Changes

- New `staging-release-checklist` capability: a menu entry in
  the MainMenu drawer, visible only on Vercel preview deploys
  (`VITE_APP_ENV === 'preview'`), that opens a modal listing
  every PR merged into `staging` whose head is not yet merged
  into `main`.
- The list is computed at runtime by calling the public GitHub
  REST API (`api.github.com`, CORS-enabled, repo is public — no
  token needed), replicating the pending-promotion algorithm
  already used by `.github/workflows/promote-to-main.yml`:
  latest merged `base=staging` PR per head branch, minus those
  with a merged `base=main` PR on the same branch.
- Each row shows the PR title + number (linked to GitHub) and
  an expandable, human-readable checklist parsed from the PR
  body's `## UI verification / QC` section, with checkboxes the
  tester can tick off. Rows missing that section link to the PR
  instead.
- Graceful degradation: rate-limit/network failure shows an
  error state plus a link to the GitHub `main...staging`
  compare view.

## Capabilities

### New Capabilities

- `staging-release-checklist`: menu-visible checklist of PRs
  merged into `staging` but not yet promoted to `main`, with
  per-PR UI-verification steps rendered as an interactive
  checklist. Preview-deploys only; production and Android
  builds never show it.

### Modified Capabilities

- None.

## Impact

- **Repo: reactive-bible (frontend) only.** `bible_research`
  untouched.
- `src/components/MainMenu.tsx` — new gated menu item.
- New component(s) for the checklist modal; new util for the
  GitHub API fetch + pending-set computation (mirroring
  `src/utils/appUpdate.ts`'s existing unauthenticated
  `api.github.com` fetch pattern).
- i18n keys in `en`/`lv`/`ru` `common.json`.
- No backend, no schema, no secrets. Unauthenticated GitHub
  API: 60 req/hr per client IP — acceptable for a QA tool.
