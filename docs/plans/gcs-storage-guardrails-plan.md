# GCS Storage Guardrails — Status: investigation / not implemented

Goal: an authenticated user (or attacker with a stolen or
self-registered account) must not be able to fill GCS via the
image-upload endpoints and generate a large bill. Guardrails must
be AUTOMATIC, not just email alerts, and must keep the whole
billing account `016064-8B1D40-715CAA` (bible-research-489314,
django-apps-7345, gen-lang-client-0833674612, geo-clash-gc798)
near free-tier levels. Builds on `gcp-cost-report.md` /
`gcp-cost-reduction-plan.md` (2026-10-19 survey).

NOTE on scope: the GCS free tier (5 GiB Standard) applies ONLY to
us-east1/us-west1/us-central1. All app buckets are in
europe-west3, so EVERY byte stored is billed (~EUR 0.023/GiB-mo).
"Stay in free tier" for images therefore means "cap stored bytes
at a level whose cost is acceptable" — or relocate the images
bucket to a US region (see Optional hardening).

## Threat surface (current code)

Upload path: `bible_research/annotations/views.py`
`_BaseImageViewSet.create` (`POST /api/v1/notes/<pk>/images/` and
`…/comments/<pk>/images/`) → `upload_original()` in
`annotations/services/image_storage.py` → GCS bucket
`<project>-images-originals` (`originals/<image_id>/source<ext>`).

Existing controls:

- Per-file size cap: `IMAGE_MAX_BYTES` = 10 MiB
  (`bible_research/settings.py`), enforced with PIL verify +
  declared-vs-actual MIME match + decompression-bomb guard.
- Per-parent caps: `IMAGE_MAX_PER_NOTE` = 10,
  `IMAGE_MAX_PER_COMMENT` = 5, enforced twice (pre-check + under
  `select_for_update` on the parent row).
- Auth: `IsAuthenticated` + `GuestReadOnly`; register endpoint is
  throttled at `register` = 10/min (`users/views.py`).
- GCS upload happens BEFORE the DB row; any failure deletes the
  blob (rollback path exists).

GAPS — the exploitable ones:

1. **No per-user quota.** `Image.uploaded_by` and
   `Image.size_bytes` columns already exist, but nothing sums
   them. An attacker creates unlimited notes and attaches 10
   images x 10 MiB to each = unbounded storage.
2. **No authenticated-user throttle.** `DEFAULT_THROTTLE_CLASSES`
   is only `AnonRateThrottle` (300/min); image upload views set
   no `throttle_classes`, so a logged-in user is completely
   unthrottled (settings.py `REST_FRAMEWORK`).
3. **No monitoring on the images buckets.** `terraform/
   monitoring.tf` alerts on `bible_audio` `total_bytes` only —
   and the whole file is gated on `alert_email_address`, which
   is NOT in `ci.auto.tfvars` and only feeds from the GitHub
   repo variable `ALERT_EMAIL_ADDRESS` (verify it is set;
   otherwise no alert resources exist at all).
4. **Billing alert is email-only.** A EUR 20/mo budget exists on
   the billing account — no Pub/Sub hook, no automated action.
5. **Versioning on `images-originals`** keeps noncurrent blobs
   30 days; attacker uploads, then deletes, and storage is still
   billed for 30 days.
6. App Engine request cap (32 MiB) and the F1 single-instance
   ceiling throttle throughput somewhat, but a patient attacker
   still accumulates ~1 GiB per few hours; 30 days of abuse at
   even 50 GiB costs ~EUR 1.2/mo and grows — the risk is
   unbounded, not the rate.

## Buckets today

Read-only `gcloud` was denied in this session; numbers are from
the 2026-10-19 survey in `gcp-cost-report.md` (re-verify with
`gcloud storage du -s` before setting thresholds).

- `bible-research-489314-bible-audio`: 0.99 GiB, EW3, versioning.
- `bible-research-489314-images-originals`: 2.3 MiB, EW3.
- `…-images-thumbnails`: ~0 B. `staging.….appspot.com`: 16.9 MiB.
- IAM: appspot runtime SA has `roles/storage.objectAdmin` on
  `images-originals` (write) and `objectViewer` on thumbnails and
  audio (read-only). `audio-generator` SA writes audio only.
