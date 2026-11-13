# Tasks

## 1. Preparation (read-only)

- [ ] 1.1 backend: Snapshot Ted-Rose's full tag tree (id, name,
  parent_tag, public) and every note->tag pair via the API;
  store the snapshot outside the repo as the revert reference
- [ ] 1.2 backend: List the 29 public notes with their current
  tags; confirm target tags with the owner (`public` is
  per-tag — moving notes must not silently change visibility)
- [ ] 1.3 both: Get owner sign-off on the deletion list
  (14 `Testing` notes, 1 "asdfsa" note, 8 empty leaves,
  emptied mid-level roots) and on the open questions in
  design.md (Mini research folding, 2nd Peter, salvageable
  Testing notes)

## 2. Create the new structure

- [ ] 2.1 backend: Create the six final top-level roots with
  their final names (rename `Research`->`Study topics`; create
  `Practical living`, `Words & revelations`; keep `Gospel
  harmony`, `Bible books`, `Sermons`)
- [ ] 2.2 backend: Create the mid-level cluster tags under
  `Study topics` (Christ & the gospel, Salvation & judgment,
  End times, Family & relationships, Christian character, Holy
  Spirit & gifts, God's sovereignty & prayer, Feasts of the
  Lord, Bible study methods, People of the Bible)
- [ ] 2.3 backend: Create the mid-level clusters under
  `Practical living` (Struggles, Dealing with people, Purpose
  & obedience, Encouragement) and under `Words & revelations`
  (For people & groups, From God)

## 3. Moves and merges (leaf-up)

- [ ] 3.1 backend: Merge duplicate-tag notes into survivors
  (Eternal judgment trio -> `Judgment`; both Fear-of-the-Lord
  tags -> `Fear of the Lord`; gift-of-prophecy pair -> `Holy
  Spirit & gifts`; `Bridegroom paradigm` handled per design)
- [ ] 3.2 backend: Move `Research` children into their new
  `Study topics` clusters per the design tree; promote `Gospel
  harmony` to top level
- [ ] 3.3 backend: Fold the `Prophecies` subtree into
  `Study topics > End times`; move `My sermons` under
  `Sermons`; move `People of the Bible > Moses` under the new
  cluster; retag the untagged 1 Kings note under `Bible books
  > 1 Kings`
- [ ] 3.4 backend: Move notes from `How to..`, `What to do..`,
  `Why..`, `Answers to questions`, `Motivational verses`,
  `Memorise`, `Pre Baptism Checklist` into the `Practical
  living` clusters; move `Word for`, `Conversations with
  God`, `Revelations and findings`, `Random > Revelations`
  into `Words & revelations`; verify
  `unique_tag_position_per_tag` never 409s (use the move
  endpoint or refresh positions)

## 4. Renames

- [ ] 4.1 backend: Rename `Sabath`->Sabbath,
  `Feast of the Boots`->Feast of Booths, `2nd Peter`->2 Peter
  (if kept), `Revelations`->Revelation, `How to live trough
  troublesome times`->through, `Memorise`->`Verses to
  memorise`, `Bible books in essence`->`Bible books`;
  normalize `Feast(s) of` forms

## 5. Deletions (after sign-off)

- [ ] 5.1 backend: `DELETE /tags/<id>/?delete_notes=true` on
  `What to do.. > Testing` (or move salvageable notes first);
  delete the "asdfsa" note under `1 Samuel`
- [ ] 5.2 backend: Delete the 8 empty leaf tags and the emptied
  mid-level roots (`God's love`, `Categorization`,
  `Motivational verses`, `End times` mid-level) — expect 409
  `tag_has_notes` if anything was missed; investigate instead
  of force-deleting

## 6. Verification

- [ ] 6.1 backend: Diff the post-change tag tree and note->tag
  mapping against the step-1.1 snapshot: 6 roots, ~85-95 tags,
  no note unintentionally lost, 29 public notes still public
- [ ] 6.2 frontend: In `reactive-bible` logged in as Ted-Rose,
  open the tags view: six roots render, expanding each root
  lists the expected note counts (`include_descendants=true`
  keeps clusters browsable), creating a note offers a sane tag
  dropdown — recheck on the `reactive-bible-tag-ux` branch if
  its mobile-UX PR is still open
