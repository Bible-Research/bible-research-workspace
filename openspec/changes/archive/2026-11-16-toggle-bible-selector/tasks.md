# Tasks

## 1. Toggle the selector from the passage button

- [x] 1.1 frontend: Pass the selector's open state (or a toggle
  callback) from `App.tsx` into `BottomNav` so the passage
  button can distinguish open from closed; verify the prop
  type-checks via `npm run build` or `tsc --noEmit`
- [x] 1.2 frontend: In `BottomNav.tsx`, make the
  current-passage button call `setBibleSelectorOpened` with the
  inverse of the current open state instead of always `true`;
  verify no other callers/behaviors change (`git diff` shows
  only the toggle wiring)

## 2. Test coverage

- [x] 2.1 frontend: Add a `BottomNav.test.tsx` case asserting a
  press while open calls the setter with `false` (and while
  closed still calls it with `true`); verify with
  `npm test -- --run src/components/BottomNav.test.tsx`
- [x] 2.2 frontend: Run `npm run lint` on the touched files and
  the 79-char check (`awk 'length > 79'`) — zero errors,
  zero warnings, no long lines introduced
