# Design

## Context

`TranslationSelector.tsx` stages picks in local state until Save
writes `activeTextFilesetId`/`activeAudioFilesetId` (persisted in
`bible-storage`, zustand schema v9). On language switch the
`languageIso` effect clears staged ids; the mirror effect then
only restores when the ACTIVE selection's owner is in the viewed
language — so revisiting a previously used language shows an
empty picker.

## Goals / Non-Goals

- Goal: per-language memory of the last saved selection, restored
  as a staged pick on language switch.
- Non-goal: changing what Save does to the active filesets, or
  auto-switching the active translation without Save.

## Decisions

- **Store the memory in `bible-storage`** as a new persisted
  field `lastSelectionByLanguage: Record<string,
  { abbr: string; textId: string | null; audioId: string | null }>`
  keyed by `language_iso` — not a separate localStorage key.
  Rationale: it is selection state owned by the same store;
  `partializeState` + `migratePersistedState` already handle
  shape/versioning (bump to v10, sanitize entries: drop
  non-object values, missing/empty `abbr`, non-string ids).
- **Record on Save only**, in `handleSave`, keyed by the saved
  translation's `language_iso`. Records the full staged triple so
  an explicit audio "None" (`null`) survives; storing only the
  `abbr` and re-deriving would lose that choice.
- **Restore in the mirror/seed path**: after the language's list
  is merged and the `languageIso` clear effect has run, if no
  owner-mirror applied and a recorded entry exists whose `abbr`
  is still in `translations` for that iso, stage it (set abbr +
  text/audio ids verbatim, skipping `applyVersionSelection`
  re-derivation). Recorded member/product ids pass through the
  existing resolve-on-save paths unchanged.
- **Ordering**: restore must wait for the per-language fetch to
  land (stale entries are validated against the merged list), so
  it hooks the same `translations` dependency the mirror effect
  already uses rather than a separate fetch.

## Risks / Trade-offs

- [Recorded ids drift from the fetched list] → validated by
  `abbr` presence at restore; a stale `abbr` falls back to
  no-memory behavior instead of staging garbage.
- [Map grows unbounded across many Other languages] → bounded
  in practice (one small entry per language ever saved); no cap
  needed.
- [Restore races with the user's fresh pick] → same
  `userInteractedRef` guard style as seeding: once the user
  touches the version picker, restore stops applying.

## Migration Plan

Pure client change: persisted schema v9 → v10 with a sanitizing
migration; old blobs gain an empty map. No backend or deploy
steps; rollback is a normal revert.

## Open Questions

None.
