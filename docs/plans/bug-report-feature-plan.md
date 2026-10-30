# Bug Report Feature — Implementation Plan

**Status:** partial — shipped differently: the menu flow went
out as the Sentry-based "Report a problem" (`fdb9a41`,
`userReport.ts` → `reportType: user-report`). The
`bug_reports` DB table, GCS screenshot storage, and
GitHub-issue public path designed below were not built.

## Goal

Add an in-app "Report a bug" flow:

- Burger menu (`MyHeader` → `MainMenu` drawer) gets a **Report a Bug**
  item that opens a modal form.
- The form collects: **issue type** (dropdown), **description**, an
  auto-captured **screenshot** of the current screen (removable /
  replaceable), and a **public/private toggle**.
- Every report stores an automatic **app-state snapshot** (route,
  active book/chapter, fileset ids, theme, platform, etc.) in the
  database.
- **Private report** (default): row in a `bug_reports` DB table +
  screenshot(s) in a dedicated GCS bucket hard-capped at **100 MB**
  total.
- **Public report**: a GitHub issue is created via the GitHub REST
  API; its URL is stored on the DB row (`github_issue_url`).
  Screenshots for public reports are **not** written to GCS
  (see limitation note below).

## Non-goals (this iteration)

- Reporter-facing "my reports" list or status tracking UI.
- Admin web UI beyond Django admin (`/admin/` already exists).
- Image attachments inside the GitHub issue body (see below).
- Screenshot annotation/redaction tools.

## Key limitation: GitHub issues cannot receive file uploads

