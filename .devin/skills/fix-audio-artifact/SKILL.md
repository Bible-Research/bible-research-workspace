---
name: fix-audio-artifact
description: Validate and repair a corrupted generated-audio
  artifact (e.g. a user reports "book.chapter:verse audio is
  broken / makes a weird sound"). Covers the served-artifact
  lookup, deep-audit validation, manifest-driven TTS repair,
  the Netskope gRPC workaround, and post-repair verification.
  Invoke when the user reports broken Latvian Glück (LVSGLU8)
  audio or asks to regenerate damaged audio files.
---

# Fix a corrupted generated-audio artifact

User reports like "Genesis 18:28 audio is broken / has a weird
breaking sound" are almost always **content-level TTS defects
baked into the stored MP3** (hard-clipped speech onsets), not
structural corruption and not delivery bugs. The full pipeline
exists: audit → manifest → in-place regeneration.

Reference docs: `bible_research/DEVELOPER_GUIDE.md`
"Audio Artifact Audit" → "Remediation workflow". Origin:
Sentry issue 158588239 (NUM.24:14), built in PRs
#122/#124/#126.

## 0. Preconditions

Work in `bible_research/`'s MAIN checkout — this is ops work
(running management commands), not code changes, so no
worktree is needed unless the run reveals a code bug to fix.

- `export PATH="/opt/homebrew/bin:$PATH"` on every exec call —
  `gcloud`/`gh` are not on agent-shell PATH.
- **Netskope CA bundle — export FIRST, before any Python
  egress.** The local machine sits behind a TLS-intercepting
  proxy; Python HTTPS (GCS downloads in the audit, pip) can
  fail or hang without it. Build once, then export on every
  call:

      CERTIFI=$(venv/bin/python -c \
        "import certifi; print(certifi.where())")
      cat "$CERTIFI" \
        "/Library/Application Support/Netskope/STAgent/data/nscacert.pem" \
        > /tmp/combined-ca-bundle.pem
      export REQUESTS_CA_BUNDLE=/tmp/combined-ca-bundle.pem \
             SSL_CERT_FILE=/tmp/combined-ca-bundle.pem \
             CURL_CA_BUNDLE=/tmp/combined-ca-bundle.pem \
             PIP_CERT=/tmp/combined-ca-bundle.pem \
             GRPC_DEFAULT_SSL_ROOTS_FILE_PATH=/tmp/combined-ca-bundle.pem

- The main checkout's `venv/` must hold the dev-only decoder:
  `venv/bin/pip install "miniaudio==1.73"` (pinned in
  `requirements-dev.txt`; `--deep` fails fast without it).
- GCS + TTS auth is ADC — `gcloud auth application-default
  print-access-token` must succeed; the signed-in account
  (user's) needs bucket write + Cloud TTS on the project.
- The CA bundle above fixes HTTPS egress; the gRPC TTS call
  still fails through the proxy — see §4.

## 1. Identify the artifact that is actually served

The audio bucket is `{GCP_PROJECT}-bible-audio`
(`bible-research-489314-bible-audio`). Layout:

    {prefix}/{FILESET}/{VOICE}/{BOOK}/{CH}_audio.mp3
    {prefix}/{FILESET}/{VOICE}/{BOOK}/{CH}_timestamps.json

- `LVSGLU8C1DA` → `audio/` prefix, verse-synthesis mode —
  the fileset the frontend requests for Glück audio today.
- `LVSGLU8C2DA` → `audio_chapter/` prefix, chapter mode —
  also served; check its artifacts if the report persists.
- `audio_gemini/` and `audio_edge/` exist (verify coverage
  with `gcloud storage ls`) but **nothing serves them** —
  do not repair them unless the user says otherwise;
  `repair_audio_files` reports them `unsupported_engine`
  anyway.
- Voice dir comes from the blob path itself; the configured
  voice is `lv-LV-Chirp3-HD-Sadachbia` (settings default).

Locate the file with
`gcloud storage ls gs://{BUCKET}/audio/{FILESET}/{VOICE}/{BOOK}/`
— never guess paths.

## 2. Validate the report before touching anything

A chapter MP3 holds all verses — the report names one verse,
the defect is usually one clipped segment. Confirm it:

    cd bible_research
    venv/bin/python manage.py audit_audio_files \
      --prefix audio --book GEN --deep \
      --report /tmp/audit.json

`--book` scopes to one book (~50 MP3s); `--deep` decodes each
MP3 and analyses **per verse** using the sidecar. Output like
`clipped: verse 28: 350 near-full-scale sample(s), longest
run 5` — check the flagged verse matches (or is adjacent to)
the reported one.
Structural-only defects (desync/truncation) surface without
`--deep`; content defects (clipping, dead silence) need it.
The `--report` JSON is the manifest the repair step consumes —
running the whole book also catches siblings (GEN.18's audit
also found a clipped GEN.50:26).

## 3. Repair (dry-run first, always)

    venv/bin/python manage.py repair_audio_files \
      --manifest /tmp/audit.json            # dry-run plan
    venv/bin/python manage.py repair_audio_files \
      --manifest /tmp/audit.json --apply    # regenerate

- Granularity is the whole chapter pair — both artifacts are
  regenerated together (sidecar first, then the MP3
  overwrites in place; a failed regen leaves the old blob
  servable).
- Repair spends real Cloud TTS characters against
  `MONTHLY_TTS_CHAR_LIMIT` — the dry-run prints remaining
  budget.
- Successful repairs are ledgered under
  `state/repair/<artifact-base>.json`; a re-run skips them
  (`--force` overrides).

## 4. The Netskope gRPC failure (you WILL hit this locally)

`--apply` prints `FAILED {name}  regen_failed:
ServiceUnavailable` per entry, and the logged traceback
carries the real signature: `503 … missing selected ALPN
property`. The TLS-intercepting proxy drops ALPN, which gRPC
requires — CA bundles do NOT help here. Run the apply step
through an injected REST-transport client (the §0 CA env
vars must already be exported — REST honours
`REQUESTS_CA_BUNDLE`/`SSL_CERT_FILE`):

    venv/bin/python manage.py shell
    >>> from google.cloud import texttospeech
    >>> from google.cloud.texttospeech_v1.services.text_to_speech \
            import transports
    >>> from bible.services.google_tts import client as tts_mod
    >>> tts_mod._default = tts_mod.GoogleTTSClient(
            client=texttospeech.TextToSpeechClient(
                transport=transports.TextToSpeechRestTransport()))
    >>> from django.core.management import call_command
    >>> call_command("repair_audio_files",
                     manifest="/tmp/audit.json", apply=True)

Chapter-mode (`audio_chapter/`) entries additionally need a
REST `_beta_client` seeded the same way from
`google.cloud.texttospeech_v1beta1.services.text_to_speech`.

## 5. Verify

Re-download the artifacts and re-run the deep audit on them —
TTS output is non-deterministic and CAN clip again:

    gcloud storage cp gs://{BUCKET}/audio/.../{CH}_audio.mp3 \
      gs://{BUCKET}/audio/.../{CH}_timestamps.json /tmp/verify/

then deep-check those bytes (`audio_deep.decode_mp3_bytes` +
`analyze_pcm` with the parsed sidecar) or just re-run the
`--deep` audit for the book — it must report **zero suspects**
before you tell the user it's fixed. Still clipped → re-run
repair with `--force` (the ledger otherwise skips it).

## 6. Report

Tell the user: which artifact/verse was confirmed corrupt,
issue code found, that it was regenerated in place (chars
spent), verification result, and suggest a listening spot-check
at `bibleresearch.vercel.app/bible/{BOOK}.{CH}` near the verse.

## Guardrails

- NEVER delete blobs (`gcloud storage rm`) as a fix — repair
  regenerates in place. Deletion only helps if a generation
  run is triggered afterwards, and the monthly job has no
  scheduler today — repair is always the safer path.
- Repair writes PROD artifacts — always show the dry-run plan
  first; only `--apply` entries the audit actually flagged.
- `audio_chapter/` (C2DA) artifacts: same flow, but `--prefix
  audio_chapter` on the audit and REST beta-client for apply.
