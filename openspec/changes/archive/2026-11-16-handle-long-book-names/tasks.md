# Tasks

## 1. Picker wrapping

- [x] 1.1 frontend: PassagePicker book rows wrap long names
  (content-driven row height, no truncation); verify a row
  with a multi-word Latvian name wraps across lines and
  `title="nav-book-{code}"` hooks still locate rows in tests

## 2. Bottom-bar marquee

- [x] 2.1 frontend: split the BottomNav passage title into a
  flex row — overflow-hidden name container + fixed-right
  chapter span; verify the chapter stays right-aligned and
  visible for any name length in a component test
- [x] 2.2 frontend: marquee animation — measure overflow,
  CSS-keyframe scroll proportional to overflow distance
  (~40-50px/s + ~1s hold) looping reveal-then-restart,
  re-measured on name/width change; verify non-overflowing
  names never animate and overflowing names scroll fully
- [x] 2.3 frontend: `bookTitleMarquee` persisted pref
  (default on, `bible-storage` version bump + migration)
  plus `prefers-reduced-motion` — both render a clipped
  name with chapter pinned; verify migration and
  reduced-motion tests

## 3. Integration

- [x] 3.1 frontend: scoped verification — vitest on
  BottomNav/PassagePicker/store files, `npm run lint` (0/0),
  `npm run build`; all green
- [x] 3.2 frontend: update `reactive-bible/AGENTS.md`
  (pref + marquee behavior) and `DEVELOPER_GUIDE.md`; verify
  docs mention `bookTitleMarquee`
