# GCP Cost Reduction — status: partial

Item 6 shipped: aggressive Artifact Registry cleanup for GAE
deploy images (bible_research #107 + #108). The remaining
levers (s2c-app Cloud Run, secrets, scheduler) are still
proposed.

Goal: bring every billable resource inside (or as close as possible
to) the GCP Always Free tier. Surveyed 2026-10-19 with `gcloud` +
Cloud Monitoring MQL (account tedis.rozenfelds@gmail.com). Builds
on `gcp-cost-report.md` — this doc adds measured 30-day usage and
concrete change proposals.

Billing state (verified via `gcloud beta billing projects describe`):

- Account `016064-8B1D40-715CAA` (open): bible-research-489314,
  django-apps-7345, gen-lang-client-0833674612, geo-clash-gc798.
  Always Free quotas are shared across this billing account.
- Account `01FF48-1CC369-F42F3B`: s2c-app-492114 — separate
  account, its own free-tier quotas and its own bill.
- The other 8 projects have billing DISABLED — they cannot accrue
  charges. Nothing to do there.

Bottom line: the main account is already near-free (est. < $3/mo
in small overages — secrets, scheduler, Artifact Registry). The
real spend is s2c-app-492114: ~$120/mo, dominated by Cloud Run
with `cpu-throttling=false` + `minScale=1`, plus a broken global
HTTPS load balancer.

## Most consuming Cloud Run services

30-day totals via Monitoring MQL (`run.googleapis.com/request_count`
and `container/billable_instance_time`), 2026-09-19..2026-10-19.
Estimates use instance-based pricing ($0.000018/vCPU-s +
$0.000002/GiB-s; CPU always allocated) where `cpu-throttling=false`,
request-based pricing ($0.000024/vCPU-s + $0.0000025/GiB-s) where
throttled. All services run 1 vCPU / 512 MiB unless noted.

| Service             | Project   | Req/30d | Bill-s/30d | Est/mo |
|---------------------|-----------|--------:|-----------:|-------:|
| s2c-backend         | s2c-app   |  10,781 |  2,592,539 | ~$49   |
| s2c-registration    | s2c-app   |  80,892 |  1,862,979 | ~$35   |
| s2c-frontend-public | s2c-app   |  90,577 |    803,058 | ~$15   |
| s2c-frontend-admin  | s2c-app   |  12,937 |    387,164 |  ~$7   |
| industry-analyser   | gen-lang  |  17,604 |     11,310 |  $0    |
| django-apps         | dja-7345  |     837 |        542 |  $0    |
| geo-clash-server    | geo-clash |       4 |         40 |  $0    |
| audio-generator JOB | bible-res |     n/a |     ~2,100 |  $0    |

Config: s2c-backend is minScale=1 + `cpu-throttling=false`; the
other three s2c services are minScale=0 but also
`cpu-throttling=false` (instance-based billing — CPU billed while
the instance is up, idle or not). industry-analyser, django-apps
and geo-clash are minScale=0 + throttled (request-based billing).
`audio-generator` is a Cloud Run JOB (runs ~monthly, ~35 min).

`s2c-backend` billable-sec ≈ 2,592,000 s = 30.0 days: it never
scaled below 1 instance all month. s2c's four services total
~5.65M billable-sec/30d ≈ **$100–107/mo** — the single biggest
GCP cost found. Every other Cloud Run workload sits far under
the free tier (shared: 180k vCPU-s + 360k GiB-s + 2M req/mo on
account `016064`; 240k/450k instance-based on `01FF48`).

## Prioritized recommendations

| # | Change          | Project    | File to edit | Cost/mo | Free target |
|---|-----------------|------------|--------------|---------|-------------|
| 1 | s2c-be min->0   | s2c-app    | gcloud cmd   | ~$49    | scale-to-0  |
| 2 | s2c x3 throttle | s2c-app    | gcloud cmd   | ~$57    | req-bill    |
| 3 | rm s2c HTTPS LB | s2c-app    | gcloud cmd   | ~$18    | domap/$0    |
| 4 | prune secrets   | s2c + main | gcloud cmd   | ~$5     | <=6/acct    |
| 5 | sched ->3 jobs  | dj + genl  | other IaC    | ~$0.50  | 3 jobs      |
| 6 | AR keep 2->1    | bible_res  | artifact*.tf | ~$0.15  | <=0.5GB     |
| 7 | BQ cost export  | billing    | console      | blind   | 10GB BQ     |

Notes on the numbering: #1–#3 save ~$120/mo (s2c account); #4–#7
save ~$2/mo and add headroom on the main account.

Already compliant — no change needed:

- `bible_research/app.yaml` F1 min1/max1 = 24 inst-hrs/day < 28
  free. Setting `min_instances: 0` would NOT reduce cost further
  (already $0) and would add cold starts. Keep as is.
- `terraform/scheduler.tf` fully commented out — keep it that way
  (AGENTS.md rule). The 3-job free quota is already consumed by
  django-apps-7345 on the same billing account.
- `MONTHLY_TTS_CHAR_LIMIT=450000` < 1M Chirp-3-HD free chars/mo.
  TTS is effectively free; could raise the cap to ~900k safely.
- Cloud Run Jobs in all projects bill per-execution only and are
  far under free tiers. `audio-generator` ran 2026-10-03 (~35 min
  ≈ 2,100 vCPU-s) — free. Note: it still runs monthly despite the
  commented-out scheduler — check what triggers it (manual gcloud?
  external cron?) before assuming it is dead.
- `_Required` log bucket (400d) is free audit logging; `_Default`
  is 30d/50GB-free — fine.
- GCS free tier only applies to US regions; the ~1 GB audio bucket
  in europe-west3 costs ~$0.02/mo — not worth relocating.

### 1. Scale s2c-backend to zero (s2c-app-492114) — ~$49/mo

s2c-backend is `minScale=1` + `cpu-throttling=false`: one instance
billed 24/7 whether or not requests arrive (measured: 2,592,539
billable-s in 30 days). Fix:

```bash
gcloud run services update s2c-backend \
  --region=europe-north1 --project=s2c-app-492114 \
  --min-instances=0 --cpu-throttling
```

Expected effect: billable-sec drops from ~2.6M to only
request-serving time (~11k req/30d → a few hundred GiB-s/vCPU-s —
well inside the instance-based free tier). Trade-off: cold starts
(~5–15 s) on the first request after idle. If s2c is a live
product, keep `--min-instances=1` ONLY if latency matters; a
cheaper middle ground is `--min-instances=1` WITH
`--cpu-throttling` (still billed while idle but at lower rate —
actually no: min-instances bills regardless; throttling just
reduces the idle rate. Prefer full scale-to-zero).

### 2. Enable CPU throttling on the other 3 s2c services — ~$57/mo

`s2c-registration`, `s2c-frontend-public`, `s2c-frontend-admin`
are all minScale=0 but `cpu-throttling=false`, so every allocated
instance bills CPU even between requests — 2.65M billable-s/30d
combined despite only ~185k requests.

```bash
for svc in s2c-registration s2c-frontend-public s2c-frontend-admin
do
  gcloud run services update "$svc" \
    --region=europe-north1 --project=s2c-app-492114 \
    --cpu-throttling
done
```

Expected effect: billing switches to request-based (CPU only while
a request is in flight); ~185k req/30d at ~200 ms each ≈ 37k
vCPU-s — under the free tier. Trade-offs: background threads/tasks
outside request handling stop working (verify nothing relies on
them — e.g. websockets, in-process schedulers); slight latency
variance. If the frontends serve long-lived SSE/websocket
connections, throttle may break them — test on
`s2c-frontend-admin` (lowest traffic) first.

### 3. Remove the s2c global HTTPS load balancer — ~$18–20/mo

Found: `s2c-lb-ip` 34.98.105.144 (IN_USE), 2 forwarding rules
(`s2c-http-fr`, `s2c-https-fr`), target proxies, url-map
`s2c-url-map` + `s2c-https-redirect`, 4 serverless-NEG backend
services, and managed cert `s2c-lb-cert-332202c2` in state
**PROVISIONING_FAILED_PERMANENTLY** — so HTTPS via the LB is
currently broken anyway. Each forwarding rule ≈ $0.025/h ≈ $18/mo
for the pair plus per-GB processing.

If the s2c product is dead: tear down the whole project — delete
the LB (forwarding rules → proxies → url-maps → backend services
→ cert), release `s2c-lb-ip`, then the 4 Run services, the 31
secrets, and the `dockerhub-proxy` AR repo. Saves ~$120/mo total.

If it is alive: the cheapest serving path is Cloud Run managed
domain mapping (`gcloud beta run domain-mappings create` per
service, free + auto SSL) or Cloudflare in front of `*.run.app`.
Both avoid the $18/mo LB baseline. Trade-off: domain mapping adds
a redirect layer and one hostname per service (no path routing
across services — the URL map's raison d'etre).

### 4. Prune Secret Manager versions — ~$4–6/mo

Active secret versions (measured): s2c-app **77**, main account
27 total (bible 5, django-apps 11, gen-lang 9, geo-clash 2). Free
tier = 6 active versions per billing account, then $0.06 each.
77 on s2c ≈ $4.26/mo; 21 over on the main account ≈ $1.26/mo.

```bash
# Destroy all non-latest ENABLED versions per secret (dry-run
# first: remove --filter to inspect what would be destroyed).
for s in $(gcloud secrets list --project=s2c-app-492114 \
    --format='value(name)'); do
  gcloud secrets versions list "$s" \
    --project=s2c-app-492114 --filter='state=ENABLED' \
    --format='value(name)' | sort -n | head -n -1 | \
  while read -r v; do
    gcloud secrets versions destroy "$v" --secret="$s" \
      --project=s2c-app-492114 --quiet
  done
done
```

Trade-off: destroyed versions are unrecoverable — keep at least
the current `latest`. On the main account the biggest win is
django-apps-7345 (11 versions): consolidate rarely-changed keys or
accept ~$0.30/mo. `secrets.tf` in bible_research is data-source
only — no terraform change; pruning is a manual/gcloud step.

### 5. Consolidate Cloud Scheduler jobs to <=3 — ~$0.50/mo

Free tier = 3 jobs per billing account. Main account has 8:
django-apps-7345 (europe-west3): `sync-bank-transactions-schedule`,
`evaluate-spending-limits-schedule`, `check-balance-alerts-schedule`
(all daily ~02:00–03:00); gen-lang-client-0833674612
(europe-west1): `trigger-scrape-housing-ads`,
`trigger-scrape-apartment-ads`, `trigger-sync-housing-regions`,
`trigger-sync-apartment-regions`, `trigger-scrape-tv-programs`.

Cheapest fix: merge the two monthly `sync-*-regions` triggers into
one job that runs both Run Jobs sequentially, and merge the two
weekly `scrape-*-ads` triggers into one. That frees 2 slots on
gen-lang. If bible's monthly TTS job ever returns, give it
django-apps' pattern: one scheduler job invoking a "fan-out" Run
Job/Workflow instead of 1 job per target. Alternatively switch to
an App Engine cron (GAE cron also counts as scheduler jobs) — no
savings. Trade-off: merged jobs serialize (longer wall time) and
one failure blocks the rest.

### 6. Artifact Registry cleanup policy — ~$0.10–0.15/mo

AR free tier = 0.5 GB per billing account. Main account holds
~1.94 GB: gae-standard 642 MB (bible) + audio-generator 108 MB +
gae-standard 489 MB (django-apps) + industry-analyser-app 441 MB +
geo-clash 264 MB. The `audio-generator` and django-apps `gae-std`
images are single-image repos; bible's `gae-standard` keeps 2
versions.

Measured 2026-10-19: bible's 642 MB is a ~512 MiB GAE runtime
image floor (one deploy image alone exceeds free tier; AR dedupes
shared layers so the 2nd kept version only adds ~72 MiB) plus a
58 MiB stale build-cache. Savings are ~1–2 cents/mo, not ~15 —
the orphaned django-apps repo (~490 MB ≈ $0.05/mo) is the bigger
AR lever. Tighten `bible_research/terraform/artifact_registry.tf`:

```hcl
  cleanup_policies {
    id     = "keep-latest-2"     # rename to keep-latest-1
    action = "KEEP"
    most_recent_versions {
-     keep_count = 2
+     keep_count = 1
    }
  }
```

(apply to the `gae_standard` repo only — keep 2 on
`audio-generator`, its image is pinned by digest for the Job.)
Expected effect: ~300 MB less → account total ~1.6 GB, still over
0.5 GB but smaller. Trade-off: one fewer rollback image for GAE
standard; acceptable since `--stop-previous-version` + version
cleanup already run on deploy. Bigger lever if more savings are
wanted: delete the orphaned `gae-standard` repo in
django-apps-7345 (no GAE app exists there — it predates the move
to Cloud Run) for another ~490 MB.

### 7. Billing export + budgets — visibility, not savings

No BigQuery billing export exists (prior report confirmed zero
datasets). Enable Console → Billing → Billing export →
standard usage cost export into a `billing_export` dataset
(BQ storage < 10 GB + queries < 1 TB/mo are free). A EUR 20/mo
budget already exists on `016064`; add a smaller budget (e.g.
EUR 5) to catch the s2c account drift and a per-project alert.
Expected effect: replaces the estimates in this doc with real
SKU-level numbers within ~48 h.

## What CANNOT reach Always Free (and cheapest alternative)

- **s2c HTTPS LB (~$18/mo)** — no free tier for global external
  LBs. Cheapest: Cloud Run managed domain mapping (free) or delete
  the stack if s2c is abandoned (cert is already dead).
- **Cloud Run always-on** — minScale>=1 can never be free; any
  warm instance exhausts the free tier in days. Alternative:
  scale-to-zero + accept cold starts (rec. 1).
- **Postgres (all Django apps)** — `DATABASE_URL` secrets point at
  external Aiven PostgreSQL (not GCP Cloud SQL; no Cloud SQL
  instances exist anywhere). Cheapest: Aiven's own free PostgreSQL
  plan, or Neon/Supabase free tier (0.5 GB), or SQLite synced to
  GCS for read-heavy workloads. This is likely the largest
  non-GCP line item.
- **geo-clash Valkey (Redis)** — `geo-clash-valkey-url` points to
  an external managed Redis (Aiven). Free alternative: Upstash
  Redis free tier (10k commands/day) or Aiven free Valkey plan.
- **Secret versions > 6/account** — prune versions (rec. 4) or
  consolidate keys into fewer secrets (e.g. one JSON secret per
  app); trade-off is coarser-grained rotation.
- **GCS in europe-west3** — free tier requires us-east1/us-west1/
  us-central1 Standard storage. The ~1 GB audio bucket costs
  ~$0.02/mo; moving it to us-central1 would make it free but adds
  latency for EU users and re-upload churn — not worth it.
- **s2c-app billing account `01FF48`** — separate account the
  current user cannot read fully; confirm ownership before
  executing recs. 1–4 there.
