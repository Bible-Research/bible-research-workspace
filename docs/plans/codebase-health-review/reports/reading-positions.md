# reading-positions/ (ReadingPositionViewSet) — code review

## Summary

Auth and per-user scoping are solid — guest reads work, guest
writes 403, cross-user rows 404 — and the `update_or_create`
upsert is race-safe on Postgres. The core defect is that book
names are validated tolerantly but stored verbatim, so
`unique_together (user, book)` can hold several rows for one
logical book and the `?book=`/`bulk` lookups silently miss them;
PUT/PATCH can also hit a 500 on the uniqueness constraint.
Verdict: **fair**.

## Endpoint contract

- Route: `annotations/urls.py:8-12` → `ReadingPositionViewSet`
  (`annotations/views.py:1831`).
- Auth: `permission_classes = [IsAuthenticated, GuestReadOnly]`
  (views.py:1841) — identical to `DEFAULT_PERMISSION_CLASSES`
  (settings.py:377-380). Guest may read, writes → 403.
- `GET /reading-positions/` → array of
  `{id, book, chapter, verse, last_accessed}`, ordered
  `-last_accessed` (model Meta, models.py:461), scoped to
  `request.user` (views.py:1846).
- `GET /reading-positions/?book=<name>` → exact, case-sensitive
  match on the stored raw string (views.py:1848-1850).
- `GET /reading-positions/<id>/` → same object; 404 for other
  users' rows (queryset scoping).
- `POST /reading-positions/` → upsert keyed on `(user, book)`;
  always returns 200 with the object — never 201, no
  created/updated flag (views.py:1854-1863). `book` must resolve
  via `get_dbt_book_id` else 400 (serializers.py:918-924).
- `PUT/PATCH /reading-positions/<id>/` → partial/full update;
  `id` and `last_accessed` read-only, `book` writable.
- `DELETE /reading-positions/<id>/` → 204; 404 cross-user.
- `POST /reading-positions/bulk/` with `{"books": [...]}` →
  `{<book>: {"chapter": int, "verse": int} | null}`
  (views.py:1865-1887); 400 `{"error"}` when `books` is not an
  array.
- Errors: 400 DRF validation, 401/403 unauthenticated or guest
  write, 404 cross-user, **500** on PATCH/PUT book collision
  (unhandled `IntegrityError`).

## Findings

### Bugs / correctness

1. **Book stored uncanonicalized — duplicates per logical book**
   (serializers.py:918-938, models.py:460). `validate_book`
   accepts anything `get_dbt_book_id` resolves — mixed case,
   `"1st Samuel"`, `"I Samuel"`, aliases (`"canticles"`), even
   fuzzy typos (`"genisis"`, bible_books.py:144,165-192) — but
   `create()` upserts on the *raw* string. `"Genesis"` and
   `"genesis"` produce two rows; `"genisis"` is stored verbatim.
   `?book=` (views.py:1850) and `bulk` (views.py:1877) then
   exact-match the raw string, so a position saved as
   `"genisis"` is invisible to a `"Genesis"` query.
2. **PATCH/PUT `book` → 500** — changing `book` to a name that
   already exists for the user violates
   `unique_together (user, book)` (models.py:460); the default
   `update` path has no `IntegrityError` handling, unlike
   `reorder`/`move` which map it to 409 (views.py:945-976). The
   upsert key probably should not be client-mutable at all.
3. **`bulk` skips all validation** — no `validate_book`, no
   canonicalization, no per-item type check (views.py:1868-1878).
   Invalid names silently return `null` instead of a 400; the
   payload differs in contract from the single upsert.
4. **No bounds on `chapter`/`verse`** — `IntegerField`s with
   defaults only (models.py:443-451); `chapter=0`, `-5`, or
   `99999` all persist. No per-book chapter-range check either.
5. **`create` always returns 200** (views.py:1860-1863) —
   tolerable for an upsert, but callers cannot distinguish
   insert from update.
6. **Upsert race is handled** — `update_or_create`
   (serializers.py:931) wraps `select_for_update` +
   `get_or_create` with an IntegrityError retry, so concurrent
   POSTs for the same book serialize correctly on Postgres.
   Semantics remain last-writer-wins on chapter/verse.
7. **Note**: the design note "upserts per user+book+chapter" is
   wrong — uniqueness is `(user, book)` only (models.py:460);
   `chapter` is a stored value, not a key dimension.

### Security & auth

- Guest handling is correct: `GuestReadOnly` passes SAFE_METHODS
  for everyone (permissions.py:27-28) and rejects guest writes
  with 403 (permissions.py:32); `IsAuthenticated` blocks truly
  anonymous requests. Verified consistent with the stated
  design.
- **Inconsistency**: `bulk` is a read implemented as POST, so
  guests get 403 on `bulk` while `GET ?book=` works — the same
  data is reachable but not via the batch path (views.py:1865).
