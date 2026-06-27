# Bible Search Feature — Implementation Plan

## 1. Goal

Replace the current `SearchModal` (a local KJV-JSON autocomplete jump-to-verse
modal) with a **full-page search experience** powered by the backend
`BibleSearchView` (which proxies DBT's `/search` endpoint via
`DBTClient.search`).

Required behaviour:

1. Dedicated `/search` route, not a modal.
2. Results grouped **by book**, books ordered in canonical (ascending) Bible
   order.
3. Each verse (or grouped block) has a **play button** that plays the chapter
   audio scoped to that verse — reusing the playlist mechanism that
   `TagNotesRoute` already drives via `setAudioPlaylistItems`.
4. Decommission the existing modal-based search entirely.

---

## 2. Current State Summary

### Backend (already done — reuse as-is)

- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/bible/services/dbt/client.py:318-373`
  — `DBTClient.search(bible_id, query, limit, page, sort_by, books)` wraps
  `GET https://b4.dbt.io/api/search`.
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/bible_research/bible/views.py:355-479`
  — `BibleSearchView` (DRF `APIView`) at `bible/search/`. Returns
  `{ data: { verses: [{book_id, chapter, verse_start, verse_text}], meta: { pagination: {...} } } }`.
  Supports `query`, `fileset_id`, `limit`, `page`, `sort_by`, `books`.
- Also supports SWORD via `_sword_search` (out of scope here; route works
  for both transparently).

**No backend changes are required.** Verified the response shape already
contains everything the frontend needs (`book_id` USFM, `chapter`,
`verse_start`, `verse_text`).

### Frontend (to be removed / replaced)

- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/components/SearchModal.tsx`
  — modal-based autocomplete over local KJV JSON. **Delete.**
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/components/SearchControl.tsx`
  — currently unused stylized button. **Delete** (verify no other refs).
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/App.tsx:39-65,116`
  — owns the `useDisclosure` modal state and `/` keydown shortcut. **Rewire**
  to navigate to `/search` and open `SearchModal` is removed.
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/components/MyHeader.tsx`
  — search icon button calls `open` prop. **Rewire** to `navigate('/search')`.

### Frontend (audio playlist machinery to reuse — no changes)

- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/types.ts:64-` — `PlaylistItem`
  shape (`itemId`, `book`, `chapter`, `startVerse`, `endVerse`, `label`).
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/hooks/useAudioPlaylist.ts`
  — already source-agnostic; works with any `PlaylistItem[]` (proven by
  its own tests: *"works with arbitrary PlaylistItem shapes"*).
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/components/Audio.tsx:18-26`
  — picks up `store.audioPlaylistItems` automatically and shows playlist
  controls in the header.
- `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/store.tsx`
  — `setAudioPlaylistItems` + `activeAudioFilesetId` already exist.

This is the same pattern used in `TagNotesRoute`'s
`useEffect`@`@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/routes/TagNotesRoute.tsx:313-344`.
We will mirror it.

### Frontend (book ordering)

- `BOOK_NAME_TO_ORDER` exists keyed by lowercase name
  (`@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/utils/bibleUtils.ts:82-89`).
- DBT returns `book_id` as USFM code (e.g. `JHN`). We need a
  **code→order** and **code→display name** map. Add small derived maps
  next to the existing ones (no duplication of `BIBLE_BOOKS`).

---

## 3. Recommended Architecture

### 3.1 New backend-facing API helper (frontend)

Add to `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/api.tsx`:

```ts
export interface SearchVerse {
  book_id: string;     // USFM e.g. 'JHN'
  chapter: number;
  verse_start: number;
  verse_text: string;
}

export interface SearchPagination {
  total: number;
  count: number;
  per_page: number;
  current_page: number;
  total_pages: number;
}

export interface SearchResponse {
  verses: SearchVerse[];
  meta: { pagination?: SearchPagination };
}

export const searchBible = async (
  query: string,
  filesetId: string,
  page = 1,
  limit = 50,
): Promise<SearchResponse> => { /* GET /api/v1/bible/search/?... */ };
```

