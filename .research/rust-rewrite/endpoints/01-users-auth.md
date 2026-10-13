# 01 — Users & token auth (register, me, obtain_auth_token)

Inventory items 17, 18, 19. Django source: `users/views.py`,
`users/serializers.py`, `users/urls.py`, `users/backends.py`,
`bible_research/urls.py:18`, `bible_research/middleware.py`,
`bible_research/authentication.py`, and DRF `rest_framework.
authtoken` (`views.py`, `serializers.py`, `models.py` — venv
`site-packages`). `users/tests.py` is an EMPTY stub (3 lines) —
no Django test cases to mirror; the test plan is derived from
the code trace + DRF semantics. `users/models.py` declares no
model — the app uses `django.contrib.auth.models.User` (table
`auth_user`) and DRF's `Token` (table `authtoken_token`).

Status: PLAN. Depends on Phase 0: `core::auth`
(`Principal`/`AuthUser`, token lookup, provisioning), `core::
errors` (`ApiError`), `AppState.pool`. Base §4.2 covers the
extractor design and the `/api/token/` backend choice; this
plan fills in endpoint-level detail.

Deployed Django is **6.1.1** (`requirements.txt`,
commit `b499e27`); the repo venv is a stale 4.2.6 — trust
requirements/git, not the venv, for version-sensitive details
(PBKDF2 iteration count, §2/§5).

## 1. Endpoint contracts

### `POST /api/v1/users/register/` — `register_user` (AllowAny)

Body (JSON; DRF default parsers also accept form + multipart):

| field              | type   | rules                                  |
|--------------------|--------|----------------------------------------|
| `username`         | string | required, non-blank, trimmed, unique   |
| `email`            | string | optional; `""`/absent skips all checks |
|                    |        | trimmed; valid email; unique (exact)   |
| `password`         | string | required, non-blank, TRIMMED, min 8    |
| `password_confirm` | string | required, non-blank, trimmed           |

201 body (key order `user`, `token`, `message`):

```json
{
  "user": {
    "id": 3,
    "username": "newuser",
    "email": "user@example.com",
    "date_joined": "2026-03-01T07:11:10.123456Z"
  },
  "token": "a1b2…<40 lowercase hex>",
  "message": "User registered successfully"
}
```

`id` is the `auth_user` bigint PK as a JSON number; `date_joined`
is DRF ISO-8601 — `Z` suffix, 6-digit micros only when nonzero.

400 = DRF field-map `{"<field>": ["<msg>", …]}`, keys in field
order (`username`, `email`, `password`, `password_confirm`).
All field errors are collected together; the object-level
password-match check only runs when field validation is clean.

