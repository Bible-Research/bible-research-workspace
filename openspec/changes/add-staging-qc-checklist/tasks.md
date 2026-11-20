# Tasks

## 1. GitHub pending-set data layer

- [x] 1.1 frontend: create `src/utils/stagingPrs.ts` —
  `fetchPendingStagingPrs()` doing the two `api.github.com`
  pulls calls (`base=staging` + `base=main`, `state=closed`,
  `per_page=100`, sort `updated`/`desc`), dedupe by `head.ref`,
  and the pending filter per design.md; verify via unit test
  with MSW fixtures that un-promoted, promoted (same-SHA), and
  replay-promoted (later `merged_at`, different SHA) cases all
  classify correctly
- [x] 1.2 frontend: add `parseQcSteps(body)` in the same module
  extracting list lines between `## UI verification / QC` and
  the next `## ` heading; verify unit tests cover numbered
  lists, `- [ ]` items, missing section (→ null), and section
  at end of body
- [x] 1.3 frontend: cover fetch-failure paths (non-OK response,
  network throw → typed error) so the modal can render the
  fallback state; verify the tests pass via
  `npm test -- --run src/utils/stagingPrs` (or the test file)

## 2. Checklist modal

- [x] 2.1 frontend: create
  `src/components/StagingChecklistModal.tsx` — Mantine modal
  listing pending PRs (title, `#N` link to `html_url`) with an
  expandable Checkbox list per PR from `parseQcSteps`, empty
  state ("nothing pending"), error state with
  `github.com/Bible-Research/reactive-bible/compare/main...staging`
  link; verify with a component test that mocked pending data
  renders checkboxes per QC step
- [x] 2.2 frontend: i18n keys `menu.stagingChecklist*` in
  `src/i18n/locales/en/common.json` (source catalog) plus `lv`
  and `ru` lazy locale files; verify `npm run build` (tsc)
  accepts the new keys

## 3. Menu integration

- [x] 3.1 frontend: add the gated entry to
  `src/components/MainMenu.tsx` —
  `import.meta.env.VITE_APP_ENV === 'preview'`, lazy-load the
  modal via `lazyWithReload` like `ReportProblemModal`; verify
  a MainMenu test shows the entry only under the preview env
  value
- [x] 3.2 frontend: update `AGENTS.md` /
  `DEVELOPER_GUIDE.md` menu-feature notes for the
  preview-gated checklist; verify doc lines stay ≤ 79 chars

## 4. Verification

- [x] 4.1 frontend: run `npm run lint` and the touched test
  files (`npm test -- --run <touched files>`) — lint at
  0 errors/0 warnings, new tests green
- [x] 4.2 frontend: verify no line exceeds 79 chars in changed
  files (`awk 'length > 79 {print FILENAME":"NR}' <files>`)

## Workflow follow-up

- Open the PR with `--base staging` per the staging-first
  workflow; the merged PR itself then becomes a checklist row
  on the next staging preview deploy.
