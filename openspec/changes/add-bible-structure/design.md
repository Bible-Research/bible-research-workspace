# Design

## Context

Book names + chapter structure are hardcoded English in three
places: `src/assets/bibleStructure.json` (USFM -> name +
chapter/verse counts, KJV versification), `BIBLE_BOOKS` in
`src/utils/bibleUtils.ts`, and `_BIBLE_BOOKS` in
`bible/utils/bible_books.py`. The picker offers all 66 books
for every translation; coverage gaps surface only as
`book_not_in_fileset` 404s after navigation.

Verified against live DBT v4: `GET /bibles/{bible_id}/book?
verify_content=true` returns `{book_id, book_id_usfx, name,
name_short, testament, book_order, chapters}` — `chapters` is a
plain chapter-number list (no per-chapter verse counts; the
`verse_count` param does not add them). Names are per-bible
orthography (`ENGKJV` -> "GENESIS", `ENGESV` -> "Genesis";
`LAVNLI` -> 66 Latvian books; `RUSSYN` -> 66 Russian books;
`LAVLVR` -> 28; `RUSCOV` -> non-book ids like `C01`).
`LVSGLU8` is a local SWORD module — not on DBT — and its
bible id 404s on DBT's books endpoint.

Caching machinery already exists: `chapter_cache.py`
(process-wide LRU), `VercelEdgeCacheMixin` for
anonymous-identical catalog endpoints, `cacheManager` 3-tier
localStorage on the client.

## Goals / Non-Goals

**Goals:**

- One provider-aware endpoint answering "what books/chapters
  does this fileset have and what are they called".
- Picker driven by per-translation coverage with
  stale-while-revalidate loading.
- Display-name layer split from the English contract identity.

**Non-Goals:**

- Per-chapter verse counts per translation (DBT's books
  endpoint doesn't return them; bundled KJV versification
  stays the verse-grid approximation — mismatches already
  fail gracefully via `book_not_in_fileset`).
- Deuterocanon/apocrypha books (filtered in v1 — routes and
  the 66-code tables can't resolve them yet).
- UI chrome localization (the `add-ui-localization` change).
- Auto-fallback text switching (explicitly rejected — text
  keeps the hint-only behavior; audio keeps `alternates`).

## Decisions

### Endpoint keyed on `fileset_id`, not `language_iso`

`GET /api/v1/bible/structure/?fileset_id=` — the caller's
natural key, and the contract returns per-translation coverage
(not merely per-language names). The backend resolves
fileset -> owning bible through the translations listing
abbr + registries. Rejected `language_iso`-only keying: it
cannot express partial canon or testament splits.

### Sourcing precedence

Per `fileset_id`, first applicable source wins:

1. DBT-backed fileset -> `v4_bible_books(bible_id,
   verify_content=true)` — the bible's own coverage and names.
2. SWORD fileset -> `pysword` `get_structure().get_books()`
   (real module coverage); names from the module if usable,
   else the canonical bible for `language_iso`.
3. `ENGKJV` -> bundled/registry English structure (names stay
   `Genesis`-style, not DBT's `GENESIS`).
4. Otherwise -> canonical bible per `language_iso` registry
   (`{'eng': 'ENGESV', 'lvs': 'LAVNLI', 'rus': 'RUSSYN'}`),
   final fallback `ENGESV`.

Rejected "always canonical-per-language names": discards
authentic per-bible naming (e.g. `LAVNLI` vs Glück differ);
rejected "always the bible's own names": caps/quirky
orthographies (`ENGKJV`, `RUSIBS` CARS titles) degrade display.

### Response contract

```
{books: [{book_id, name, name_short, testament,
          chapters: [1,2,...]}]}
```

Own contract, DBT as a replaceable source — if DBT is ever
dropped, only the sourcing registry changes. Filter entries
to the recognized USFM set (drops `C01`-style entries and
deuterocanon for v1).

### Caching

View uses `VercelEdgeCacheMixin` (catalog class, like
`bible/translations/`: `max-age=86400,
stale-while-revalidate=604800`) + a module-level dict/LRU in
the provider layer (24h TTL). Client caches per bible abbr —
testament-split filesets share one structure.

### Frontend structure resolution (stale-while-revalidate)

`structureFor(fileset)` resolution order: localStorage (per
bible abbr) -> bundled locale catalog (`en`/`lv`/`ru` names +
`bibleStructure.json` coverage) -> previous structure held
rendered -> fetch. Swap names+coverage atomically. A new
Zustand slice (`structureByBible`, transient — persistence via
cacheManager only).

### Sync callers keep the bundled fallback

`parseBibleRef`, `decodeVerses` clamping, `scriptureMention`
validation and `getAdjacentChapters` are synchronous; they
keep reading `bibleStructure.json` (the 66-book canon is a
safe superset for parsing). The loaded structure overlays
*display*; mention/URL matching gains localized names
additively via a `localizedBookName(code)` lookup fed by the
active structure.

### Greyed-but-clickable uncovered books

Books absent from the loaded structure render dimmed (not the
`disabled` attribute — it swallows clicks); click -> localized
notification `"{translation} doesn't include {book}"`, no
navigation. Audio `alternates`/substitution untouched.

## Risks / Trade-offs

- SWORD module names may be historical/odd (Glück
  orthography) -> fall back to `LAVNLI` names; verify module
  name quality during implementation.
- Partial-canon translations show fewer picker books -> by
  design; greying keeps discoverability.
- Stale structure lets a click reach an uncovered book ->
  existing `book_not_in_fileset` notification path handles
  it; self-corrects when structure lands.
- Endpoint adds a provider call per cold fileset -> bounded
  by LRU + edge + client caches (~once per language per
  deploy).
- Testament-split products -> structure cached per bible
  abbr, not per fileset id.

## Migration Plan

1. Backend PR: endpoint + provider surface + registry +
   tests. Ships dormant (no caller yet).
2. Frontend PR: structure slice + picker/book-name display +
   SWR + greyed books.
3. Rollback: both changes are additive — revert PRs; the
   endpoint can stay harmlessly deployed.

## Open Questions

- `name` vs `name_short` per surface (picker column width vs
  compact mentions) — pick at implementation; default `name`.