GitHub's REST API has **no endpoint for attaching images** to issues
(the web UI's drag-drop uses an undocumented internal endpoint).
Options considered:

1. *Recommended* — match the requirement literally: public reports
   do not persist screenshots anywhere. The issue body carries the
   description + app-state JSON + environment metadata. The DB row
   keeps text + state only.
2. Alternative — store the screenshot in GCS anyway for N days
   (lifecycle-deleted) and embed a signed URL in the issue body.
   Violates "no file storage for public reports" and V4 signed URLs
   expire after max 7 days anyway, so the link rots. Not recommended.

This plan implements option 1.

---

## Current codebase (findings)

### Backend (`bible_research/`)

- Reusable GCS pipeline already exists:
  `annotations/services/image_storage.py` provides
  `upload_original(image_id, file) -> (gs_uri, size_bytes,
  content_type)`, `signed_image_url`, `delete_original`; validates
  MIME type + magic bytes (Pillow) + `IMAGE_MAX_BYTES` (10 MB).
- `annotations/models.py` `Image` is a generic attachment with XOR
  FK to `Note`/`Comment` (`image_exactly_one_parent` check
  constraint) and `size_bytes` — the pattern to mirror.
- Settings block (`bible_research/settings.py` ~L238): all
  `IMAGE_*`/`AUDIO_*` config via `os.environ` with defaults.
- Terraform: `terraform/images_storage.tf` — bucket + IAM pattern
  to copy. App Engine service account gets `objectAdmin`.
- `DeviceAndCountryMiddleware` auto-provisions a `guest` user, so
  unauthenticated reporters are still attributable.
- Django admin is enabled; models there are registered manually
  (`annotations/admin.py`).
- No existing GitHub integration. `requests` is already a
  dependency (used by the DBT client — verify in
  `requirements.txt`).
- ID convention: prefixed string PKs ≤18 chars (`TAG…`, `CMNT_…`,
  `IMG_…`).

### Frontend (`reactive-bible/`)

- `MyHeader.tsx` renders the Mantine `Burger`; `App.tsx` owns
  `mainMenuOpened` state; `MainMenu.tsx` renders a right-side
  `Drawer` with `Stack` of items — new entries follow the
  `Group` + `Text` pattern (see "New Note", which opens
  `AddStandaloneNoteModal`).
- `api.tsx` holds all backend calls; `utils/apiClient.ts` provides
  `publicFetch` (attaches token if present, never logs out) and
  `authenticatedFetch` (401 → logout). Both hardcode
  `Content-Type: application/json` — multipart uploads must strip
  that header (see Frontend §3).
- `useBibleStore` (`store.tsx`) + `useAuthStore`
  (`stores/authStore.tsx`) hold everything needed for the state
  snapshot. **Never include `token`.**
- `@mantine/dropzone` is already a dependency (unused today) —
  free UI for extra manual attachments.
- No screenshot library present. Capacitor plugins are already in
  use (`@capacitor/core`, Android wrapper).

---

## Backend implementation

### 1. New `feedback` app

Create `bible_research/feedback/` (`apps.py`, `models.py`,
`serializers.py`, `views.py`, `urls.py`, `admin.py`, `tests.py`,
`services/`). Bug reports are platform feedback, not annotations —
a separate app avoids coupling and keeps `annotations` scoped to
user content. Register in `INSTALLED_APPS` and include its urls
under `/api/v1/` in `bible_research/urls.py`.

(Alternative, lower churn: put everything in `annotations/`. Same
model/view code either way.)

### 2. `BugReport` model

```python
def generate_bug_report_id():
    return f"BUG_{str(uuid.uuid4()).upper().replace('-', '')[:13]}"

class BugReport(models.Model):
    ISSUE_TYPES = [
        ("api", "API / backend"),
        ("frontend", "Frontend logic"),
        ("ui", "UI / layout"),
        ("ux", "UX / usability"),
        ("feature_request", "Feature request"),
        ("other", "Other"),
    ]
    id = models.CharField(max_length=18, primary_key=True,
                          default=generate_bug_report_id,
                          editable=False)
    user = models.ForeignKey(User, on_delete=models.SET_NULL,
                             null=True, blank=True,
                             related_name="bug_reports")
    issue_type = models.CharField(max_length=20,
                                  choices=ISSUE_TYPES)
    description = models.TextField()
    public = models.BooleanField(default=False)
    app_state = models.JSONField(
        default=dict,
        help_text="Client-side snapshot: route, book/chapter, "
                  "filesets, UA, viewport, platform, app version.")
    page_url = models.CharField(max_length=512, blank=True)
    app_version = models.CharField(max_length=64, blank=True)
    github_issue_url = models.URLField(
        max_length=512, null=True, blank=True,
        help_text="Set when a public issue was created.")
    github_error = models.CharField(
        max_length=256, blank=True,
        help_text="Last GitHub API failure, for audit.")
    created_at = models.DateTimeField(auto_now_add=True)
```

`Meta.ordering = ["-created_at"]`, index on
`("issue_type", "created_at")`.

### 3. `BugReportScreenshot` model

Mirror `Image` but in `feedback` (avoids a cross-app FK and the
XOR-constraint churn on `Image`):

```python
def generate_screenshot_id():
    return f"SCR_{str(uuid.uuid4()).upper().replace('-', '')[:13]}"

class BugReportScreenshot(models.Model):
    id = models.CharField(max_length=18, primary_key=True,
                          default=generate_screenshot_id)
    report = models.ForeignKey(BugReport, on_delete=models.CASCADE,
                               related_name="screenshots")
    storage_url = models.CharField(max_length=1024)
    content_type = models.CharField(max_length=64, blank=True)
    size_bytes = models.PositiveIntegerField(null=True, blank=True)
    uploaded_by = models.ForeignKey(User, on_delete=models.SET_NULL,
                                    null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

`size_bytes` is what makes the 100 MB cap cheap to enforce.

### 4. Storage service — `feedback/services/screenshot_storage.py`

Reuse, don't fork: refactor `image_storage.upload_original` into a
bucket-parameterized helper, e.g.

```python
def upload_image_to_bucket(bucket_name, object_prefix,
                           image_id, file) -> (gs_uri, size, ctype)
```

keeping `upload_original(image_id, file)` as a thin wrapper so the
note/comment path is untouched. The new module calls it with
`settings.BUG_REPORT_BUCKET` and object prefix
`bug-reports/{report_id}/{screenshot_id}`. Same signed-URL and
delete helpers apply (they derive bucket from the stored `gs://`
URL, so `signed_image_url`/`delete_original` already work for any
bucket).

