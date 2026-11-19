# Design

## Context

`bookTitleMarquee` is persisted (`partializeState`), defaults
on, and `BottomNav`/`Marquee` already honor it plus
`prefers-reduced-motion`. The only missing piece is a UI that
writes it — deferred until the `/settings` page existed
(`add-note-ref-translation-setting`).

## Goals / Non-Goals

**Goals:**

- A labeled switch on `/settings` bound to
  `setBookTitleMarquee`.

**Non-Goals:**

- No store or schema changes — the field already persists.
- No BottomNav/Marquee logic changes.
- No other deferred settings — this change adds exactly one
  control.

## Decisions

### D1 — Mantine `Switch` matching the theme toggle's style

The control mirrors the MainMenu theme `Switch` (label +
on/off) rather than introducing a new form pattern; copy lives
in `settings.*` i18n keys (`en`/`lv`/`ru`).

### D2 — Placement below the note-translation control

Single column order on the page: Note reference translation →
Book title marquee. New preferences append below.

## Risks / Trade-offs

- None material — the preference and its consumers are already
  shipped and tested; this is a write surface only.

## Migration Plan

Ships after `add-note-ref-translation-setting` (needs the
page). Rollback = revert the control; the stored field remains
harmless.
