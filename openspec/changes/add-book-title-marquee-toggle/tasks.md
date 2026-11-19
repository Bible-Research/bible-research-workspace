# Tasks

All work is in `reactive-bible/` (frontend only). Repo rules:
worktree off `origin/main`, PR `--base staging`, ≤79 chars/line,
scoped `npm test`/`npm run lint` verification only.
Depends on `add-note-ref-translation-setting` (needs
`/settings`) — implement that change first.

## 1. Marquee toggle on the settings page

- [ ] 1.1 Add a labeled `Switch` to `SettingsRoute` bound to
  `bookTitleMarquee`/`setBookTitleMarquee`, placed below the
  note-reference-translation control — verify a settings test
  toggles the persisted field and reflects the stored value
  on load
- [ ] 1.2 Add `settings.bookTitleMarquee` (+hint) keys to
  `en`/`lv`/`ru` `common.json` — verify i18n completeness
  tests pass

## 2. Docs + checks

- [ ] 2.1 Update `reactive-bible/AGENTS.md`: the marquee
  preference now has its settings UI — verify the doc matches
- [ ] 2.2 Run `npm run lint` (0/0) and the 79-char check on
  touched files — verify clean

## Workflow follow-up

- Archive the change once the staging PR merges cleanly.
