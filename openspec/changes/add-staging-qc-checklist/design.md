# Design

## Context

See proposal.md — Why. Grounding facts verified in the repo:

- `Bible-Research/reactive-bible` is a public repo, and the app
  already calls `api.github.com` unauthenticated from the
  browser (`src/utils/appUpdate.ts` `fetchNewestApkUrl`);
  api.github.com sends `Access-Control-Allow-Origin: *`.
- `vite.config.ts` bakes `import.meta.env.VITE_APP_ENV` =
  `VERCEL_ENV` (`'production' | 'preview' | 'development'`);
  Android builds override with `VITE_APP_ENV=android[-debug]`.
- MainMenu (`src/components/MainMenu.tsx`) is a Mantine Drawer;
  modals follow the lazy `lazyWithReload` + `Suspense` pattern
  used by `ReportProblemModal`.
- `.github/workflows/promote-to-main.yml` already defines
  "pending promotion" authoritatively; PR bodies carry a
  mandatory `## UI verification / QC` section (create-pr
  skill), which is the source of the checkable steps.

## Goals / Non-Goals

**Goals:**
- Menu entry visible only when `VITE_APP_ENV === 'preview'`.
- Modal lists pending PRs (title, #number, GitHub link) with
  the PR's QC section rendered as tickable checkboxes.
- Runtime fetch on modal open; failures degrade to an error
  state + link to the GitHub `main...staging` compare view.

**Non-Goals:**
- No GitHub auth, no backend proxy, no persistence of checkbox
  state across sessions/devices.
- No "mark PR verified" write-back — the checklist is a local
  aid, not a release gate.
- No build-time JSON generation or workflow changes.

## Decisions

**Runtime fetch over build-time snapshot.** The app's semantics
are "merged into `staging`, not in `main`", which can change
without a redeploy (a promotion run or a new merge). A
prebuild-generated `staging-changes.json` would only refresh on
rebuild; runtime fetch is always current, and the
unauthenticated-API precedent already exists. Cost: ~2–3 API
calls per open, well inside the 60 req/hr anonymous quota.

**Pending-set algorithm mirrors promote-to-main.yml.**

```
GET /repos/Bible-Research/reactive-bible/pulls
    ?state=closed&base=staging&per_page=100
    &sort=updated&direction=desc
  -> keep merged_at != null
  -> dedupe by head.ref (latest merged_at wins)
GET .../pulls?state=closed&base=main&per_page=100
    &sort=updated&direction=desc
  -> keep merged_at != null
  -> index by head.ref: {head.sha, merged_at}
pending = staged PR where no main entry has
          head.sha == staged head.sha
          OR main.merged_at > staged.merged_at
```

The `head.sha` clause is the workflow's clean-squash check;
the `merged_at` ordering clause covers the workflow's
commit-search fallback (replayed promotions land under a new
SHA on the same branch). Deliberately NOT the
`compare/main...staging` shortcut: after a partially failed
train, promoted squash commits remain on staging until the
reset and would show as false pending.

**QC section parsing, no markdown renderer.** Extract the slice
of `body` between `## UI verification / QC` and the next
`## `-level heading, then take ordered/unordered list lines
(`1. `, `- `, `- [ ]`) as steps. Each step → a Mantine
`Checkbox`. PRs without the section get a "see PR body" link.
Plain-text rendering keeps the modal light — the section is
authored as numbered steps already.

**Placement follows existing patterns.** New
`src/utils/stagingPrs.ts` (fetch + pending computation + QC
parsing — pure, unit-testable), new lazy
`src/components/StagingChecklistModal.tsx`, one gated entry in
MainMenu (Tabler `IconChecklist`), i18n keys under
`menu.stagingChecklist*` in `en` (source catalog) plus `lv`/`ru`
lazy locales.

**Checkbox state is component-local** (`Record<number,
Set<number>>` keyed by PR then step index). Session-only per
the spec.

## Risks / Trade-offs

- Anonymous rate limit (60/hr per IP) — a heavy QA session plus
  the APK-url check could exhaust it → error state links to
  `github.com/.../compare/main...staging`, which is the manual
  equivalent.
- Pulls-list pagination caps at 100; >100 merged staging PRs
  since the last release is implausible (the workflow itself
  errors at 200) → accepted; note in code.
- Pulls-list responses include `body`; ~100 bodies is a heavy
  payload for one request → accepted for a preview-only QA
  tool; can switch to per-PR fetches on expand if it bites.
- A `main` PR merged for a reused branch name before the latest
  staging merge → handled by the `merged_at` ordering clause;
  a pre-existing older `main` merge never suppresses a newer
  staging merge.

## Open Questions

- None blocking. If Vercel ever flips the repo private, the
  fetch breaks — the error state already covers the UX.
