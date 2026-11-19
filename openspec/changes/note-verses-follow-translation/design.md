# Design

## Context

`NoteVerseList` renders `note.verses[].text` verbatim — text
the backend embedded at note serialization time in the note's
creation-time fileset. `useVerseTexts(refs, filesetId,
translations)` already knows how to fetch display text for
arbitrary refs per resolved fileset, and the main change's
`useNoteRefFilesetId()` supplies the override → selection
ordering.

## Goals / Non-Goals

**Goals:**

- Card verse text matches the resolved note-ref translation.
- Zero regression on resilience: stored text is always the
  placeholder/fallback.

**Non-Goals:**

- No backend change: `note.verses` enrichment and stored text
  stay as-is (they remain the fallback and the folded/share
  payload).
- `note.headings` stay creation-time — section headings are
  not re-fetched (a translation's own headings may not exist
  for the fileset; out of scope).
- Verse selection/toolbar behavior unchanged.

## Decisions

### D1 — Overlay, don't replace

`NoteVerseList` maps `note.verses` to `VerseRef[]`, feeds them
through `useVerseTexts` with `useNoteRefFilesetId()`, and
overlays the returned `text` onto each rendered verse keyed by
`(bookId, chapter, verse)`. Rendering always starts from
stored text; the fetched text swaps in when it arrives. Folded
and expanded views share the same overlay source so both stay
consistent.

### D2 — Book-name keying via USFM

`note.verses[].book` may be a display name; convert with
`toUsfmCode` for the `VerseRef` and for keying the overlay —
the same trap as the popover's `resolveTextFileset` call.

### D3 — Failures degrade silently to stored text

`useVerseTexts` already swallows fetch failures and returns
`[]` — exactly the desired fallback posture, so no new error
surface is needed. `note.error`/`error_code` rendering stays
unchanged for the genuinely-broken upstream case.

## Risks / Trade-offs

- Extra fetch per rendered card — bounded: `useVerseTexts`
  batches per book/chapter, `getVersesInChapter` is
  cache-first, and cards already re-render lazily via the
  intersection observer.
- Stale-text flash: stored translation shows until the fetch
  lands — accepted, per the spec's fallback requirement.
- Copyright caching cap is unchanged — verses go through the
  same cacheManager LRU as the popover.

## Migration Plan

Frontend-only; lands after `add-note-ref-translation-setting`
(consumes its hook). Rollback = revert to stored-text
rendering.
