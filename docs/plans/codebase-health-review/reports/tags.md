# tags/ (TagViewSet) — code review

## Summary

TagViewSet is a compact ModelViewSet whose security rests almost
entirely on queryset scoping; reads and writes are correctly
partitioned between anonymous, guest, device, and registered users,
and the DELETE `?delete_notes=` 409 guard is a genuinely good
addition. The weaknesses are latent rather than exposed: the
subtree walk and CASCADE delete ignore ownership, `perform_update`
skips the ownership check its own docstring promises, and the
serializer permits self-parent cycles. Verdict: **fair**.

## Endpoint contract

Routes: `annotations/urls.py:6` (DefaultRouter, basename `tag`).

- `GET /api/v1/tags/` — `AllowAny` (views.py:132-133).
  Authenticated → 200, unpaginated array of own tags
  `{id, name, parent_tag, created_at, updated_at}` ordered by
  name (views.py:152-160; `user` is a HiddenField, write-only).
  Anonymous → 200 `[]` (views.py:169). No `pagination_class` and
  no `DEFAULT_PAGINATION_CLASS` (settings.py:368-382), so the
  full list is always returned.
- `GET /api/v1/tags/<pk>/` — authenticated: own tag → 200,
  another user's tag → 404. Anonymous: 200 only when the tag has
  ≥1 public note, else 404 (views.py:165-168). Response also
  leaks `parent_tag` id of a private parent (id only).
- `POST /api/v1/tags/` — `IsAuthenticated` + `GuestReadOnly`
  (views.py:134). Body `{name, parent_tag?}` → 201 + tag object.
  400 on blank/overlong name, duplicate `(user, name,
  parent_tag)` via auto `UniqueTogetherValidator`, or
  `parent_tag` outside the caller's tags
  (serializers.py:86-105). Anonymous → 401/403 in tests; in
  production the middleware auto-provisions a device user for
  non-safe methods (middleware.py:30-42), so "anonymous" writes
  actually succeed under a new device account.
- `PUT/PATCH /api/v1/tags/<pk>/` — same permissions → 200; 404
  for other users' tags (queryset scoping, no explicit owner
  check); 403 for `guest`.
- `DELETE /api/v1/tags/<pk>/[?delete_notes=true]` — 204 on
  success. 409 `{error: 'tag_has_notes', message, note_count}`
  when the tag subtree still has notes and `delete_notes` is not
  exactly `'true'` (views.py:243-269). Cascade deletes
  descendant tags and their notes/comments/images via
  `Tag.parent_tag` / `Note.tag` CASCADE (models.py:47-55,104-112).

## Findings

### Bugs / correctness

1. **No cycle prevention on `parent_tag`.** `TagSerializer` has
   no `validate_parent_tag`; `PATCH /tags/<id>/
   {parent_tag: <same id>}` is accepted (the scoped queryset at
   serializers.py:94-96 includes the tag itself), and A→B→A
   loops are equally possible. `_subtree_tag_ids` terminates
   thanks to its visited set (views.py:280-290), so delete won't
   hang, but the hierarchy is silently corrupted.
2. **`perform_create` guest fallback is dead and latently
   harmful.** `create` requires `IsAuthenticated` +
   `GuestReadOnly` (views.py:134), so the `else` branch
   (views.py:185-200) is unreachable; its final
   `serializer.save()` would create a `user=NULL` tag
   (models.py:33-40 allows null) that no queryset can ever see.
3. **Delete guard is not atomic.** `destroy` counts notes then
   deletes outside any transaction (views.py:247-270); a note
   created in the gap is silently cascade-deleted despite the
   409 guard existing to prevent exactly that.
4. **Duplicate top-level names are validated but not
   DB-enforced.** `unique_together = (user, name, parent_tag)`
   (models.py:65) can't enforce uniqueness when `parent_tag` is
   NULL (Postgres treats NULLs as distinct); the real
   `unique_top_level_tag_name` constraint was dropped in
   migration 0003. The auto `UniqueTogetherValidator` still
   returns 400 for API callers, but concurrent creates or direct
   ORM writes can persist duplicates.

### Security & auth

5. **`_subtree_tag_ids` and the cascade ignore ownership.** The
   descendant walk at views.py:283-287 queries
   `Tag.objects.filter(parent_tag_id__in=frontier)` with no user
   filter, and `instance.delete()` CASCADEs across owners
   (views.py:302). Cross-user parent links cannot be created via
   the API (serializers.py:94-96 scopes the queryset) but are
   possible via admin/shell; if one exists, `note_count` mixes
   in a stranger's notes (false 409) or their tag+notes are
   destroyed.
6. **`perform_update`/`perform_destroy` lack the ownership
   check** their docstrings promise (views.py:202-213,292-306;
   contrast NoteViewSet at views.py:388-431). Safe today only
   because `get_queryset` scopes to `user=user`; if that ever
   widens (e.g. shared tags), `serializer.save()` would apply
   the HiddenField `user` default and silently *reassign*
   ownership to the caller.
7. **`parent_tag` field's class-level queryset is
   `Tag.objects.all()`** (serializers.py:62-67). It is rescoped
   in `__init__` only when a request is in context; any
   serializer instantiated without one (scripts, future code
   paths) accepts any user's tag as parent. Fail-closed default
   should be `none()`.
