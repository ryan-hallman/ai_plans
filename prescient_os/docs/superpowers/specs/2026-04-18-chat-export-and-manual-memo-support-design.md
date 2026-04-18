# Chat Export and Manual Memo Support Design

## Purpose

This design defines a narrow support slice for the retrieval benchmark:

- export chat histories from user-profile space
- normalize them into a sparse, provider-agnostic format
- import them as evidence into the benchmark corpus
- support a small set of manually curated, AI-assisted memos as point-in-time benchmark documents

This is not KE product modeling work. It is benchmark support for testing retrieval methodology with more realistic founder/company data.

## Scope

### In scope

- raw chat export from provider/user-profile space
- sparse normalized session format
- manifest for exported sessions
- import of normalized sessions as benchmark evidence documents
- manually created point-in-time memos produced in separate AI-assisted sessions

### Out of scope

- automated artifact promotion
- ontology-heavy company modeling
- memo automation or review workflows
- synthesis evaluation
- ACL / audience modeling
- productized chat ingestion workflows

## Design goals

- preserve provenance and time context
- keep export format simple enough to run from multiple workstations
- avoid overdesigning provider-specific schemas
- support later promotion of evidence into benchmark memos without making that part of the export pipeline
- keep the benchmark focused on retrieval methodology

## Export model

The export should use a sparse, two-layer structure:

- `manifest.yaml`
- `raw/<provider>/<session_id>.*`
- `normalized/sessions/<session_id>.yaml`

### Raw layer

The raw layer preserves provider-native session exports exactly as obtained from the user-profile space. This is the audit and reprocessing source of truth.

### Normalized layer

Each normalized session is the canonical, provider-agnostic representation used for later import and retrieval indexing.

Required fields:

- `session_id`
- `source_provider`
- `source_app`
- `exported_at`
- `raw_filename`
- `content_hash`
- `messages`
  - `index`
  - `role`
  - `content_text`

Optional fields:

- `title`
- `model`
- `has_tool_calls`
- `system_prompt_present`
- `provider_metadata`
- message-level `timestamp`

### Messages

Messages should preserve roles explicitly:

- `system`
- `user`
- `assistant`
- `tool` if present and easily exportable

System messages should be preserved in normalized sessions but excluded from retrieval indexing by default later.

## Import posture

Imported chat sessions should be treated as **evidence documents**, not as knowledge objects.

The import path should preserve:

- ordered message content
- timestamps when available
- provider/source metadata
- enough provenance to support later point-in-time curation

The import path should not attempt to infer business entities, company ontology, or durable artifacts.

## Manual memo layer

Point-in-time memos are allowed, but only as manually curated benchmark fixtures.

Workflow:

1. export/import chats as evidence
2. choose one or more sessions or ranges
3. use a saved AI prompt/template in a separate session to draft a memo
4. human reviews/finalizes
5. save the memo as a normal corpus document with source references

This keeps memos useful for retrieval testing without introducing an automated curation system.

### Memo shape

Recommended fields:

- `memo_id`
- `title`
- `as_of`
- `topic_bucket`
- `summary`
- `key_points`
- `open_questions` (optional)
- `source_refs`
  - `session_id`
  - optional message ranges
  - optional git refs

## Organization model

For PrescientOS benchmark materials, durable organization should be topic-first, with time-based memos as supporting artifacts.

Recommended top-level buckets:

- `Product Strategy`
- `Knowledge Engine`
- `Evaluation and Benchmarks`
- `Corpus and Source Material`
- `Engineering and Infrastructure`
- `Go-to-Market and Market Learning`
- `Company Operations`

Time-based memos such as weekly or monthly updates should exist as artifact types with timestamps, not as the primary taxonomy.

## Why this is the right boundary

This design intentionally separates:

- evidence capture
- later manual curation
- future KE product modeling

That keeps the current effort aligned to the benchmark goal:

- get realistic founder/company data into the engine
- preserve point-in-time differences and pivots
- test retrieval methodology with better support data

without turning benchmark support tooling into a second product surface.

## Next implementation target

The next implementation slice should cover:

- sparse export schema
- manifest format
- importer for normalized sessions
- saved prompt/template for manual memo drafting

It should not include automated memo generation, ontology work, or retrieval+synthesis scoring expansion.
