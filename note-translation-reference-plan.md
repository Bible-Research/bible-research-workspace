# Store & Display Translation Reference on Notes — AI Agent Implementation Plan

> **Audience:** Windsurf AI agent (Cascade).
> Execute each step in order. Do not skip ahead. After each step, verify the result before proceeding.

---

## Context & Background

Two repos live under `~/personal_data/p_projects/Bible Research/`:

- **`bible_research/`** — Django REST API.
- **`reactive-bible/`** — React + Mantine + Zustand frontend.

**Goal:** When a user creates a note, record which Bible translation (fileset) they were reading at the time. Display this translation name on the `NoteCard` in the notes view so readers know which version the verse text comes from.

### Current state

- **`Note` model** (`annotations/models.py`): Has `user`, `tag`, `note_text`, `public`, `verses` (M2M via `NoteVerse`). **No translation reference.**
- **`NoteSerializer.create()`** (`annotations/serializers.py`): Accepts `tag`, `note_text`, `verse_references`. **No fileset/translation field.**
- **`NoteSerializer.to_representation()`**: Fetches verse text from DBT using `dbt_client.get_verses(dbt_book_id, chapter)` with the **default** `ENGESV` fileset — it ignores which translation was actually used when the note was created.
- **Frontend `addTagNote()`** (`api.tsx`): Sends `{ tag, note_text, verse_references }`. **No fileset_id.**
- **Frontend `NoteCard`** (`NoteCard.tsx`): Shows heading, verse text, note text, action buttons. **No translation label.**
- **Frontend store**: Has `activeTextFilesetId` (e.g. `ENGESH`) and `translations` array with `Translation.abbr` and `Translation.name`.

### Key observations

1. The `Note` model needs a new field to store the text fileset_id used at creation time.
2. `NoteSerializer.to_representation()` should use the stored fileset_id (instead of default `ENGESV`) when fetching verse text from DBT.
3. The API response should include the translation name so the frontend can display it.
4. The frontend must send `fileset_id` when creating a note, and display the translation name when viewing.

### Design rules

- **Lines ≤ 79 chars** in all Python files.
- **Backward compatible:** Existing notes with no `fileset_id` should still work (default to `ENGESV`).
- **Migration safe:** The new field is nullable so existing rows are unaffected.

---

## Step 1 — Backend: Add `text_fileset_id` to `Note` model

### 1a. Edit `bible_research/annotations/models.py`

Add a new field to the `Note` class, after the `public` field and before the `verses` M2M:

```python
text_fileset_id = models.CharField(
    max_length=50,
    blank=True,
    null=True,
    help_text=(
        "The DBT text fileset ID active when "
        "this note was created "
        "(e.g., 'ENGESV', 'ENGESH')."
    )
)
```

### 1b. Create and run the migration

```bash
cd bible_research
python manage.py makemigrations annotations -n add_text_fileset_to_note
python manage.py migrate
```

### 1b — Test

Verify the migration applied cleanly:
```bash
cd bible_research
python manage.py showmigrations annotations
```
Confirm the new migration is `[X]`.

---

## Step 2 — Backend: Update `NoteSerializer`

### 2a. Edit `bible_research/annotations/serializers.py`

**In `NoteSerializer.Meta.fields`**, add `'text_fileset_id'`:

```python
fields = [
    'id', 'user', 'note_text', 'public',
    'text_fileset_id',
    'created_at', 'updated_at',
    'tag', 'verse_references'
]
```

**Add the field definition** in the serializer class body (it should be writable on create but read-only after):

```python
text_fileset_id = serializers.CharField(
    max_length=50,
    required=False,
    allow_null=True,
    allow_blank=True,
    help_text=(
        "The text fileset ID of the translation "
        "active when creating this note."
    )
)
```

### 2b. Update `to_representation()` to use stored fileset

In the `to_representation()` method, change the `get_verses` call to use the note's stored `text_fileset_id` when available:

Current code:
```python
dbt_client = DBTClient()
verse_text = dbt_client.get_verses(
    dbt_book_id, chapter, **kwargs
)
```

