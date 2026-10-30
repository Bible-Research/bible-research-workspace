# Tag Management — Mobile UX Findings (Galaxy-class phone)

Status: investigation complete (read-only; no app source modified)

Method: Playwright 1.64.1, `devices['Galaxy S9+']` — 320x658 CSS px
viewport, deviceScaleFactor 4.5, mobile Chrome UA, touch enabled —
against the local dev server (`localhost:5173`) logged in as
`Tester5`. All screenshots are in `./screenshots/`.

Code referenced:
`reactive-bible/src/routes/TagManagementRoute.tsx`,
`reactive-bible/src/components/TagTree.tsx`,
`CreateTagModal.tsx`, `EditTagModal.tsx`, `LoginPage.tsx`.

## Screenshot inventory

- `01-login.png` — `/login` on the Galaxy viewport: centered card,
  full-width inputs and "Sign in" button; footer link wraps to
  "Create one now" on its own line.
- `02-login-filled.png` — same page with credentials typed.
- `03-tags-initial.png` — `/tags` as loaded (full capture; the app
  uses a fixed-viewport layout so this equals the visible page).
- `03b-tags-viewport.png` — first viewport of `/tags`: header bar,
  "Tag Management" title, "New Tag" button, search field, "40 tags",
  ~6 tag rows.
- `04-tags-expanded.png` — "Atonement" and "Attributes of God"
  expanded; level-1 children ("Assurance", "Holiness of God",
  "Sovereignty") indented, names wrapping to two lines.
- `05-tags-deeper.png` — deeper expansion mid-list ("New Covenant",
  "E2E-Tag-1373" → "E2E29621", "Testing nesting").
- `06-search.png` — search "the": 7 matches, `forceExpand` shows
  3-level chain Prophecy → Messianic → In the Psalms; level-2 name
  wraps.
- `07-create-modal.png` — "Create New Tag" modal: fits the viewport,
  name input, parent Select, Cancel / Create Tag buttons.
- `08-create-modal-parent-dropdown.png` — parent-tag Select opened:
  dropdown renders *above* the input, covering the Tag Name field
  and the modal title.
- `09-edit-modal.png` — "Edit Tag" modal for "Atonement": fits,
  orange child-tags warning, Created/Updated dates, buttons fine.
- `10-scrolled.png` — scrolled list; an "Edit tag" tooltip is left
  floating over the next row after a tap.
- `11-after-delete-dismiss.png` — page after tapping a trash icon
  and dismissing the native confirm; tag list unchanged.
- `12-landscape.png` — 740x360 landscape: title + New Tag share a
  row; ~2.5 tag rows visible; search capped at 400px wastes space.

## Measured facts (DOM)

- Row action buttons (`ActionIcon size="sm"`): **22 x 22 CSS px**
  each (view / edit / delete / chevron), **4 px gaps** between them.
- Row height: ~44 px for single-line names, ~60-70 px when the name
  wraps.
- Indent: `level * 24` px margin-left per depth (`TagTree.tsx:68`).
- Name column at 320 px: ~150 px at depth 0 → ~102 px at depth 2
  after indent + 3 action icons + chevron + gaps.
- Folder/tag type icons render at computed `color:
  rgb(193,194,197)` — the `var(--mantine-color-blue-6)` passed to
  `IconFolder`/`IconFolderOpen`/`IconTag` (`TagTree.tsx:104-112`)
  **does not resolve** under Mantine v6, so all type icons fall
  back to the same gray. Parent (blue folder) vs leaf (gray tag)
  distinction is lost; only the chevron vs 28 px spacer differs.
- Native confirm text on delete (dismissed, not confirmed):
  `Are you sure you want to delete "Atonement"? Notes assigned to
  it will lose this tag, and its 1 child tag(s) may be affected.
  This action cannot be undone.`

## Problems, most severe first

1. **Action buttons are ~half the minimum touch size and packed
   4 px apart.** 22 px targets vs the 44 px (Apple) / 48 dp
   (Android) guidance, with only 4 px between view/edit/delete
   (`03b`). The destructive red trash sits immediately right of
   the gray pencil — a right-thumb tap aimed at Edit can land on
   Delete. Mitigation is only the `window.confirm`. *Idea:* bump
   ActionIcons to `size="lg"`+ (~38-44 px) and `spacing="xs"` on
   the action group; on small screens consider hiding edit/delete
   behind a single "more" menu or an expandable row actions bar.

