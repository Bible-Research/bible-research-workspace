# Display Translation Copyright at End of Chapter — AI Agent Implementation Plan

> **Audience:** Windsurf AI agent (Cascade).
> Execute each step in order. Do not skip ahead. After each step, verify the result before proceeding. Mark each step done before moving to the next.

---

## Context & Background

Two repos live under `~/personal_data/p_projects/Bible Research/`:

- **`bible_research/`** — Django REST API. Proxies the DBT v4 API.
- **`reactive-bible/`** — React + Mantine + Zustand frontend.

**Goal:** Display copyright information for the active Bible translation at the bottom of every chapter view. This is both a legal requirement and a user-facing feature.

### DBT Copyright API (researched)

```
GET /bibles/{bible_id}/copyright?v=4
```

- `bible_id` = the Bible abbreviation (e.g. `ENGESV`, `ENGESH`). This is the `abbr` field returned by the `/bibles` endpoint and stored as `Translation.abbr` in the frontend.
- The generated client already has `BiblesApi.v4_bible_copyright(bible_id, v)` in `bible_research/bible/services/dbt/dbt_client/openapi_client/api/bibles_api.py`.

**Response shape** (`v4_bible.copyright`) — array of fileset copyright objects:
```json
[
  {
    "id": "ENGESV",
    "type": "text_plain",
    "size": "C",
    "copyright": {
      "copyright_date": "2001",
      "copyright": "© 2001 Crossway Bibles",
      "copyright_description": "The Holy Bible, English Standard Version...",
      "open_access": 0
    }
  },
  {
    "id": "ENGESVO2DA",
    "type": "audio",
    "size": "OT",
    "copyright": { ... }
  }
]
```

Each fileset under the Bible has its own copyright entry. For displaying at chapter end, we want the **text fileset's copyright** (matching the active text fileset).

### Key identifiers in the frontend

- `Translation.abbr` — the `bible_id` for the copyright API (e.g. `ENGESV`)
- `activeTextFilesetId` — the specific text fileset (e.g. `ENGESH`). This may or may not equal `abbr`.
- The frontend already has `translations` in the Zustand store, so we can look up the `abbr` for the currently active text fileset.

### Design rules

- **Cache aggressively:** Copyright data is static per Bible. Cache it permanently (no expiration).
- **Backend proxies DBT:** Frontend never calls DBT directly.
- **Lines ≤ 79 chars** in all Python files.
- **Graceful degradation:** If copyright fetch fails, chapter still renders normally with no copyright footer.

---

## Step 1 — Backend: `DBTClient.get_copyright()`

### 1a. Edit `bible_research/bible/services/dbt/client.py`

Add a new method after `get_timestamps()` and before `search()`:

```python
def get_copyright(
    self,
    bible_id: str,
    **kwargs
) -> Dict[str, Any]:
    """Get copyright info for a Bible."""
    return self._make_request(
        self.bibles_api.v4_bible_copyright,
        bible_id,
        **kwargs
    )
```

No new imports needed — `BiblesApi` is already imported and instantiated as `self.bibles_api`.

### 1a — Test

Edit `bible_research/bible/tests/test_dbt_client.py`. Add:

```python
def test_get_copyright_calls_api(dbt_client):
    """
    Test that get_copyright calls v4_bible_copyright
    with correct args.
    """
    dbt_client.bibles_api.v4_bible_copyright = Mock(
        return_value=[{"id": "ENGESV", "copyright": {}}]
    )

    result = dbt_client.get_copyright("ENGESV")

    dbt_client.bibles_api.v4_bible_copyright \
        .assert_called_once_with("ENGESV", v=4)
    assert result == [
        {"id": "ENGESV", "copyright": {}}
    ]
```

Run: `cd bible_research && python -m pytest bible/tests/test_dbt_client.py::test_get_copyright_calls_api -v`

---

## Step 2 — Backend: `CopyrightView` endpoint

### 2a. Edit `bible_research/bible/views.py`

Add a new view class after `AudioTimestampView`:

