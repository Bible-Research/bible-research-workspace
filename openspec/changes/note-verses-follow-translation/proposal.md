# Proposal

## Why

A note's linked verses (`note.verses`, backend-enriched at
creation/fetch time) render their stored text in whatever
translation the note was created with — so once
`add-note-ref-translation-setting` lands, the `@…` previews
follow the reader's translation (or the pin) while the note
card's own verse text stays frozen in the creation-time
translation. The two note-reference surfaces contradict each
other on every card.

## What Changes

- `NoteVerseList` re-renders stored verse text in the resolved
  note-reference translation: the pinned override wins, else
  the selected translation. Stored `note.verses[].text`
  remains the rendered fallback while the re-fetch is in
  flight, offline, or failed — a card never blanks its verses.
- Depends on `add-note-ref-translation-setting` landing first
  (it provides `useNoteRefFilesetId` and the resolver);
  without the pin field this change still switches cards to
  the selected translation.

## Capabilities

### New Capabilities

<!-- None — extends the existing note-ref-translation
     capability. -->

### Modified Capabilities

- `note-ref-translation`: adds the requirement that a note
  card's linked-verse text follows the resolved
  note-reference translation with stored text as fallback.

## Impact

- `reactive-bible` only: `src/components/NoteVerseList.tsx`,
  `src/utils/noteVerses.ts` (overlay helpers), possibly
  `src/hooks/useVerseTexts.ts` reuse. No backend changes.
- Behavior change is user-visible: note cards may show a
  different translation than the stored payload.
