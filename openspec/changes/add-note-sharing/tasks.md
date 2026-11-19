# Tasks

## 1. Backend — models and permission

- [x] 1.1 backend: Add `NoteShare`, `TagShare`
  (`include_descendants`), `SavedNote` models plus
  `Note.published`, `Tag.published`,
  `Note.copied_from` (`SET_NULL`),
  `Note.note_text_plain`, and `Meta.permissions`
  `publish_note`; write the model code and tell the user
  the `makemigrations` command to run (agents never run
  it). Verify: `python manage.py check` passes; migration
  file content reviewed by user.
- [x] 1.2 backend: Data migration (or management command)
  backfilling `note_text_plain` from TipTap JSON for
  existing notes; plain-text extraction util shared with
  the serializer save path. Verify: unit test asserts
  JSON→text parity on a fixture with marks, paragraphs,
  and verse embeds.
- [x] 1.3 backend: Coordinate with the in-flight
  `tag-tree-restructure` change — confirm
  `subtree_tag_ids` remains the subtree primitive
  `include_descendants` resolves through. Verify: grep
  that change's spec/design for hierarchy-altering
  requirements before implementing task 3.1.

## 2. Backend — publishing gate and visibility

- [x] 2.1 backend: Enforce the publish gate on every
  EFFECTIVE `published=true` write — note create/PATCH,
  `notes/bulk/` items, `Tag.published` inheritance, and
  the new tag `set-published` action (mirroring
  `set-public`; `false` clears `published` only). Both
  coercion directions: `published`⇒`public`, and
  clearing `public` clears `published` (incl.
  `set-public` cascade). Verify: tests — non-publisher
  400s (incl. bulk + published-tag inheritance),
  publisher succeeds, `published∧¬public` unreachable,
  unpublish keeps `public=true`.
- [x] 2.2 backend: Add `can_publish` to `UserSerializer`
  (`/me/`). Verify: test asserts the flag tracks
  `has_perm` for a group-granted and a plain user.
- [x] 2.3 backend: Extend `get_accessible_notes_qs` with
  `Exists` subqueries for NoteShare and TagShare
  (subtree-aware via `subtree_tag_ids`); add read-only
  `author {id, username}` to `NoteSerializer`. Verify:
  tests cover sharee-visible private note, subtree grant,
  moved-out note 404, author populated.
- [x] 2.4 backend: `published`/`q` params on
  `GET /notes/` — published-only feed ordered
  `-created_at` via `NotesPagination`, `icontains` on
  `note_text_plain`, anonymous-allowed,
  `include_verse_text=False` on list responses; `q`
  forces published scope, `q`+`tag_id`/`note_pk` → 400;
  add `notes-search` user-throttle scope. Verify: tests —
  private/shared notes never in `q` results, ordering,
  param-conflict 400, no per-note enrichment.

## 3. Backend — share endpoints

- [x] 3.1 backend: `POST/DELETE /notes/<id>/share/` and
  `/tags/<id>/share/` (revoke via
  `…/share/<username>/`; `iexact` resolution,
  silent-200, self-share no-op, tag re-share UPSERTS
  `include_descendants`, 100-recipient cap);
  owner-scoped — non-owner share attempts 404; owner-only
  `GET <id>/shares/`; `share` throttle scope in
  `DEFAULT_THROTTLE_RATES` + conftest override. Verify:
  tests — identical body on unknown username, recipient
  list owner-only, revoke → sharee 404, sharee share
  attempt 404, scope upsert.
- [x] 3.2 backend: `GET /notes/shared-with-me/` returning
  `{notes, tag_shares}` (direct grants + shared-tag
  notes + the tag grants themselves with name and
  `include_descendants`); extend `GET /tags/<id>/` to
  resolve for sharees with scoped `note_count`;
  `include_verse_text=False` on the listing. Verify:
  tests — both grant kinds, empty shared tag still
  appears via tag_shares, non-shared tag 404s.

## 4. Backend — bookmarks and copy

