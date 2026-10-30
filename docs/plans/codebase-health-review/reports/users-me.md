# users/me/ + api/token/ — code review

## Summary

Both endpoints are thin wrappers around correct-ish code, but the
auth surface is undermined by `DeviceAndCountryMiddleware`: `/me/`
effectively never returns 401 because anonymous callers are
silently provisioned a writable per-device (or shared `guest`)
account first, and the composed device username can collide with a
registered username, yielding a login bypass. `/api/token/` is
stock DRF `obtain_auth_token` with zero throttling — unlimited
credential guessing. Verdict: **poor**.

## Endpoint contract

**`GET /api/v1/users/me/`** → `current_user`
(users/views.py:115-125, users/urls.py:6)

- Auth: `Authorization: Token|Bearer <key>` or session
  (settings.py:369-376). Params: none.
- 200 → `{id: int, username: str, email: str,
  date_joined: iso8601}` via `UserSerializer`
  (users/serializers.py:89-97).
- Intended 401 `{"detail": ...}` for anonymous callers — in
  practice unreachable: the path is not in
  `PUBLIC_READ_PREFIXES` (middleware.py:16-20), so anonymous
  requests are auto-authenticated before the view runs
  (middleware.py:35-42) and get 200 for a freshly created device
  user or the shared `guest` account.
- 500 possible if the guest fallback itself raises inside
  `auto_authenticate` (middleware.py:160-182 is unguarded).

**`POST /api/token/`** → DRF `obtain_auth_token`
(bible_research/urls.py:18)

- Body `{username, password}`; accepts form, multipart, JSON
  (authtoken/views.py:11). Username matching is
  case-insensitive (users/backends.py:38-40).
- 200 → `{token: "<40-char hex>"}` (created if absent).
- 400 → `{non_field_errors: ["Unable to log in with provided
  credentials."]}` for bad credentials or `is_active=False`;
  400 field errors for missing fields.
- No 429 — `throttle_classes = ()` on the view
  (venv .../rest_framework/authtoken/views.py:9) and no
  `DEFAULT_THROTTLE_*` in settings.py:368-382.

## Findings

### Bugs / correctness

1. **`/me/` can never 401 and mints users.** `IsAuthenticated`
   (users/views.py:116) is dead code: any anonymous request —
   including HEAD/OPTIONS CORS preflights — is provisioned a user
   first (middleware.py:30-42,125-159). Undocumented behavior;
   DEVELOPER_GUIDE.md:316 implies a real auth requirement.
2. **Shared writable accounts.** With no `Accept-Language`/
   `User-Agent`, the composed username is `"None Unknown Device"`
   (middleware.py:51,104,132) — one account shared by every
   header-less client, and `GuestReadOnly` only blocks the
   literal `guest` username (permissions.py:32), so this shared
   account can write notes/tags cross-client.
3. **Fake emails stored and returned.** Device/guest users get
   `email="<username>@example.com"` (middleware.py:145,170);
   `/me/` returns these indistinguishable from real emails.
4. **Guest fallback can 500.** A DB error inside the `except`
   block (middleware.py:163-174) propagates uncaught.
5. **Session pollution.** `login()` at middleware.py:153-157
   runs even when the request carries a valid Bearer token (the
   middleware sees anonymous before DRF auth runs), so every API
   call also creates a device-user session row and cookie.

### Security & auth

6. **Login bypass via username collision.** The device username
   is `f"{primary_language} {device}"` from attacker-controlled
   headers (middleware.py:132); `User.objects.get` then logs the
   request in as that user with no credentials
   (middleware.py:138,152). Any registered username containing a
   space (allowed — `username` is an unrestricted `CharField`,
   users/serializers.py:23-31) is impersonatable by crafting
   `Accept-Language` + `User-Agent`. `/me/` then confirms the
   stolen identity.
7. **`/api/token/` is unthrottled.** DRF's view explicitly sets
   `throttle_classes = ()`; the project configures no
   `DEFAULT_THROTTLE_CLASSES`/`_RATES` (settings.py:368-382), no
   django-axes, no middleware rate limiting. Unlimited online
   brute force; each attempt also does a user lookup + possible
   `create_user` + session write via the middleware, amplifying
   the DB cost per guess.
8. **Enumeration.** Token failure responses are uniform 400s
   (good), and `CaseInsensitiveAuthBackend` runs a dummy hash
   for unknown users (users/backends.py:41-46) — partial timing
   equalization; the extra `iexact` queries in the
   `MultipleObjectsReturned` path (backends.py:47-57) leave a
   residual oracle. Adjacent: `register/` leaks username/email
   existence via `UniqueValidator` 400s
   (users/serializers.py:16-31).
