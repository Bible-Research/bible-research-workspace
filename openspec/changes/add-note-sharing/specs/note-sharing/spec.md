# Spec Delta

## Purpose

Defines the note-sharing contract across both repos: tiered
note visibility (private / link-public / published), targeted
per-user grants on notes and tags, username-based share
resolution, bookmarks, copy-to-my-notes, the published
directory feed and search, and publish-permission gating.

## ADDED Requirements

### Requirement: Tiered note visibility

A note SHALL carry two independent flags: `public`
(anyone with the link may read — unchanged existing
semantics) and `published` (listed in the public directory
feed and search). The invariant `published ⇒ public` SHALL
hold in both directions: setting `published` coerces
`public=true`, and clearing `public` clears `published`
— including via the tag `set-public` cascade.

#### Scenario: Publishing coerces the public flag

- **WHEN** an authorized caller sets `published=true` on a
  note with `public=false`
- **THEN** the note is stored with both `published=true`
  and `public=true`

#### Scenario: Unpublishing keeps link-sharing

- **WHEN** a note with `published=true` and `public=true`
  is updated to `published=false`
- **THEN** the note remains `public=true` and is still
  readable by link, but disappears from directory feed and
  search results

#### Scenario: Clearing public clears published

- **WHEN** `public=false` is set on a `published` note —
  directly or via `set-public` on its tag
- **THEN** the note's `published` flag is cleared too;
  `published=true ∧ public=false` is unreachable

#### Scenario: Private note invisible to non-owners

- **WHEN** a caller requests a note that is neither
  `public` nor `published` and holds no share grant on it
  or its tag
- **THEN** the API responds 404 and reveals nothing about
  the note's existence

### Requirement: Publishing requires a permission

Any write whose EFFECTIVE result is `published=true` SHALL
require `annotations.publish_note` (granted via Django
admin) — note create/PATCH, bulk create, `set-published`,
and inheritance from a published tag. Violations fail 400
in the annotations shape. `/me/` SHALL expose
`can_publish`.

#### Scenario: Non-publisher cannot publish

- **WHEN** an authenticated user without
  `annotations.publish_note` submits `published=true` or
  creates a note under a tag whose `published` default
  they cannot satisfy
- **THEN** the API responds 400 in the annotations error
  shape and no flag is stored

#### Scenario: Publisher flag exposed to client

- **WHEN** an authenticated user requests
  `GET /api/v1/users/me/`
- **THEN** the response contains `can_publish` reflecting
  `has_perm('annotations.publish_note')`

#### Scenario: Link-sharing stays ungated

- **WHEN** any authenticated user sets `public=true`
  without the publish permission
- **THEN** the request succeeds — only `published` is
  permission-gated

#### Scenario: Guest cannot publish or share

- **WHEN** the legacy `guest` account attempts a publish,
  share, bookmark, or copy write
- **THEN** `GuestReadOnly` responds 403 and
  `/me/` reports `can_publish=false`

### Requirement: Tag publishing mirrors tag publicity

`Tag.published` SHALL mirror `Tag.public`: new notes under
a published tag default to `published` (subject to the
permission gate), and a `set-published` action SHALL flip
the tag's flag and cascade it to direct notes — the
`false` direction clears `published` only, leaving each
note's `public` untouched.

#### Scenario: Notes inherit a published tag's flag

- **WHEN** a publisher creates a note under their
  published tag omitting `published`
- **THEN** the note is stored `published=true` and
  `public=true`

#### Scenario: Tag unpublish cascade preserves links

- **WHEN** a publisher posts `set-published {public:
  false}` semantics (`{"published": false}`)
- **THEN** the tag and its direct notes clear `published`
  while each note keeps its current `public` flag

### Requirement: Note share grants

`POST /api/v1/notes/<id>/share/` with `{"username"}`
SHALL create a per-recipient grant on an owned note.
Sharees gain read access plus comment ability; editing
stays owner-only. Grants per note SHALL be capped at 100.
`DELETE /api/v1/notes/<id>/share/<username>/` SHALL
revoke access immediately. Share endpoints SHALL be
owner-scoped: non-owners receive 404 and cannot create
transitive grants.

#### Scenario: Sharee can read a shared private note

- **WHEN** the owner shares a private note with username
  `alice` and `alice` retrieves it
- **THEN** the note is returned with `is_owner=false`

#### Scenario: Sharee can comment but not edit or re-share

- **WHEN** `alice` comments on a note shared with her
- **THEN** the comment is created (moderated by the
  existing note-owner delete rule); and **WHEN** `alice`
  PATCHes the note or hits its share endpoint
- **THEN** the API responds 404/403 as an owner-scoped
  resource

#### Scenario: Revoking a share

- **WHEN** the owner revokes `alice`'s grant on a private
  note and `alice` retrieves it
- **THEN** the API responds 404

### Requirement: Share resolution is not a user oracle