Change to:
```python
dbt_client = DBTClient()
fileset = instance.text_fileset_id or 'ENGESV'
verse_text = dbt_client.get_verses(
    dbt_book_id,
    chapter,
    bible_id=fileset,
    **kwargs
)
```

### 2c. Add `translation_name` to the response

Still in `to_representation()`, after building the representation, add a human-readable translation name. The simplest approach: derive it from the `text_fileset_id`. We can use the `fileset_id` directly as a label for now, or look it up. Since the frontend already has the full translation list, the cleanest approach is to just return `text_fileset_id` and let the frontend resolve the display name.

However, for convenience (especially shared/public notes where the viewer may not have the same translations loaded), also add a `translation_label`:

```python
representation['text_fileset_id'] = (
    instance.text_fileset_id or 'ENGESV'
)
```

This is already handled by the serializer field, but ensures it's present even for old notes.

### 2d — Test

Edit `bible_research/annotations/tests.py`. Add a test:

```python
def test_note_with_fileset_id(self):
    """Test that notes save and return text_fileset_id."""
    tag = self.test_tag_serializer()
    note_data = {
        'note_text': 'Test note with fileset',
        'tag': tag.id,
        'text_fileset_id': 'ENGESH',
        'verse_references': [
            {
                'book': 'John',
                'chapter': 3,
                'verse': 16,
            }
        ]
    }
    serializer = NoteSerializer(
        data=note_data,
        context=self.context
    )
    self.assertTrue(serializer.is_valid())
    note = serializer.save()
    self.assertEqual(
        note.text_fileset_id, 'ENGESH'
    )
```

Also add a test for backward compatibility:

```python
def test_note_without_fileset_id(self):
    """Existing notes without fileset_id still work."""
    tag = self.test_tag_serializer()
    note_data = {
        'note_text': 'Legacy note',
        'tag': tag.id,
        'verse_references': [
            {
                'book': 'John',
                'chapter': 3,
                'verse': 16,
            }
        ]
    }
    serializer = NoteSerializer(
        data=note_data,
        context=self.context
    )
    self.assertTrue(serializer.is_valid())
    note = serializer.save()
    self.assertIsNone(note.text_fileset_id)
```

Run: `cd bible_research && python -m pytest annotations/tests.py -v -k "test_note_with_fileset_id or test_note_without_fileset_id"`

---

## Step 3 — Backend: Manual API test

### 3a. Start Django

```bash
cd bible_research
source venv/bin/activate
python manage.py runserver 8000
```

### 3b. Create a note with fileset_id

```bash
# Get an auth token first (adjust credentials)
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login/ \
  -H 'Content-Type: application/json' \
  -d '{"username":"testuser","password":"password123"}' | python3 -c "import sys,json; print(json.load(sys.stdin).get('token',''))")

# Create a note with text_fileset_id
curl -X POST http://localhost:8000/api/v1/notes/ \
  -H "Authorization: Token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tag": "<a_valid_tag_id>",
    "note_text": "Test note with translation",
    "text_fileset_id": "ENGESH",
    "verse_references": [
      {"book": "John", "chapter": 3, "verse": 16}
    ]
  }'
```

### 3c. Retrieve and verify

```bash
# Get notes for the tag
curl http://localhost:8000/api/v1/notes/?tag_id=<tag_id> \
  -H "Authorization: Token $TOKEN"
```

Verify the response includes:
- `"text_fileset_id": "ENGESH"`
- Verse text is fetched using `ENGESH` fileset (not default `ENGESV`)

### 3d. Test backward compatibility

```bash
# Fetch an existing note (created before this change)
curl http://localhost:8000/api/v1/notes/<old_note_id>/ \
  -H "Authorization: Token $TOKEN"
```

Verify:
- `text_fileset_id` is `null` or absent
- Verse text still loads (falls back to `ENGESV`)

Stop the dev server after testing.

---

## Step 4 — Frontend: Update TypeScript types

### 4a. Edit `reactive-bible/src/types.ts`

Add `text_fileset_id` to the `Note` interface:

