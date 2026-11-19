# Tasks

All work is in `reactive-bible/` (frontend only). Repo rules:
worktree off `origin/main`, PR `--base staging`, ≤79 chars/line,
scoped `npm test`/`npm run lint` verification only.
Depends on `add-note-ref-translation-setting` (needs
`useNoteRefFilesetId`) — implement that change first.

## 1. Overlay re-resolved verse text on note cards

- [ ] 1.1 In `NoteVerseList`, map `note.verses` to `VerseRef[]`
  via `toUsfmCode` inside `useMemo` (a new array per render
  would refire the hook into a fetch loop), resolve
  `useNoteRefFilesetId()` through `resolveTextFileset` to a
  concrete id with the ESV fallback, and feed `useVerseTexts`
  with `enabled` gated by the card's visibility latch (off
  for the `__overlay` drag scope) — overlay returned text per
  `(bookId, chapter, verse)` on both folded and expanded
  rendering; verify a NoteVerseList test shows
  pinned-translation text after the fetch resolves and that
  off-screen/overlay cards issue no request
- [ ] 1.2 Keep stored `v.text` as the rendered fallback while
  loading, offline, or failed (no error row replaces verses
  when stored text exists) — verify tests for the
  fetch-failure and offline cases keep stored text

## 2. Docs + checks

- [ ] 2.1 Update `reactive-bible/AGENTS.md` +
  `DEVELOPER_GUIDE.md`: note cards re-resolve verse text to
  the note-reference translation; `note.verses`/`headings`
  remain the stored fallback — verify text matches behavior
- [ ] 2.2 Run `npm run lint` (0/0) and the 79-char check on
  touched files — verify clean

## Workflow follow-up

- Archive the change once the staging PR merges cleanly.
