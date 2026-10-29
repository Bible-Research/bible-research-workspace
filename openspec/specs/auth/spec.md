# Auth Specification

## Purpose

Defines how callers authenticate to the REST API (token schemes,
no anonymous provisioning, the read-only guest account) and how
the frontend's two fetch helpers handle tokens and 401
responses differently for public versus user-owned resources.

## Requirements

### Requirement: Token authentication schemes

The API SHALL authenticate requests via DRF
`TokenAuthentication` (`Authorization: Token …`) and
`BearerTokenAuthentication` (`Authorization: Bearer …`). The
DRF API MUST NOT accept cookie sessions; Django session auth
exists for `/admin` only. Tokens are issued via
`POST /api/token/`.

#### Scenario: Token header authenticates request

- **WHEN** a request carries `Authorization: Token <key>` with
  a valid token
- **THEN** the request is processed as that token's user

#### Scenario: Bearer variant authenticates request

- **WHEN** a request carries `Authorization: Bearer <key>`
  with a valid token
- **THEN** the request is processed as that token's user

#### Scenario: Cookie session rejected by API

- **WHEN** a request to an `/api/v1/` endpoint carries only a
  Django session cookie
- **THEN** it is treated as anonymous, not as the logged-in
  user

### Requirement: No anonymous auto-provisioning

Anonymous requests SHALL arrive as a real `AnonymousUser` — no
middleware-created per-device `User` rows. Every write endpoint
MUST require `IsAuthenticated` explicitly, so anonymous writes
return 401; `/me/` SHALL return 401 for anonymous callers.

#### Scenario: Anonymous write is rejected

- **WHEN** an unauthenticated caller POSTs to a write endpoint
  such as `notes/` or `tags/`
- **THEN** the API responds 401

#### Scenario: Anonymous read of public content

- **WHEN** an unauthenticated caller GETs bible content
- **THEN** the request succeeds without any user account being
  created

### Requirement: Guest account is read-only

The legacy `guest` account SHALL be read-only: `GuestReadOnly`
(wired into `DEFAULT_PERMISSION_CLASSES` plus the image and
reading-position views that override permissions) rejects its
writes with 403. It exists only to own rows created before
provisioning was removed and is reachable only via a
previously issued token.

#### Scenario: Guest write is forbidden

- **WHEN** the `guest` account attempts a write with its token
- **THEN** the API responds 403, not 401

### Requirement: User-scoped querysets

Serializers SHALL scope tag and note querysets to
`request.user`; anonymous callers get `.none()`.
`GET /api/v1/comments/counts/` SHALL remain `AllowAny` so share
pages can count public-note comments.

#### Scenario: Anonymous user sees no notes

- **WHEN** an unauthenticated caller lists `notes/` or `tags/`
- **THEN** the response contains an empty result set

#### Scenario: Comment counts stay public

- **WHEN** an unauthenticated caller requests
  `comments/counts/`
- **THEN** the API returns counts for public-note comments

### Requirement: Frontend fetch helper semantics

The frontend SHALL use two helpers from
`src/utils/apiClient.ts`: `publicFetch` attaches the token when
present, retries a 401 once anonymously (a stale token must not
break public content), and never logs out;
`authenticatedFetch` logs out and shows a "session expired"
notification on 401. `publicFetch` is for bible content,
`authenticatedFetch` for user-owned resources. Network-failure
retry SHALL fire only for idempotent methods (GET/HEAD/
OPTIONS).

#### Scenario: Stale token on public content

- **WHEN** `publicFetch` receives a 401 while carrying a stale
  token
- **THEN** it retries once without the token and never logs the
  user out

#### Scenario: 401 on user-owned resource

- **WHEN** `authenticatedFetch` receives a 401
- **THEN** the user is logged out and a "session expired"
  notification is shown

#### Scenario: Non-idempotent request is not retried

- **WHEN** a POST/PUT/DELETE fails with a network error
- **THEN** no automatic network-failure retry is attempted
