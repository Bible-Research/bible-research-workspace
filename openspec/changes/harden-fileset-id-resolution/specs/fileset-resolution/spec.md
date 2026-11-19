# Spec Delta

## Purpose

Defines how stored client fileset selections — product ids,
member ids and raw ids — resolve to concrete per-book provider
filesets, and how drifted or unroutable selections degrade
instead of reaching the API.

## ADDED Requirements

### Requirement: Product ids never reach the API

A stored id carrying the client-side product form
(`{abbr}:{kind}:{n}`, i.e. containing `:`) that matches no text
option — exact, member, or drifted-prefix lookup — SHALL
resolve as unresolvable and SHALL NOT be sent upstream
verbatim. Non-product unknown ids SHALL keep passing through,
since raw provider fileset ids are routable. The bundled-KJV
fast paths (`ENGKJV` and `ENGKJV:*` → the offline bundle)
SHALL keep resolving before this rule applies.

#### Scenario: Orphaned product id does not 400

- **WHEN** the stored selection is `ENGNIV:text:1` and no
  loaded translation owns an `ENGNIV` text option
- **THEN** resolution reports "unresolvable" and no request
  carrying `ENGNIV:text:1` is sent

#### Scenario: Raw DBT id still routes

- **WHEN** the stored selection is a plain provider fileset id
  that no loaded translation owns
- **THEN** it is sent upstream verbatim (legacy persisted
  selections keep working)

### Requirement: Reader self-heals a broken selection

When the persisted text selection resolves as unresolvable,
the reader SHALL recover to a valid selection — the
locale-default text fileset (a different value than the
persisted one, so the reset actually takes effect) — and tell
the user, rather than showing an empty chapter or a
client-input error.

#### Scenario: Drifted selection resets on load

- **WHEN** the app loads with an unresolvable product-id text
  selection
- **THEN** the selection resets to the locale default, a
  notification explains the reset, and the chapter renders
  normally

#### Scenario: Recovery happens once

- **WHEN** the unresolvable selection has been reset
- **THEN** no further resets or notifications fire until a
  new unresolvable value is stored

### Requirement: Audio selections degrade the same way

A stored audio id carrying the product form that matches no
audio option SHALL resolve as unresolvable rather than being
sent upstream verbatim — the same rule as for text ids.

#### Scenario: Orphaned audio product id does not 400

- **WHEN** the stored audio selection is `FOO:audio:1` and no
  loaded translation owns it
- **THEN** resolution reports "unresolvable" and no audio
  request carries `FOO:audio:1`

### Requirement: Note-reference surfaces skip unresolvable ids

A pinned or selected note-reference fileset that resolves as
unresolvable SHALL be skipped in the override → selection →
fallback chain, so previews and note dialogs fall back cleanly
instead of surfacing a client-input error.

#### Scenario: Unresolvable pin falls through

- **WHEN** the pinned fileset is an unresolvable product id
- **THEN** note-reference text resolves as if no pin were set

#### Scenario: Unresolvable selection falls back to ESV

- **WHEN** the selected fileset is also unresolvable for
  note-reference text
- **THEN** the preview requests the passage in the ESV fileset