Share endpoints SHALL return identical success whether or
not the username resolves — never revealing account
existence. Matching SHALL be `iexact`. Re-sharing SHALL
be idempotent (note shares) or an upsert (tag shares).
Self-share is a no-op. An owner-only endpoint SHALL list
actual recipients. Share mutations SHALL be throttled.

#### Scenario: Unknown username still succeeds

- **WHEN** the owner shares a note with a username no
  account matches
- **THEN** the API responds with the same success body as
  for a real recipient and creates no grant

#### Scenario: Re-sharing a tag upserts the scope toggle

- **WHEN** the owner re-shares a tag with the same
  recipient but a different `include_descendants` value
- **THEN** the existing grant is updated to the new scope
  rather than silently ignored

#### Scenario: Owner inspects recipients

- **WHEN** the owner requests the share list for their
  note or tag
- **THEN** the response lists each recipient username and
  grant timestamp; sharees and anonymous callers cannot
  access it

### Requirement: Tag share grants with subtree scope

`POST /api/v1/tags/<id>/share/` with `{"username",
"include_descendants"}` SHALL grant the recipient access
to notes under the shared tag. With
`include_descendants=true` the grant SHALL cover the
tag's whole descendant subtree, resolved live — notes and
child tags created after the grant are covered, and notes
moved out stop being shared.

#### Scenario: Live subtree resolution

- **WHEN** a tag is shared with `include_descendants=true`
  and the owner later adds a note to a child tag
- **THEN** the sharee can read that new note

#### Scenario: Direct-tag-only share

- **WHEN** a tag is shared with
  `include_descendants=false` and the sharee requests a
  note under a child tag
- **THEN** the API responds 404

#### Scenario: Moved-out note stops being shared

- **WHEN** a note is moved out of a shared tag into an
  unshared tag and the sharee retrieves it
- **THEN** the API responds 404 unless the note is
  otherwise accessible (public/published/own share)

### Requirement: Shared visibility flows through one scope

Every note-reading surface (retrieve, list, order-index,
comments, comment counts, note images) SHALL apply a
single visibility scope: own notes, `public`/`published`
notes, notes with a direct share grant, and notes under
shared tags honoring `include_descendants`. Inaccessible
notes SHALL 404 without leaking existence.
`notes/linked/` SHALL stay own-notes-only.

#### Scenario: Comments on a shared note

- **WHEN** a sharee lists comments on a note shared via a
  tag grant
- **THEN** the comment tree is returned exactly as for a
  public note, and the sharee may comment

#### Scenario: Shared tag's private notes stay hidden

- **WHEN** a sharee lists notes of a tag that was never
  shared with them and is not theirs
- **THEN** only notes otherwise accessible
  (public/published) are returned

### Requirement: Shared-with-me listing

`GET /api/v1/notes/shared-with-me/` SHALL return both the
notes granted to the caller (direct grants plus shared-tag
notes) AND the tag grants (`tag id`, `name`,
`include_descendants`) so shared tags can render as
read-only nodes. Shared tags SHALL also be retrievable
individually by sharees. The frontend SHALL render them
in a "Shared with me" section, never in the user's own
tag tree.

#### Scenario: Sharee browses shared notes

- **WHEN** `alice` requests the shared-with-me listing
- **THEN** it contains every note granted to her plus a
  `tag_shares` array describing each shared tag, all with
  `is_owner=false` and `author` populated

#### Scenario: Sharee retrieves a shared tag by id

- **WHEN** `alice` requests `GET /tags/<id>/` for a tag
  shared with her
- **THEN** the tag is returned with a public-scoped
  `note_count`; a non-shared foreign tag still 404s

### Requirement: Bookmarks

An authenticated user SHALL be able to bookmark any
accessible note they do not own via
`POST /api/v1/notes/<id>/bookmark/` (idempotent) and
remove it. `GET /api/v1/notes/bookmarks/` SHALL list
bookmarks; entries whose target lost accessibility SHALL
be reported unavailable rather than dropped. A deleted
note's bookmark is removed with it (CASCADE).

#### Scenario: Bookmark an accessible note

- **WHEN** `alice` bookmarks a published note she does not
  own
- **THEN** the bookmark is created and appears in her
  bookmark listing with the full note payload

#### Scenario: Repeat bookmark is a no-op

- **WHEN** `alice` bookmarks the same note twice
- **THEN** the API returns success both times with exactly
  one bookmark row

#### Scenario: Dead bookmark shows unavailable

- **WHEN** a bookmarked note is unpublished and
  unpublicized (or its share revoked) and `alice` lists
  bookmarks
- **THEN** the entry is present with an unavailable
  marker instead of note content

#### Scenario: Bookmark own or inaccessible note rejected

- **WHEN** a user bookmarks their own note or one they
  cannot access
- **THEN** the API responds 400 for own and 404 for
  inaccessible

### Requirement: Copy to my notes