```python
class CopyrightView(APIView):
    """Return copyright info for a Bible translation."""

    def get(self, request, format=None):
        bible_id = request.query_params.get('bible_id')

        if not bible_id:
            return Response(
                {"error": "bible_id is required."},
                status=status.HTTP_400_BAD_REQUEST
            )

        try:
            dbt_client = DBTClient()
            result = dbt_client.get_copyright(bible_id)

            # result is a list of fileset copyright
            # objects. Normalize into a clean response.
            filesets = []
            for item in (result or []):
                cr = item.get("copyright") or {}
                filesets.append({
                    "id": item.get("id"),
                    "type": item.get("type"),
                    "size": item.get("size"),
                    "copyright": cr.get("copyright", ""),
                    "copyright_date": cr.get(
                        "copyright_date", ""
                    ),
                    "copyright_description": cr.get(
                        "copyright_description", ""
                    ),
                })
            return Response({"data": filesets})
        except Exception as e:
            logger.exception(
                "Error fetching copyright: %s", e
            )
            return Response(
                {"error": str(e)},
                status=status.HTTP_400_BAD_REQUEST
            )
```

### 2b. Edit `bible_research/bible/urls.py`

Add the route:
```python
path(
    'bible/copyright/',
    views.CopyrightView.as_view(),
    name='bible-copyright'
),
```

### 2b — Test

Create `bible_research/bible/tests/test_copyright_view.py`:

- Mock `DBTClient.get_copyright` to return sample data with nested `copyright` objects.
- Use DRF's `APIRequestFactory` to call the view.
- Assert 200 response with correct flattened `{ "data": [...] }` shape.
- Assert 400 when `bible_id` is missing.
- Assert 400 when DBT raises an exception.

Run: `cd bible_research && python -m pytest bible/tests/test_copyright_view.py -v`

---

## Step 3 — Backend: Manual API test

### 3a. Start the Django dev server

```bash
cd bible_research
source venv/bin/activate
python manage.py runserver 8000
```

### 3b. Test with curl

```bash
# ESV
curl "http://localhost:8000/api/v1/bible/copyright/?bible_id=ENGESV"

# ESH (English Standard Hearing — used in default frontend config)
curl "http://localhost:8000/api/v1/bible/copyright/?bible_id=ENGESH"

# KJV
curl "http://localhost:8000/api/v1/bible/copyright/?bible_id=ENGKJV"
```

### 3c. Verify and record

- Confirm the response has `"data": [...]` with `copyright`, `copyright_date`, `copyright_description` fields.
- Note which `type` values appear (e.g. `text_plain`, `audio`, `audio_drama`). The frontend will filter for the text fileset's copyright.
- Check whether `bible_id` = `Translation.abbr` works for all translations. Record any discrepancies.
- **Important:** Check if the `abbr` used in the frontend matches what DBT expects. The translations endpoint returns `abbr` from the `/bibles` response, which is the same as `Bible.id` — so it should work as `bible_id`.

Stop the dev server after testing.

---

## Step 4 — Frontend: TypeScript types

### 4a. Edit `reactive-bible/src/types.ts`

Append at the end of the file:

```typescript
export interface FilesetCopyright {
  id: string;
  type: string;
  size: string;
  copyright: string;
  copyright_date: string;
  copyright_description: string;
}
```

No tests needed for types.

---

## Step 5 — Frontend: Copyright cache

### 5a. Edit `reactive-bible/src/utils/cacheManager.ts`

Add a new section after the existing cache sections (e.g. after CACHE STATS or after timestamp cache if it exists).

Key: `bible_copyright_cache`

Interface:
```typescript
interface CopyrightCache {
  [bibleId: string]: FilesetCopyright[];
}
```

Functions to add:
- `getCachedCopyright(bibleId: string): FilesetCopyright[] | null`
- `cacheCopyright(bibleId: string, data: FilesetCopyright[]): void`
- `clearCopyrightCache(): void`

Cache key format: just `bibleId` (e.g. `ENGESV`).
No expiration — copyright is immutable.

Import `FilesetCopyright` from `'../types'`.

### 5a — Test

Create `reactive-bible/src/utils/__tests__/cacheManager.copyright.test.ts`:
- Test `cacheCopyright` stores data and `getCachedCopyright` retrieves it.
- Test cache miss returns `null`.
- Test `clearCopyrightCache` removes all entries.
- Mock `localStorage`.

Run: `cd reactive-bible && npx vitest run src/utils/__tests__/cacheManager.copyright.test.ts`

---

## Step 6 — Frontend: API function

### 6a. Edit `reactive-bible/src/api.tsx`

