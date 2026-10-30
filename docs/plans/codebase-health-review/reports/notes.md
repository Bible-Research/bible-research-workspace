# notes/ (NoteViewSet + reorder/order-index/move) — code review

## Summary

The reorder/move position machinery is unusually well-hardened —
two-phase temp-position writes, `select_for_update`, finite-float
validation, and a friendly 409 contract — and test coverage is
broad. The weak spots are elsewhere: `NoteSerializer` fires one
blocking provider call plus two extra DB queries per note on list
reads, verse-less notes get an inconsistent `tag` shape, and
`tag_position` is writable on the plain serializer where a
collision 500s instead of 409ing.

Verdict: fair

## Endpoint contract

Base: `/api/v1/notes/` (`annotations/urls.py:7`).

- `GET /notes/` — list. Params: `tag_id` (`untagged` sentinel =
  null tag, views.py:596-601), `public=true` (own ∪ public),
  `ordering` (`custom|-custom|created|-created|verse|-verse`,
  views.py:438-505), `page`, `page_size` (default 25, max 100,
  pagination.py:17-19), `fileset_id` (verse-text translation,
  views.py:340-346). Anonymous: public notes only.
- `GET /notes/<pk>/` — own or public; else 404.
- `POST /notes/` — `{note_text, public, tag, tag_position,
  verse_references[]}`; verse refs resolve against local `Verse`
  rows, missing verse → 400 (serializers.py:254-267).
- `PATCH /notes/<pk>/` — same fields; `verse_references` present
  replaces all links (serializers.py:274-323). Owner only (403).
- `DELETE /notes/<pk>/` — owner only (403/404).
- `POST /notes/bulk/` — `{tag_id, notes:[{note_text, public,
  verse_references}]}`; tag queryset scoped to user's tags →
  foreign tag 400; atomic; returns serialized list, 201.
- `POST /notes/linked/` — `{verse_references[]}` (1-500); own
  notes matching any ref; `{count, results}`; no provider calls.
- `POST /notes/reorder/` — `{tag_id, updates:[{note_id,
  position}]}`, max 100; two-phase rewrite under one transaction;
  204. Errors: 400 `validation_error`, 409 `position_conflict` /
  `database_error` (views.py:732-976).
- `GET /notes/order-index/?tag_id=` — unpaginated compact rows
  `{id, tag_position, book, chapter, verse_start, verse_end}`,
  canonical order; anonymous allowed (public only).
- `POST /notes/move/` — `{tag_id, note_id, target_note_id,
  placement: before|after}`; midpoint insert, rebalances 1..N on
  float exhaustion; 200 `{note_id, tag_position, rebalanced}`;
  same 400/409 codes (views.py:1064-1290).
- Provider degradation: serializer adds `error`/`error_code`
  (`rate_limited`|`not_found`|`provider_error`) to the note
  payload; refs kept, `text` empty (serializers.py:547-560,
  bible/utils/provider_errors.py:36-74).

## Findings

### Bugs / correctness

- **Inconsistent response shape for verse-less notes.** The early
  return at serializers.py:343-344 skips both the
  `verses`/`headings` keys and the nested-tag expansion at
  serializers.py:566-567. A note with no verses returns `tag` as
  a bare PK string and no `verses`/`headings`; a versed note
  returns a full tag object and both keys. Clients must sniff the
  shape.
- **`tag_position` is client-writable on NoteSerializer**
  (serializers.py:203-207 — it is a plain model field, not
  read-only). `POST`/`PATCH` with a position that collides in the
  tag raises `IntegrityError` from `super().update` /
  `Note.objects.create` — unhandled → 500, bypassing the 409
  `position_conflict` contract reorder/move carefully implement.
  Same for PATCH re-tagging a note into a tag where its position
  is already taken.
- **`order-index` ignores the `untagged` sentinel.** views.py:1025
  filters `tag_id='untagged'` literally → always `[]`, while list
  (views.py:596-598) and comments/counts (views.py:1538-1539)
  special-case it to `tag__isnull=True`. Untagged notes also have
  no reorder/move path at all — maybe intentional, undocumented.
- **Bulk create has no size cap.** `BulkNoteCreateSerializer.notes`
  (serializers.py:599-602) lacks `max_length` — reorder caps at
  100, `linked` at 500 refs, comment-counts at 200 ids, but one
  `notes/bulk/` body can create unlimited rows + NoteVerse links.