### 5. The 100 MB cap

GCS has **no native bucket-size quota**, so enforce in app code:

- `settings.BUG_REPORT_STORAGE_MAX_BYTES` —
  default `100 * 1024 * 1024`.
- Before each upload, inside `transaction.atomic()` with a row lock
  on the parent `BugReport` (same pattern as
  `NoteImageViewSet.create`):

  ```python
  used = (BugReportScreenshot.objects
          .aggregate(Sum("size_bytes"))["size_bytes__sum"] or 0)
  if used + incoming_size > settings.BUG_REPORT_STORAGE_MAX_BYTES:
      raise ValidationError({"file": "Screenshot storage quota "
                                     "reached. Report saved without "
                                     "image."})
  ```

  Report creation itself must still succeed — drop the screenshot,
  not the report.
- Terraform lifecycle rule (below) is the backstop that reclaims
  space over time.
- Frontend also compresses the screenshot (JPEG ~0.7, capped width)
  so typical shots are 100–400 KB — hundreds of reports fit in
  100 MB.

### 6. GitHub service — `feedback/services/github_issues.py`

```python
def create_issue(title, body, labels) -> str  # returns html_url
```

- `POST https://api.github.com/repos/{GITHUB_REPO}/issues` with
  `Authorization: Bearer {GITHUB_TOKEN}`,
  `Accept: application/vnd.github+json`, `X-GitHub-Api-Version`.
- `title` = `f"[{issue_type}] {description[:60]}"` (truncate).
- `body` = description + `## App state` fenced-json block +
  metadata (page_url, app_version, user, timestamp).
- Label mapping: `api→api`, `frontend→frontend`, `ui→ui`, `ux→ux`,
  `feature_request→enhancement`, `other→bug`.
- 10 s timeout, `raise_for_status`; on failure log + store message
  in `github_error`, still return 201 (report is persisted; a
  management command or admin retry can come later).

Optional: map `issue_type` → different repos via
`GITHUB_REPO` (default `Bible-Research/reactive-bible`) vs a
`GITHUB_REPO_BACKEND` for `api` reports. Decide with the user;
single repo + labels is simpler.

### 7. Endpoint — `POST /api/v1/bug-reports/`

`feedback/views.py`, `ModelViewSet` restricted to `create` (plus
staff-only `list`/`retrieve` via `IsAdminUser` if wanted — Django
admin may suffice).

- `parser_classes = [MultiPartParser]` — single request carries:
  `description` (str, required, cap ~10k chars),
  `issue_type` (choice, required),
  `public` (`"true"|"false"`),
  `app_state` (JSON string, parse + cap ~64 KB, strip unexpected
  nested keys server-side — allowlist the known keys),
  `page_url`, `app_version`,
  `screenshot` (file, optional, repeatable or single — MVP: one
  `screenshot` field; `@mantine/dropzone` can still send one).
- Permission: `AllowAny` — the device middleware supplies `guest`
  attribution. Add a DRF throttle
  (`AnonRateThrottle`/`UserRateThrottle`, e.g.
  `BUG_REPORT_RATE = "10/day"`) to bound DB growth and GitHub-issue
  spam.
- Flow:
  1. `BugReport.objects.create(...)` (no screenshot yet).
  2. If `public`: call `create_issue(...)`; on success set
     `github_issue_url`. **Discard any uploaded file.**
  3. If `not public` and file present: quota check → upload to
     `BUG_REPORT_BUCKET` → `BugReportScreenshot.objects.create`.
  4. Response `201`: `{id, issue_type, public, github_issue_url,
     created_at}` following the existing `{"error": ...}` failure
     contract for 4xx/5xx.

### 8. Serializer + admin

- `BugReportSerializer` (read: all fields + nested screenshot
  signed_urls for staff; write: the multipart fields above).
- `BugReportAdmin`: `list_display` = id, issue_type, public, user,
  github_issue_url (as link), created_at; `list_filter` on
  issue_type/public; `search_fields` description/user; inline
  `BugReportScreenshot` with `readonly_fields` showing
  `signed_image_url` output (`format_html` link) so reviewers can
  open screenshots directly from admin.

