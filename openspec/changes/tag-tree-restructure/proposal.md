# Proposal

## Why

User `Ted-Rose`'s production tag tree has grown organically to 21
top-level tags and 127 tags for 737 notes: `Research` alone holds
49 % of all notes, four parallel top-levels cover the same
"practical question" dimension, eschatology is split across two
trees, and ~65 % of tags hold 2 notes or fewer. Browsing the tree
and choosing a tag for a new note is harder than it should be. A
one-time data reorganization — executable entirely through the
existing API — collapses the tree to 6 roots and roughly 90 tags.

## What Changes

- Rename `Research` -> `Study topics` and reorganize its children
  into mid-level clusters (Christ & the gospel, Salvation &
  judgment, End times, Family & relationships, Christian
  character, Holy Spirit & gifts, God's sovereignty & prayer,
  Feasts of the Lord, Bible study methods, People of the Bible).
- Promote `Gospel harmony` (138 notes) to a standalone top-level
  study project.
- Merge the four question-style roots (`How to..`, `What to
  do..`, `Why..`, `Answers to questions`) plus `Motivational
  verses`, `Memorise` and `Pre Baptism Checklist` into one
  `Practical living` root with four mid-level clusters.
- Unify eschatology: fold the `Prophecies` subtree into
  `Study topics > End times`.
- Merge verified same-topic duplicate tags (`Fear of the Lord`
  x2, the `Eternal judgment` trio, the gift-of-prophecy pair,
  `Marriage` + `Bridegroom paradigm`).
- Unify sermons: move the `My sermons` root under `Sermons`.
- Merge `Word for`, `Conversations with God`, `Revelations and
  findings` and `Random > Revelations` into `Words & revelations`.
- Delete junk data (needs owner sign-off): `What to do.. >
  Testing` with its 14 app-testing notes, one "asdfsa" note, 8
  empty leaf tags and emptied mid-level roots.
- Rename misspelled/inconsistent tags (`Sabath`->Sabbath,
  `Feast of the Boots`->Booths, `2nd Peter`->2 Peter,
  `Revelations`->Revelation, `trough`->through; normalize
  `Feast(s) of` forms).
- No code, schema or API changes — data-only reorganization via
  existing PATCH/DELETE/move endpoints.

## Capabilities

### New Capabilities

None — this change introduces no new system behavior.

### Modified Capabilities

None — tag CRUD, move and delete semantics are unchanged; only
one user's production data is reshaped. `skip_specs: true` is
set in `.openspec.yaml` accordingly.

## Impact

- **Repos:** none directly — execution uses the existing
  `bible_research` API (and/or the `reactive-bible` UI) against
  production data only.
- **Data:** user `Ted-Rose` (id 7): 127 tags -> ~85-95, 21 roots
  -> 6; ~16 notes deleted (14 junk `Testing` notes + 1 "asdfsa"
  note, pending sign-off); every remaining note ends in a
  meaningful two-or-three-level path, kept browsable by
  `include_descendants=true`.
- **Coordination:** tag-UX work may be in flight in the
  `reactive-bible-tag-ux` worktree
  (branch `fix/tag-management-mobile-ux`; `docs/plans/
  tag-management-mobile-ux/` records PR #203 open, review clean).
  Those UI changes do not block this data cleanup, but the same
  tag screens will be used to verify the result — recheck status
  before execution.
- **Access:** the read-only `ai_agent` DB user cannot execute
  this; it needs a write-capable path — the API with the owner's
  token, or ~100 UI operations performed by the owner.