- **Parking math loses precision on extreme positions.**
  `_park_notes_for_renumber` writes `base - idx - 1`
  (views.py:88-92); when `|base|` ≳ 2^52 (reachable via
  client-supplied reorder positions → `extra_floor`, or real
  negatives accumulated by 'before first' moves) the decrements
  collapse to identical floats → phase-1 `bulk_update`
  self-collides → misleading 409. `MOVE_GAP_EPSILON` guards
  midpoints but not the park range.
- **Multi-book/multi-chapter labeling quirk.** Verses sort by
  verse number only (serializers.py:340-342); `verses[0]` fixes
  the book+chapter fetched (serializers.py:346-348) and every
  verse is labeled with it. A note linking John 3:16 + John 4:1
  labels everything "John 4" and leaves chapter-3 text empty —
  acknowledged by test_verse_labeling_multi_chapter_quirk
  (tests.py:2499-2517) but still wrong data.
- **`Verse.objects.get(book__iexact=…)` is not unique-safe.**
  `Verse` has no uniqueness on (book, chapter, verse)
  (bible/models.py:9-20); a duplicate row raises
  `MultipleObjectsReturned`, caught by the generic `except
  Exception` and re-raised as a 400 that embeds `str(e)`
  (serializers.py:264-267, 308-311) — masks a data bug and
  leaks exception text into the response.
- **Duplicate verse links allowed.** `NoteVerse` has no
  uniqueness (the `unique_together` is commented out,
  models.py:397-400); duplicate `verse_references` create
  duplicate links that surface as repeated verses in responses.
- Minor: `perform_create` logs `validated_data['text']` — the
  field is `note_text`, so it always logs 'N/A' (views.py:355).
  "Successfully updated/deleted" debug logs run before the write
  (views.py:400-403, 432-435). Reorder accepts `position: true`
  (floats to 1.0) and never type-checks `tag_id` (harmless —
  str() coercion → 400, but sloppy).

### Security & auth

- Scoping is solid overall: reads via `get_accessible_notes_qs`
  (own ∪ public, views.py:59-69); writes gated by
  `IsAuthenticated + GuestReadOnly` (views.py:329-338); tag
  assignment restricted to the caller's own tags inside the
  serializer (serializers.py:184-199); owner checks in
  `perform_update`/`perform_destroy` (views.py:373-436);
  reorder/move scope queries to `user=user` (views.py:901-905,
  1138-1145).
- **Existence oracle in the nested image routes** (in scope as
  `notes/<pk>/…`): `NoteImageViewSet._get_note` 404s a missing
  note but raises 403 for an existing inaccessible one
  (views.py:1593-1604); `CommentImageViewSet` is analogous
  (views.py:1694-1709). `CommentViewSet` deliberately 404s both
  cases (views.py:1350-1359, 1385-1387) — inconsistent policy.
- **Anonymous reads amplify provider cost.** `list`/`retrieve`/
  `order-index` are `AllowAny`; each serialized note triggers an
  upstream provider call (see Performance), so unauthenticated
  traffic can burn ESV/API.Bible/DBT rate limits.
- `unique_tag_position_per_tag` is tag-global, not per-user
  (models.py:152-158). Foreign notes can only share a tag via
  non-API writes, but when they do, their positions legitimately
  collide and produce 409s — acknowledged at views.py:1133-1137.

### Performance

- **N+1 + upstream call per note on list/retrieve.** `get_queryset`
  (views.py:507-614) has no `select_related('tag')` or
  `prefetch_related('verses')`; `to_representation` then hits the
  DB for `instance.verses.all()` and `instance.tag` per note
  (serializers.py:340-342, 566-567) *and* makes one blocking
  provider call per note (serializers.py:377-506). A default page
  of 25 → ~50 extra queries + up to 25 sequential HTTP calls —
  the dominant hot spot. `linked` (views.py:709-722) and
  `order_index` (views.py:1023-1040) already demonstrate the
  prefetch/no-enrichment fix; no chapter-level caching, so notes
  sharing a chapter re-fetch it.
- `ordering=verse` annotates three `Min` aggregates plus a
  ~66-branch `Case` built per request (views.py:453-480);
  `BOOK_ORDER_MAP`/`Case`/`Min` imports happen inside the method
  body (views.py:450-451).
- `linked` ORs up to 500 predicates into one query
  (views.py:702-708) — capped, but heavy SQL at the cap.
- `NoteSerializer.create`/`update` resolve verses with one
  `.get()` per reference (serializers.py:245-249, 293-297) while
  `BulkNoteCreateSerializer._resolve_verses` already does it in
  one query (serializers.py:632-673) — inconsistent.

