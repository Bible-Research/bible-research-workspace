# POST /api/v1/users/register/ (register_user) — code review

## Summary

`register_user` (users/views.py:61) is a small, mostly correct FBV:
passwords are hashed via `create_user`, the field whitelist prevents
mass-assignment, and the 201 contract matches the docs and frontend.
However, it accepts weak passwords despite configured Django
validators, has no rate limiting, never reconciles with the device
user that `DeviceAndCountryMiddleware` creates on the very same
request, and has zero backend tests. Verdict: **fair** — usable but
with real security and lifecycle gaps.

## Endpoint contract

- Route: `users/urls.py:5`, included at `api/v1/users/`
  (`bible_research/urls.py:17`). Decorated `@api_view(['POST'])`
  + `@permission_classes([AllowAny])` (users/views.py:59-60).
- Request JSON: `username` (req), `email` (opt, blank allowed),
  `password` (req, min_length=8), `password_confirm` (req)
  (users/serializers.py:13-46).
- 201: `{user: {id, username, email, date_joined}, token, message}`
  (users/views.py:84-92); token via `Token.objects.get_or_create`
  (users/views.py:78).
- 400: DRF error dict `{field: [msg]}` on validation failure,
  including `password_confirm` mismatch (serializers.py:59-69).
- Non-POST -> 405; duplicate-registration race -> unhandled
  `IntegrityError` -> 500 (no catch around `create_user`,
  serializers.py:81-85).

## Findings

### Bugs / correctness

1. **Every anonymous registration first creates a junk device user.**
   `DeviceAndCountryMiddleware.auto_authenticate` runs for any
   non-GET, non-admin path when the session has no user
   (middleware.py:35-42); `POST /api/v1/users/register/` qualifies.
   So each attempt does a `User.objects.get/create_user` for
   `"{lang} {device}"` (middleware.py:132-149) and a session `login()`
   (middleware.py:152-157) *before* the view runs. The view then
   ignores `request.user` and creates a second, unrelated account —
   no upgrade/linking, and the response sets a session cookie bound
   to the *device* user while the returned token belongs to the
   *registered* user (split identity via
   `CSRFExemptSessionAuthentication`, authentication.py:19-31).
   Notes written before signup stay on the orphaned device account.
2. **Concurrent duplicate registration -> 500.** UniqueValidator
   checks then `create_user` (serializers.py:17-31, 81-85); the
   DB unique constraint racing raises `IntegrityError`, never
   mapped to a 400/409.
3. **Case-variant usernames allowed.** `UniqueValidator` uses exact
   match (serializers.py:26-29) while login is case-insensitive
   (backends.py:38-57). Registering `alice` when `Alice` exists
   succeeds; a later `ALICE` login falls back to `matches.first()`
   (backends.py:51-57) — nondeterministic which account a
   case-variant login hits. Tests (users/tests.py:89-103) encode
   tolerance rather than prevention; registration is the right
   place to prevent it.
4. Email uniqueness is also case-sensitive (serializers.py:17-21);
   no normalization before the uniqueness check.

### Security & auth

1. **Configured password validators are dead code.** Settings define
   `AUTH_PASSWORD_VALIDATORS` — similarity, min-length, common,
   numeric (settings.py:313-338) — but the serializer never calls
   `django.contrib.auth.password_validation.validate_password`.
   Only `min_length=8` is enforced (serializers.py:36): `12345678`
   and `password1` are accepted.
2. **No throttling anywhere.** `REST_FRAMEWORK` (settings.py:368-382)
   has no `DEFAULT_THROTTLE_CLASSES`/rates and no view sets
   `throttle_classes` — so the FBV doesn't "skip" controls other
   views have; nothing is throttled. Still, an unauthenticated
   account-creation endpoint is the worst place to be unthrottled:
   unlimited registrations, and (per bug 1) each call can also
   mint a device user. Same gap applies to `POST /api/token/`
   (bible_research/urls.py:18) — credential brute force.
3. **User enumeration.** Distinct messages reveal whether a
   username/email exists (serializers.py:19, 28). Standard for
   registration UX, but with no throttling it is a cheap bulk
   oracle.
4. Mass-assignment is safe: `Meta.fields` is an explicit whitelist
   and `create()` passes only username/email/password
   (serializers.py:48-86) — `is_staff`/`is_superuser` cannot be
   injected. Passwords are hashed (`create_user`,
   serializers.py:81) and never logged (views.py:68-70 logs only
   the username).
