# Self-generated verse timestamps for Latvian DBT audio —
Status: implemented (Bible-Research/bible_research#116)

Goal: for Latvian DBT audio filesets that have no DBT verse
timestamps, generate our own `*_timestamps.json` artifacts with
Google Cloud Speech-to-Text (STT) so the frontend verse
follow-along works for them — while staying inside the STT V1
free tier (60 min/month) using the same metering pattern as the
TTS audio generator (`bible/services/google_tts/` +
`generate_chapter_audio`).

## Current state (verified live 2026-10-30)

`GET /api/v1/bible/translations/?language_iso=lvs` offers:

| Fileset | Name | Source | Timestamps? |
|---|---|---|---|
| LATBSLN1DA(+opus16) | LAVNLI 2012 NT plain | DBT | yes (DBT) |
| LATBSLN2DA(+opus16) | LAVNLI 2012 NT drama | DBT | yes (DBT) |
| LATBSLP1DA(+opus16) | LAVNLI OT partial (sparse) | DBT | NO — `{"data":[]}` |
| LATBSLP2DA(+opus16) | LAVNLI OT partial drama | DBT | NO — `{"data":[]}` |
| LATLVRN1DA(+opus16) | LVR 1953 BFBS NT | DBT | NO — `{"data":[]}` |
| LATLVRP1DA(+opus16) | LVR 1953 OT partial | DBT | NO — `{"data":[]}` |
| LVSGLU8C1DA / C2DA | GLU8 generated TTS | GCS | yes (generated) |

So the gap is `LATLVRN1DA`, `LATLVRP1DA`, `LATBSLP1DA`,
`LATBSLP2DA`. The `-opus16` variants are the same recordings in
opus — timestamps generated for the base fileset apply to them
(the frontend's `resolveTimestampsFilesetId` already strips the
suffix; the serving fallback must do the same).

Neither LAVLVR nor LAVNLI has a text fileset in DBT
(`text_options: []`), so verse alignment needs a reference text
(see "Reference text" below).

## How the existing TTS generator works (what we mirror)

- `bible/management/commands/generate_chapter_audio.py` — the
  monthly run: acquire per-month lock → read persisted monthly
  usage → build in-memory `CharBudget` → for each pending
  chapter: pre-flight `budget.can_afford(...)`, generate,
  upload, then persist the spend delta in `finally` (billed
  spend is never lost, even mid-chapter failure).
- `bible/services/google_tts/budget.py` — `CharBudget` with
  `can_afford`/`consume`/`BudgetExceeded`.
- `bible/services/storage/gcs.py` —
  `read_monthly_usage`/`increment_monthly_usage` (object
  `state/usage/{fileset}/{YYYY-MM}.json`, `if_generation_match`
  optimistic concurrency), `acquire_run_lock`/
  `mark_run_complete` (object `state/lock/{fs}/{mode}/{YYYY-MM}`,
  stale after `LOCK_STALE_HOURS`), `list_completed_chapters`
  (progress derived from artifact presence — no DB).
- Scheduling: `all-jobs` Cloud Run Job runs
  `run_all_jobs`; `JOBS` table in
  `bible/management/commands/run_all_jobs.py` gates sub-jobs on
  `dom:3`; env vars (incl. `MONTHLY_TTS_CHAR_LIMIT=450000`) in
  `terraform/all_jobs.tf`. Prod runs are manual —
  `gcloud run jobs execute all-jobs`.
- Serving: `GET /api/v1/bible/timestamps/` →
  `provider_for(fileset_id).get_timestamps` →
  `{"data": [{"verse_start", "timestamp"}]}`.

## GCP STT facts that shape the design

- Free tier: 60 min/month, **V1 API only** (`speech.SpeechClient`
  — package `google-cloud-speech`). V2/Chirp has no free tier.
- Word offsets: `RecognitionConfig(
  language_code="lv-LV", enable_word_time_offsets=True,
  encoding=MP3, model="latest_long")` — MP3 input is supported
  by V1, so DBT MP3s need no transcoding.
- Chapters exceed the 1-min sync limit → use
  `long_running_recognize` with `RecognitionAudio(uri="gs://…")`.
  Staging audio must live in GCS (see layout below).
- Billing counts every successfully processed request in
  1-second increments, including empty results — so consume the
  chapter's full duration once the LRO completes, even if the
  result is empty. Network/LRO failures before a response are
  not billed.
- DBT chapter rows already carry `duration` — use it for budget
  pre-flight without downloading the MP3 first.

## Reference text for verse alignment

STT yields word-level times; we still must map words to verse
numbers by aligning the transcript to a known chapter text.

- `LATLVR*` (1953 Glück revision) → reference = **GLU8 SWORD
  text** (`SwordClient.get_chapter_verses("LVSGLU8", …)`). LVR
  1953 is a revision of Glück, so a high lexical match is
  expected; fuzzy alignment covers the residual differences.
- `LATBSLP*` (LAVNLI 2012) → no text exists in our stack.
  Task order: (1) check api.scripture.api.bible for a Latvian
  text bible (`GET /bibles?language=lav` — note `ApiBibleClient`
  has NO list-bibles method today, so this needs a small new
  client method or a raw `requests` call with `API_BIBLE_KEY`);
  (2) if none, fall back to GLU8 with a lower
  `MIN_ALIGNMENT_SCORE` and mark artifacts low-confidence;
  (3) if alignment fails outright, skip those filesets in phase
  1 and ship `LATLVR*` only — do not block the feature on it.
- Alignment algorithm (`google_stt/aligner.py`, pure functions,
  fully unit-testable):
  1. Normalize words: lowercase, strip punctuation/digits,
     keep Latvian diacritics.
  2. `difflib.SequenceMatcher` between STT word list and the
     concatenated verse-word list; matching blocks give anchor
     words.
  3. Each verse's first matched word → its STT index →
     timestamp. Verses without a matched anchor interpolate
     linearly between neighbouring anchored verse boundaries
     proportional to word count. Chapter fails if < 2 anchors.
  4. Leading spoken metadata ("Jāņa evaņģēlijs, trešā nodaļa")
     simply never matches — no special-casing needed.
  5. Record `alignment_score` = matched reference words /
     total reference words; upload even when low but flag
     `"low_confidence": true` below `MIN_ALIGNMENT_SCORE`
     (default 0.5) so artifacts are inspectable later.

## GCS layout (bucket `${project}-bible-audio`)

```
timestamps_generated/{FILESET}/{BOOK}/{CH}_timestamps.json
stt_staging/{FILESET}/{BOOK}/{CH}.mp3        # LRO input
state/stt-usage/{YYYY-MM}.json              # GLOBAL seconds
                                           # counter (STT bills
                                           # per account)
state/stt-lock/{FILESET}/{YYYY-MM}.json     # per-fileset
                                           # monthly lock
```

Artifact payload shape (same `data` contract as SWORD mode):

```json
{"data": [{"verse_start": 1, "timestamp": 0.0}, ...],
 "duration_seconds": 312.5,
 "file_size_bytes": 2501234,
 "meta": {"source": "stt_v1", "reference_text": "LVSGLU8",
          "alignment_score": 0.91, "low_confidence": false,
          "generated_at": "…"}}
```

- Progress is derived like TTS: a chapter is done iff its
  `timestamps_generated/…` object exists (`list_…` helper
  mirroring `list_completed_chapters`).
- `stt_staging/` objects get a 7-day lifecycle delete rule in
  `terraform/audio_storage.tf` (belt-and-suspenders; the job
  also deletes them after the LRO finishes). NOTE: the bucket
  has object versioning enabled, so a `Delete` lifecycle action
  only makes blobs noncurrent — bytes are actually freed by the
  existing noncurrent-after-30-days rule (~37-day effective
  retention). Harmless at these sizes; do not disable
  versioning.

## Serving path (contract unchanged — frontend untouched)

`DbtProvider.get_timestamps` (providers.py ~894): today it
returns `[]` when DBT `data` is null/empty. Change: when DBT
data is empty AND the fileset (with a `-opus16` suffix stripped)
is in a new `GENERATED_TIMESTAMPS_FILESETS` registry, read
`timestamps_generated/{base}/{BOOK}/{CH}_timestamps.json` via a
new `gcs.read_generated_timestamps_json` (locmem-cached like
`read_timestamps_json`, separate `v1:gen-ts:` cache-key prefix).
Artifact missing → return `[]` (preserves today's 200-empty
contract; no 404 change, no new `error_code`).

If the DBT timestamps call itself raises for a registered
fileset, try the generated artifact before propagating: artifact
found → return it; not found → re-raise (today's 502 behavior
is preserved when we have nothing better).

The App Engine service account already has
`roles/storage.objectViewer` on the whole bucket
(`audio_storage.tf`), so `timestamps_generated/` reads need no
new IAM — don't add a redundant grant.

## New code

- `bible/services/google_stt/__init__.py`, `client.py`
  (`GoogleSTTClient.long_running_recognize_uri(gs_uri)` →
  `[(word, start_s, end_s), …]`; raises `QuotaExceeded` on
  `gax.ResourceExhausted`; singleton `get_default_stt_client`),
  `budget.py` (`SecondsBudget` — clone of `CharBudget` in
  seconds), `aligner.py` (pure normalization + alignment).
- `bible/services/dbt/timestamps_registry.py` —
  `GENERATED_TIMESTAMPS_FILESETS` mapping base fileset id →
  reference-text descriptor (`{"reference": "LVSGLU8"}` to
  start; extensible to an API.Bible bible id later).
- `bible/services/storage/gcs.py` — add
  `generated_timestamps_object_path`,
  `upload_generated_timestamps`, `read_generated_timestamps_json`
  (with its own cache key), `list_generated_chapters`,
  `stt_staging_object_path` + upload/delete helpers,
  `read_stt_monthly_usage`/`increment_stt_monthly_usage`
  (reuse the `if_generation_match` read-modify-write pattern),
  `acquire_stt_run_lock`/`mark_stt_run_complete` (same shape as
  the TTS lock, `stt-lock` prefix).
- `bible/management/commands/generate_audio_timestamps.py` —
  flags: `--fileset-ids a,b,c` (required), `--book`, `--dry-run`,
  `--max-chapters`. Loop mirrors `_run_mode`:
  per fileset: acquire stt lock → read global usage →
  `SecondsBudget(cap - used)` → worklist → per chapter (below).

  **Worklist**: covered chapters come from
  `DBTClient.get_books(fileset_id)` — but VERIFY the live
  response shape for `LATBSLP1DA` before building on it
  (first implementation step): `chapters` is `list[int]` in one
  DBT schema and a comma-separated string in another, may be
  null, and items key on `book_id` (the existing
  `DBTClient.get_chapters` filters on `id` — a latent bug; do
  NOT copy it). Pass `verify_content=true` so the listing
  reflects the *fileset's* real coverage, not the bible's.
  Minus `list_generated_chapters` = pending work.

  Per chapter:
  1. `DBTClient.get_timestamps` — non-empty `data` on a 200 →
     skip (DBT coverage may appear later; never spend STT
     budget where DBT already serves timestamps). An exception
     or non-200 is treated as "no DBT coverage" → proceed to
     generate, but log a warning (these filesets are verified
     to lack timestamps; a raise must NOT silently skip the
     chapter every run).
  2. Pre-flight `budget.can_afford(ceil(duration))` where
     `duration` comes from the DBT chapter row — it can be
     `None`; in that case download the MP3 and measure with
     mutagen before deciding. Budget can't afford → stop,
     `reason="budget_exhausted"`.
  3. Download MP3 from `rows[0]["path"]`
     (`DbtProvider._chapter_rows`-style fetch via
     `DBTClient.get_verses`), upload to `stt_staging/`, run
     `long_running_recognize` and poll
     `operation.result(timeout=…)`. Do not set
     `sample_rate_hertz` — let STT infer it from the MP3
     header (DBT files are 44.1 kHz; an explicit wrong rate
     breaks recognition).
  4. `budget.consume(ceil(duration))` once the LRO completes
     and persist the delta via `increment_stt_monthly_usage`
     in `finally` (same never-lose-spend semantics as TTS —
     an empty-but-successful response is still billed).
  5. Align STT words to the reference text, upload
     `timestamps_generated` JSON, delete staging blob.
  6. On generic per-chapter exception: log + continue
     (TTS semantics); `QuotaExceeded`/`BudgetExceeded` → stop
     with reason.
- `bible/management/commands/run_all_jobs.py` — JOBS entry:
  `("generate_audio_timestamps",
    ["--fileset-ids",
     "LATLVRN1DA,LATLVRP1DA,LATBSLP1DA,LATBSLP2DA"],
    "dom:10")` (a different day from the TTS `dom:3` run so the
  two never contend for the job's 5400s timeout).

## Settings / env / infra

- `bible_research/settings.py`: `MONTHLY_STT_SECONDS_LIMIT`
  (default 3300 = 55 min — headroom under 60 min),
  `STT_LANGUAGE_CODE` (default `lv-LV`), `STT_MODEL` (default
  `latest_long`), all via `_audio_settings_source`.
- `terraform/all_jobs.tf`: `MONTHLY_STT_SECONDS_LIMIT=3300`
  env var on the job (overridable later).
- `terraform/audio_storage.tf`: lifecycle rule deleting
  `stt_staging/` objects after 7 days.
- Enable `speech.googleapis.com` (`google_project_service`) and
  grant the `audio-generator` SA the minimal role needed for
  `longrunningrecognize` — verify exact role during
  implementation; the SA already has bucket `objectAdmin` for
  staging.
- `requirements.txt`: add `google-cloud-speech` pinned to a
  version ≥7 days old (check `pip index`/`venv` for what the
  app currently ships).

## Tasks

backend: 1. Add `google-cloud-speech` dep + settings keys +
   STT API/role Terraform.
backend: 1a. Verify live `v4_bible_books` response shape for
   `LATBSLP1DA` (chapters list vs CSV vs null; `book_id` key;
   `verify_content=true` coverage) before building the
   worklist on it.
backend: 2. `google_stt/` package: client, budget, aligner.
backend: 3. `gcs.py` helpers (generated timestamps,
   stt_staging, stt-usage, stt-lock).
backend: 4. `timestamps_registry.py` + `DbtProvider.
   get_timestamps` fallback (incl. `-opus16` stripping).
backend: 5. `generate_audio_timestamps` command + JOBS entry
   + `all_jobs.tf` env var.
backend: 6. Tests mirroring `test_audio_quota_budget.py` /
   `test_audio_run_lock.py` / command tests (mock STT client,
   DBT client, gcs); aligner unit tests incl. low-score path;
   provider-fallback test (empty DBT → artifact → `[]`).
backend: 7. Update `bible_research/DEVELOPER_GUIDE.md`
   (timestamps section) per workspace rule 6.

No frontend changes — the `/timestamps` contract is unchanged.

## Risks / open questions

- **Free-tier throughput**: 55 min/month means the ~9 h of
  `LATLVRN1DA` NT audio takes ~10 months to cover. Accepted
  (user asked for free tier); `MONTHLY_STT_SECONDS_LIMIT` is a
  single env var away from a paid budget if that ever changes.
  Alternatives documented for the record: STT V2 dynamic batch
  (~$0.003/min, ~$2 for the whole NT) or local Whisper
  (free, no GCP) — both rejected for now.
- **GLU8≠LVR wording drift**: mitigated by fuzzy alignment +
  `alignment_score`; spot-check a chapter (e.g. John 3) in the
  implementation PR's UI-verification section.
- **Dramatized audio (`LATBSLP2DA`)**: music/voice acting hurts
  STT accuracy — kept in scope with low-confidence flagging.
- **V1 model support for lv-LV**: verify `latest_long` accepts
  `lv-LV`; fall back to `default` if not (config var).

## Verification

- Unit tests green; dry-run prints worklist/budget without GCS
  writes.
- Manual: `python manage.py generate_audio_timestamps
  --fileset-ids LATLVRN1DA --book JON --max-chapters 1` against
  a dev bucket (or prod bucket in a maintenance window),
  then `GET /api/v1/bible/timestamps/?fileset_id=LATLVRN1DA&
  book=John&chapter=3` returns `{"data":[…]}` where it used to
  be empty; UI: verse highlight follows audio on the 1953 LVR.
