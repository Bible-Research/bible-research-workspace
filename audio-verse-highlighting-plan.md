# Audio Bible Verse Highlighting & Auto-Scroll — AI Agent Implementation Plan

> **Audience:** Windsurf AI agent (Cascade).
> Execute each step in order. Do not skip ahead. After each step, verify the result before proceeding. Mark each step done before moving to the next.

---

## Context & Background

Two repos live under `~/personal_data/p_projects/Bible Research/`:

- **`bible_research/`** — Django REST API (Python). Proxies the Digital Bible Platform (DBT) v4 API. Already serves text and audio Bible content.
- **`reactive-bible/`** — React + Mantine + Zustand frontend (TypeScript). Already streams audio via Howler.js and displays verses.

**Goal:** When the user plays chapter audio, highlight the verse currently being read and auto-scroll to it. Architect the solution so it can later support "note audio" mode (jumping between specific verses from a user's notes).

### DBT Timestamps API (already researched)

```
GET /timestamps/{fileset_id}/{book}/{chapter}?v=4
```

Response:
```json
{
  "data": [
    { "book": "JHN", "chapter": 1, "verse_start": 1, "timestamp": 0.0 },
    { "book": "JHN", "chapter": 1, "verse_start": 2, "timestamp": 8.52 }
  ]
}
```

- Each entry = start time (seconds) for that verse.
- End time of verse N = start time of verse N+1. Last verse ends at audio duration.
- Generated client class: `bible_research/bible/services/dbt/dbt_client/openapi_client/api/audio_timing_api.py` → method `v4_timestamps_verse(fileset_id, book, chapter, v)`.
- The `fileset_id` used for timestamps may differ from the streaming fileset. Step 1c will verify this.
- `bible_research/bible/utils/bible_books.py` has `get_audio_bible_id()` which constructs testament-aware fileset IDs.

### Key design rules

- **`audioActiveVerse` (new) vs `activeVerses` (existing):** Keep separate. `activeVerses` is user-tap-controlled. `audioActiveVerse` is audio-driven, read-only. Both can be visually active simultaneously with different styles.
- **Timestamp cache never expires:** Timestamps are immutable metadata.
- **Backend proxies DBT:** Frontend never calls DBT directly.
- **Lines ≤ 79 chars** in all Python files.

---

## Step 1 — Backend: `DBTClient.get_timestamps()`

### 1a. Edit `bible_research/bible/services/dbt/client.py`

1. Add this import at the top alongside the existing API imports:
   ```python
   from openapi_client.api.audio_timing_api import AudioTimingApi
   ```
2. In `DBTClient.__init__`, after the line that creates `self.annotations_api`, add:
   ```python
   self.audio_timing_api = AudioTimingApi(self.api_client)
   ```
3. Add a new method **after** `get_verses()` and **before** `search()`:
   ```python
   def get_timestamps(
       self,
       fileset_id: str,
       book: str,
       chapter: str,
       **kwargs
   ) -> Dict[str, Any]:
       return self._make_request(
           self.audio_timing_api.v4_timestamps_verse,
           fileset_id,
           book,
           chapter,
           **kwargs
       )
   ```
   Keep all lines ≤ 79 characters.

### 1a — Test

Run only the edited file's tests:
```bash
cd bible_research
python -m pytest bible/tests/test_dbt_client.py -v
```
If no test file exists for the client, create `bible/tests/test_dbt_client.py` with:
- A test that mocks `AudioTimingApi.v4_timestamps_verse` and asserts `get_timestamps()` calls it with the right args and returns the result.
- A test that verifies `_make_request` injects `v=4` by default.

---

## Step 2 — Backend: `AudioTimestampView` endpoint

### 2a. Edit `bible_research/bible/views.py`

Add a new view class after `BiblePassageView`:

```python
class AudioTimestampView(APIView):
    """Return audio timestamps for a chapter."""

    def get(self, request, format=None):
        fileset_id = request.query_params.get('fileset_id')
        book = request.query_params.get('book')
        chapter = request.query_params.get('chapter')

        if not all([fileset_id, book, chapter]):
            return Response(
                {
                    "error":
                    "fileset_id, book, and chapter "
                    "are required."
                },
                status=status.HTTP_400_BAD_REQUEST
            )

        try:
            dbt_client = DBTClient()
            result = dbt_client.get_timestamps(
                fileset_id, book, chapter
            )
            timestamps = [
                {
                    "verse_start": item.get(
                        "verse_start"
                    ),
                    "timestamp": item.get(
                        "timestamp"
                    ),
                }
                for item in result.get("data", [])
            ]
            return Response({"data": timestamps})
        except Exception as e:
            logger.exception(
                "Error fetching timestamps: %s", e
            )
            return Response(
                {"error": str(e)},
                status=status.HTTP_400_BAD_REQUEST
            )
```

Import `DBTClient` if not already imported in views.py (it is used via serializers currently — add a direct import).

### 2b. Edit `bible_research/bible/urls.py`

Add the route:
```python
path(
    'bible/timestamps/',
    views.AudioTimestampView.as_view(),
    name='audio-timestamps'
),
```

### 2b — Test

Create `bible/tests/test_timestamp_view.py`:
- Mock `DBTClient.get_timestamps` to return sample data.
- Use DRF's `APIRequestFactory` to call the view.
- Assert 200 response with correct `{ "data": [...] }` shape.
- Assert 400 when `fileset_id` is missing.
- Assert 400 when DBT raises an exception.

Run: `cd bible_research && python -m pytest bible/tests/test_timestamp_view.py -v`

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
# Test with a known ESV audio fileset (Old Testament book)
curl "http://localhost:8000/api/v1/bible/timestamps/?fileset_id=ENGESVO2DA&book=GEN&chapter=1"

# Test with a New Testament book
curl "http://localhost:8000/api/v1/bible/timestamps/?fileset_id=ENGESVN2DA&book=JHN&chapter=3"

# Test with the ESH fileset used by the frontend
curl "http://localhost:8000/api/v1/bible/timestamps/?fileset_id=ENGESHN1DA&book=JHN&chapter=1"
```

### 3c. Verify and record

- **If the response has `"data": [...]` with `verse_start` and `timestamp` fields:** Proceed.
- **If the fileset_id from streaming doesn't work for timestamps:** Note which fileset_id format works. You may need to strip the codec suffix (e.g. `ENGESHN1DA-opus16` → `ENGESHN1DA`) or use a different base. Update the view to handle this mapping.
- **Record the working fileset_id pattern** in a comment at the top of `AudioTimestampView`.

Stop the dev server after testing.

---

## Step 4 — Frontend: TypeScript types

### 4a. Edit `reactive-bible/src/types.ts`

Append at the end of the file:
```typescript
export interface VerseTimestamp {
  verse_start: number;
  timestamp: number;
}
```

No tests needed for types.

---

## Step 5 — Frontend: Timestamp cache

### 5a. Edit `reactive-bible/src/utils/cacheManager.ts`

Add a new section after the `// CACHE STATS` section at the bottom.

Key: `bible_timestamp_cache`
Interface:
```typescript
interface TimestampCache {
  [key: string]: VerseTimestamp[];
}
```

Functions to add:
- `getCachedTimestamps(filesetId: string, book: string, chapter: number): VerseTimestamp[] | null`
- `cacheTimestamps(filesetId: string, book: string, chapter: number, timestamps: VerseTimestamp[]): void`
- `clearTimestampCache(): void`

Cache key format: `${filesetId}:${book}:${chapter}`
No expiration — timestamps are immutable.

Import `VerseTimestamp` from `'../types'`.

### 5a — Test

Create `reactive-bible/src/utils/__tests__/cacheManager.timestamp.test.ts`:
- Test `cacheTimestamps` stores data and `getCachedTimestamps` retrieves it.
- Test cache miss returns `null`.
- Test `clearTimestampCache` removes all entries.
- Mock `localStorage` with a simple in-memory implementation.

Run: `cd reactive-bible && npx vitest run src/utils/__tests__/cacheManager.timestamp.test.ts`

---

## Step 6 — Frontend: API function

### 6a. Edit `reactive-bible/src/api.tsx`

Add after the `prefetchAdjacentChapters` function (before the Notes/Tags section):

```typescript
import { VerseTimestamp } from './types';
import {
  getCachedTimestamps,
  cacheTimestamps,
} from './utils/cacheManager';
```

(Merge these into the existing import blocks at the top of the file.)

Function:
```typescript
export const getAudioTimestamps = async (
  book: string,
  chapter: number,
  filesetId: string
): Promise<VerseTimestamp[]> => {
  const cached = getCachedTimestamps(filesetId, book, chapter);
  if (cached) {
    return cached;
  }

  try {
    const url =
      `${API_BASE_URL}/api/v1/bible/timestamps/` +
      `?fileset_id=${encodeURIComponent(filesetId)}` +
      `&book=${encodeURIComponent(book)}` +
      `&chapter=${chapter}`;
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`Timestamps fetch failed: ${response.statusText}`);
    }
    const data = await response.json();
    const timestamps: VerseTimestamp[] = data.data || [];
    cacheTimestamps(filesetId, book, chapter, timestamps);
    return timestamps;
  } catch (error) {
    console.warn('Failed to fetch audio timestamps:', error);
    return []; // Graceful degradation: no highlighting
  }
};
```

Note: The function returns `[]` on error — this means audio still plays, just without highlighting. This is intentional graceful degradation.

The `book` parameter here is the book **name** (e.g. "John"), but the API expects a book **ID** (e.g. "JHN"). Check how `getBibleAudioUrl` handles this — it sends the book name to the Django API which does the conversion. Do the same here: the Django `AudioTimestampView` should accept a book name and convert it using `get_dbt_book_id()`, OR the frontend should send the `activeBookShort` (which is already the short code like "Joh"). Inspect the store's `activeBookShort` field and the `get_dbt_book_id` mapping to decide.

### 6a — Test

Create `reactive-bible/src/__tests__/api.timestamps.test.ts`:
- Mock `fetch` and `getCachedTimestamps`.
- Test cache hit returns cached data without fetch.
- Test successful fetch returns and caches data.
- Test fetch failure returns `[]`.

Run: `cd reactive-bible && npx vitest run src/__tests__/api.timestamps.test.ts`

---

## Step 7 — Frontend: Zustand store additions

### 7a. Edit `reactive-bible/src/store.tsx`

In the `BibleState` interface, add:
```typescript
audioActiveVerse: number | null;
setAudioActiveVerse: (verse: number | null) => void;
```

In `initialState`, add:
```typescript
audioActiveVerse: null,
```

In the store creator, add:
```typescript
setAudioActiveVerse: (audioActiveVerse) =>
  set({ audioActiveVerse }),
```

Also, in the existing `setActiveChapter` action, clear `audioActiveVerse`:
```typescript
setActiveChapter: (activeChapter) => set({
  activeChapter,
  activeVerses: [],
  audioActiveVerse: null,
}),
```

Do the same for `setActiveBook`.

### 7a — Test

If `reactive-bible/src/__tests__/store.test.ts` exists, add tests there. Otherwise create it:
- Test `setAudioActiveVerse(5)` sets the value.
- Test `setAudioActiveVerse(null)` clears it.
- Test `setActiveChapter` resets `audioActiveVerse` to `null`.

Run: `cd reactive-bible && npx vitest run src/__tests__/store.test.ts`

---

## Step 8 — Frontend: `useVerseHighlighter` hook

### 8a. Create `reactive-bible/src/hooks/useVerseHighlighter.ts`

```typescript
import { useEffect, useRef } from 'react';
import { Howl } from 'howler';
import { useBibleStore } from '../store';
import { VerseTimestamp } from '../types';

/**
 * Polls audio playback position and sets the
 * audio-active verse based on timestamps.
 *
 * Designed to work for both chapter-play and
 * future note-play modes.
 */
export const useVerseHighlighter = (
  audio: Howl | null,
  isPlaying: boolean,
  timestamps: VerseTimestamp[]
) => {
  const setAudioActiveVerse = useBibleStore(
    (s) => s.setAudioActiveVerse
  );
  const intervalRef = useRef<ReturnType<typeof setInterval>>();

  useEffect(() => {
    if (!audio || !isPlaying || timestamps.length === 0) {
      return;
    }

    intervalRef.current = setInterval(() => {
      const currentTime = audio.seek() as number;
      if (typeof currentTime !== 'number') return;

      // Binary search for the active verse
      let lo = 0;
      let hi = timestamps.length - 1;
      let activeVerse = timestamps[0].verse_start;

      while (lo <= hi) {
        const mid = Math.floor((lo + hi) / 2);
        if (timestamps[mid].timestamp <= currentTime) {
          activeVerse = timestamps[mid].verse_start;
          lo = mid + 1;
        } else {
          hi = mid - 1;
        }
      }

      setAudioActiveVerse(activeVerse);
    }, 100);

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [audio, isPlaying, timestamps, setAudioActiveVerse]);

  // Clear on unmount
  useEffect(() => {
    return () => setAudioActiveVerse(null);
  }, [setAudioActiveVerse]);
};
```

### 8a — Test

Create `reactive-bible/src/hooks/__tests__/useVerseHighlighter.test.ts`:
- Use `@testing-library/react-hooks` (or `renderHook` from `@testing-library/react`).
- Mock `Howl` with a fake `seek()` that returns a fixed value.
- Provide sample timestamps: `[{verse_start:1, timestamp:0}, {verse_start:2, timestamp:5}, {verse_start:3, timestamp:10}]`.
- Set fake seek to 7.5 → assert `audioActiveVerse` is 2.
- Set fake seek to 0.5 → assert `audioActiveVerse` is 1.
- Set fake seek to 15 → assert `audioActiveVerse` is 3.

Run: `cd reactive-bible && npx vitest run src/hooks/__tests__/useVerseHighlighter.test.ts`

---

## Step 9 — Frontend: Update `Verse.tsx` for audio highlighting

### 9a. Edit `reactive-bible/src/components/Verse.tsx`

1. Read `audioActiveVerse` from the store:
   ```typescript
   const audioActiveVerse = useBibleStore(
     (state) => state.audioActiveVerse
   );
   const isAudioActive = audioActiveVerse === verse;
   ```

2. Add a new style class in `useStyles`:
   ```typescript
   linkAudioActive: {
     borderLeft: `3px solid ${theme.colors.blue[5]}`,
     backgroundColor:
       theme.colorScheme === 'dark'
         ? theme.fn.rgba(theme.colors.blue[9], 0.15)
         : theme.fn.rgba(theme.colors.blue[1], 0.5),
   },
   ```

3. Add the class to the `cx()` call:
   ```typescript
   className={cx(classes.link, {
     [classes.linkActive]: isActive,
     [classes.linkAudioActive]: isAudioActive,
   })}
   ```

4. Add auto-scroll when `isAudioActive` changes. In the existing `useEffect` for scrolling, add a second effect:
   ```typescript
   useEffect(() => {
     if (isAudioActive) {
       ref.current?.scrollIntoView({
         block: 'center',
         behavior: 'smooth',
       });
     }
   }, [isAudioActive]);
   ```
   Make sure `ref` is always attached (currently it's conditional on `isActive`). Change:
   ```
   ref={isActive ? ref : null}
   ```
   to:
   ```
   ref={ref}
   ```

### 9a — Test

No unit test needed for styling. This will be verified via manual testing in Step 11.

---

## Step 10 — Frontend: Wire timestamps into `Audio.tsx`

### 10a. Edit `reactive-bible/src/components/Audio.tsx`

1. Import the new hook and API function:
   ```typescript
   import { useVerseHighlighter } from '../hooks/useVerseHighlighter';
   import { getAudioTimestamps } from '../api';
   import { VerseTimestamp } from '../types';
   ```

2. Add state for timestamps:
   ```typescript
   const [timestamps, setTimestamps] = useState<VerseTimestamp[]>([]);
   ```

3. Read `activeBookShort` from the store (needed for the API call):
   ```typescript
   const activeBookShort = useBibleStore(
     (state) => state.activeBookShort
   );
   ```

4. Fetch timestamps when audio starts playing. Inside the `loadAndPlayAudio` async function, after the audio URL is obtained and before `new Howl(...)`, add:
   ```typescript
   // Fetch timestamps in parallel (non-blocking)
   getAudioTimestamps(
     activeBookShort, activeChapter, activeAudioFilesetId
   ).then(setTimestamps).catch(() => setTimestamps([]));
   ```

5. Clear timestamps on chapter/book change. In the existing `useEffect` that resets audio:
   ```typescript
   useEffect(() => {
     if (audio) {
       audio.unload();
       setAudio(null);
     }
     setTimestamps([]);
   }, [activeBook, activeChapter, activeAudioFilesetId]);
   ```

6. Activate the hook at the component level:
   ```typescript
   useVerseHighlighter(audio, isPlaying, timestamps);
   ```

### 10a — Notes on the book parameter

The Django `AudioTimestampView` receives a `book` query param. Currently `BiblePassageView` receives a full book name (e.g. "John") and converts it via `get_dbt_book_id()`. For the timestamp view, decide:
- **Option A:** Send `activeBookShort` (e.g. "Joh") and have Django convert it. But `get_dbt_book_id()` maps full names, not short codes.
- **Option B:** Send `activeBook` (e.g. "John") and have Django convert it (same as existing flow).
- **Option C:** Send the DBT book code directly (e.g. "JHN") — but the frontend doesn't currently store this.

**Recommended:** Use Option B — send `activeBook` (the full name). Update `AudioTimestampView` to accept a `passage`-style book name and convert using `get_dbt_book_id()`. This is consistent with `BiblePassageView`.

---

## Step 11 — Manual testing

### 11a. Start both servers

```bash
# Terminal 1: Django
cd bible_research && source venv/bin/activate && python manage.py runserver 8000

# Terminal 2: React (with local API override)
cd reactive-bible && VITE_API_BASE_URL=http://localhost:8000 npm run dev
```

### 11b. Test scenarios

Perform these checks in the browser:

1. **Basic playback with highlighting:**
   - Navigate to John 1.
   - Click the play button.
   - Verify: The audio plays. The verse being read gets a blue left-border highlight. The highlight moves to the next verse as the audio progresses. The page auto-scrolls to keep the active verse in view.

2. **Pause and resume:**
   - Pause the audio mid-chapter.
   - Verify: Highlighting stops (stays on last verse, does not flicker).
   - Resume playback.
   - Verify: Highlighting resumes from the correct verse.

3. **Seek:**
   - Drag the audio slider to a later point.
   - Verify: Highlighting jumps to the correct verse for that timestamp.

4. **Chapter change during playback:**
   - While audio is playing, navigate to a different chapter.
   - Verify: Old highlighting clears. If auto-play continues, new timestamps load and highlighting works for the new chapter.

5. **User verse selection coexists:**
   - While audio is playing, tap a verse manually.
   - Verify: The tapped verse gets the `linkActive` style AND the audio-active verse keeps its `linkAudioActive` style. Both are visible simultaneously.

6. **Graceful degradation:**
   - Temporarily break the timestamp endpoint (e.g. stop Django).
   - Click play in the frontend.
   - Verify: Audio still plays normally, just without verse highlighting. No errors in the UI.

7. **Different translations:**
   - Switch to a different Bible translation that has audio.
   - Play a chapter.
   - Verify: Timestamps load correctly (or gracefully degrade if timestamps aren't available for that fileset).

### 11c. Browser console checks

- No errors or warnings related to timestamps when playing.
- Confirm `getCachedTimestamps` cache hit on second play of same chapter (check for absence of network request in DevTools Network tab).

---

## Data Flow Diagram

```
User clicks Play
      |
      |---> Audio.tsx: load audio URL (existing)
      |         └── Howl plays
      |
      └---> Audio.tsx: fetch timestamps (new, parallel)
                └── getAudioTimestamps() -> cache -> VerseTimestamp[]
                        |
                        v
               useVerseHighlighter hook
                 polls audio.seek() @ 100ms
                 binary-search timestamps
                        |
                        v
               store.setAudioActiveVerse(N)
                        |
                        v
               Verse.tsx re-renders
                 highlight + scrollIntoView
```

---

## Future: Note Audio Mode (do NOT implement now)

When this feature is needed, introduce an `AudioPlaylist` abstraction:

```typescript
interface AudioPlaylistItem {
  book: string;
  chapter: number;
  filesetId: string;
  startVerse?: number; // for note-mode
  endVerse?: number;   // for note-mode
}

interface AudioPlaylistState {
  items: AudioPlaylistItem[];
  currentIndex: number;
  mode: 'chapter' | 'notes';
}
```

- **Chapter mode** (this implementation): single-item playlist.
- **Notes mode:** N items from note verse references. When a verse range ends (based on timestamps), advance to next playlist item. The `useVerseHighlighter` hook needs no changes — it works on whatever timestamps are provided.
- `Audio.tsx` would be refactored to consume a playlist instead of directly reading `activeBook`/`activeChapter`.