- No ownership holes: `get_queryset` and `bulk` both filter on
  `request.user` (views.py:1846, 1876); the upsert assigns the
  context user (serializers.py:928); cross-user detail access
  404s. `perform_update`/`perform_destroy` checks are
  unnecessary because scoping already confines instances.
- No secrets/exceptions echoed; invalid book is reflected in the
  400 message — acceptable, consistent with siblings.

### Performance

- Bound is ~66 rows per user; single indexed query for list
  (`reading_pos_user_book_idx`, `reading_pos_user_time_idx`,
  models.py:462-471) — appropriate, no N+1.
- `bulk` has **no cap on `books` length** — an unbounded `IN`
  clause (contrast `NOTE_IDS_CAP = 200`, views.py:47).
- `bulk` materializes all positions then builds the result map
  in Python — fine at this scale.
- `update_or_create` costs SELECT FOR UPDATE + UPDATE per POST;
  reasonable.

### Code quality & consistency

- `permission_classes` at views.py:1841 duplicates the global
  default exactly — dead code that suggests an intentional
  override where none exists.
- `create` calls `serializer.save()` directly and skips
  `perform_create` (views.py:1854-1863); user assignment lives in
  `serializer.create` via context instead. Works, but breaks the
  documented convention ("`perform_create` assigns the request
  user").
- `bulk` error body `{'error': 'books must be an array'}`
  (views.py:1871) doesn't follow the sibling convention of
  `{'error': 'validation_error', 'message': ...}` used by
  `reorder`/`move`/`order_index` (e.g. views.py:763-769).
- No `@extend_schema` on `bulk` or the `book` query param —
  undocumented in OpenAPI despite drf-spectacular.
- No logging anywhere in the viewset, unlike every sibling.
- `bulk` result keys echo raw client input, so `"Genesis"` and
  `"genesis"` are distinct keys over the same stored data.

### Test coverage

Only two tests exist (`tests.py:2008-2041`,
`ReadingPositionViewSetTest`): valid create → 200 and invalid
book → 400.

Missing: upsert-updates-existing-row; case/alias variants
creating duplicates (would pin finding 1); list scoping vs other
users; `?book=` filter; retrieve/PATCH/DELETE incl. cross-user
404; PATCH book-collision → pins finding 2; guest read allowed /
guest write → 403; `bulk` happy path, non-array → 400, missing
book → null; `chapter`/`verse` bounds → 400.

## Refactoring recommendations

1. **P0 — Canonicalize `book` before storing** — in
   `validate_book`, return
   `get_book_name_from_id(get_dbt_book_id(value))`
   (bible_books.py:85-101) so all accepted spellings persist as
   the title-cased canonical name. Restores the
   one-row-per-book invariant and makes `?book=`/`bulk`
   deterministic. Needs a data note: existing rows may hold
   raw variants. Effort: S.
2. **P0 — Make `book` immutable after create** — drop it from
   writable fields on update (or `read_only` + explicit upsert),
   removing the PATCH→500 `IntegrityError` path entirely.
   Alternatively catch `IntegrityError` in `perform_update` and
   return 409 like `reorder` does (views.py:945-961).
   Effort: S.
3. **P1 — Normalize `bulk` input through the same lookup** —
   map each requested name via `get_dbt_book_id` →
   `get_book_name_from_id` before the `book__in` query, reject
   unresolvable names with 400, and cap the array (e.g. 100,
   mirroring `NOTE_IDS_CAP`). Effort: S.
4. **P1 — Allow guest reads on `bulk`** — either switch the
   endpoint to `GET` (safe method; params/query or repeated
   `book=` keys) or add an explicit permission exception for the
   `bulk` action, so guest clients get parity with `GET ?book=`.
   Effort: S.
5. **P1 — Add `MinValueValidator(1)` to `chapter`/`verse`** on
   the serializer (models.py:443-451 stay as-is); optionally a
   per-book chapter ceiling via `get_dbt_book_id`. Effort: S.
6. **P2 — Remove the redundant `permission_classes` line**
   (views.py:1841) or comment why it exists; add a short
   docstring note that defaults already cover it. Effort: S.
7. **P2 — Route create through `perform_create`** and keep the
   upsert in the serializer, returning 201 on insert vs 200 on
   update (track `created` from `update_or_create`). Effort: S.
8. **P2 — Unify error shape in `bulk`** —
   `{'error': 'validation_error', 'message': ...}` plus
   `@extend_schema` for the request body and the `?book=` query
   param. Effort: S.
9. **P2 — Expand `ReadingPositionViewSetTest`** with the
   missing cases listed above, prioritizing duplicate-variant
   upserts, guest write 403, and `bulk`. Effort: M.
10. **P2 — Add request logging** consistent with sibling
    viewsets (`logger.info` on create/update/delete).
    Effort: S.