`POST /api/v1/notes/<id>/copy/` SHALL create a new note
owned by the caller duplicating the source's text and
verse links, `public=false`, `published=false`, with
`copied_from` provenance that survives source deletion
(`SET_NULL`) and is serialized to the copy's owner only.
`POST /api/v1/tags/<id>/copy/` SHALL copy every
accessible note in a tag into a caller-owned tag,
atomically; a source holding more than 100 accessible
notes SHALL fail 400 rather than truncate.

#### Scenario: Fork a public note

- **WHEN** `alice` copies a published note into her tag
- **THEN** a new note owned by `alice` is returned
  (`201`) with the same text and verse links, private
  flags, and `copied_from` set to the source

#### Scenario: Partial tag copy respects visibility

- **WHEN** `alice` copies a tag containing notes she can
  and cannot see
- **THEN** only notes accessible to her are copied

#### Scenario: Copying own note rejected

- **WHEN** the owner copies their own note
- **THEN** the API responds 400

#### Scenario: Source author hidden on published copies

- **WHEN** a copied note is itself published and an
  anonymous reader fetches it
- **THEN** the payload does not expose `copied_from` —
  no link back to the source's existence or owner

### Requirement: Published directory and search

`GET /api/v1/notes/?published=true` SHALL list published
notes for any caller including anonymous, paginated and
ordered by recency (`-created_at`) by default. `q` SHALL
match published notes against a stored plain-text form of
`note_text` — never raw TipTap JSON — and SHALL force
published scope regardless of other params; combining
`q` with `tag_id` or `note_pk` SHALL 400. `q` SHALL be
throttled. Directory/search responses SHALL NOT perform
per-note upstream verse enrichment.

#### Scenario: Anonymous searches the directory

- **WHEN** an anonymous caller requests
  `notes/?q=forgiveness`
- **THEN** only published notes whose plain text matches
  are returned, newest first, without verse-text
  enrichment

#### Scenario: Search never leaks private notes

- **WHEN** an authenticated caller searches with `q`
- **THEN** results are still restricted to published
  notes — private, link-public, and merely-shared notes
  are excluded

#### Scenario: q conflicts with scoped params

- **WHEN** a caller requests `notes/?q=x&tag_id=TAG…`
- **THEN** the API responds 400 in the annotations error
  shape

#### Scenario: Legacy public filter unaffected

- **WHEN** a caller requests `notes/?public=true`
  without `q`
- **THEN** existing behavior is preserved (own ∪ public
  for authenticated, public-only anonymous)

### Requirement: Author attribution

Note payloads SHALL include a read-only
`author {id, username}` for every serialized note, so
shared, published, and bookmarked notes identify their
creator.

#### Scenario: Shared note identifies its author

- **WHEN** a sharee or anonymous caller reads an
  accessible note owned by someone else
- **THEN** the payload's `author.username` names the owner

### Requirement: Share dialog UX

Owned notes and tags SHALL offer a share dialog: free-text
username with honest result copy ("if that username
exists, they'll get access"), an `include_descendants`
toggle on tags with a "future notes are shared too" hint,
an owner-visible recipient list that refreshes after
submit, and per-recipient revoke.

#### Scenario: Owner shares a tag from the dialog

- **WHEN** the owner submits a username with
  `include_descendants` checked
- **THEN** the recipient list refreshes to show the grant
  and the success copy makes no existence claim

### Requirement: Sharing surfaces and navigation

The frontend SHALL provide navigation entries and pages
for: a "Shared with me" section (read-only shared tag
nodes), an Explore page for the published feed with
search, and a Bookmarks list showing unavailable rows —
each with an empty state.

#### Scenario: Anonymous can browse the directory

- **WHEN** an anonymous visitor opens the app menu
- **THEN** the Explore entry is available while the
  auth-only Shared-with-me and Bookmarks entries are
  hidden or prompt login

### Requirement: Access-aware note actions

Save/Copy actions SHALL appear only on accessible notes
the viewer does not own; anonymous readers SHALL see a
"Log in to save" link with a return path instead.
Publish controls SHALL render only when `can_publish`,
and a published note's `public` checkbox SHALL be
checked and disabled.

#### Scenario: Publish toggle hidden for non-publishers

- **WHEN** `/me/` reports `can_publish=false`
- **THEN** the note editor and tag controls show no
  publish option

#### Scenario: Anonymous reader sees login prompt

- **WHEN** an anonymous visitor opens a public note page
- **THEN** Save/Copy controls are replaced by a "Log in
  to save" link carrying a return path

#### Scenario: Published note keeps public locked

- **WHEN** a publisher edits a published note
- **THEN** the `public` checkbox is checked and disabled,
  matching the server-side flag invariant

#### Scenario: Visitor saves a public note

- **WHEN** an authenticated visitor opens a public note
  page and chooses "Save"
- **THEN** the note is bookmarked and appears in their
  bookmarks list
