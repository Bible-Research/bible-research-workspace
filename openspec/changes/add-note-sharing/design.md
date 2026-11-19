# Design

## Context

See `proposal.md` — Why/What. The relevant current state:

- `annotations/views.py::get_accessible_notes_qs` is the
  single visibility funnel — every note-reading surface
  (retrieve, list, order-index, comments, images, counts)
  already flows through it, so widening it widens
  everything consistently.
- `Note.public` means "anyone with the link can read" and
  is open to every authenticated user; `Tag.public` +
  `set-public` cascade the flag to direct notes.
- `note_text` stores TipTap JSON — unusable for ranking or
  clean substring search.
- `Image` rows point at GCS blobs and a `post_delete`
  receiver deletes the blob — copying an Image row without
  copying the blob orphans the file when either row dies.
- Sharing must never become a user-existence oracle:
  registration already enforces whitespace-free,
  case-insensitive (`iexact`) usernames.
- `subtree_tag_ids(tag, user=owner)` exists and bounds
  descendant resolution to one owner's tree — the same
  primitive tag-share resolution reuses.

## Goals / Non-Goals

**Goals:**

- Tiered visibility: private / link-shared (`public`) /
  directory-listed (`published`), with `published`
  restricted to holders of a new permission.
- Explicit per-recipient grants on notes and tags,
  resolved live (grants follow content moves/additions).
- Username-typed sharing with a silent-success response —
  no user enumeration.
- Bookmarks, note copy, whole-tag copy; published-only
  feed + text search; author attribution.
- One contract across both repos; additive API only.

**Non-Goals:**

- Edit/collaboration access for sharees (view + comment
  only). Concurrent-write and ordering semantics need a
  dedicated change.
- Sharee self-removal ("leave this share"), share
  expiration, share invites/acceptance flows, per-user
  blocklists, notification emails.
- Image copying on fork (see Risks); full Postgres FTS
  ranking (see Search decision).
- Moderation dashboards, reporting flows, or a public
  user-profile page.

## Decisions

### 1. Two flags, not one: `public` (link) + `published`
###    (directory)

`public` keeps today's semantics untouched — backward
compatible, and link-sharing stays free for everyone.
`published` (default false) is the directory flag: only
`published=true` notes appear in `?published=true` feed
and `?q=` results.

**Invariant — both directions**: `published ⇒ public`.
Upward: setting `published` coerces `public=true` (a
directory entry that 404s for anonymous viewers is a
broken contract). Downward: clearing `public` clears
`published` too — including through the `set-public`
tag cascade — so `published=true ∧ public=false` is
unreachable. Unpublishing (`published=false`) leaves
`public` alone: the link keeps working. The frontend
mirrors this by rendering the `public` checkbox
locked-on while a note is published.

*Alternative considered*: reuse `public` as the directory
flag and gate it. Rejected — ordinary users would lose
link-sharing, and every existing public note would
silently join the directory.

### 2. Publish gate = `annotations.publish_note`
###    permission

`Note.Meta.permissions` gains `("publish_note", ...)`.
Enforced server-side via
`request.user.has_perm('annotations.publish_note')` on
every write whose EFFECTIVE result is `published=true`:
note create/PATCH, `notes/bulk/` items, the new tag
`set-published` action, AND inheritance from a
`Tag.published` default — a user whose permission is
revoked can't sneak published notes in through an old
published tag. Violations return the annotations 400
contract (`{'error': 'validation_error', 'message'}`) —
not 403 — since a non-publisher should not be able to
submit the flag at all; the frontend never shows the
toggle. `Tag.published` mirrors `Tag.public`: new notes
default to `published` under it, and `set-published`
flips the tag flag plus cascades to direct notes — the
`false` direction clears `published` only, preserving
each note's `public` (the invariant's downward coercion
is one-way).

*Alternatives*: `is_staff` (conflates admin access),
Group-name check (stringly-typed, same admin UX anyway).
A permission is assignable per-user or via a group in
admin and composes with future moderation permissions.