- [x] 4.1 backend: `POST/DELETE /notes/<id>/bookmark/`
  (idempotent repeat) + `GET /notes/bookmarks/` with
  per-row `available` annotation from the same
  visibility `Exists` expression; reject own (400) and
  inaccessible (404) targets. Verify: tests —
  bookmark→unavailable transition, repeat bookmark
  single row, deleted note cascades the row away.
- [x] 4.2 backend: `POST /notes/<id>/copy/` and
  `POST /tags/<id>/copy/` (>100 accessible notes → 400,
  atomic, fresh `tag_position`s appended, `copied_from`
  SET_NULL serialized owner-only, `note_text_plain`
  populated on every create path, images NOT copied).
  Verify: tests — fork payload, partial tag copy
  respects visibility, own-note copy 400, published copy
  hides `copied_from` from anonymous readers.
- [x] 4.3 backend: Update DEVELOPER_GUIDE.md + AGENTS.md
  endpoint/model docs for all of groups 2–4. Verify: doc
  diffs cover every new route and flag.

## 5. Frontend — api layer and auth

- [x] 5.1 frontend: `src/api/sharing.ts`,
  `bookmarks.ts`, `copy` functions; `Note`/`User` types
  gain `author`, `published`, `can_publish`;
  `sharedWithMe`/`bookmarks`/`exploreNotes(published,q)`
  fetches. Verify: `npm run build` typechecks; MSW
  handlers added.
- [x] 5.2 frontend: authStore surfaces `can_publish`;
  `published` plumbed through NoteForm's
  submit/autosave/draft/flush chain and
  `addTagNote`/`editNote` signatures (mirrors the
  existing `isPublic` plumbing); publish toggle on notes
  AND the tag control wherever `setTagPublic` lives —
  both gated on `can_publish`; `public` checkbox locked
  on while a note is published. Verify: unit tests —
  toggle absent when false, checkbox locked when
  published.

## 6. Frontend — sharing UI

- [x] 6.1 frontend: Share dialog on owned notes/tags —
  free-text username with honest copy ("if that username
  exists…"), `include_descendants` toggle + "future
  notes are shared too" hint on tags, recipient list
  refreshing after submit, revoke. Verify: vitest covers
  submit, recipient rendering, revoke.
- [x] 6.2 frontend: "Shared with me" section (read-only
  tag nodes from `tag_shares`, `is_owner=false` cards
  hide edit, show author) + empty state. Verify:
  route/component test renders shared notes and a shared
  empty tag without edit affordances.
- [x] 6.3 frontend: Explore route — published feed +
  search box (`q` debounce), note cards with author,
  paginated, empty state; MSW handlers for the
  anonymous feed/search. Verify: test renders feed +
  search call params.
- [x] 6.4 frontend: Navigation entries for Explore /
  Shared-with-me / Bookmarks in `MainMenu` (auth-optional
  vs auth-required items matching its existing pattern —
  Explore is anonymous-safe). Verify: menu test shows
  Explore to anonymous, hides the auth-only entries.

## 7. Frontend — bookmarks and copy

- [x] 7.1 frontend: Save/bookmark + "Copy to my notes"
  actions on NoteDetailRoute and note cards for non-owned
  accessible notes — replaced by a "Log in to save" link
  (return path) for anonymous readers; bookmarks list
  route with unavailable-state rows + remove + empty
  state. Verify: tests — bookmark flow, unavailable row,
  copy target-tag picker, anonymous CTA.
- [x] 7.2 frontend: Author attribution on note cards in
  shared/public contexts; i18n strings (en + lv + ru) for
  all new UI. Verify: lint clean, `npm test` for touched
  files green.
- [x] 7.3 frontend: Update DEVELOPER_GUIDE.md/AGENTS.md
  "Known gaps"/behavior notes. Verify: doc diff covers
  new routes and the sharing model.

## 8. Integration

- [ ] 8.1 both: End-to-end check on staging — first grant
  `publish_note` to Tester5 via Django admin on
  `stagingdb`, then share note+tag between
  Tester5/Daniels, bookmark, copy, publish-gate UI,
  anonymous explore search; spot-check the Capacitor
  Android shell renders the new routes. Verify: manual
  QC steps from the spec scenarios pass on the Vercel
  staging deployment against `stagingdb`.
