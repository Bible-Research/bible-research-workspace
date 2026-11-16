# Spec Delta

## ADDED Requirements

### Requirement: Structure resolution uses provider dispatch

Book/chapter structure lookups for a `fileset_id` SHALL route
through the same facade and dispatch order as passage and audio
requests (`provider_for`: ESV registry → API.Bible registry →
SWORD registry → DBT), with provider-specific structure
sources only where the provider cannot answer.

#### Scenario: SWORD fileset resolves via SWORD provider

- **WHEN** a structure request carries `LVSGLU8`
- **THEN** the facade resolves the SWORD provider and the local
  module supplies coverage — not a DBT lookup

#### Scenario: Unknown fileset falls through to DBT

- **WHEN** a structure request carries a fileset id present in
  no registry
- **THEN** the DBT provider supplies the structure, keyed on
  the fileset's owning bible