### 9. Settings additions (`bible_research/settings.py`)

```python
BUG_REPORT_BUCKET = os.environ.get(
    "BUG_REPORT_BUCKET", f"{_gcp_project}-bug-reports")
BUG_REPORT_STORAGE_MAX_BYTES = int(os.environ.get(
    "BUG_REPORT_STORAGE_MAX_BYTES", 100 * 1024 * 1024))
GITHUB_TOKEN = os.environ.get("GITHUB_TOKEN") or config.get(...)
GITHUB_REPO = os.environ.get(
    "GITHUB_REPO", "Bible-Research/reactive-bible")
```

`GITHUB_TOKEN` goes to GCP Secret Manager in prod (see existing
secret handling in `app.yaml`/`config_template.yaml`); never commit
it.

### 10. Terraform — `terraform/bug_reports.tf`

Copy `images_storage.tf` shape:

- `google_storage_bucket.bug_reports`
  (`${var.project_id}-bug-reports`, `uniform_bucket_level_access`,
  `force_destroy = false`).
- Lifecycle: `AbortIncompleteMultipartUpload` age 7; `Delete` age
  90 (tuneable) — bounds storage as the app-level cap's backstop.
- IAM: `appspot` service account `roles/storage.objectAdmin`.
- Output `bug_reports_bucket`.
- Set `BUG_REPORT_BUCKET` env on App Engine (wherever
  `IMAGE_BUCKET_ORIGINALS` is supplied — `app.yaml` or terraform
  app-engine config).
- CORS: not needed (uploads go through the API, not direct-to-GCS).

### 11. Migrations

Per workspace rules, **the user runs**:

```bash
python manage.py makemigrations feedback
python manage.py migrate
```

### 12. Backend tests — `feedback/tests.py`

Mirror `annotations/test_images.py`: mock `_get_client`/bucket via
`unittest.mock.patch`; `override_settings(BUG_REPORT_BUCKET=...)`.
Cover:

- private report + screenshot → DB row + mocked GCS upload;
- quota path → report saved, screenshot rejected;
- public report → `create_issue` mocked; `github_issue_url` stored;
  uploaded file ignored (no GCS call asserted);
- GitHub failure → 201 still, `github_error` populated;
- missing/invalid `issue_type`, oversized `app_state` → 400.

---

## Frontend implementation

### 1. `src/components/ReportBugModal.tsx`

Mantine `Modal` (v6 API, mirroring `AddStandaloneNoteModal`):

- `Select` — "Issue type" — required: API / Frontend / UI / UX /
  Feature request / Other (`data` maps to backend choices).