- CI: `github-deployer` SA (via Workload Identity Federation,
  `terraform/wif.tf`) already holds `roles/storage.admin` +
  `roles/resourcemanager.projectIamAdmin` — it can measure bucket
  size AND revoke IAM bindings without any new credentials.

## Industry practices (web research; sources at the end)

- **GCS has no native bucket size quota** (confirmed:
  cloud.google.com/storage/quotas — only rate limits and a
  1 TB/zone cap on Rapid/zonal buckets). Standard workarounds:
  app-side quota enforcement (the norm), monitoring alerts on
  `storage/total_bytes`, or a staging-bucket + notification
  pattern (overkill here).
- **`storage.googleapis.com/storage/total_bytes` and
  `storage/object_count` are daily-sampled gauges** — an alert on
  them can lag ~24-48 h. Fine as a safety net, useless as a
  fast kill switch.
- **Official kill-switch pattern** (Google docs "Disable billing
  with notifications" / "Control resource usage with
  notifications"): Budget → Pub/Sub topic → Cloud Run function →
  action. Actions range from selective (revoke a role, stop VMs)
  to nuclear (`projects.updateBillingInfo` to detach billing —
  takes down GAE and everything else; recovery is manual).
- **Cloud Monitoring notification channels** support Pub/Sub and
  webhooks; a monitoring alert can also drive automation, subject
  to the daily-sample lag above.
- **Rate limiting layers**: DRF throttling (already in use),
  App Engine firewall (IP rules only — weak vs distributed
  abuse), Cloud Armor (needs a load balancer — the app sits on
  the default `appspot.com` front end, so N/A without new infra).
- **GitHub Actions scheduled workflows** can impersonate a GCP
  service account via WIF (already configured here). Caveat:
  GitHub disables schedules after ~60 days of repo inactivity,
  and cron runs can be delayed during peak times — treat as a
  best-effort net, not a hard SLA.
- **Free tier reference** (cloud.google.com/storage): 5 GiB
  Standard in us-east1/us-west1/us-central1, 5,000 Class A +
  50,000 Class B ops/mo, 100 GiB egress NA-only, per billing
  account.

## Options comparison

- **A. Per-upload check of total bucket size in the view**
  (user's idea). Stops abuse? yes; immediate; small effort.
  Weakness: `total_bytes` is daily-sampled, so the check reads
  stale data, and listing blobs per upload is expensive and
  racy. Enforce on DB sums instead — see L1.
- **B. Per-user DB quota + upload throttle (recommended L1).**
  Stops abuse at the source; immediate; small effort.
  Weakness: locmem throttle cache is per-process — fine on
  the single F1 instance, weaker if GAE ever scales >1.
- **C. Monitoring alert → email (existing pattern).**
  Does NOT stop abuse — a human must act; ~24-48 h lag;
  tiny effort; needs ALERT_EMAIL_ADDRESS set.
- **D. Monitoring alert → Pub/Sub → Cloud Run function that
  revokes write IAM.** Stops abuse; ~24-48 h lag; medium
  effort (new Pub/Sub + function infra in terraform).
  Weakness: same daily metric lag; adds APIs to keep enabled.
- **E. Budget → Pub/Sub → Cloud Run function kill switch.**
  Stops abuse; hours-to-days lag (billing export latency);
  medium effort. Selective IAM revoke is fine; auto-disabling
  billing is nuclear — kills the whole app, manual recovery.
- **F. Scheduled GitHub Action: `gcloud storage du` + revoke
  IAM (user's idea).** Stops abuse; <=1 h (hourly cron);
  small effort — WIF + SA rights already exist. Weakness:
  best-effort cron; GitHub disables schedules on idle repos.
- **G. App Engine firewall / Cloud Armor.** Partial at best;
  medium+ effort. IP rules don't stop one authenticated
  user; Armor needs a load balancer — not recommended.

## Recommended layered design

### Layer 1 — App-level limits (primary defense; do first)

All in `bible_research/` (Django app). No new infra, no cost.

- **L1a. Per-user storage quota.** In
  `_BaseImageViewSet.create` (annotations/views.py, before the
  `upload_original` call and again inside the `transaction.atomic`
  block under the parent row lock — plus lock the user row
  `User.objects.select_for_update().get(pk=user.pk)` so two
  parallel uploads by one user cannot both pass), check:

  ```python
  agg = Image.objects.filter(uploaded_by=user).aggregate(
      total=Sum('size_bytes'), cnt=Count('id'))
  ```

  Reject with 403/ValidationError when `total + file.size`
  exceeds `settings.IMAGE_USER_QUOTA_BYTES` (suggest 100-200 MiB)
  or `cnt` exceeds `settings.IMAGE_USER_QUOTA_COUNT` (suggest
  200). Both settings follow the existing
  `IMAGE_MAX_BYTES`-style env-var pattern in
  `bible_research/settings.py`. No migration needed — the
  columns already exist.
- **L1b. Upload rate throttle.** Add a `UserRateThrottle`
  subclass + `throttle_scope = 'image_upload'` on
  `_BaseImageViewSet` (or set `throttle_classes` in `create`),
  with `'image_upload': '30/hour'` and optionally a day rate via
  a second scope (`200/day`) in `DEFAULT_THROTTLE_RATES`. Locmem
  cache is sufficient while GAE runs a single F1 instance —
  note this in settings comments (multi-instance would weaken
  it; acceptable residual risk).
- **L1c. Global kill switch.** `IMAGE_UPLOADS_ENABLED` env
  setting (default true); `create` returns 503 when false. Lets
  the operator stop uploads with `gcloud app deploy`-free env
  change... (App Engine env changes do need a redeploy — still
  faster than code edits; alternatively read it from an env var
  already plumbed through `app.yaml`.)
- **L1d. Tighten `IMAGE_MAX_BYTES`** 10 MiB → 5 MiB (real photos
  are fine at 5 MiB; halves worst-case per-upload damage). Pure
  settings change.

Layer 1 alone caps worst-case user damage at
quota_bytes × active_users and per-day growth at
throttle × 5 MiB. It is the cheapest and fastest control — the
layers below are backstops for "the app-level check itself is
bypassed or buggy".

### Layer 2 — GCP monitoring alerts (detection net; terraform)

Extend `bible_research/terraform/monitoring.tf`:

- Add `google_monitoring_alert_policy` for
  `images-originals` `storage/total_bytes` (suggest threshold
  ~1 GiB, well above the current 2.3 MiB) and optionally
  `storage/object_count`. Reuse the same gated email channel
  pattern — requires the `ALERT_EMAIL_ADDRESS` repo variable to
  actually be set (manual step, below).
- Optional second channel: Pub/Sub notification channel feeding
  a small Cloud Run function that revokes the appspot SA's
  `roles/storage.objectAdmin` binding on `images-originals`
  (Option D). Recommend as phase 2 — adds `pubsub` API +
  function infra to terraform; both fit in free tiers.

### Layer 3 — Scheduled GitHub Action guardrail (automatic net)

New workflow
`bible_research/.github/workflows/storage-guardrail.yml`,
modeled on `cleanup-artifacts.yml`
(same WIF auth stanza; `github-deployer` already has
`roles/storage.admin` + `projectIamAdmin`):

```yaml
on:
  schedule: [{ cron: '23 * * * *' }]   # hourly
  workflow_dispatch:
permissions: { contents: read, id-token: write }
```

Steps:

1. `google-github-actions/auth@v3` + `setup-gcloud@v3` exactly
   as in `cleanup-artifacts.yml`.
2. Measure both writable buckets:

   ```bash
   BYTES=$(gcloud storage du -s \
     "gs://${GCP_PROJECT_ID}-images-originals" --total-bytes)
   ```

   (also check `-bible-audio` and `-images-thumbnails`.)
3. On breach (suggest 2 GiB images, 6 GiB total):

   ```bash
   APPSPOT_SA="${GCP_PROJECT_ID}@appspot.gserviceaccount.com"
   gcloud storage buckets remove-iam-policy-binding \
     "gs://${GCP_PROJECT_ID}-images-originals" \
     --member="serviceAccount:${APPSPOT_SA}" \
     --role="roles/storage.objectAdmin"
   ```

   Then `gh issue create` (needs `issues: write` permission) or
   `gh api` to file an alert issue titled "GCS guardrail tripped".
4. Document restore in the issue body: `terraform apply` (the
   binding lives in `images_storage.tf` — apply repairs the
   drift) or `gcloud storage buckets add-iam-policy-binding`.

Properties: revoking `objectAdmin` breaks uploads AND deletes
(the app starts returning errors on upload immediately).
Existing images keep serving: signed-URL generation only needs
`iam.serviceAccountTokenCreator`, which the appspot SA holds on
itself, plus object GET permission — note the current binding is
`objectAdmin` only, so verify reads still work after revocation
(objectAdmin includes read). Cheaper alternative action: swap
the binding for an IAM deny policy — plain revoke is simpler
and terraform-reversible. This layer catches anything Layer 1
misses within ~1 hour, well before the daily metric exists.

### Layer 4 — Billing-account backstop (manual / GCP console)

- Keep the EUR 20 budget; LOWER alert thresholds (e.g. EUR 2, 5,
  10) so abuse pages the owner early — console change.
- Do NOT auto-disable billing on the shared account: it would
  kill GAE + the other projects and recovery is manual. If a
  programmatic budget response is wanted later, use Option E
  scoped to `bible-research-489314` only, still revoking IAM
  rather than detaching billing.

### Optional hardening (not required)

- **Relocate images buckets to us-central1** → the first 5 GiB
  becomes free-tier covered. Trade-offs: EU latency, re-upload
  of existing 2.3 MiB (trivial), terraform location change.
- **Disable versioning on `images-originals`** (or shorten
  noncurrent retention): deleted attacker/legit images bill for
  30 days extra under versioning. Recovery value for user images
  is low — evaluate dropping `versioning { enabled = true }`.
- **Egress watch**: anonymous reads of public-note images are
  throttled (300/min anon), and signed URLs cap at 10 min TTL —
  egress abuse is bounded but worth a `sent_bytes` alert later.

## Manual steps required (cannot be done by the agent)

1. Verify/set GitHub repo variable `ALERT_EMAIL_ADDRESS`
   (Settings → Variables) — without it `monitoring.tf` creates
   nothing.
2. Console → Billing → Budgets: add lower alert thresholds /
   a second small budget as above.
3. If implementing the Pub/Sub kill switch: enable
   `pubsub.googleapis.com` + Cloud Build/Run-functions APIs in
   `main.tf`'s service list, and wire the budget → topic link in
   the Console (budgets are not terraform-managed here).
4. Decide quota values (`IMAGE_USER_QUOTA_BYTES` etc.) and set
   them via env/`app.yaml` if non-default.
5. Note: migrations for the quota check are NOT needed (uses
   existing columns) — do not run makemigrations.

## Implementation order

1. L1a + L1b + L1d (one PR in `bible_research/`; tests in
   `annotations/test_images.py`).
2. Layer 3 workflow (one PR; test via `workflow_dispatch` with a
   dry-run flag that only reports size, then flip `DRY_RUN`).
3. Layer 2 terraform alert on `images-originals` (requires step
   1 of manual steps).
4. Layers 4 / Pub/Sub kill switch / bucket relocation — later,
   only if wanted.

## Sources

- cloud.google.com/storage/quotas (no bucket size cap)
- cloud.google.com/storage (free tier figures)
- docs.cloud.google.com/billing/docs/how-to/disable-billing-
  with-notifications and /control-usage (budget→Pub/Sub→function)
- cloud.google.com/monitoring (total_bytes daily sampling;
  Pub/Sub/webhook channels)
- Repo: `annotations/views.py`, `annotations/services/
  image_storage.py`, `bible_research/settings.py`,
  `terraform/{images,audio}_storage.tf`, `monitoring.tf`,
  `iam.tf`, `wif.tf`, `.github/workflows/cleanup-artifacts.yml`
