**Status:** not implemented

# Backend CI / test / DB hardening plan

- **Date:** 2026-10-23
- **Source:** joint 3-agent investigation of the `bible_research`
  test suite, GitHub Actions, and database setup.
- **Scope:** repo `bible_research` (workflow + code changes via PR),
  plus GitHub repo settings and GCP IAM/state (no commits possible —
  user/admin actions, exact commands given).
- **Suite baseline:** 816 tests pass locally in ~110 s, 1 xfailed;
  coverage % unmeasured (PyPI unreachable from this machine —
  `pip install` fails with a self-signed-cert error; CI can install).

## 0. Findings → where each is fixed

| # | Finding | Fix |
|---|---|---|
| F1 | `Run tests` is not a required check; a red PR merged and
deployed to prod on Oct 14 | Phase 1.2 |
| F2 | `deploy.yml` has no `concurrency` group and does not wait on
tests — Cloud Run Job update raced once already | Phase 1.1 |
| F3 | terraform `apply` on main fails with
`403 iam.serviceAccounts.delete` on the old `audio-scheduler` SA;
guaranteed to re-fail on next terraform merge | Phase 2 |
| F4 | Local `config.yaml` authenticates to prod as **writable**
`avnadmin` (documented `ai_agent` read-only user is commented out) |
Phase 0 |
| F5 | `test_single_query_assertion` is `@expectedFailure` — a query-
count regression on `comments/counts/` is permanently muted |
Phase 3.1 |
| F6 | No coverage measurement anywhere | Phase 3.2 |
| F7 | Upstream API drift is invisible — every provider test mocks
the boundary; `dbt_integration_test.py` (live DBT) is outside
`testpaths` and never runs in CI | Phase 3.3 |
| F8 | Tests run on SQLite while prod is Postgres 16 | Phase 3.6
(stretch) |
| F9 | Small hygiene: `test_tag_serializer` returns a value
(DeprecationWarning); `manage.py test` uses the file `db.sqlite3`
instead of `:memory:`; AGENTS.md says Django 5.2 but
`requirements.txt` pins `Django==6.1.2` | Phase 3.5 |

## Phase 0 — restore the read-only local DB user (no PR)

`config.yaml` is gitignored; this is a local-only fix.

1. In `bible_research/config.yaml` swap the commented lines back:
   `USER: ai_agent` active, `avnadmin` commented (lines 5–8).
   Keep `avnadmin` available in comments for the rare case a
   write is genuinely needed.
2. Verify: `python manage.py shell -c "from django.db import
   connection; print(connection.settings_dict['USER'])"` →
   `ai_agent`.
3. Note the risk that remains: an accidental `migrate` attempt as
   `ai_agent` fails with `InsufficientPrivilege` — that is the
   desired behavior.

## Phase 1 — make red tests block merge and stop deploy races

### 1.1 Repo change (PR): serialize deploys