- `Textarea` — "What happened?" — required, autosize.
- Screenshot preview `Image` + buttons: "Retake", "Remove"; hidden
  entirely when `public` is on (explain via `Text size="xs"` that
  screenshots can't be attached to public issues).
- `Switch` — "Make this report public" — caption: "Creates a public
  GitHub issue. Anyone can see the description and app state."
- Submit `Button` with loading state; on success
  `showNotification` (green) — if `github_issue_url` returned,
  include an `Anchor` to it.

### 2. `src/utils/bugReport.ts`

- `collectAppState()` — read `useBibleStore.getState()` and
  `useAuthStore.getState()`; return an explicit allowlisted object:
  `route` (`location.pathname`), `activeBookId`, `activeChapter`,
  `bibleVersion`, `activeTextFilesetId`, `activeAudioFilesetId`,
  `verseSelection`, `showNotes`, `versesFolded`,
  `username` (never token), `userAgent`, `viewport`,
  `platform` (`isNativeApp()` from `utils/nativeAudio`),
  `timestamp`. Explicit fields — **not** a raw store dump — so
  persisted-store changes can't leak data later.
- `captureScreenshot()` —
  - Web: dynamic `import("html2canvas")` → canvas →
    `toBlob("image/jpeg", 0.7)`; scale `windowWidth` to ≤1600 px.
    Dynamic import keeps ~45 KB out of the main bundle.
    Caveat: cross-origin images taint the canvas — wrap in
    try/catch, treat failure as "no screenshot".
  - Native (Capacitor): `@capacitor/screenshot` plugin's
    `Screenshot.take()` returns base64 → blob. Detect via
    `isNativeApp()`.
  - Fallback: return `null`; form still submits.

New deps: `html2canvas` (pin a version ≥7 days old) and
`@capacitor/screenshot` (native only).

### 3. `src/api.tsx`

```ts
export const submitBugReport = async (formData: FormData) => {
  const response = await publicFetch(
    `${API_BASE_URL}/api/v1/bug-reports/`,
    { method: "POST", body: formData,
      headers: { "Content-Type": "" } } // see note
  );
  ...
};
```

`publicFetch` merges `getAuthHeaders()` which sets
`Content-Type: application/json`. For multipart the browser must
set the boundary — either extend `apiClient.ts` with a
`multipartFetch` that omits `Content-Type` (cleaner), or delete
the header before send. Prefer the helper; it also serves future
note-image uploads (already a known gap in AGENTS.md).

### 4. `MainMenu.tsx`

- New `Group` item **"Report a Bug"** (`IconBug` from
  `@tabler/icons-react`) after "Tag Management"; opens
  `ReportBugModal` via local `useState` — same pattern as
  `newNoteOpened`/`AddStandaloneNoteModal`. Mount the modal next to
  it inside the fragment. Keep the drawer open/close behaviour
  consistent (`onClose()` first, like "New Note" does NOT — check:
  "New Note" keeps the drawer open behind the modal; match that
  since the modal is self-contained).

### 5. Tests

- `ReportBugModal.test.tsx` + `MainMenu.test.tsx` additions:
  MSW handler for `POST /api/v1/bug-reports/`; mock
  `utils/bugReport` (`vi.mock`) so html2canvas never runs under
  happy-dom. Assert payload fields, public toggle hiding the
  screenshot section, success notification, `title` test hooks
  (`title="report-bug-submit"` etc.) per convention.

---

## Privacy & security checklist

- `app_state` allowlist — never `token`, never note bodies, never
  full localStorage.
- Server re-validates everything: `issue_type` choice, description
  length, `app_state` size + schema, image MIME + magic bytes
  (inherited from `image_storage`), per-file 10 MB cap.
- Throttle the endpoint; public reports call GitHub — treat spam
  as a GitHub-abuse vector.
- `BUG_REPORT_BUCKET` private (uniform access, no allUsers);
  reviewers use signed URLs via Django admin.
- `GITHUB_TOKEN` scoped `issues:write` on the target repo only;
  Secret Manager in prod.
- Description/app-state are user-supplied text — GitHub API treats
  body as markdown; no HTML is rendered backend-side, so XSS risk
  is limited to admin views (Django escapes by default).

## Open questions for the user

1. Which repo should public issues target —
   `Bible-Research/reactive-bible` for everything, or split
   `api`-type reports to `bible_research`? (Plan defaults to one
   repo + labels.)
2. Should guests (unauthenticated) be allowed to file **public**
   reports? Recommend: yes for private, but require login for
   public (attribution + spam control on the GitHub side).
3. Retention: is 90-day auto-delete on screenshots acceptable, or
   keep until manually deleted (cap blocks new uploads at 100 MB)?

## Task order

1. Terraform bucket + IAM (`bug_reports.tf`) — apply manually.
2. Backend: `feedback` app, models, storage refactor, GitHub
   service, endpoint, admin, tests.
3. User runs `makemigrations` + `migrate`; set env vars.
4. Frontend: `bugReport.ts` utils, `api.tsx` fn, `multipartFetch`,
   `ReportBugModal`, `MainMenu` item, tests.
5. Docs: update both `DEVELOPER_GUIDE.md` files (endpoint +
   feature) — required by workspace rules.
6. Manual end-to-end: dev server → menu → report a bug → verify DB
   row via admin, GCS object, and (public) GitHub issue.