8. Scoping itself is correct: user A cannot list/retrieve/
   update/delete user B's tags (404, views.py:152-160);
   anonymous list is `[]`; anonymous retrieve requires a public
   note (views.py:165-169); `guest` writes → 403
   (permissions.py:27-32).

### Performance

9. **`get_queryset` runs `queryset.count()` purely for a debug
   log** (views.py:156-159) — one extra `COUNT(*)` on every
   request including retrieve/update/destroy.
10. No N+1 in list: `parent_tag` serializes as pk, so no join is
    needed. `_subtree_tag_ids` BFS costs one query per tree
    level — fine for realistic depths. `destroy`'s
    `tag_id__in` count and the anonymous `notes__public` join
    both ride existing FK indexes; `Note.public` is unindexed
    but the join is tag-driven, so impact is negligible.
11. Tags list is unpaginated (see contract); acceptable at
    current scale, worth revisiting if users accumulate
    thousands of tags.

### Code quality & consistency

12. Deviates from the project convention "enforce ownership in
    `perform_update`/`perform_destroy` via `PermissionDenied`"
    (AGENTS.md): TagViewSet relies solely on queryset scoping
    while NoteViewSet does both — the docstrings at
    views.py:204-205 and 294-295 claim checks that don't exist.
13. Dead code: `perform_create`'s unauthenticated/guest branch
    (views.py:185-200) and the anonymous branch of
    `TagSerializer.__init__` (serializers.py:97-105) are
    unreachable through the API.
14. `name` accepts whitespace-only strings; no strip/normalize
    (serializers.py:71). Duplicate-tag IntegrityError on a race
    would surface as 500, not 400 — no `IntegrityError` handling
    on create/update (unlike the careful
    `_is_tag_position_conflict` handling for notes,
    views.py:95-112).
15. Verbose per-request `logger.info` in `get_queryset`/
    `perform_*` plus eager f-strings — noisy but consistent with
    the file's style.

### Test coverage

Covered well:
- Anonymous matrix: public-note retrieve 200, private-only 404,
  empty list, write rejections (tests.py:1652-1747).
- Guest read allowed / all writes 403 incl. delete-cascade block
  (tests.py:2945-2987).
- Delete guard: 409, `delete_notes=false`, cascade on `true`,
  descendant notes counted (tests.py:3078-3165).
- Middleware keeps anonymous GETs anonymous (tests.py:3208-3212).
- Serializer-level create (tests.py:65-83).

Gaps:
- No authenticated cross-user tag test (A GET/PATCH/DELETE B's
  tag → 404) — the core scoping guarantee is untested.
- No duplicate-name → 400 test; no `parent_tag` pointing at
  another user's tag → 400 test.
- No update happy path (rename own tag → 200).
- No self-parent/cycle test (would currently pass silently —
  see finding 1).

## Refactoring recommendations

1. **P0 — None.** No actively exploitable hole found under
   current call paths.
2. **P1 — Add explicit ownership checks in `perform_update`/
   `perform_destroy`.** Mirror NoteViewSet
   (views.py:388-431): raise `PermissionDenied` when
   `instance.user != request.user`. Why: docstrings already
   promise it, it's the stated convention, and it removes the
   silent-ownership-reassignment latent bug (finding 6).
   Effort: S.
3. **P1 — Scope the delete subtree to the owner.** Pass
   `user=instance.user` into `_subtree_tag_ids`'s
   `Tag.objects.filter(...)` (views.py:284-286) and 409 if any
   descendant is foreign-owned before cascading. Why: prevents
   cross-user data destruction if a bad parent link ever exists
   (finding 5). Effort: S.
4. **P1 — Add `validate_parent_tag`.** Reject `value ==
   self.instance` and any value that is already a descendant
   (reuse `_subtree_tag_ids`). Why: cycle corruption is
   currently accepted input (finding 1). Effort: S.
5. **P2 — Remove dead code.** Delete `perform_create`'s
   unreachable guest/`save()` fallback (views.py:185-200) and
   simplify to `serializer.save(user=request.user)`; default the
   `parent_tag` class-level queryset to `Tag.objects.none()`
   (serializers.py:62-67). Why: dead paths hide a NULL-user
   orphan bug (findings 2, 7). Effort: S.
6. **P2 — Drop the logging `count()`** in `get_queryset`
   (views.py:156-159) or downgrade to lazy `logger.debug` with
   `%s` args. Why: one useless `COUNT(*)` per request (finding
   9). Effort: S.
7. **P2 — Restore DB-level top-level name uniqueness.** Add a
   conditional `UniqueConstraint(fields=['user','name'],
   condition=Q(parent_tag__isnull=True))` to `Tag.Meta`
   (requires a new migration — have a human run it) and map
   `IntegrityError` → 400/409 in the view. Why: closes the
   race/ORM duplicate gap left by migration 0003 (finding 4).
   Effort: M.
8. **P2 — Backfill tests.** Authenticated cross-user 404s,
   duplicate-name 400, foreign `parent_tag` 400, self-parent
   400, rename happy path. Why: the scoping guarantees this
   resource depends on have no regression net. Effort: S.
