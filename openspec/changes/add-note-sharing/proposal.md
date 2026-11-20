# Proposal

## Why

Today the only way to show another person a note is the
`public` flag: an all-or-nothing switch that makes a note
readable by anyone who has the link. There is no way to
share with a specific user, no way to discover public
content inside the app (the `?public=true` list exists but
is unsurfaced and unsearchable), no way to keep a public
note found via a link, and no editorial control over what
reaches a platform-wide audience — anyone can set
`public=true`.

The product wants a real sharing layer: targeted
user-to-user sharing of notes and tags, a curated public
directory where published notes are browsable and
searchable, bookmarks for saved public notes, and copying
shared/public content into one's own account — with
publishing rights restricted to approved users.

## What Changes

- **Visibility tiers on notes**: keep `public` as the
  link-sharing flag (unchanged semantics, open to everyone);
  add `published` — a stricter flag that makes a note
  appear in the public directory feed and search results.
  `published` writes are gated by a new
  `annotations.publish_note` permission granted via Django
  admin (not `is_staff`).
- **Targeted sharing**: new `NoteShare` and `TagShare`
  grant models — one row per (note|tag, recipient). Tag
  shares carry an `include_descendants` toggle the sharer
  picks at share time; access is resolved at read time
  through the tag/subtree, so notes added later are covered
  and notes moved out stop being shared. Recipients get
  view + comment access; edits stay owner-only.
- **Share by username**: free-text username entry (no user
  list, no autocomplete); the share endpoint answers
  identically whether or not the username resolves —
  sharing is never a user-existence oracle.
- **Visibility funnel extension**:
  `get_accessible_notes_qs` gains share grants, so shared
  notes/tags flow through every existing surface
  (retrieve, order-index, comments, images, counts) with
  no contract change.
- **Bookmarks**: `SavedNote` rows let any authenticated
  user bookmark an accessible note they do not own; a
  bookmarked note that becomes inaccessible shows as
  "unavailable" until the user removes the reference.
- **Copy (fork)**: `POST /notes/<id>/copy/` materializes
  an accessible note as an owned note (new row,
  `copied_from` provenance); `POST /tags/<id>/copy/` bulk
  -copies every accessible note in a tag into one of the
  copier's tags.
- **Directory feed + search**: `GET /notes/` gains
  `?published=true` and `?q=` filters scoped to published
  notes; searching requires a stored plain-text form of
  `note_text` (the field stores TipTap JSON, which cannot
  be ranked or cleanly matched).
- **Author attribution**: note payloads gain a read-only
  `author {id, username}` so shared/published notes
  identify their creator.
- **Frontend**: Explore route (published feed + search),
  share dialogs on notes and tags (username + subtree
  toggle), a "Shared with me" section, a bookmarks list
  with unavailable-state, "Save"/"Copy to my notes"
  actions on the note detail page, author attribution in
  note cards, and a publish toggle shown only when
  `/me/` reports `can_publish`.
- **`/me/` payload** gains `can_publish` (computed from
  `has_perm('annotations.publish_note')`).

## Capabilities

### New Capabilities

- `note-sharing`: the whole sharing/visibility contract —
  tiered note visibility (private / link-public /
  published), targeted note and tag grants with subtree
  scope, username-based share resolution without a user
  oracle, bookmarks, copy-to-my-notes, the published
  directory feed and search, publish-permission gating,
  and author attribution on non-owned notes.

### Modified Capabilities

<!-- auth's write-scoping requirement is unchanged: tag FK
     querysets on note/tag writes stay owner-only. The /me/
     can_publish field is covered by note-sharing rather
     than redefining auth behavior. -->

## Impact

- **Repos**: both — `bible_research` (models, migrations,
  viewsets, serializers, permission check, `/me/` payload)
  and `reactive-bible` (new routes, dialogs, api modules,
  store slices, i18n strings).
- **APIs**: `GET /api/v1/notes/` gains `published`/`q`
  params; new actions `share`, `unshare`, `bookmark`,
  `unbookmark`, `copy` on notes; `share`/`unshare`/`copy`
  on tags; new `bookmarks`/`shared-with-me` listings;
  `GET /api/v1/users/me/` gains `can_publish`. All
  additive — no breaking contract changes; `public`
  link-sharing is untouched.
- **DB**: three new tables (`NoteShare`, `TagShare`,
  `SavedNote`), `Note.published`, `Note.copied_from`,
  `Note.note_text_plain` (or equivalent search column),
  `Note.Meta.permissions` entry — a human-run
  `makemigrations` per repo rules.
- **Cross-repo contract**: note payloads gain `author` and
  `published`; share/bookmark/copy endpoints join the
  annotations error-contract shape
  (`{'error', 'message'}`).
- **In-flight changes**: `tag-tree-restructure` may alter
  subtree semantics that `include_descendants` relies on —
  coordinate before implementing tag-share resolution.