Notes:
- Use existing `API_BASE_URL` pattern.
- No auth required (search is public; backend doesn't enforce auth here).
- Add **AbortController** so rapid typing cancels stale requests.

### 3.2 New route `/search`

Create `@/Users/tedis.rozenfelds/personal_data/p_projects/Bible Research/reactive-bible/src/routes/SearchRoute.tsx`:

Responsibilities:
1. Read `?q=` and `?page=` from URL (`useSearchParams`) — URL is source of
   truth → shareable links + back/forward navigation.
2. Render search input bound to `q` (debounced 300 ms before pushing to URL).
3. Use `activeTextFilesetId` from store as `fileset_id`.
4. Fetch via `searchBible` in `useEffect([q, filesetId, page])`. Show
   loading / error / empty states.
5. Group results by `book_id`, sort book groups by USFM order
   (`BOOK_CODE_TO_ORDER` — new tiny derived map).
6. Inside each group, sort by `chapter`, then `verse_start`.
7. Render groups as collapsible `Accordion` (Mantine), each item showing
   `{book} {chapter}:{verse_start}` + verse text + a play `ActionIcon`.
8. Clicking the verse text → navigate to
   `/bible/{book_name}/{chapter}` and set `setActiveVerses([verse_start])`.
9. Clicking play on a single verse → set
   `audioPlaylistItems = [singleItemForThatVerse]` and rely on `Audio.tsx`
   to start playback. (Same UX as Notes view.)
10. Optional: a "Play all results" button → push **all** verses as a single
    playlist in book-canonical order.
11. Pagination controls bound to `meta.pagination`.

### 3.3 `PlaylistItem` construction from a search hit

```ts
const toPlaylistItem = (v: SearchVerse, i: number, total: number): PlaylistItem => ({
  itemId: `search-${v.book_id}-${v.chapter}-${v.verse_start}`,
  book: BOOK_CODE_TO_NAME[v.book_id],         // store expects name
  chapter: v.chapter,
  startVerse: v.verse_start,
  endVerse: v.verse_start,
  label: `Result ${i + 1}/${total} – ${BOOK_CODE_TO_NAME[v.book_id]} ` +
         `${v.chapter}:${v.verse_start}`,
});
```

Add to `bibleUtils.ts`:

```ts
export const BOOK_CODE_TO_NAME = BIBLE_BOOKS.reduce((acc, b) => {
  acc[b.code] = b.name; return acc;
}, {} as Record<string, string>);

export const BOOK_CODE_TO_ORDER = BIBLE_BOOKS.reduce((acc, b, i) => {
  acc[b.code] = i; return acc;
}, {} as Record<string, number>);
```

Display name capitalisation: `BIBLE_BOOKS.name` is lowercase. For display,
wrap with a small `titleCase()` helper or extend the table with a
`displayName` field. **Recommended**: add `displayName` (cleaner, no
runtime regex), but as a minimal first pass use a tiny title-case utility
to avoid touching the data table.

### 3.4 Cleanup

- Delete `SearchModal.tsx` and `SearchControl.tsx`.
- Remove imports + `useDisclosure` modal wiring from `App.tsx`.
- In `App.tsx` keydown handler: replace `modalFn.open()` with
  `navigate('/search')` (move handler into a component that has
  `useNavigate`, e.g. wrap it in `AppRoutes` or do `window.location` push;
  cleanest: a small `useKeyboardShortcuts()` hook called inside a child
  of `<BrowserRouter>`).
- Update `MyHeader` `open` prop to a `onSearchClick` callback that
  navigates to `/search` (or replace prop usage with `useNavigate` inside
  the header itself — simpler).
- Add `<Route path="/search" element={<SearchRoute />} />` to
  `routes/index.tsx`.

### 3.5 Tests

- Add `routes/__tests__/SearchRoute.test.tsx`:
  - renders, debounced fetch fires once
  - groups results by book in canonical order
  - clicking play sets `audioPlaylistItems` with a single-verse item
  - clicking "Play all" sets a multi-item playlist in book order
  - pagination updates URL
- Add `api` test for `searchBible` (mock fetch, asserts URL params).
- Snapshot/visual: not required.

### 3.6 Out of scope (future enhancements)

- Search-within-book filter (DBT supports `books` param).
- Highlighting matched terms in verse text.
- Persisted recent searches.
- Server-side relevance ranking (DBT `sort_by`).

---

## 4. File-by-File Change List

| File | Action |
|---|---|
| `reactive-bible/src/api.tsx` | **Add** `searchBible()` + types |
| `reactive-bible/src/utils/bibleUtils.ts` | **Add** `BOOK_CODE_TO_NAME`, `BOOK_CODE_TO_ORDER` |
| `reactive-bible/src/routes/SearchRoute.tsx` | **Create** |
| `reactive-bible/src/routes/index.tsx` | **Add** `/search` route |
| `reactive-bible/src/App.tsx` | **Remove** modal + `useDisclosure`; rewire `/` shortcut to navigate |
| `reactive-bible/src/components/MyHeader.tsx` | **Rewire** search button to `navigate('/search')`; drop `open` prop |
| `reactive-bible/src/components/SearchModal.tsx` | **Delete** |
| `reactive-bible/src/components/SearchControl.tsx` | **Delete** (verify unused) |
| `reactive-bible/src/routes/__tests__/SearchRoute.test.tsx` | **Create** |

Backend: **no changes**.

---

## 5. Suggested Implementation Order

1. Add `searchBible` API helper + `BOOK_CODE_TO_*` maps (small, isolated).
2. Build `SearchRoute` with grouping + pagination, no audio yet.
3. Wire route + delete modal/control + update header + `/` shortcut.
4. Hook up `setAudioPlaylistItems` (per-verse play + play-all).
5. Tests.
6. Manual QA against a deployed backend (`ENGESH` / `ENGESV` fileset).

---

## 6. Risks / Open Questions

- **Audio coverage for OT/NT-only filesets**: `useAudioPlaylist` already
  uses `resolveTimestampsFilesetId` + `filesetCoversTestament` to handle
  this; nothing extra needed.
- **DBT search across all books at once**: confirmed via
  `BibleSearchView` proxy — returns mixed books in one page. Frontend
  must group client-side (done in plan).
- **Sort order across pages**: DBT pagination ordering is unspecified.
  Since requirement is *"book ascending order"*, grouping is per-page.
  If a single canonical full-corpus ordering is required, we'd need to
  fetch all pages up front (size could be large) — recommend **per-page
  grouping** initially and revisit if UX feedback demands a global sort.
- **`/` keydown shortcut**: must not fire while focus is in input
  elements (existing handler doesn't guard against this; carry forward
  the same behaviour or add a guard while we're touching it).