Merge these into the existing import blocks at the top of the file:

```typescript
import { VerseTimestamp, FilesetCopyright } from './types';
import {
  getCachedTimestamps,
  cacheTimestamps,
  getCachedCopyright,
  cacheCopyright,
} from './utils/cacheManager';
```

(If `VerseTimestamp`/timestamp cache imports already exist from the audio highlighting plan, just add the new ones.)

Add the function after the audio/timestamp functions, before the Notes/Tags section:

```typescript
/**
 * Fetch copyright info for a Bible translation.
 * @param bibleId - The Bible abbreviation (e.g. "ENGESV")
 * @returns Array of fileset copyright objects
 */
export const getCopyrightInfo = async (
  bibleId: string
): Promise<FilesetCopyright[]> => {
  const cached = getCachedCopyright(bibleId);
  if (cached) {
    return cached;
  }

  try {
    const url =
      `${API_BASE_URL}/api/v1/bible/copyright/` +
      `?bible_id=${encodeURIComponent(bibleId)}`;
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(
        `Copyright fetch failed: ${response.statusText}`
      );
    }
    const data = await response.json();
    const copyrights: FilesetCopyright[] = data.data || [];
    cacheCopyright(bibleId, copyrights);
    return copyrights;
  } catch (error) {
    console.warn('Failed to fetch copyright info:', error);
    return [];
  }
};
```

### 6a — Test

Create `reactive-bible/src/__tests__/api.copyright.test.ts`:
- Mock `fetch` and `getCachedCopyright`.
- Test cache hit returns cached data without fetch.
- Test successful fetch returns and caches data.
- Test fetch failure returns `[]`.

Run: `cd reactive-bible && npx vitest run src/__tests__/api.copyright.test.ts`

---

## Step 7 — Frontend: `CopyrightNotice` component

### 7a. Create `reactive-bible/src/components/CopyrightNotice.tsx`

```typescript
import { useEffect, useState } from 'react';
import { Text, Box } from '@mantine/core';
import { useBibleStore } from '../store';
import { getCopyrightInfo } from '../api';
import { FilesetCopyright } from '../types';
import { shallow } from 'zustand/shallow';

const CopyrightNotice = () => {
  const { activeTextFilesetId, translations } = useBibleStore(
    (state) => ({
      activeTextFilesetId: state.activeTextFilesetId,
      translations: state.translations,
    }),
    shallow
  );
  const [copyright, setCopyright] = useState<string>('');

  useEffect(() => {
    if (!activeTextFilesetId || translations.length === 0) {
      setCopyright('');
      return;
    }

    // Find the bible_id (abbr) for the active text fileset
    const translation = translations.find((t) =>
      t.filesets.some((f) => f.id === activeTextFilesetId)
    );
    if (!translation) {
      setCopyright('');
      return;
    }

    const bibleId = translation.abbr;

    getCopyrightInfo(bibleId).then((data) => {
      // Find the text fileset's copyright
      const textCr = data.find(
        (c) => c.id === activeTextFilesetId
      );
      // Fallback: use first text_plain type, then first entry
      const cr =
        textCr ||
        data.find((c) => c.type === 'text_plain') ||
        data[0];

      if (cr) {
        setCopyright(
          cr.copyright_description || cr.copyright || ''
        );
      } else {
        setCopyright('');
      }
    });
  }, [activeTextFilesetId, translations]);

  if (!copyright) return null;

  return (
    <Box py="md" px="sm">
      <Text size="xs" color="dimmed" align="center" italic>
        {copyright}
      </Text>
    </Box>
  );
};

export default CopyrightNotice;
```

### 7a — Test

Create `reactive-bible/src/components/__tests__/CopyrightNotice.test.tsx`:
- Mock `getCopyrightInfo` to return sample data.
- Mock the Zustand store with `activeTextFilesetId: 'ENGESV'` and a matching translation.
- Render the component, assert the copyright text appears.
- Test with empty translations → assert nothing renders.
- Test with API returning `[]` → assert nothing renders.

Run: `cd reactive-bible && npx vitest run src/components/__tests__/CopyrightNotice.test.tsx`

---

## Step 8 — Frontend: Wire into `PassageView.tsx`

### 8a. Edit `reactive-bible/src/components/PassageView.tsx`