5. `AllowAny` (views.py:60) is correct and intentional — it
   overrides the default `IsAuthenticated` + `GuestReadOnly`
   (settings.py:377-380); `@api_view` still runs inside APIView so
   defaults would otherwise apply.

### Performance

1. Registration costs extra writes unrelated to its purpose:
   device-user `get`/`create` + session row from `login()`
   (middleware.py:137-157). An attacker rotating User-Agent /
   Accept-Language headers creates unbounded device-user rows
   without ever completing registration — DB growth on an
   unauthenticated path.
2. `Token.objects.get_or_create` (views.py:78) is fine — the user
   is always fresh, so it is one extra SELECT; `create` would do.

### Code quality & consistency

1. **FBV diverges from convention.** Every other view is a ViewSet
   or APIView (bible/views.py:156,348,475,607,902;
   annotations/views.py:115,309,1293,1492,...); `users/` is the
   only `@api_view` usage (views.py:59,115). Works, but inconsistent
   and easier to mis-decorate.
2. Manual `is_valid()`/`Response(400)` (views.py:75-101) rather
   than `raise_exception=True` — stylistic, acceptable.
3. The `users` logger is absent from `LOGGING` (settings.py:450-465
   configure only `django`, `bible`, `annotations`): `logger.info`
   calls at views.py:68-82 are dropped (no root handler;
   WARNING+ hits `lastResort` stderr unformatted). Observability
   of registration events is weaker than the code suggests.
4. `extend_schema` docs (views.py:24-58) match behavior; the
   "Password must be at least 8 characters" claim is accurate but
   understates that no other policy applies.

### Test coverage

- **None for this endpoint.** `users/tests.py` (117 lines) contains
  only `ObtainAuthTokenTestCase` for `/api/token/` — no tests for
  `register_user` or `UserRegistrationSerializer`: nothing for the
  201 contract, token issuance, password hashing, duplicate
  username/email, `password_confirm` mismatch, min length, or the
  middleware side-effects above. Frontend register tests
  (reactive-bible `authStore.register.test.tsx`, `RegisterPage.test.tsx`)
  use MSW mocks and exercise no backend code.

## Refactoring recommendations

1. **P0 — Enforce `validate_password` in the serializer**
   (serializers.py `validate()`): call
   `django.contrib.auth.password_validation.validate_password`
   with the pending user so the configured similarity/common/
   numeric validators actually run. Why: `12345678` passes today.
   Effort: S.
2. **P0 — Add throttling**: set `AnonRateThrottle` (or a scoped
   throttle) for `register` and `/api/token/` via
   `DEFAULT_THROTTLE_CLASSES`/`DEFAULT_THROTTLE_RATES` or
   `@throttle_classes`. Why: unbounded account creation,
   enumeration, and credential brute force. Effort: S-M.
3. **P0 — Exclude the register path from device auto-provisioning**
   in middleware (middleware.py:35-42) or skip `login()` there.
   Why: every signup currently mints a junk device user and a
   session for the wrong account. Effort: S.
4. **P1 — Define the device-user → registered-user story**: either
   upgrade `request.user` when it is an auto-provisioned account
   (set username/password on it so prior notes carry over) or
   explicitly document the split. Why: pre-registration data is
   silently orphaned today. Effort: M (product decision + code).
5. **P1 — Enforce case-insensitive username uniqueness**
   (`UniqueValidator(queryset=..., lookup='iexact')`) or normalize
   case at registration. Why: login is case-insensitive; variant
   accounts make auth nondeterministic (backends.py:47-57).
   Effort: S.
6. **P1 — Catch `IntegrityError` around `create_user`** (or wrap in
   `transaction.atomic` + re-validate) and return 400/409. Why:
   race between UniqueValidator and insert yields a 500.
   Effort: S.
7. **P2 — Normalize email** (strip/lowercase) before uniqueness;
   optionally soften the "already exists" messages once throttling
   lands. Why: reduces duplicate accounts and enumeration signal.
   Effort: S.
8. **P2 — Convert to `APIView`** (e.g. `RegisterView`) and add a
   `users` logger to `LOGGING`. Why: project convention and lost
   INFO logs. Effort: S.
9. **P2 — Add backend tests** for the 201 contract, token
   issuance, hashing, duplicate/case-variant usernames, password
   mismatch, min length, and middleware interaction. Why: zero
   coverage on an auth-critical path. Effort: M.
