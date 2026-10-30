# Tag Management — Mobile UX Improvement Plan

Status: in progress (reactive-bible PR #203 open, review clean)
Repo: `reactive-bible` (React 18 + Mantine v6)
Evidence: `./mobile-ux-findings.md` + `./screenshots/` (Playwright,
`devices['Galaxy S9+']`, 320x658, logged in as Tester5 against the
production API via `localhost:5173`).

## Background

`/tags` (`src/routes/TagManagementRoute.tsx` + `TagTree` +
`CreateTagModal`/`EditTagModal`) gained a lot of capability in the
Oct-2026 batch merges: hierarchical tree with ancestor-preserving
search (`#164`), auth-gated actions and silent refresh (`#141`,
`#145`), extracted tree utils (`#157`), narrow selectors (`#153`),
plus earlier force-refresh-on-nav and Enter-to-submit in modals.
None of those commits touched mobile layout — the page is a
desktop-first tree squeezed onto a 320-360 px phone screen.

## Verified problems (ranked)

1. Row action icons render at **22x22 px with 4 px gaps** — half
   the 44-48 px touch-target guidance; the red delete sits 4 px
   from edit on every row.
2. **24 px/level indent** + ~110 px of icons leaves ~100 px for
   names at depth 2; "Holiness of God" wraps at depth 1 already.
3. Delete uses a bare **`window.confirm`** — unstyled on Android,
   long message, no danger emphasis.
4. Folder/tag icon colors are **broken**: `var(--mantine-color-
   blue-6)` doesn't resolve under Mantine v6 — everything renders
   gray, so the expand affordance is only the chevron.
5. **~55% of the first viewport is chrome** (app header, `order=2`
   title, wrapped New Tag button, search, count) — ~6 rows visible.
6. Parent-tag `Select` dropdown flips upward and covers the Tag
   Name field and modal title.
7. Hover `Tooltip`s fire and linger on touch, covering sibling
   icons.
8. Leaf rows are dead taps (only parents toggle on row tap).
9. Minor: post-login lands on a stale stored route, not `/tags`.

## Implementation

All work in `reactive-bible`, one feature branch (e.g.
`fix/tag-management-mobile-ux`), staged per workspace rules.
Constraints: 79-char lines, `npm run lint` clean, vitest green.

### Phase 1 — quick wins, low risk

- [ ] **Fix icon colors** (`TagTree.tsx`): replace the unresolved
  `var(--mantine-color-blue-6)` on `IconFolder`/`IconFolderOpen`/
  `IconTag` with `theme.colors.blue[6]` / `theme.colors.gray[6]`
  via `useMantineTheme()`.
- [ ] **Bigger touch targets**: bump the row `ActionIcon`s from
  `size="sm"` to `size="lg"` (~38 px) and the chevron likewise;
  raise the action `Group` spacing from `4` to `xs`. Verify rows
  don't grow past ~56 px.
- [ ] **Replace `window.confirm`** (`TagManagementRoute.tsx`) with
  `openConfirmModal` from `@mantine/modals` (provider already in
  `App.tsx`): red `Delete` confirm button, keep the child-count
  warning copy, `confirmProps` labelled clearly. Update
  `TagManagementRoute.test.tsx` which likely stubs
  `window.confirm`.
- [ ] **Leaf row tap = View notes**: in `TagTree.tsx`, when a node
  has no children, call `onViewNotes` on `Paper` click instead of
  nothing; keep cursor pointer.
- [ ] **Tooltips off on touch**: gate the three `Tooltip`s behind
  `useMediaQuery('(pointer: fine)')` (or Mantine's
  `Tooltip disabled` under a coarse-pointer check) so they don't
  linger over adjacent icons on phones.
- [ ] **Narrow the indent on small screens**: `level * 24` ->
  `level * 14` under the `sm` breakpoint (keep 24 on desktop), via
  `useMediaQuery` in `TagTree`. Also add `lineClamp={1}`/ellipsis
  to the name `Text` so long names truncate instead of wrapping.

### Phase 2 — layout restructuring

- [ ] **Collapse row actions into one menu on `xs`**: on
  hover-capable desktop keep inline icons; on `xs`/coarse pointers
  render a single `IconDotsVertical` `Menu` with View notes /
  Edit / Delete (Delete last, red, separated by `Menu.Divider`).
  Frees ~70 px per row for the tag name and kills the
  edit-vs-delete mis-tap entirely.
- [ ] **Compact header on `xs`** (`TagManagementRoute.tsx`):
  `Title` `order={3}` (or `size="lg"`) on small screens; put the
  New Tag button as a compact icon button next to the title on one
  row; search `TextInput` full-width (drop `maxWidth: 400` under
  `sm`); merge the "N tags" count into the search row.
- [ ] **Indent rail instead of margin** (optional, if 14 px still
  tight): replace `marginLeft` with a 2 px left border on a fixed
  ~8 px rail so every row keeps near-full width at any depth.
- [ ] **Fix the parent Select** (`CreateTagModal`/`EditTagModal`):
  add `withinPortal` and `dropdownPosition="bottom"` so the
  dropdown can't cover the Tag Name field; consider
  `maxDropdownHeight` so more than ~4 options show. Optionally
  `fullScreen` modals on `xs` (`size` responsive prop).
- [ ] **Edit modal cleanup on mobile** (optional): collapse the
  Created/Updated dates behind a subtle line on `xs`.

### Phase 3 — verification

- [ ] `npm run lint` — 0 errors/warnings; `awk` 79-char check on
  edited files.
- [ ] `npm test -- --run` scoped to `TagTree`, `TagManagementRoute`,
  `CreateTagModal`, `EditTagModal` test files; update tests for
  the confirm-modal swap and menu-based actions.
- [ ] Re-run the Playwright script from `/tmp/tag-ux` (recreate if
  gone: `npm i playwright@1.64.1` + `npx playwright install
  chromium`) at `devices['Galaxy S9+']` against `localhost:5173`,
  capture before/after equivalents of `03b`, `04`, `08`; eyeball
  for regressions. Also capture one desktop-width shot to confirm
  the desktop layout is unchanged.
- [ ] Update `reactive-bible/DEVELOPER_GUIDE.md` (workspace rule:
  contract doc after functionality changes).
- [ ] Manually exercise on a real phone or Chrome DevTools device
  emulation: expand/collapse, open each action, delete-cancel a
  tag, run a search.

## Out of scope

- Backend/API changes — none needed; `/api/v1/tags/` contract
  stays as-is.
- Redesigning the tree into a flat list or drag-and-drop
  reparenting — the hierarchy itself is fine, the density isn't.
- The post-login redirect quirk — separate auth-flow issue, note
  as a follow-up rather than bundling.
- `noteCounts` prop is unused by this route — leave as-is.