### Code quality & consistency

- views.py is 1887 lines hosting 7 viewsets + helpers; the notes
  portion alone is ~980 lines with ~150 lines of hand-rolled body
  validation in `reorder`/`move` that reads like a serializer.
- Canonical ordering (`tag_position` ASC nulls-last,
  `-created_at`) is duplicated in `_apply_ordering`,
  `order_index`, and `move` (views.py:482-505, 1030-1033,
  1141-1144).
- `reorder` returns inside `transaction.atomic()` for the
  missing-notes 400 (views.py:908-921) — correct but surprising;
  hoisting the check would clarify that the commit is empty.
- Docstrings and comments are genuinely excellent (the
  parking/epsilon rationale is well explained); f-strings are
  used for logging throughout instead of lazy `%s` args.

### Test coverage

Strong. Reorder has swap/duplicate/occupied/foreign-note/
non-object/non-numeric/non-string-id/non-finite/non-dict cases
(tests.py:1750-2005); move covers midpoint, append, prepend,
rebalance incl. negative-position parking, and cross-user /
cross-tag rejection (tests.py:2595-2886); order-index covers
order, shape, labeling quirks, visibility, 150-note unpaginated
(tests.py:2389-2592); `linked` asserts 2 queries and zero
provider calls (tests.py:1413-1443); anonymous/guest matrices
covered. Ran: 83 note-related tests pass, 2 `expectedFailure`
(test_note_serializer, test_single_query_assertion).
Gaps: no test for a `PATCH` `tag_position` collision (the 500
above); no test pinning the verse-less `tag` shape; no
concurrency test for the reorder parking race (hard, acceptable).

## Refactoring recommendations

1. **P0 — Kill the serialization N+1.** Add
   `select_related('tag').prefetch_related('verses')` to
   `NoteViewSet.get_queryset`, and add a per-request cache of
   provider chapter fetches keyed by `(fileset_id, dbt_book_id,
   chapter)` in serializer context. Why: 25 sequential provider
   calls per page is the resource's biggest latency/429 risk.
   Effort: M.
2. **P0 — Fix `to_representation` early return.** Emit
   `verses: []`, `headings: []`, and the expanded `tag` object
   unconditionally so the response shape never depends on verse
   linkage. Why: silent contract break clients must sniff.
   Effort: S.
3. **P1 — Stop letting `tag_position` bypass the 409 path.**
   Either make it read-only on `NoteSerializer` (positions move
   only via reorder/move) or wrap create/update in the same
   `IntegrityError` → 409 `position_conflict` mapping. Why:
   today a single PATCH 500s on a normal collision. Effort: S.
4. **P1 — Cap `notes/bulk/`.** `max_length=100` on the `notes`
   field to match the reorder cap. Why: unbounded row creation
   per request. Effort: S.
5. **P1 — Align `order-index` with the `untagged` sentinel** (or
   reject it with a documented 400). Why: `tag_id=untagged`
   silently returns `[]` while every other endpoint resolves it.
   Effort: S.
6. **P2 — One canonical-ordering helper + one ordering map**;
   reuse `_resolve_verses` in `NoteSerializer`; move reorder/move
   body validation into a DRF serializer. Why: three copies of
   the same ordering already exist; hand-validation is ~150 lines.
   Effort: M.
7. **P2 — Precision-safe parking.** Compute temp positions with
   `math.nextafter(base, -inf)` stepping (or park below
   `base - len(notes) - margin` after checking representability)
   and treat unrepresentable ranges as a rebalance. Why: removes
   the false-409 edge. Effort: S.
8. **P2 — Unify the inaccessible-note policy in image routes**:
   404 (not 403) for notes the requester can't see, matching
   `CommentViewSet`. Why: closes the existence oracle and one
   convention. Effort: S.
9. **P2 — Data integrity for verses:** dedupe `verse_references`
   in validation; enforce/resolve `Verse` uniqueness (unique
   constraint or `filter().first()` + explicit handling) instead
   of `.get()` + catch-all. Why: duplicate links and masked
   `MultipleObjectsReturned`. Effort: M.
10. **P2 — Small cleanups:** fix the `text`→`note_text` log bug;
    move success logs after the writes; catch `OperationalError`
    deadlock in reorder/move → 409; hoist module-level imports in
    `_apply_ordering`. Why: correctness of logs/errors, hygiene.
    Effort: S.
