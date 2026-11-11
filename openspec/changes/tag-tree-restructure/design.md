# Design

Data source for all analysis: production Aiven Postgres via the
read-only `ai_agent` connection, 2026-11-05. Distilled from the
original audit doc `docs/plans/tag-restructure-proposal.md`
(untracked draft, superseded by this change).

## Context

Model constraints that shape every decision:

- A note has exactly one tag (`Note.tag` FK) — tags behave as
  folders, not labels; a note can only live in one place.
- `?tag_id=<pk>&include_descendants=true` merges a whole subtree
  when listing, so mid-level "cluster" tags remain browsable —
  deeper grouping does not hide notes.
- Deleting a tag while notes remain anywhere in its subtree is
  409-blocked (`tag_has_notes`); `?delete_notes=true`
  force-deletes. Child tags CASCADE-delete with their parent.

Current state (user `Ted-Rose`, id 7): 127 tags, 736 tagged notes
+ 1 untagged (737 total), 29 notes public, 21 top-level tags, max
depth 3 (root -> child -> grandchild). Note-count distribution
per tag: 0 notes x29, 1 x40, 2 x14, 3-9 x24, 10+ x19 — 65 % of
all tags hold at most 2 notes; 19 tags carry the bulk.

### Problems found

1. `Research` is a dumping ground: 361 notes (49 % of all) under
   41 children of wildly mixed granularity — projects (Gospel
   harmony 138/947 verses, Once saved always saved? 52, Take
   every thought captive 56), normal topics (Jesus is God 12,
   Avoid Balaam's error 13, Feasts of the Lord 9) and one-note
   stubs (Alcohol, Millennium, Worship, The law, Vibe
   researching, Mini research, ...).
2. Four parallel "practical question" top-levels: `How to..`
   (70), `What to do..` (18), `Why..` (6), `Answers to questions`
   (12) — 106 notes, all the same dimension: applied Christian
   living phrased as questions.
3. Eschatology split across two trees: `Prophecies` (End times,
   Rapture pre/post, New world, Wealth) vs `Research` children
   (Eternal judgment, Millennium, Nobody knows the hour?, The
   day of judgement, Will God punish Christians?, The glory that
   is to be revealed, Once saved always saved?).
4. Same-topic duplicate tags, verified by zero shared verses:
   `Fear of the Lord` (2) + `The fear of the Lord is hatred of
   evil` (6); `Eternal judgment` (8) + `The day of judgement`
   (2) + `Will God punish Christians?` (1); `Gift of prophecy`
   (1) + `Gifts of the Holy Spirit` (1); `Marriage` (2) +
   `Bridegroom paradigm` (1).
5. Meta and orphan tags: `Mini research`, `Vibe researching`,
   `How to study Bible`, `Word study`; `Categorization > Titles
   of Jesus` (meta-tag with one child); singleton top-levels
   `Core Christ teachings > Reconcile`, `People of the Bible >
   Moses`, `God's love > God's desire to be with us`; `Random >
   Bookmarks` (1), `Random > Revelations` (1, duplicates intent
   of `Revelations and findings`).
6. Junk data: `What to do.. > Testing` — 14 app-testing notes
   ("haha", "Chjj ghjj", "Testing now", "Edited the first note",
   2026-01..2026-09); `Bible books > 1 Samuel` — one "asdfsa asd
   sad asd das sda" note; 8 empty leaf tags (`2nd Peter`,
   `Forgive`, `God's desire to be with us`, `Why Jesus came?`,
   `Saints in the end times`, `Sexual immorality`,
   `Brivibas nedelas nogale 2026-05`, `Abel`); 1 untagged note
   (1 Kings 1 verses — belongs under `Bible books > 1 Kings`).
7. Naming inconsistencies: `Sabath`, `Feast of the Boots`,
   `2nd Peter` (breaks the `1 Peter` convention), `Revelations`,
   `How to live trough`; mixed `Feast of`/`Feasts of` plurality;
   inconsistent trailing question marks.
8. Structural quirks: `Word for > Man of peace > It is okay to
   be lowly` (topic nested under a person); inconsistent depth
   under `Prophecies > End times`; `Sermons` vs `My sermons` as
   separate roots.

## Goals / Non-Goals

Goals:

- 6 top-level tags, ~85-95 tags total (from 127).
- Every note in a meaningful two-or-three-level path.
- Zero accidental note loss: the 409 `tag_has_notes` guard is
  relied on as the safety mechanism; notes are deleted only via
  the explicit, signed-off deletion list.

Non-goals:

- No backend/frontend code or schema changes.
- No tag-model semantics changes (single-tag, folder-like model
  stays).
