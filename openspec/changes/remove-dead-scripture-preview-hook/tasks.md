# Tasks

All work is in `reactive-bible/` (frontend only). Repo rules:
worktree off `origin/main`, PR `--base staging`, ≤79 chars/line,
scoped `npm test`/`npm run lint` verification only.

## 1. Delete the dead hook

- [ ] 1.1 Confirm `src/hooks/useScripturePreview.ts` is only
  imported by its own test (`rg -l "useScripturePreview" src`
  excluding `useScriptureRefPreview`), then delete
  `src/hooks/useScripturePreview.ts` and
  `src/hooks/useScripturePreview.test.ts` — verify `npm test
  -- --run` and `npm run lint` stay green (0 errors/warnings)

## Workflow follow-up

- Archive the change once the staging PR merges cleanly.
