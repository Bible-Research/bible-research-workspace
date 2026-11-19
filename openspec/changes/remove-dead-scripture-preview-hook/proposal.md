# Proposal

## Why

`src/hooks/useScripturePreview.ts` is dead code: the
`useScriptureRefPreview` popover flow superseded it and only
its own test file still imports it. Dead hooks drift out of
sync with the real preview behavior and waste test time.

## What Changes

- Delete `src/hooks/useScripturePreview.ts` and
  `src/hooks/useScripturePreview.test.ts`.
- No behavior change — this is a pure cleanup, so the change
  declares `skip_specs: true`.

## Capabilities

### New Capabilities

<!-- None — no spec-level behavior change. -->

### Modified Capabilities

<!-- None — no spec-level behavior change. -->

## Impact

- `reactive-bible` only; two files removed. Verified by
  confirming no other imports exist and the suite stays green.