- No changes to other users' data.
- Not a recurring governance policy — one-time cleanup only.

## Decisions

### D1: Execute through the existing API, not direct SQL

Everything is possible through existing endpoints as `Ted-Rose`:
create/rename/reparent via PATCH `parent_tag`/`name`, move notes
via PATCH `note.tag` (or `notes/move/`), delete via
`DELETE /tags/<id>/?delete_notes=true`. The 409 `tag_has_notes`
guard makes "move notes, then delete emptied tag" inherently
safe. A bulk script could run it in one transaction via the API
with the owner's token; direct DB writes are rejected (the
`ai_agent` user is read-only, and bypassing serializer-level
guards risks constraint violations).

### D2: Merge by moving notes, not by repointing tags

Duplicate-tag merges (Fear of the Lord, Eternal judgment, ...)
move each note to the surviving tag, then delete the emptied
duplicate. `unique_tag_position_per_tag` can collide on bulk
reassignment — the move endpoint resolves positions, so prefer
it over raw PATCH loops, or refresh positions afterwards.

### D3: Order destructive ops leaf-up

Reparenting a tag into its own descendant is 400-blocked, so
collapse chains by moving children/notes first and deleting
emptied parents last. Emptied mid-level roots (`God's love`,
`Categorization`, `Motivational verses`, the `End times`
mid-level) are deleted only after their subtrees are fully
moved.

### D4: Public notes reviewed before reshuffle

`public` is per-tag; moving a public note to a private tag does
not flip the flag. `Avoid Balaam's error` is the only public tag;
all 29 public notes are listed and their target tags confirmed
with the owner before execution.

## Target structure (21 roots -> 6)

```
Study topics                        (rename of "Research")
+- Christ & the gospel
|   +- Jesus is God                 12
|   +- Jesus sacrifice               2
|   +- Titles of Jesus               1   <- from Categorization
+- Salvation & judgment
|   +- Once saved always saved?     52
|   +- Judgment                     11   <- merge Eternal judgment +
|   |                                      The day of judgement +
|   |                                      Will God punish Christians?
|   +- Avoid Balaam's error         13   [public]
|   +- The law                       1
+- End times                        36   <- absorbs Prophecies
|   +- Rapture                       2+23 (Pre-trib 18, Post-trib 5)
|   +- The glory to be revealed      6
|   +- New world                     3
|   +- Millennium                    1
|   +- Nobody knows the hour         1
|   +- Antichrist                    1   (delete empty "Saints in
|   |                                    the end times")
+- Family & relationships           30
|   +- Family is church             21+1 (child: To one another)
|   +- Women                         4
|   +- Marriage                      2
|   +- Bridegroom paradigm           1
|   +- Child raising                 1
+- Christian character              64
|   +- Take every thought captive   56
|   +- Fear of the Lord              8   <- merge both FoL tags
|   +- Hasty temper                  2
|   +- Reconcile                     1   <- from Core Christ teachings
|   +- Vengeance is the Lord's       1
|   +- Worship                       1
|   +- Alcohol                       1
+- Holy Spirit & gifts               2   <- merge Gift of prophecy +
|   |                                    Gifts of the Holy Spirit
+- God's sovereignty & prayer        9
|   +- Creation - God relationship   4
|   +- Seated in heavenly places     3
|   +- Pray as all depends on God... 2
+- Feasts of the Lord                9   (rename children:
|   |                                    Sabbath, Booths,
|   |                                    Firstfruits, Weeks, ...)
+- Bible study methods               4
|   +- Word study                    1
|   +- How to study the Bible        1
|   +- Mini research                 1   (or fold into parent)
|   +- Vibe researching              1   (or fold into parent)
+- People of the Bible               1
    +- Moses                         1   <- move whole subtree

Gospel harmony                     138   <- promote from Research;
                                            standalone project

Bible books                        100   (rename "Bible books in
+- 1 Kings                           49    essence")
+- 2 Samuel                          22
+- Jeremiah                          13
+- Philippians                       12
+- 1 Peter                            3
+- 1 Samuel                           1    (delete "asdfsa" junk note)
+- 2 Peter                            0    (rename "2nd Peter",
                                           or delete if unwanted)

Sermons                             90
+- My sermons                       17   <- move root "My sermons"
|   +- Lord of Hosts                17
|   +- Why Jesus came?               0   -> delete or fill
+- Grooming the Next Generation     34
+- DM_Riga                           3+23 (date children unchanged)
+- DM_Jelgava                        1+13 (date children unchanged)
+- Church                            0+2
|   +- MK - Draudze                  2   (delete empty "Brivibas
|   |                                    nedelas nogale 2026-05")
+- Viri runa                         1

Practical living                    97   <- merge How to.. + What to
+- Struggles                         50    do.. + Why.. + Answers to
|   +- Living through troublesome    16    questions + Motivational
|   |    times                            verses + Memorise +
|   +- Avoiding burnout              11    Pre Baptism Checklist
|   +- Not anxious about anything    10
|   +- Why should we suffer?          7
|   +- Pre Baptism Checklist          6
|   +- Constant change & uncertainty  1
+- Dealing with people               31
|   +- Aggressors                    24
|   +- The accuser                    6
|   +- Enemies                        1
+- Purpose & obedience               16
|   +- Why should we follow God?      5
|   +- Meaning of life                5
|   +- What God wants us to do?       3
|   +- Making disciples               2
|   +- Why not to sin?                1
+- Encouragement                      4
    +- Keep on fighting               2
    +- Verses to memorise             2  (rename "Memorise")

Words & revelations                 33   <- merge Word for +
+- For people & groups                   Conversations with God +
|   +- AT                           5    Revelations and findings +
|   +- Dove                         3    Random > Revelations
|   +- Kids                         4    (child: SAR; delete empty
|   +- Man of peace                 7     Abel; lift "It is okay to
+- From God                        14     be lowly" notes up or keep
    +- It depends not on human      9     child)
    |    will or exertion...
    +- Passage for the season       1
    +- Revelations and findings     2   <- merge Random > Revelations
                                           + top-level singleton
```

