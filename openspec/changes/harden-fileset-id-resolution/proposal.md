# Proposal

## Why

A stored text selection can hold a product id
(`{abbr}:text:{n}`) whose owning translation later disappears
from the merged `translations` list (cleared storage, list
drift). `resolveTextFileset` then passes the raw id through to
the API, but `:` fails the backend's `FILESET_ID_RE` charset —
the request 400s `invalid_parameter` instead of degrading like
any other stale selection. Product ids are client-side
constructs and are never routable upstream.

## What Changes

- `resolveTextFileset` stops passing through unresolvable
  product-shaped ids: an id containing the `{abbr}:{kind}:{n}`
  separator that matches no text option (exact, member, or
  drifted-prefix lookup) resolves to `null` instead of leaking
  to the API. Non-product unknown ids keep passing through —
  raw DBT fileset ids are routable.
- Callers' null handling applies per surface: the note-ref
  preview falls back through its override → selection → ESV
  chain; the reader treats an unresolvable stored selection as
  a broken persisted value and self-heals to the locale-default
  fileset (one-time, with a notification) rather than
  blanking the chapter.
- The settings-page pin uses the same rule: an unresolvable
  pinned product id behaves as "no pin" and the settings page
  offers to clear it.

## Capabilities

### New Capabilities

- `fileset-resolution`: how stored client fileset selections
  (product ids, member ids, raw ids) resolve to concrete
  per-book provider filesets, and how drifted or unroutable
  selections degrade.

### Modified Capabilities

<!-- None. -->

## Impact

- `reactive-bible` only: `src/utils/filesetGroups.ts`,
  `src/components/PassageView.tsx` (or a store-level self-heal
  subscription), note-ref resolver consumers, settings page.
- No backend or API changes; the change prevents a class of
  avoidable 400s.