| case                         | body                                       |
|------------------------------|--------------------------------------------|
| field missing                | `{"f": ["This field is required."]}`       |
| explicit `null`              | `{"f": ["This field may not be null."]}`   |
| `""` (see §3 whitespace      | `{"f": ["This field may not be blank."]}`  |
|  quirk)                      |                                            |
| `password` < 8 after trim    | `{"password": ["Ensure this field has at   |
|                              |   least 8 characters."]}`                  |
| username taken (exact)       | `{"username": ["A user with this username  |
|                              |   already exists."]}`                      |
| non-blank email taken        | `{"email": ["A user with this email        |
|                              |   already exists."]}`                      |
| bad email shape              | `{"email": ["Enter a valid email           |
|                              |   address."]}`                             |
| bool/array/object for a char | `{"f": ["Not a valid string."]}`           |
| field (email:                | (`{"email": ["Enter a valid email          |
|                              |   address."]}`)                            |
| password mismatch            | `{"password_confirm": ["Password fields    |
|                              |   didn't match."]}`                        |

Auth: `AllowAny`. In prod the device middleware still
auto-provisions a device/guest user before the view runs — a
harmless side-effect (Rust drops the session write per §4.2).
A valid `Authorization` header is accepted but ignored; register
always creates a NEW user.

GET/PUT/etc. → 405 `{"detail": "Method \"GET\" not allowed."}`.

### `GET /api/v1/users/me/` — `current_user` (IsAuthenticated)

200 body = `UserSerializer` — exactly:

```json
{"id": 3, "username": "…", "email": "…",
 "date_joined": "…Z"}
```

Auth: `IsAuthenticated` → `AuthUser` extractor. Under prod
`AUTO_PROVISION_USERS`, device and `guest` users ARE
authenticated (base §4.2) — a guest `/me` returns the `guest`
row `{id, "guest", "guest@example.com", …}` with 200. Anonymous
only happens with provisioning off (tests):

- no credentials → 401
  `{"detail": "Authentication credentials were not provided."}`
- bad token → 401 `{"detail": "Invalid token."}` +
  `WWW-Authenticate: Token` (header value = first auth class's
  scheme; DRF test defaults would emit `Basic` — see §5)

POST/PUT/DELETE → 405 `{"detail": "Method \"POST\" not
allowed."}`.

### `POST /api/token/` — `obtain_auth_token` (outside `/api/v1/`)

Mounted at root in `bible_research/urls.py:18`; the Rust router
must NOT nest it under `/api/v1`. `permission_classes = ()` —
fully public; provisioning still runs in prod (side-effect
only). Parsers: JSON, `application/x-www-form-urlencoded`,
multipart.

Body: `{"username": "…", "password": "…"}` — `username` is a
DRF CharField (whitespace TRIMMED); `password` has
`trim_whitespace=False` (leading/trailing spaces are PART of
the credential).

200 `{"token": "<40 lowercase hex>"}` — `get_or_create`: an
existing token is returned unchanged; a new one is
`secrets.token_hex(20)` (40 hex chars from 20 random bytes).
Tokens are stored plaintext and never rotated/expired.

400 bodies (verbatim):

- field missing → `{"username": ["This field is required."], …}`
- field `""` → `{"<f>": ["This field may not be blank."]}`
- fields present but falsy after processing — e.g. `username:
  "   "` (trims to `""`) →
  `{"non_field_errors": ["Must include \"username\" and
  \"password\"."]}`
- bad credentials →
  `{"non_field_errors": ["Unable to log in with provided
  credentials."]}` — identical for unknown user and wrong
  password (Django blunts timing by hashing anyway on a miss)

Username lookup is CASE-INSENSITIVE (`username__iexact`) via
`CaseInsensitiveAuthBackend` — the FIRST entry in
`AUTHENTICATION_BACKENDS` (settings.py:286-289).

Quirk to reproduce: that backend checks only `check_password`
and returns the user — it never calls `user_can_authenticate`,
so an `is_active=false` user with the right password still gets
a token (the DRF comment assuming inactive ⇒ rejection applies
only to the plain `ModelBackend`, which runs second).

GET → 405 `{"detail": "Method \"GET\" not allowed."}`.

## 2. Current Django behavior trace

### register_user (users/views.py:61-101)

1. `UserRegistrationSerializer(data=…)` + `is_valid()`
   (serializers.py:8-86). Both `username` and `email` are
   DECLARED fields, so no model validators are inherited — no
   `UnicodeUsernameValidator` regex, no `max_length` on either.
   Per-field pipeline (`fields.py` `run_validation`):
   `required`/`null` checks → `''` vs `allow_blank` →
   `to_internal_value` (type check; `str()` for int/float, bool
   + containers → 'invalid'; then `.strip()` because
   `trim_whitespace` defaults True) → validators in order:
   `min_length`/`max_length` (password only: min 8),
   `ProhibitNullCharactersValidator` (`\x00` → "Null characters
   are not allowed."), `ProhibitSurrogateCharactersValidator`
   ("Surrogate characters are not allowed."), then the explicit
   `UniqueValidator` — `SELECT EXISTS(SELECT 1 FROM auth_user
   WHERE <field> = %s)` with lookup `exact` (case-SENSITIVE).
   Blank `email` `""` short-circuits BEFORE the unique query.
   Errors collect into an ordered field map; `validate()` runs
   only if that map is empty → compares `password` vs
   `password_confirm` → `ValidationError({'password_confirm':
   "Password fields didn't match."})` → field-keyed entry.
   → 400 with `serializer.errors`.
2. `serializer.save()` → `create()` pops `password_confirm` →
   `User.objects.create_user(username, email or '', password)`:
   - `normalize_username` = `unicodedata.normalize('NFKC')` —
     full-width `ｕｓｅｒ` stores as `user` (post-validation!),
   - `normalize_email` — strip, `rsplit('@',1)`, lowercase the
     DOMAIN only (`User@EXAMPLE.COM` → `User@example.com`),
   - `set_password` → `make_password` →
     `pbkdf2_sha256$<iters>$<salt>$<b64>`; salt =
     `get_random_string(12)` over `[a-zA-Z0-9]`; `<iters>` =
     `PBKDF2PasswordHasher.iterations` of the DEPLOYED version
     (6.x = 1_200_000; stale venv 4.2 = 600_000 — see §5),
   - INSERT `auth_user (password, last_login, is_superuser,
     username, first_name, last_name, email, is_staff,
     is_active, date_joined)` VALUES (hash, NULL, false, uname,
     '', '', email, false, true, now()).
3. `Token.objects.get_or_create(user=user)` → SELECT
   `authtoken_token WHERE user_id=%s`; miss → INSERT with
   `key=token_hex(20)`, `created=now()` (`models.py:29-45`).
   Race → IntegrityError → retry SELECT.
4. 201 `{user: UserSerializer(user).data, token: key,
   message: 'User registered successfully'}`.

### current_user (users/views.py:117-125)

Trivial: `IsAuthenticated` gate, then `UserSerializer(
request.user)` → 200. Zero extra queries beyond auth.

### obtain_auth_token (DRF `authtoken/views.py:26-31`)

1. `AuthTokenSerializer` field validation (messages above);
   `validate()` → `authenticate(request, username=…,
   password=…)` which iterates `AUTHENTICATION_BACKENDS`:
   a. `CaseInsensitiveAuthBackend` (backends.py:18-52):
      `User.objects.get(username__iexact=u)` → Postgres
      `UPPER("auth_user"."username"::text) = UPPER(%s)`;
      `DoesNotExist` → `User().set_password(password)` — runs a
      full PBKDF2 for timing blunting → None. Hit →
      `user.check_password(password)` → return user (NO
      `is_active` check). `MultipleObjectsReturned` (two users
      differing only by case — possible because the DB unique
      index is case-sensitive and register's UniqueValidator is
      `exact`) propagates → Django 500.
   b. `ModelBackend`: `get_by_natural_key` exact-match +
      `check_password AND user_can_authenticate`. Reached only
      when (a) missed; an exact-case hit that (a) found but
      rejected has the same password → still rejected.
      Net observable behavior: ONE iexact lookup + password
      check; `is_active` never consulted.
2. `check_password` internals: stored `!…` unusable passwords
   (all provisioned device/guest users) → False. On a success,
   Django's `must_update` rehashes when stored iterations are
   below the current default and saves via the `setter`
   callback — the `password` column can be UPDATED by a token
   login (see §5, DB-only divergence).
3. No `login()` for the target user → `last_login` untouched.
   (The middleware's device-user `login()` does bump the
   DEVICE user's `last_login` and writes a `django_session`
   row — both dropped per base §4.2.)
4. `Token.objects.get_or_create(user=user)` — identical to
   register's step 3 → 200 `{"token": key}`.

## 3. Rust implementation plan

Files (base §3/§6 — one module per endpoint group):

```text
crates/server/src/routes/users/
├── mod.rs        # router() + shared helpers
├── dto.rs        # UserDto, RegisterResponse, TokenResponse
├── validate.rs   # drf_char() + field-error map builder
├── register.rs   # POST /register/
├── me.rs         # GET /me/
└── token.rs      # POST handler for /api/token/
crates/core/src/auth.rs  # += get_or_create_token()
```

Routes (`crates/server/src/routes/mod.rs`):

```rust
.nest("/api/v1/users", routes::users::router())
//   users::router() =
//   Router::new().route("/register/", post(register))
//                .route("/me/",        get(me))
.route("/api/token/", post(users::token::obtain))
// ^ root level — NOT under /api/v1
```

Register `#[utoipa::path]` + `openapi.rs` entries on all three
(tag `Users`; the Django side has no schema decorator on
`/api/token/` — keep it minimal/public).

### Body decoding (`DrfBody` extractor, shared helper)

Read raw bytes + `Content-Type`: `application/json` →
`serde_json::from_slice` into `serde_json::Map`; form-urlencoded
→ `serde_urlencoded` into the same map; `multipart/form-data` →
`axum::extract::Multipart` field collection (needed for
`/api/token/` parity; register can share it). Unsupported type →
415 `{"detail": "Unsupported media type \"{ct}\" in request."}`.
Malformed JSON → 400 `{"detail": "JSON parse error - {msg}"}`
(text won't match CPython's json verbatim — flag for parity
normalization, §5). A map (not typed struct) preserves DRF's
missing vs `null` vs wrong-type distinction.

`drf_char(map, name, opts{required, allow_blank, min_length,
trim}) -> Result<Option<String>, FieldErr>` mirroring
`run_validation` order:

1. absent → required? "This field is required." : Ok(None)
2. `null` → "This field may not be null."
3. `""` → allow_blank? Ok("") : "This field may not be blank."
4. wrong type → bool/containers "Not a valid string." (email:
   "Enter a valid email address."); number → `to_string()` —
   Python `str()`: `123` → `"123"`, `3.0` → `"3.0"` (f64 with
   zero fraction prints `3`, Python prints `3.0` — edge,
   see §5)
5. `trim` → `.trim()`; `""` AFTER trim is NOT re-flagged blank
   (DRF quirk — validators then run on `""`)
6. `min_length` → "Ensure this field has at least {n}
   characters."
7. `\x00` → "Null characters are not allowed."; surrogate
   code points can't occur in valid UTF-8 — skip, note it.
8. caller-supplied validators (unique checks below)

Accumulate into an ORDERED map (`serde_json::Map` with
`preserve_order`, or `IndexMap`) in field order
username → email → password → password_confirm →
`ApiError::Validation` → 400. Email adds a syntax check
("Enter a valid email address.") — a pragmatic `contains('@')
&& domain has '.'`-level port of Django's `EmailValidator` is
enough; exact regex parity is overkill (§5).

### register handler

```rust
async fn register(State(s): State<AppState>, body: DrfBody)
    -> ApiResult<(StatusCode, Json<RegisterResponse>)>
```

1. Validate all four fields → field map; unique checks only
   for values that passed earlier stages:
   `SELECT EXISTS(SELECT 1 FROM auth_user WHERE username = $1)`
   and `… WHERE email = $1` (exact, case-sensitive — NOT
   iexact).
2. Field map non-empty → 400. Else compare passwords →
   `{"password_confirm": ["Password fields didn't match."]}`.
3. Normalize: `username.nfkc().collect::<String>()` (`unicode-
   normalization`); email = `email.strip()` then
   `rsplit_once('@')` → `name + "@" + domain.to_lowercase()`.
4. `password_hash` via `djangohashers` → `pbkdf2_sha256`
   `$1_200_000` iterations, 12-char `[a-zA-Z0-9]` salt (const
   `PASSWORD_HASH_ITERATIONS` in `core::auth`, matching the
   deployed Django — §5).
5. Insert:

```sql
INSERT INTO auth_user
    (password, last_login, is_superuser, username,
     first_name, last_name, email, is_staff, is_active,
     date_joined)
VALUES ($1, NULL, false, $2, '', '', $3, false, true, now())
RETURNING id
```

   `is_unique_violation()` on `auth_user_username_key` → 500
   (Django IntegrityError parity — race or NFKC collision).
   `value too long` (username > 150 chars — serializer has NO
   max_length) → also 500 parity.
6. `core::auth::get_or_create_token(pool, id)`:

```sql
SELECT key FROM authtoken_token WHERE user_id = $1
-- miss →
INSERT INTO authtoken_token (key, user_id, created)
VALUES ($1, $2, now())
ON CONFLICT (user_id) DO NOTHING
-- $1 = 40 lowercase hex of 20 random bytes (rand::rng());
-- re-SELECT if no row landed (get_or_create retry parity)
```

7. `(StatusCode::CREATED, Json({user: UserDto, token,
   message: "User registered successfully"}))` — preserve key
   order `user, token, message` (`preserve_order` map or a
   struct serializing fields in that order).

### me handler

```rust
async fn me(AuthUser(u): AuthUser) -> Json<UserDto>
```

`AuthUser` already resolved the `auth_user` row in the
extractor — ensure `Principal`'s SELECT fetches `id, username,
email, date_joined` (plus `password` if the token handler wants
the same row shape). Serialize → 200. Guest/device users pass
`is_authenticated` (base §4.2): a guest `/me` is a 200 with the
guest row, NOT an error — no special-casing.

### obtain handler (`/api/token/`)

```rust
async fn obtain(State(s): State<AppState>, body: DrfBody)
    -> ApiResult<Json<TokenResponse>>
```

1. `drf_char` username (required, trim) + password (required,
   NO trim); field errors → 400 field map.
2. Falsy pair → 400 `{"non_field_errors": ["Must include
   \"username\" and \"password\"."]}` (whitespace-only
   username lands here; pure `""` already died at step 1).
3. ONE lookup reproducing the backend chain:

```sql
SELECT id, password FROM auth_user
WHERE UPPER(username::text) = UPPER($1)
```

   - `fetch_all` returning >1 → `MultipleObjectsReturned`
     parity → 500.
   - 0 rows → run one full PBKDF2 `make_password` (timing
     blunting, backends.py:42-46) → 400 `{"non_field_errors":
     ["Unable to log in with provided credentials."]}`.
   - 1 row → `djangohashers::check_password(password, row.
     password)`; `!` unusable marker → false → same 400.
     `is_active` NOT consulted — deliberate quirk parity.
4. `get_or_create_token` (shared with register) →
   `Json({"token": key})` 200.
5. Optional parity: stored iteration count <
   `PASSWORD_HASH_ITERATIONS` → rehash + `UPDATE auth_user SET
   password=$1 WHERE id=$2` (Django `must_update` side-effect).

`UserDto` — `Serialize` only:
`{id: i64, username: String, email: String, date_joined:
DateTime<Utc>}`. Date format: `to_rfc3339_opts(SecondsFormat::
AutoSi, true)` — DRF's `isoformat()` omits the fraction when
micros are 0 and emits 6 digits otherwise; AutoSi matches that
for µs-precision columns. Slight refinement of base §4.3's
`Micros` (which always prints `.000000`) — centralize as
`core::fmt::drf_dt()` so all plans share it.

### Edge cases checklist

- register under prod provisioning → request.user is the device
  user; IGNORED (AllowAny) — create the new user anyway.
- `username` colliding with a PROVISIONED device username
  (`"lv-LV Android 12; Mobile"`, `"guest"`) → unique 400, same
  as any taken name.
- whitespace-only username `"   "` → trims to `""` at
  `to_internal_value`, validators pass (`WHERE username=''`
  misses) → Django CREATES a `""`-username user → 201. Reproduce
  verbatim; flag in §5.
- NFKC post-validation collision (`Ｊｏｈｎ` unique-checks as
  full-width, stores as `John`) → unique violation → 500.
- numeric `username: 123` → `"123"` accepted.
- `password` trimmed BOTH at validation AND before hashing —
  `"  abcdefgh  "` registers hash of `"abcdefgh"`.
- `/api/token/` for a provisioned unusable-password user →
  check_password false → 400 (they can never get a token).
- Case-variant pair exists (`JohnDoe` + `johndoe`) → iexact
  returns 2 rows → 500.
- `WWW-Authenticate` on 401s: `Token` (first auth class). Body
  contract is the load-bearing part; header is secondary.
- Trailing slashes: Django `APPEND_SLASH` would 301
  `/api/token` → `/api/token/`; axum registers the slash form —
  router-wide decision, defer to base plan (same note as
  endpoint 05).

## 4. Test plan

`users/tests.py` is empty — nothing to mirror literally. These
are contract-derived HTTP tests (`tests/users_auth.rs`,
axum-test + scratch Postgres, `AUTO_PROVISION_USERS=false`,
seed rows in `auth_user`/`authtoken_token` via SQL).

register:

1. happy path (no email) → 201: shape, key order, `id` int,
   token = 40 `[0-9a-f]`; DB row has `is_active=t`,
   `is_staff/is_superuser=f`, `last_login NULL`,
   `first_name/last_name=''`, `email=''`, `password` matching
   `^pbkdf2_sha256\$1200000\$`.
2. with email → `User@EXAMPLE.COM` stored `User@example.com`.
3. `{}` → 400 with all three required-field messages and NO
   `email` key; `email: null` → "may not be null.".
4. blank `""` fields → "may not be blank." each.
5. `password` 7 chars → min-length message verbatim; 8 → ok;
   `"  abcdef  "` (8 with spaces → 6) → min error.
6. taken username → unique message; `johndoe` vs existing
   `JohnDoe` → 201 second row (exact-match quirk) — then see
   token edge test 18.
7. taken email → unique message; `email: ""` always allowed
   even when other `""`-email rows exist.
8. mismatch → `password_confirm` message; field error +
   mismatch → ONLY the field errors (validate() skipped).
9. bad email (`"notanemail"`) → "Enter a valid email
   address."
10. `username: 123` → 201 stored `"123"`; `username: true` →
    "Not a valid string."; `email: 123` → "Enter a valid
    email address."
11. whitespace username `"   "` → 201 `""` (quirk; assert the
    contract as implemented, flag §5).
12. NFKC `"Ｔｅｓｔ"` → stored `"Test"`.
13. register while presenting a valid token → still 201 new
    user.
14. GET/PUT/DELETE on `/api/v1/users/register/` → 405
    `{"detail": "Method \"GET\" not allowed."}`.
15. register → `POST /api/token/` same creds → SAME token key.

me:

16. seeded user + `Authorization: Token <k>` → 200 dto;
    `Bearer <k>` also → 200.
17. anonymous → 401 not-provided body; garbage token → 401
    `Invalid token.` + `WWW-Authenticate: Token`.
18. seeded `guest` row + its token → 200 guest dto.
19. POST → 405.

token:

20. good creds → 200 `{"token"}`; second call → identical key
    (no rotation).
21. `JohnDoe` registered, login `JOHNDOE` → 200 (iexact).
22. wrong password → 400 `non_field_errors` "Unable to log
    in…"; unknown user → byte-identical body.
23. `is_active=false` + correct password → **200** (quirk —
    assert implemented behavior, flag §5).
24. unusable-password (`!…`) user → 400.
25. `username: "   "` → "Must include…" non_field error;
    `""`/`null`/missing → field errors.
26. `application/x-www-form-urlencoded` body → 200; multipart
    → 200 (if implemented, else §5 deviation).
27. password `" spaced "` for a user registered with spaces
    via raw SQL → 200 (no trim); padded wrong guess → 400.
28. GET on `/api/token/` → 405.

Unit: `drf_char` matrix (missing/null/blank/types/trim/
min_length), `normalize_email`, NFKC helper, `drf_dt` at
zero/nonzero micros, token-key length/charset.

Parity corpus (`tests/parity/`, base §4.15): register happy +
dup-username + short-password + mismatch + bad-email; me with
token + anon; token good + wrong-password + case-variant
username + form-encoded body. Normalize `id`, `date_joined`,
`token` before diffing.

## 5. Open questions / risks

- **PBKDF2 iteration count.** Deployed Django is 6.1.1 →
  1,200,000 (6.0 release notes); the stale venv reads 600,000.
  Verify what `PBKDF2PasswordHasher.iterations` the prod build
  uses and confirm `djangohashers` lets us SET the count on
  `make_password` (crate defaults lag Django). Not a hard
  blocker — `check_password` honors the stored count — but new
  hashes should match prod's work factor.
- **`is_active` ignored on token login** (CaseInsensitive
  backend never checks it). Reproducing verbatim is a security
  quirk — flag to the owner whether to keep parity or gate on
  `is_active` in v2. Plan says: reproduce + comment.
- **`must_update` rehash side-effect** on `/api/token/`
  success — DB-only divergence if skipped (Django upgrades
  old-iteration hashes; Rust wouldn't). Cheap to add; decide
  at impl.
- **Empty-username user** (`"   "` → `""`) and **username >
  150 chars** (no serializer max → DB `value too long` → 500)
  are real Django behaviors; reproducing 201/500 verbatim is
  planned. A kinder contract (400) would deviate — owner's
  call.
- **`iexact` multi-row → 500** (case-variant users). Django
  returns an HTML 500; Rust `ApiError::Internal` is a JSON-ish
  500 — bodies can't match, status parity is enough.
- **Multipart parsing on `/api/token/`** — `parser_classes`
  includes it; browsers/frontend use JSON or form. Implement
  json+form; multipart optional. Document if skipped.
- **JSON parse error text** — DRF embeds CPython's json message
  (`"JSON parse error - Expecting value: line 1 column 1"`).
  serde_json's wording differs; keep the `{"detail": "JSON
  parse error - …"}` SHAPE and normalize in the parity
  differ.
- **EmailValidator strictness** — Django's regex is elaborate
  (internationalized domains etc.). A pragmatic check risks
  accepting an address Django rejects (or vice versa); edge
  cases are rare — note and revisit only if parity corpus
  flags it.
- **float coercion text** — Python `str(3.0)` = `"3.0"` vs
  Rust `3.0f64.to_string()` = `"3"`: differs only if a client
  sends a float `username`, vanishingly rare; note in
  `drf_char` docs.
- **`WWW-Authenticate` value** under test-config DRF defaults
  would be `Basic`; prod Token-first gives `Token`. Rust emits
  `Token` unconditionally — matches prod, diverges from the
  conftest world; acceptable (base §4.2 already made this
  trade).
- **Session/login side-effects dropped** — middleware
  `login()` wrote `django_session` rows + bumped the DEVICE
  user's `last_login` on every `/api/token/` call; per §4.2
  sessions are not ported. No response-body impact.
- **`auth_user.email` unique is app-level only** (no DB
  constraint) — concurrent registers can both pass the
  existence check; same in Django, no new risk.