2. **Deep hierarchy squeezes names to ~100 px.** 24 px/level
   margin-left plus a fixed ~110 px of icons leaves "In the Psalms"
   (depth 2) wrapping to two lines (`06`), and depth 3+ would be
   ~80 px — a few characters per line. *Idea:* reduce indent to
   12-16 px on narrow viewports, or replace margin-indent with a
   thin vertical rail so the row keeps full width; truncate long
   names with ellipsis instead of wrapping.

3. **Delete confirmation is a bare `window.confirm`.**
   (`TagManagementRoute.tsx:95-103`) On Android Chrome it renders
   as an unstyled system dialog with a long 3-sentence message —
   the destructive verb is buried mid-paragraph and the dialog
   can't emphasize Delete or be reached by mistake-proofing (e.g.
   requiring the tag name). Headless Chromium won't even render it
   for screenshot. *Idea:* replace with a Mantine `ConfirmModal`/
   `openConfirmModal` — styled, shows child/note impact clearly,
   and can place the danger button deliberately.

4. **Parent-vs-leaf visual distinction is broken.** The intended
   blue folder icons render the same gray as tag icons because
   `var(--mantine-color-blue-6)` isn't a valid variable in this
   Mantine v6 setup (`TagTree.tsx:104-112`). On mobile, scanning a
   flat list to spot expandable rows depends on the tiny chevron
   alone. *Idea:* use `theme.colors.blue[6]` / `#228be6` directly,
   or wrap icons in `useMantineTheme()`.

5. **~55% of the first viewport is chrome, not tags.** At 320x658
   the fixed app header (search icon, play icon, "Change
   Translation" text, hamburger), the large `Title order=2`, the
   wrapped "New Tag" button row, the search box and the "40 tags"
   count consume ~360 px before the first row (`03b`); only ~5-6
   rows fit above the fold, ~2.5 in landscape (`12`). *Idea:*
   shrink the page title on mobile (`order=3`/`size="lg"`), merge
   title + New Tag + search into a tighter toolbar; shorten the
   "Change Translation" header label to an icon on `xs`
   breakpoints.

6. **Search results keep ancestors but names still wrap.**
   `forceExpand` works well (`06`) — matched descendants stay
   visible under parents — but combined with the indent issue a
   depth-2 match occupies ~1/3 more vertical space than needed.

7. **The Select dropdown opens upward over the form.** With little
   space below, Mantine flips the parent-tag dropdown above the
   input (`08`), covering Tag Name and the modal title. It works —
   it's `searchable` — but it occludes the field you just filled
   and the list shows only ~4 options. *Idea:* `withinPortal` +
   `dropdownPosition="bottom"`, or a full-screen picker on
   `xs` viewports.

8. **Tooltips fire and linger on touch.** Shot `10` shows an
   "Edit tag" tooltip left hovering over the next row after a tap;
   on mobile tooltips add noise and can cover the very icon the
   user is aiming at. *Idea:* gate `Tooltip` on hover-capable
   pointers (`withinPortal disabled`/`disabled` under
   `(pointer: coarse)` media query).

9. **Minor: post-login redirect.** Logging in via `/login` landed
   on a stale stored route (`/notes/tag/TAG7AD4...`) rather than
   `/tags`; harmless but worth noting if testers expect to return
   to where they were bounced from.

10. **Minor: leaf rows are dead taps.** Parent rows toggle on any
    tap (whole `Paper` is clickable, `TagTree.tsx:79`), but leaf
    rows do nothing on tap — inconsistent; users may expect a leaf
    tap to open its notes (the eye icon's job). *Idea:* make leaf
    row tap = View notes, and/or expand hit area of the chevron.

## What already works well on mobile

- Modals (`07`, `09`) fit the 320 px viewport cleanly; buttons and
  inputs are full-width and legible; edit modal's child-tag warning
  and dates are readable.
- The list lives in an inner `ScrollArea`, so the header/title/
  search stay pinned while rows scroll — good orientation.
- Search filtering with ancestor retention (`filteredTags` +
  `forceExpand`) is genuinely good UX on a small screen.
- Row padding gives ~44 px row height — rows themselves are fine
  targets; it's the icon cluster that isn't.
- Login page is clean: autofocus, full-width fields, tappable eye
  toggle; only nit is the "Create one now" link wrapping.
