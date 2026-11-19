# Design

## Context

`resolveTextFileset` (`src/utils/filesetGroups.ts`) resolves a
stored id through `findTextOption` — exact option id → member
id → drifted `{abbr}:{kind}` prefix — and falls back to
returning the raw `storedId`. That passthrough is correct for
raw DBT ids but wrong for product ids: `{abbr}:text:{n}` is a
client-side construct, `:` fails `FILESET_ID_RE`, and the API
answers 400 `invalid_parameter`. `PassageView` treats a `null`
resolution as "no text selection" and blanks the chapter, so
the reader needs an explicit recovery path, not just a null.

## Goals / Non-Goals

**Goals:**

- Product-shaped ids that resolve to nothing never reach the
  wire.
- A broken persisted reader selection self-heals with a
  visible notice.
- Note-reference surfaces fall through their existing chain.

**Non-Goals:**

- No change to valid id resolution (members, `_ET` splits,
  `ENGKJV` fast path).
- No backend change — `FILESET_ID_RE` stays strict.
- No retroactive cleanup of already-persisted bad values
  beyond the self-heal write.

## Decisions

### D1 — `:` is the product-id marker

`resolveTextFileset` returns `null` (unresolvable) when the
stored id contains `:` and `findTextOption` misses all three
lookup paths. Plain unknown ids keep passing through — a raw
id like `ENGESVN2DA` may be a perfectly routable DBT fileset
the merged list simply doesn't know. `resolveAudioFileset`
gets the same treatment (`findAudioOption` miss + `:` →
unresolvable) — it currently leaks orphaned `FOO:audio:1`
ids to the audio endpoints identically.

### D2 — Reader self-heal lives in the store subscriber

A `useBibleStore.subscribe` guard (same pattern as the
existing `loadStructure` subscription) watches
`activeTextFilesetId` + `translations`: when the selection is
product-shaped and unresolvable once `translations` is
populated, it writes the locale default —
`DEFAULT_TRANSLATION_BY_LOCALE[resolveUiLanguage()]` (or a
new `resolveLocaleDefaultTextFileset()` helper) — NOT
`resolveInitialTextFileset()`, which reads back the persisted
broken value, and fires one Mantine notification. Doing it at
the store layer covers every entry point (PassageView,
pickers, prefetch) and keeps PassageView's `null → empty`
branch for the genuinely-no-selection case. Guard against
re-fire loops: the reset id is always resolvable, and a
`translations` refresh that re-unresolves a (different) value
is the only legitimate re-trigger. The vanilla `subscribe`
misses hydration (registered after `create`), so the guard
also needs a one-shot kick on startup — e.g. the same check
invoked once after store creation or on first
`setTranslations`.

### D3 — Note-ref chain skips unresolvable ids

`useNoteRefFilesetId` treats an unresolvable pin as absent
(fall through to the selection), and an unresolvable
selection as absent (fall through to `ENGESV_API`). The
settings page shows a "pinned translation unavailable" state
with a clear/reset action when its stored pin doesn't
resolve.

### D4 — `?? storedId` re-leaks get plugged at the call sites

Two consumers re-leak a null resolution back into a request:
`resolveNotesFilesetId` (`notesSlice.ts`) does
`resolveTextFileset(...) ?? storedId` — return `undefined`
instead, so `GET /notes/` drops a broken `fileset_id` rather
than proxying it upstream — and `useVerseTexts` does
`resolveTextFileset(...) ?? filesetId` — skip the fetch on
`null` instead of issuing a guaranteed-400 request.

## Risks / Trade-offs

- Resetting `activeTextFilesetId` mutates a user preference —
  mitigated by the notification and by only firing when the
  value is provably unroutable (product-shaped AND owner
  absent), never on a mere coverage gap.
- Self-heal depends on `translations` being loaded; the
  subscriber must wait for a non-empty list before judging a
  product id unresolvable, or every cold start resets.

## Migration Plan

Frontend-only via the `staging`-first flow. Rollback = revert
— worst case returns to the 400 behavior.
