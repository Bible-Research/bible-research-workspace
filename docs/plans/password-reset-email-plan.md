# Plan: Optional email at registration + forgot-password reset

Status: plan (issue: Bible-Research/bible-research-workspace)
Scope: `bible_research` (backend) + `reactive-bible` (frontend)

## User request

- Registration form should let users add an email address (optional).
- Login form should have "Forgot password": user enters their email
  and gets a password reset by email.

## Investigation — what already exists

### Backend (`bible_research`)

- `users/serializers.py` — `UserRegistrationSerializer` ALREADY
  accepts `email`: `EmailField(required=False, allow_blank=True)`
  with a `UniqueValidator`. No change needed for registration.
- `users/views.py` — `register_user` returns
  `{user: {id, username, email, date_joined}, token, message}`.
- `users/urls.py` — only `register/` and `me/`. No reset endpoints.
- `bible_research/urls.py` — users app mounted at `/api/v1/users/`;
  token login at `POST /api/token/` (DRF `obtain_auth_token`).
- NO email infrastructure at all: no `EMAIL_*` settings, no
  `send_mail` usage, no SMTP/transactional-mail dependency.
- Secrets/config pattern: GCP prod reads Secret Manager via
  `bible_research/gcp.py::get_secret`; local dev reads
  `config.yaml` (template: `config_template.yaml`).
- GOTCHA — `bible_research/middleware.py`
  (`DeviceAndCountryMiddleware`) auto-provisions users with
  placeholder emails `"{username}@example.com"` (which contain
  spaces — not real addresses) and unusable passwords. Password
  reset lookups MUST exclude these, same rules as Django's
  `PasswordResetForm.get_users()`: `email__iexact`, `is_active`,
  `has_usable_password()`.
- Error contract for the API: failures return `{"error": "..."}`.
- `users/tests.py` is an empty stub; pytest + in-memory SQLite,
  no `config.yaml` needed (`conftest.py`).

### Frontend (`reactive-bible`)

- `RegisterPage.tsx` ALREADY has "Email (optional)" field, and
  `authStore.register(username, password, passwordConfirm,
  email?)` ALREADY posts it. Registration half is DONE.
- `LoginPage.tsx` has no "Forgot password" link.
- `src/routes/index.tsx` — public routes `/login`, `/register`;
  new auth pages register here.
- Auth calls are raw `fetch` to `API_BASE_URL` (`src/config.ts`)
  inside `authStore.tsx` — not in `src/api.tsx`. Follow that
  pattern for the two new calls.
- UI: Mantine v6 (`TextInput`, `PasswordInput`, `Alert`,
  `Anchor`), notifications via `@mantine/notifications`.
- Tests: vitest + RTL + MSW (`src/mocks/`).
- Capacitor Android app exists; emailed links open the web URL —
  App Links are not configured, acceptable for v1 (reset happens
  in browser).

## Design decisions

- NEVER email the password itself or a generated plaintext
  password. Standard flow: email a single-use, time-limited
  reset LINK; the confirm endpoint sets the new password.
  Uses `django.contrib.auth.tokens.default_token_generator`
  + `urlsafe_base64_encode` — no new model/table required
  (no migration needed).
- Token lifetime: `PASSWORD_RESET_TIMEOUT` (Django default 3
  days; consider setting 24h explicitly).
- Reset request ALWAYS returns 200 + generic message regardless
  of whether the email exists (prevents account enumeration).
- On successful reset: `set_password()` + delete the user's DRF
  `Token` rows so old sessions are invalidated.