## Deletions and renames

Deletions (notes destroyed — requires owner sign-off):

- `What to do.. > Testing` + its 14 junk notes
  (`?delete_notes=true`, or move 2-3 salvageable notes first).
- `Bible books > 1 Samuel > "asdfsa"` note — delete note, keep
  tag.
- Empty leaves (no notes lost): `2nd Peter` (or rename),
  `Forgive`, `God's desire to be with us`, `Why Jesus came?`,
  `Saints in the end times`, `Sexual immorality`,
  `Brivibas nedelas nogale 2026-05`, `Abel`.
- Emptied mid-level roots after moves: `God's love`,
  `Categorization`, `Motivational verses`, `End times` mid-level.

Renames: `Sabath`->Sabbath, `Feast of the Boots`->Feast of
Booths, `2nd Peter`->2 Peter, `Revelations`->Revelation,
`How to live trough troublesome times`->through; normalize
`Feast(s) of` forms.

## Risks / Trade-offs

- [Note loss via wrong move target] -> rely on the 409
  `tag_has_notes` guard; take an API-level snapshot of the full
  note->tag mapping before the first write so any move can be
  diffed and reverted.
- [`unique_tag_position_per_tag` collisions when merging notes
  into one tag] -> use the move endpoint (resolves positions) or
  refresh positions after bulk reassignment.
- [Top-level name uniqueness
  (`unique_top_level_tag_name_per_user`, serializer-enforced)]
  -> create roots with final names directly; no temporary
  placeholders that could collide.
- [Reparent-into-descendant 400] -> order moves leaf-up; never
  reparent before the target's descendants are relocated.
- [Public-note visibility regressions] -> `public` is per-tag;
  review the 29 public notes before moving any of them.
- [~100 sequential UI/API ops is error-prone] -> prefer a single
  scripted run with the owner's token; script is idempotent
  (name lookups, not hardcoded pks).
- [In-flight tag-UX frontend work (`reactive-bible-tag-ux`
  worktree, `fix/tag-management-mobile-ux`)] -> UI behavior
  changes there do not affect API semantics; re-verify the tag
  screens after this cleanup lands.

## Migration Plan

1. Snapshot: dump all of Ted-Rose's tags (id, name, parent,
   public) and note->tag pairs via the API — committed nowhere,
   kept as the revert reference.
2. Create new mid-level cluster tags under their final roots.
3. Move notes leaf-up (merge duplicates first, then cluster
   moves), then reparent surviving subtrees.
4. Renames (PATCH `name`).
5. Deletions: junk notes via `?delete_notes=true`, then emptied
   leaves/mid-levels.
6. Verify: 6 roots, expected note counts per cluster,
   `include_descendants=true` browsing, 29 public notes intact.

Rollback: no automatic rollback — revert is manual re-moves
guided by the step-1 snapshot. This is acceptable because every
operation is small, guarded and individually reversible.

## Open Questions

- Fold `Mini research`/`Vibe researching` into `Bible study
  methods` directly, or keep as named children? Owner's call.
- Rename or delete `2nd Peter` (empty leaf)?
- Salvage any of the 14 `Testing` notes before force-delete?