```typescript
export interface Note {
  id: string;
  note_text: string;
  public: boolean;
  is_owner: boolean;
  created_at: string;
  updated_at: string;
  tag: Tag;
  verses: Verse[];
  text_fileset_id?: string | null;
}
```

### 4b. Also update the duplicate `Note` interface in `api.tsx`

There is a duplicate `Note` interface at the bottom of `reactive-bible/src/api.tsx` (line ~571). Add the same field there:

```typescript
export interface Note {
  id: string;
  note_text: string;
  public: boolean;
  is_owner: boolean;
  created_at: string;
  updated_at: string;
  tag: Tag;
  verses: NoteVerse[];
  text_fileset_id?: string | null;
}
```

**Note to agent:** Ideally the duplicate interface in `api.tsx` should be removed and the one in `types.ts` used everywhere. But to minimize scope, just add the field to both for now.

No tests needed for types.

---

## Step 5 — Frontend: Send `text_fileset_id` when creating notes

### 5a. Edit `reactive-bible/src/api.tsx` — `addTagNote()`

Add `filesetId` parameter:

```typescript
export const addTagNote = async (
  tagId: string,
  tagNoteText: string,
  verseReferences: {
    book: string;
    chapter: number;
    verse: number;
  }[],
  textFilesetId?: string | null
) => {
  const body = JSON.stringify({
    tag: tagId,
    note_text: tagNoteText,
    verse_references: verseReferences,
    text_fileset_id: textFilesetId || undefined,
  });
  // ... rest unchanged
```

### 5b. Edit `reactive-bible/src/components/AddTagNoteModal.tsx`

Read `activeTextFilesetId` from the store and pass it to `addTagNote`:

```typescript
const {
  tags, getTags, activeVerses,
  activeBook, activeChapter, setActiveVerses,
  activeTextFilesetId,
} = useBibleStore((state) => ({
  tags: state.tags,
  getTags: state.getTags,
  activeVerses: state.activeVerses,
  activeBook: state.activeBook,
  activeChapter: state.activeChapter,
  setActiveVerses: state.setActiveVerses,
  activeTextFilesetId: state.activeTextFilesetId,
}));
```

In `handleSubmit`, pass it:

```typescript
await addTagNote(
  tagId,
  text,
  verseReferences,
  activeTextFilesetId
);
```

### 5b — Test

If `AddTagNoteModal` has existing tests, update them to verify `text_fileset_id` is included in the POST body. Otherwise create `reactive-bible/src/components/__tests__/AddTagNoteModal.test.tsx`:

- Mock `addTagNote` and the Zustand store with `activeTextFilesetId: 'ENGESH'`.
- Submit the form.
- Assert `addTagNote` was called with `'ENGESH'` as the 4th argument.

Run: `cd reactive-bible && npx vitest run src/components/__tests__/AddTagNoteModal.test.tsx`

---

## Step 6 — Frontend: Display translation on `NoteCard`

### 6a. Edit `reactive-bible/src/components/NoteCard.tsx`

Add a helper to resolve the translation display name, and show it below the heading.

Import what's needed:

```typescript
import { useBibleStore } from '../store';
```

Inside the component, resolve the name:

```typescript
const translations = useBibleStore(
  (state) => state.translations
);

const translationLabel = (() => {
  if (!note.text_fileset_id) return null;
  const match = translations.find((t) =>
    t.filesets.some((f) => f.id === note.text_fileset_id)
  );
  return match?.name || note.text_fileset_id;
})();
```

In the JSX, add the label after the heading `<Title>`, inside the `<Group position="apart">`:

```tsx
<Group position="apart" mb={0}>
  <Box>
    <Title order={4}>{heading}</Title>
    {translationLabel && (
      <Text size="xs" color="dimmed">
        {translationLabel}
      </Text>
    )}
  </Box>
  <Group spacing="xs">
    {/* ... existing buttons ... */}
  </Group>
</Group>
```

Import `Box` if not already imported (it is already imported).

### 6a — Test

Create `reactive-bible/src/components/__tests__/NoteCard.test.tsx`:

- Render `NoteCard` with a note that has `text_fileset_id: 'ENGESH'`.
- Mock the store with translations containing a matching fileset.
- Assert the translation name (e.g. "English Standard Hearing") appears in the rendered output.
- Render with `text_fileset_id: null` → assert no translation label.
- Render with `text_fileset_id: 'UNKNOWN'` → assert the raw fileset ID is shown as fallback.

Run: `cd reactive-bible && npx vitest run src/components/__tests__/NoteCard.test.tsx`

---

## Step 7 — Manual testing

### 7a. Start both servers

```bash
# Terminal 1: Django
cd bible_research && source venv/bin/activate && python manage.py runserver 8000

# Terminal 2: React
cd reactive-bible && VITE_API_BASE_URL=http://localhost:8000 npm run dev
```

### 7b. Test scenarios

1. **New note stores translation:**
   - Navigate to a chapter in the Bible view.
   - Select a verse, open the "Add note" modal.
   - Submit a note.
   - Navigate to the notes view for that tag.
   - Verify: The note card shows the translation name (e.g. "English Standard Hearing") below the verse heading.

2. **Translation changes between notes:**
   - Switch to a different translation (e.g. KJV).
   - Select a verse and create another note.
   - View the notes.
   - Verify: The new note shows "King James Version" and the old note shows the previous translation.

3. **Legacy notes without fileset_id:**
   - View notes that were created before this change.
   - Verify: They render normally. No translation label appears (or a default label). Verse text still loads.

4. **Shared/public notes:**
   - View a shared note link as an unauthenticated user.
   - Verify: The translation label still shows (it comes from the API response field, not just the local translations list). If the viewer doesn't have that translation in their store, the raw fileset ID is shown as fallback.

5. **Edit note doesn't change fileset:**
   - Edit an existing note's text (not its verses).
   - Verify: The `text_fileset_id` remains unchanged (it was set at creation time).

### 7c. Browser console checks

- No errors when creating or viewing notes.
- Confirm the POST body to `/api/v1/notes/` includes `text_fileset_id`.
- Confirm the GET response from `/api/v1/notes/` includes `text_fileset_id` for new notes.

---

## Data Flow Diagram

```
Creating a note:

  AddTagNoteModal
    reads activeTextFilesetId from store (e.g. "ENGESH")
      |
      v
  addTagNote(tagId, text, verseRefs, "ENGESH")
    → POST /api/v1/notes/ { ..., text_fileset_id: "ENGESH" }
      |
      v
  NoteSerializer.create()
    saves text_fileset_id to Note model
      |
      v
  Database: Note row has text_fileset_id="ENGESH"


Viewing a note:

  GET /api/v1/notes/?tag_id=X
      |
      v
  NoteSerializer.to_representation()
    → dbt_client.get_verses(book, ch, bible_id="ENGESH")
    → response includes text_fileset_id="ENGESH"
      |
      v
  Frontend: NoteCard receives note.text_fileset_id
    → looks up Translation.name from store
    → displays "English Standard Hearing" below heading
```

---

## Summary of files to edit

| File | Action |
|------|--------|
| `bible_research/annotations/models.py` | Add `text_fileset_id` CharField to `Note` |
| `bible_research/annotations/migrations/` | Auto-generated migration |
| `bible_research/annotations/serializers.py` | Add field to serializer, use it in `to_representation()` |
| `bible_research/annotations/tests.py` | Add tests for fileset_id on notes |
| `reactive-bible/src/types.ts` | Add `text_fileset_id` to `Note` interface |
| `reactive-bible/src/api.tsx` | Add `text_fileset_id` to `Note` interface + `addTagNote()` param |
| `reactive-bible/src/components/AddTagNoteModal.tsx` | Send `activeTextFilesetId` with note creation |
| `reactive-bible/src/components/__tests__/AddTagNoteModal.test.tsx` | Create — test fileset sent |
| `reactive-bible/src/components/NoteCard.tsx` | Display translation name from `text_fileset_id` |
| `reactive-bible/src/components/__tests__/NoteCard.test.tsx` | Create — test translation label |