- Since email is optional, users who never set one cannot reset —
  expected; surface this honestly in UI copy ("only works if you
  added an email to your account").

## Backend plan (`bible_research` repo, feature branch)

1. `users/serializers.py`
   - `PasswordResetRequestSerializer`: `email` field; helper
     `get_users()` mirroring `PasswordResetForm.get_users`
     (iexact email, `is_active=True`, `has_usable_password()`,
     excludes `@example.com` placeholders implicitly via
     `has_usable_password`).
   - `PasswordResetConfirmSerializer`: `uid`, `token`,
     `new_password` (min_length=8), `new_password_confirm`;
     `validate()` decodes uid (`urlsafe_base64_decode`),
     checks `default_token_generator.check_token(user, token)`
     and password match.
2. `users/views.py`
   - `password_reset_request` (POST, `AllowAny`): for each
     matching user build
     `{FRONTEND_BASE_URL}/reset-password/{uidb64}/{token}`,
     `send_mail` (subject + plain-text body), log outcome;
     always `200 {"message": ...}`.
   - `password_reset_confirm` (POST, `AllowAny`): on valid
     serializer `user.set_password(...)`, delete user's DRF
     tokens, `200 {"message"}`; invalid → `400 {"error": ...}`.
   - drf-spectacular `@extend_schema` docs matching existing
     style.
3. `users/urls.py`: add `password-reset/` and
   `password-reset-confirm/`.
4. `bible_research/settings.py`
   - `FRONTEND_BASE_URL` per env (prod: Vercel URL; local:
     `http://localhost:5173`; config.yaml-overridable).
   - `EMAIL_BACKEND`: default console backend locally
     (overrideable via config.yaml); GCP branch reads SMTP or
     API-key settings from Secret Manager.
   - `DEFAULT_FROM_EMAIL` setting.
   - DECISION NEEDED (email provider on App Engine): port 25 is
     blocked on GAE; use SMTP over 587 (e.g. Gmail/SendGrid
     SMTP) or a transactional HTTP API (SendGrid/Mailgun).
     Simplest portable: SMTP 587 + `django.core.mail`.
   - Update `config_template.yaml` with the new keys.
5. `users/tests.py` — new tests:
   - request: real email → 1 message in `mail.outbox` containing
     uid/token; unknown email → 200 + 0 emails; placeholder
     `@example.com` user → ignored; malformed email → 400.
   - confirm: valid → password changed + tokens deleted; bad
     token/uid → 400; confirm mismatch → 400; short password →
     400. Use `override_settings(EMAIL_BACKEND=locmem)`.
6. Update `DEVELOPER_GUIDE.md` (endpoint table + notes).

## Frontend plan (`reactive-bible` repo, feature branch)

1. New `src/components/ForgotPasswordPage.tsx` — email input,
   submit, generic success screen ("If an account exists..., "
   incl. note that reset needs a saved email). Mantine v6,
   same `Container`/`Paper` layout as `LoginPage`.
2. New `src/components/ResetPasswordPage.tsx` — reads
   `uid`/`token` via `useParams`, password + confirm inputs,
   client-side match/length checks (mirror `RegisterPage`),
   submit → success notification + `navigate('/login')`;
   invalid/expired link state with link back to
   `/forgot-password`.
3. `LoginPage.tsx` — "Forgot password?" `Anchor`/`Link` under
   the password field → `/forgot-password`.
4. `src/routes/index.tsx` — add public routes
   `/forgot-password` and `/reset-password/:uid/:token`.
5. API helpers `requestPasswordReset(email)` and
   `confirmPasswordReset(uid, token, pw, pwConfirm)` — plain
   `fetch` + `API_BASE_URL`, consistent with `authStore.tsx`
   (they carry no auth state, so export as standalone helpers;
   keep them in `authStore.tsx` or a small module — follow the
   existing file layout).
6. MSW handlers + tests for both pages (happy path, unknown
   email still shows success, bad token shows error).
7. Update `DEVELOPER_GUIDE.md`.

## API contract (to implement)

- `POST /api/v1/users/password-reset/`
  body `{email}` → always `200 {"message": "..."}`
- `POST /api/v1/users/password-reset-confirm/`
  body `{uid, token, new_password, new_password_confirm}`
  → `200 {"message": "..."}` or `400 {"error": "..."}`

## Acceptance criteria

- Register with and without email still works (already true).
- "Forgot password" on `/login` → enter email → email with
  working reset link received (console backend locally).
- Link opens `/reset-password/:uid/:token` → set new password →
  log in with new password; old password fails; old auth token
  rejected.
- Unknown emails get identical response (no enumeration).
- Auto-provisioned device users can never trigger emails.

## Out of scope / follow-ups

- Email verification at signup; login-by-email.
- Rate limiting the reset endpoint (recommend follow-up issue).
- Android App Links for reset links; HTML email template.