In `.github/workflows/deploy.yml` add:

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false
```

`cancel-in-progress: false` queues the second push instead of
cancelling mid-deploy. This alone fixes the Cloud Run Job
`ABORTED: Conflict for resource 'audio-generator'` race.

Optional (decide in the PR): gate deploy on test success by
replacing the `push` trigger with `workflow_run` on "Run tests"
(`types: [completed]`, then `if: github.event.workflow_run
.conclusion == 'success'`). Trade-off: doubles red-turnaround
visibility but `workflow_dispatch` still works for manual deploys.
Recommended minimum is the concurrency block plus Phase 1.2 —
with required checks, red code can no longer reach main anyway.

### 1.2 GitHub settings (user action — requires repo admin)

Two overlapping guards exist today (verified via `gh api`):

- **Classic branch protection on `main`**: requires only the
  `terraform` status check (hence `terraform-skip.yml`).
- **Ruleset "main rulesets" (id 15652770)**: requires a PR,
  1 approval, squash-only, Copilot review — **no required
  status checks**.

Add the `test` check (job name in `test.yml`; verify the exact
check-run name with `gh api repos/Bible-Research/bible_research/
commits/<sha>/check-runs`) to whichever protection is meant to be
authoritative — safest is both:

- UI: Settings → Rules → edit the `main` ruleset → "Require
  status checks to pass" → add `test` (and keep `terraform`).
- Or REST: `gh api repos/Bible-Research/bible_research/rulesets/
  15652770` → PUT the same JSON with a `required_status_checks`
  rule entry listing `test`.
- For classic protection:
  `gh api -X PUT repos/Bible-Research/bible_research/branches/
  main/protection/required_status_checks/contexts -f
  contexts[]=test -f contexts[]=terraform`
  (check exact endpoint shape for the current API version; the
  classic endpoint replaces the contexts array wholesale, so
  include `terraform` to keep it required).

Verification: open a throwaway PR, confirm the merge button is
blocked until `test` is green.

## Phase 2 — terraform apply repair

Root cause (verified in `terraform/iam.tf`): `github_deployer`
already has `roles/iam.serviceAccountAdmin` scoped by an IAM
condition matching SA **email** resource names
(`endsWith("/serviceAccounts/audio-scheduler@…")` etc.). The
delete call presents the resource in **numeric-UID form**
(`projects/-/serviceAccounts/107647814527877652766`), so the
condition never matches → 403. The deployer also has unconditional
`roles/resourcemanager.projectIamAdmin`, so CI can edit its own
IAM bindings if needed.

**Option A (recommended — the SA is meant to be gone anyway):**

1. Delete the real SA with owner creds (user runs):
   `gcloud iam service-accounts delete
   audio-scheduler@bible-research-489314.iam.gserviceaccount.com
   --project=bible-research-489314`
2. Drop it from state so apply stops planning a destroy:
   `cd terraform && terraform state rm
   google_service_account.audio_scheduler`
3. `terraform plan` should show no destroy for that SA.

**Option B (keep the delete in IaC):** extend the condition in
`iam.tf` `github_deployer_service_account_admin_scoped` to also
match the numeric UID form for each managed SA
(`resource.name.endsWith("/serviceAccounts/<21-digit-uid>")` —
UIDs via `gcloud iam service-accounts describe <email>`).
Fragile if an SA is recreated (new UID), so prefer A for this
one-off; document the caveat in `iam.tf` comments either way —
the same 403 will hit any future SA delete attempted by CI.

## Phase 3 — test-suite hardening (repo PR, one branch)

### 3.1 Un-mute the comment-counts query test

`annotations/tests.py:964` `test_single_query_assertion` asserts
≤2 queries on `comments/counts/` and is `@expectedFailure`.

1. Measure the real count (run the test locally and read the
   failure diff).
2. If it's an N+1 → fix `CommentCountView` (likely a missing
   `select_related`/annotation on the per-note count query).
3. If the count is legitimately higher → set the correct bound in
   `assertNumQueries`, drop `@expectedFailure`, and comment why.
4. Either way the test must assert a real number — delete the
   `expectedFailure` decorator.

### 3.2 Coverage in CI

- Add `pytest-cov` install + flags to `test.yml`:
  `pip install -r requirements.txt pytest-cov`, then
  `python -m pytest -q --cov=bible --cov=annotations --cov=users
  --cov-report=term-missing`.
- Do **not** add `--fail-under` yet — first run establishes the
  baseline; record it in this file's status header / a follow-up
  plan before gating.

### 3.3 Upstream-drift alarm (live DBT test)

- Add a `DBT_KEY` repo secret (real key; ask the user to create it
  — agents must not handle the secret value).
- New workflow `live-dbt.yml`: weekly cron + `workflow_dispatch`,
  runs `python -m pytest bible/services/dbt/
  dbt_integration_test.py -q` with `DBT_KEY` env. Non-blocking
  (informational) — a failure means DBT v4 changed shape and the
  mocked contract is stale.

### 3.4 FIXME-pinned bugs — decision pass, not code

`AGENTS.md` lists 4–5 tests that pin buggy prod behavior
(`test_reading_positions.py:179`, `test_tag_isolation.py:131`,
`test_note_isolation.py:231`, `test_images.py:1033`). For each:
either fix the code+test now, or open/link a GitHub issue and
convert the test to `xfail(reason="issue #N", strict=True)` so a
future fix surfaces as XPASS rather than a confusing failure.
Product decisions belong to the user — this phase only annotates.

### 3.5 Small hygiene (same PR)

- `annotations/tests.py` `test_tag_serializer`: remove the
  non-None return (silences the DeprecationWarning).
- `bible_research/settings.py` test branch (`'test' in sys.argv`):
  `db.sqlite3` → `':memory:'` so `manage.py test` can never touch
  the dev SQLite file.
- `bible_research/AGENTS.md` + workspace `AGENTS.md`: Django
  "5.2" → "6.x" (requirements pins 6.1.2; venv has 6.1.1 — note
  the pin/env skew too, `pip install -r requirements.txt` fixes).
- `DEVELOPER_GUIDE.md`: add a line noting Django 6.x (the guide
  still says 4.2.6).

### 3.6 Stretch — Postgres in CI

`test.yml` variant with `services: postgres:16` and a conftest
path honoring `DATABASE_URL` env (conftest currently hardcodes
sqlite `:memory:` — needs an `if os.environ.get('DATABASE_URL')`
branch). Catches PG-only behavior (collations, `distinct('on')`).
Worthwhile but separable; ship as its own PR if time allows.

## Phase 4 — coverage additions (separate PRs, lowest priority)

- `google_tts/client.py` — only 1 direct test today.
- Untested management commands: `extract_book_ids`,
  `import_esv_verses`, `populate_tag_positions` — the two audio
  generators already have test files to copy the pattern from.
- `dbt_client` has 5 tests; view-level error paths are covered
  via mocks — add client unit tests only if Phase 3.2 shows a
  gap that matters.

## Verification

- Phase 0: `connection.settings_dict['USER'] == 'ai_agent'`; a
  deliberate write attempt fails with `InsufficientPrivilege`.
- Phase 1: throwaway PR shows merge blocked on `test`; two rapid
  merges to main queue rather than race (`gh run list`).
- Phase 2: `terraform plan` on main shows no `audio_scheduler`
  destroy; next terraform-touching merge's apply is green.
- Phase 3: `pytest -q` green with zero `expectedFailure`/
  xfail entries; `term-missing` coverage table in CI logs;
  `live-dbt.yml` runs green on manual dispatch.
- All changed files ≤79 chars (`awk 'length>79{print FILENAME":"
  NR}' <file>`); commits `Type: Capitalized message`; feature
  branch only; stage files by path.

## Out of scope / deliberate non-changes

- No widening of `github_deployer` to unscoped
  `serviceAccountAdmin` — `iam.tf` comments document why the
  scoped grant exists (compromised-CI blast radius).
- No `makemigrations`/`migrate`/`sqlmigrate` by any agent — the
  prod `django_migrations` orphan rows (rewritten history) stay
  untouched; they are benign.
- No changes to `scheduler.tf` — the monthly TTS scheduler stays
  commented out per repo rules.
- Vercel dead-code branch in `settings.py` — documented, left in
  place (harmless; removal is a separate cleanup decision).