9. **`/me/` leakage is modest but real.** `id` is a sequential
   integer (user-count oracle) and `date_joined`/`username`
   reveal device metadata; no `is_staff`/`last_login` exposure —
   the field list itself is fine. The real gap: nothing
   distinguishes guest / device / registered accounts, so
   clients must string-match `username == 'guest'`.
10. **Tokens are permanent.** No expiry or rotation on
    `authtoken` tokens; a leaked token is valid forever.

### Performance

- Every non-public request pays ≥1 `SELECT` + `login()` session
  write, plus `INSERT` on first sight of each UA/language combo
  (middleware.py:138-157) — this runs on `/api/token/` too.
- `print()` calls on every request (middleware.py:44,71,117,158,
  161,182) spam stdout in production.

### Code quality & consistency

- FBVs (`@api_view`) here vs. project convention of APIView /
  ViewSets (bible/views.py `BiblePassageView`, annotations
  viewsets; AGENTS.md "APIView for custom endpoints").
- `/api/token/` sits outside the `/api/v1/` prefix — the only
  unversioned API route (bible_research/urls.py:18).
- `current_user` logs username at INFO per call
  (users/views.py:121-123) — noise.
- `UserSerializer` declares `read_only_fields`
  (users/serializers.py:97) though only used for reads —
  harmless but suggests a write path that doesn't exist.
- Imports in users/views.py:3-8 mix third-party and django
  ordering (rest_framework.authtoken after decorators, django
  import last) — deviates from the stated import convention.

### Test coverage

- users/tests.py covers only `/api/token/` — 5 regression tests
  for case-insensitive login and inactive users (issue #53,
  users/tests.py:18-117). Solid for that narrow path.
- **Zero coverage** for `/me/` and `register/`; no test asserts
  what `/me/` returns for guest or device users.
- conftest.py:41-48 omits `DeviceAndCountryMiddleware` and sets
  no `REST_FRAMEWORK` — pytest cannot reproduce production
  auto-provisioning or default-permission behavior (a workaround
  already exists at bible/tests/test_public_read_permissions.py:
  9-17). No throttling tests (nothing to test yet).

## Refactoring recommendations

1. **[P0] Stop auto-provisioning on auth/identity routes.**
   Exempt `/api/token/`, `/api/v1/users/`, schema/docs, and
   HEAD/OPTIONS preflights from `auto_authenticate`
   (middleware.py:30-42). *Why:* restores real 401 semantics for
   `/me/` and stops user creation on preflights/login attempts.
   *Effort:* S.
2. **[P0] Prevent device-username impersonation.** Generate
   device usernames from an unguessable, reserved namespace
   (e.g. `device_<hash>` or UUID stored per header fingerprint)
   instead of `f"{lang} {device}"`, and reject registration of
   usernames that match the reserved pattern or contain spaces.
   *Why:* closes the silent login-as-victim path
   (middleware.py:132-152). *Effort:* M.
3. **[P0] Rate-limit `/api/token/`.** Subclass
   `ObtainAuthToken` with a scoped throttle (e.g. `AnonRateThrottle`
   at ~5/min per IP plus per-username) and wire
   `DEFAULT_THROTTLE_RATES`. *Why:* unlimited brute force today;
   also reduces per-attempt DB writes. *Effort:* S.
4. **[P1] Make account kind explicit in `/me/`.** Add a computed
   field (e.g. `account_kind: guest|device|registered`) to
   `UserSerializer` instead of writing fake `@example.com`
   emails; document it. *Why:* lets clients and reviewers
   distinguish identities without username heuristics. *Effort:* S.
5. **[P1] Add tests for `/me/` and the middleware contract.**
   Cover real user, guest, device-provisioned, and truly
   anonymous callers; align conftest's `MIDDLEWARE`/
   `REST_FRAMEWORK` with prod or override per test. Add a
   throttle test once #3 lands. *Effort:* M.
6. **[P2] Token hygiene.** Add expiry/rotation (django-rest-
   knox or `Token.created` reaping) and move the endpoint under
   `/api/v1/` for versioning consistency (keep the old route
   during transition). *Effort:* M.
7. **[P2] Clean up logging & conventions.** Replace middleware
   `print()`s with `logger.debug`, drop the per-call INFO log in
   `current_user`, convert users' FBVs to APIViews per project
   convention, fix import ordering. *Effort:* S.