`/me/` (`UserSerializer`) gains `can_publish` =
`has_perm(...)` — the frontend's only signal for showing
the publish toggle.

### 3. Grants are rows, resolved at read time

```python
class NoteShare(models.Model):
    id      — SHR… prefixed PK
    note    FK -> Note (CASCADE)
    owner   FK -> User (CASCADE)   # denormalized for audit
    shared_with FK -> User (CASCADE)
    created_at
    Meta: unique_together (note, shared_with)

class TagShare(models.Model):
    id      — TSH… prefixed PK
    tag     FK -> Tag (CASCADE)
    owner   FK -> User (CASCADE)
    shared_with FK -> User (CASCADE)
    include_descendants = models.BooleanField(default=False)
    created_at
    Meta: unique_together (tag, shared_with)
```

No per-note rows for tag shares: `include_descendants` is
evaluated through `subtree_tag_ids(tag, user=owner)` at
query time, so notes added to a shared tag are visible
immediately and notes moved out stop being shared — a
materialized row set would go stale on both counts.

Visibility becomes:

```
accessible(user) = own
    | public
    | published                    (implies public anyway)
    | NoteShare.shared_with = user
    | note.tag ∈ {t : TagShare.shared_with = user}
    | note.tag ∈ descendants(shared_tag)   # iff include_desc
```

implemented inside `get_accessible_notes_qs` so comments,
images, counts and order-index inherit it for free.

**Shares survive flag changes**: unsharing a public flag
does not revoke explicit grants; the owner can always
unshare or delete. Documented behavior, not an accident.

**Owner-scoped only**: share/unshare/recipient-list
endpoints resolve through the owner queryset, so a
sharee hitting `share/` on a note they can merely read
gets 404 — no transitive re-sharing. Grants per note and
per tag are capped at 100 recipients (probing is
throttled, but a single owner shouldn't mint thousands
of grants either).

*Alternative*: share-invitation rows with accept/reject —
deferred (Non-Goals); direct grants match Google Keep /
Evernote behavior.

### 4. Username sharing, silent success

`POST /notes/<id>/share/ {"username": "…"}` and
`POST /tags/<id>/share/ {"username", "include_descendants"}`:

- Resolve with `username__iexact` (matches the login
  backend's case-insensitivity).
- On any resolvable-or-not outcome return 200
  `{"shared": true}` — never 404/400 on an unknown name;
  the endpoint MUST NOT confirm whether a username exists.
- Note shares are idempotent no-ops on repeat; tag
  shares are an UPSERT — re-sharing the same recipient
  with a flipped `include_descendants` updates the
  existing grant, otherwise "idempotent" would silently
  keep the old scope (the worst possible surprise in
  both directions).
- Self-share is a no-op success (owner already sees it).
- Owner reads back the truth: `GET /notes/<id>/shares/`
  (owner-only) lists `{shared_with: username, created_at}`
  — typos are visible to the person who made them.
- `DELETE /notes/<id>/share/<username>/` and
  `/tags/<id>/share/<username>/` revoke (usernames are
  whitespace-free, safe in paths).
- Share mutations get a dedicated throttle scope
  (`share`, in `DEFAULT_THROTTLE_RATES` with a conftest
  override) — silent-200 stops the answer, not the
  probing.
- Sharing a private note is allowed: grant + flag are
  independent axes.
- Share-success UX copy is honest about the oracle-free
  design: "if that username exists, they'll get access" —
  never "shared with bob".

*Alternative considered*: 404 on unknown username — better
typo UX but turns the endpoint into an oracle; the
owner-visible share list recovers most of the UX anyway.

### 5. Bookmarks = `SavedNote` pointer rows

```python
class SavedNote(models.Model):
    id    — SVD… prefixed PK
    user  FK -> User (CASCADE)
    note  FK -> Note (CASCADE)
    created_at
    Meta: unique_together (user, note)
```

- `POST /notes/<id>/bookmark/` + `DELETE` — target must be
  accessible AND not owned by the requester (400 own,
  404 inaccessible). Repeat bookmarks are idempotent
  no-ops — `unique_together` alone would turn a
  double-tap into an IntegrityError 500.
- `GET /notes/bookmarks/` lists the user's bookmarks.
  Each row serializes the note if still accessible, else
  `{note_id, available: false}` — the frontend renders a
  greyed "no longer available" row with a remove action.
- Deliberate asymmetry: a note whose VISIBILITY is lost
  (unpublished, unshared) shows as unavailable, but a
  DELETED note's bookmark cascades away — there is no
  reference left to preserve. Stated so nobody "fixes"
  it later.

*Alternative*: auto-remove dead bookmarks — rejected;
losing a saved reference silently is worse than an
unavailable badge.

### 6. Copy = fork into a new owned row

- `POST /notes/<id>/copy/ {"tag_id": <mine>|"untagged"}`
  — creates a NEW `Note` owned by the requester:
  `note_text` duplicated, `verses` re-linked via a new
  `NoteVerse` set, `public=false`, `published=false`,
  `copied_from=<source>` (new nullable self-FK,
  `on_delete=SET_NULL` — CASCADE would destroy copies
  with the source). `copied_from` is serialized to the
  copy's OWNER only — exposing it on a published copy
  would leak the source note's existence/owner to
  anonymous readers. Source must be accessible and not
  already owned (400 for own-note copies). Returns 201
  with the new note payload.
- `POST /tags/<id>/copy/ {"tag_id": <mine>}` — bulk-copies
  every note in the source tag that is accessible to the
  requester (respecting their visibility — a sharee
  copying a partially-shared tag gets only what they can
  see). Atomic; a source with more than 100 accessible
  notes fails 400 rather than silently truncating —
  the user splits or narrows first.
- New notes land in the target tag at appended
  `tag_position`s — never reuse source positions
  (unique_tag_position_per_tag is per tag but stale
  floats would scramble ordering). Every create path —
  serializer, note copy, tag copy, bulk — populates
  `note_text_plain` or copies surface in search with no
  text to match.

*Alternative*: bookmark-as-subscription with live view —
rejected earlier by product; copy gives permanence.

### 7. Search: `q=` over a plain-text column

- `GET /api/v1/notes/?published=true&q=<term>` — `q`
  FORCES published scope for every caller, even combined
  with `public=true`; combined with `tag_id`/`note_pk` it
  400s (explicit beats silently ignoring one param).
  Own-note search is a separate future concern; the same
  param can gain a `scope=own` later.
- Feed/search default to `-created_at` and ride
  `NotesPagination` — the default `custom` ordering is
  `tag_position`, which is per-tag-per-owner and
  meaningless across a multi-author directory.
- Add `Note.note_text_plain` (TextField; a plain btree
  index won't help `%term%` icontains, so no index — the
  published subset is small), maintained by stripping
  TipTap JSON to text on every create path. `icontains`
  for v1 — correct results without `SearchVector`
  migration complexity; upgrade to Postgres FTS only
  when the corpus justifies it.
- `q` gets a scoped `UserRateThrottle` (e.g.
  `notes-search`, mirroring the `search` scope on
  `bible/search/`) — an unthrottled `icontains` scan is
  a cheap DoS for authenticated users.
- List responses (feed, search, shared-with-me,
  bookmarks) set `include_verse_text=False` — verse refs
  stay, text is empty. Otherwise a 25-note anonymous
  page fans out to ~25 blocking upstream Bible-provider
  calls (latency + provider quota), exactly why
  `notes/linked/` already opts out.
- Results serialize through the normal note payload +
  `author` — no snippet/highlight contract in v1.
- Search is published-only, so `AllowAny` applies like
  the rest of the public surface — anonymous users can
  search the directory.

*Alternative*: `icontains` on the raw JSON field — matches
markup keys and can't rank or produce clean snippets.

### 8. Author attribution

`NoteSerializer` adds read-only `author {id, username}`
(alongside existing `is_owner`). Comments already expose
an author object — same shape. Anonymous/private-own
reads include it too (owner sees own name harmlessly);
the username of a *published* note's author is public
information by definition.

*Trade-off*: username is a login identifier — exposing it
on public notes slightly widens the attack surface
(phishing/credential guessing, already throttled at
`POST /api/token/`). Accepted: attribution is core to a
directory product; a display-name field can layer on
later without breaking the contract.

### 9. Tag visibility for sharees

`GET /tags/` stays owner-only. `GET
/notes/shared-with-me/` returns `{notes: [...],
tag_shares: [{tag id, name, include_descendants,
created_at}]}` — the tag grants are needed to render
read-only tag nodes in a "Shared with me" section; a
notes-only payload can't render an empty shared tag.
`GET /tags/<id>/` additionally resolves for sharees of
that tag (mirroring the anonymous "tag with public
notes" retrieve), with `note_count` filtered to what
the sharee can see. Shared tags render as read-only
nodes — NOT spliced into the user's own tag tree, which
would collide with per-user `unique_together` naming
and ownership assumptions throughout the tree UI.
Acknowledged leak: a shared note's embedded tag object
exposes the owner's tag name/id (Google Keep-like,
accepted); `notes/linked/` deliberately stays
own-notes-only — shared ≠ own.

## Risks / Trade-offs

- **Image orphaning on copy** → v1 copies do not clone
  images (note payload omits them; documented in the
  copy response or spec). Copying Image rows against the
  same `storage_url` would let either row's delete purge
  the blob out from under the other; proper fix is blob
  duplication at copy time — follow-up change.
- **`note_text_plain` drift** → maintained in one place
  (serializer save); backfill migration for existing
  rows; tests assert plain/JSON parity on create+PATCH.
- **Subtree shares vs `tag-tree-restructure`** (in-flight
  change) → `subtree_tag_ids` is the single resolution
  primitive; if that change alters hierarchy semantics,
  tag-share resolution updates in one place. Flag as a
  coordination point in tasks.
- **Share-grant N+1 on note lists** → shares join via
  `Exists` subqueries in the funnel queryset rather than
  Python loops; paginated endpoints stay O(1) queries.
- **Bookmark availability check per row** → annotate
  `available` via the same `Exists` expression reused
  from the funnel — no per-row queries.
- **Permission grant UX** → none in-app (admin only);
  acceptable: publishing is editorial staff work.
- **Grandfathered public notes** → stay public but never
  published; they keep working by link without entering
  the directory. No data migration needed beyond
  backfilling `note_text_plain`.

## Migration Plan

1. Backend: models + `makemigrations` (human-run per repo
   rules) — `Note.published`, `Note.copied_from`,
   `Note.note_text_plain`, `Meta.permissions`, three new
   tables; data migration backfilling `note_text_plain`.
2. Backend: funnel extension, share/bookmark/copy
   endpoints, `published`/`q` params, `author` field,
   `can_publish` on `/me/`, `set-published` tag action.
   Deploy order irrelevant vs frontend (all additive).
3. Frontend: api modules, share dialog, Explore route,
   Shared-with-me + Bookmarks sections, note-detail
   actions, publish toggle behind `can_publish`, i18n.
4. Admin grants `publish_note` to approved users
   (per-user or a "Publishers" group) — zero code.
5. Rollback: additive only — revert code; new tables and
   columns can stay harmlessly or be dropped by a human.

## Open Questions

- Should `published` notes' *comments* stay visible after
  unpublishing (the note goes link-only but keeps an
  existing discussion)? Default: yes — comments follow
  note visibility, nothing special-cased.
- Whole-tag copy: copy tag structure too (recreate the
  tag subtree under the copier's tree)? Default: flat —
  all accessible notes land in the one target tag.