1. Import the component:
   ```typescript
   import CopyrightNotice from './CopyrightNotice';
   ```

2. Add `<CopyrightNotice />` after the verses list, inside the `<Box>` but after the verse map:
   ```tsx
   <Box pb={showAudioPlayer ? 120 : 0}>
     {verses.map((verse) => (
       <Verse verse={verse.verse} key={verse.verse} text={verse.text} />
     ))}
     <CopyrightNotice />
   </Box>
   ```

This places the copyright notice at the bottom of the chapter, before any audio player padding.

### 8a — Test

No new unit test needed for this wiring. Verified via manual testing in Step 9.

---

## Step 9 — Manual testing

### 9a. Start both servers

```bash
# Terminal 1: Django
cd bible_research && source venv/bin/activate && python manage.py runserver 8000

# Terminal 2: React (with local API override)
cd reactive-bible && VITE_API_BASE_URL=http://localhost:8000 npm run dev
```

### 9b. Test scenarios

1. **Copyright appears for default translation:**
   - Open the app (default: John 1, ESH translation).
   - Scroll to the bottom of the chapter.
   - Verify: A small, dimmed, italic copyright notice appears below the last verse.

2. **Copyright updates on translation change:**
   - Switch to a different Bible translation (e.g. KJV).
   - Verify: Copyright text updates to reflect the new translation. For KJV (public domain), the text may differ or be empty — verify graceful handling.

3. **Copyright persists across chapters:**
   - Navigate to a different chapter.
   - Verify: Copyright still shows at the bottom (same translation = same copyright).

4. **Copyright with audio player:**
   - Start audio playback so the audio player bar appears at the bottom.
   - Verify: Copyright notice is above the audio player (not hidden behind it). The `pb={showAudioPlayer ? 120 : 0}` padding should ensure this.

5. **Graceful degradation:**
   - Stop the Django server.
   - Reload the React app and navigate to a chapter.
   - Verify: Chapter renders normally. No copyright footer shows. No errors in UI.

6. **Cache verification:**
   - Open DevTools Network tab.
   - Navigate between chapters within the same translation.
   - Verify: Copyright API is called only once. Subsequent chapters reuse cached data (no new network request for `/bible/copyright/`).

### 9c. Browser console checks

- No errors or warnings related to copyright fetching.
- Confirm cache hit on second chapter navigation (no duplicate network request).

---

## Data Flow Diagram

```
PassageView renders
      |
      └──> <CopyrightNotice />
                |
                ├── reads activeTextFilesetId from store
                ├── finds Translation.abbr from translations[]
                |
                └── getCopyrightInfo(abbr)
                        |
                        ├── cache hit? → return cached
                        └── fetch /api/v1/bible/copyright/?bible_id=ENGESV
                                |
                                v
                          Django CopyrightView
                            → DBTClient.get_copyright("ENGESV")
                              → BiblesApi.v4_bible_copyright()
                                |
                                v
                          Response: [{id, type, copyright, ...}]
                                |
                                v
                          CopyrightNotice finds text fileset match
                          Renders: <Text italic dimmed>© ...</Text>
```

---

## Summary of files to edit

| File | Action |
|------|--------|
| `bible_research/bible/services/dbt/client.py` | Add `get_copyright()` method |
| `bible_research/bible/tests/test_dbt_client.py` | Add test for `get_copyright()` |
| `bible_research/bible/views.py` | Add `CopyrightView` class |
| `bible_research/bible/urls.py` | Add `/bible/copyright/` route |
| `bible_research/bible/tests/test_copyright_view.py` | Create — test the view |
| `reactive-bible/src/types.ts` | Add `FilesetCopyright` interface |
| `reactive-bible/src/utils/cacheManager.ts` | Add copyright cache section |
| `reactive-bible/src/utils/__tests__/cacheManager.copyright.test.ts` | Create — test cache |
| `reactive-bible/src/api.tsx` | Add `getCopyrightInfo()` function |
| `reactive-bible/src/__tests__/api.copyright.test.ts` | Create — test API function |
| `reactive-bible/src/components/CopyrightNotice.tsx` | Create — the component |
| `reactive-bible/src/components/__tests__/CopyrightNotice.test.tsx` | Create — test component |
| `reactive-bible/src/components/PassageView.tsx` | Wire in `<CopyrightNotice />` |
