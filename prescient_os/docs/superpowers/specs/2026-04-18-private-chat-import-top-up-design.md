# Private Chat Import And Top-Up Design

## Goal

Build a repeatable import path for private founder chat history that:

- ingests Claude archive exports
- ingests live local Claude history
- ingests live local Codex history
- preserves raw source material in `prescient_os_data`
- normalizes main-thread conversations into sparse session records
- supports incremental top-up without duplication
- cuts frozen benchmark snapshots for retrieval evaluation

This is benchmark-support infrastructure. It is not a KE artifact-promotion workflow and it is not a general product ingestion surface yet.

## Why This Slice Exists

The active POC has shifted from Peloton-only benchmarking toward retrieval testing on PrescientOS's own history. That gives us:

- a corpus the user can judge for correctness without reading public filings line by line
- real pivots, superseded claims, and point-in-time conflicts
- a corpus that can keep growing with the company

The immediate purpose is to get realistic private data into the retrieval benchmark. The design should support later reuse, but it should stay narrow.

## Repo Split

### `prescient_os`

Owns the reusable import capability:

- provider adapters/parsers
- normalized session models
- import metadata models
- dedup and drift-detection logic
- snapshot-cut logic

This code is portable product capability. Claude/Codex normalization is not inherently private or benchmark-specific.

### `prescient_os_data`

Owns private machine-local state:

- raw mirrored source files
- import state / source manifests
- machine-local source configuration
- normalized session outputs
- frozen benchmark snapshots

This repo is the private execution surface. It holds sensitive data and should remain separate from the product code repo.

## Source Coverage For V1

The first importer supports exactly three source kinds:

- `claude_archive`
  - `~/prescient_os-chat-history-2026-04-18.tar.gz`
- `claude_live`
  - local Claude history/session files on the current machine
- `codex_live`
  - local Codex history/session files on the current machine

The importer should treat the archive as a Claude source, not a generic bundle.

## Scope Boundary

The importer should normalize only the main conversation thread into retrieval content.

Preserve in raw, but do not index in V1:

- Claude subagent transcripts
- Claude tool-result payloads
- other provider-specific auxiliary files that are not part of the main parent conversation

Those files remain useful for provenance and later parsing improvements, but including them in retrieval text now would widen scope and increase noise.

## Identity And Deduplication

Canonical identity:

- `source_provider + session_id`

Drift detection:

- `content_hash`

Meaning:

- if a session is unseen, import it
- if a session is seen and the hash is unchanged, skip it
- if a session is seen and the hash changed, update the raw mirror, record the drift, and rebuild the normalized session

The canonical identity is not the content hash. Provider/session identity is more stable for incremental top-up, while the hash helps detect changed upstream session files.

## Data Layout In `prescient_os_data`

### Mutable raw mirror and derived normalized layer

Recommended top-level shape:

- `raw/claude/...`
- `raw/codex/...`
- `imports/chat_sources_manifest.yaml`
- `normalized/sessions/<provider>__<session_id>.yaml`
- `snapshots/<snapshot_id>/...`

### Import state

`imports/chat_sources_manifest.yaml` should track one record per imported session:

- `source_kind`
  - `claude_archive`
  - `claude_live`
  - `codex_live`
- `source_provider`
- `session_id`
- `raw_relpath`
- `normalized_relpath`
- `content_hash`
- `first_seen_at`
- `last_seen_at`
- `source_label`
  - machine/archive label useful for debugging provenance

This file is the key to idempotent top-up behavior.

## Normalized Session Shape

The importer should emit the same sparse normalized session format already established for the private corpus scaffold:

- `session_id`
- `source_provider`
- `source_app`
- `title` if known
- `exported_at`
- `raw_filename`
- `content_hash`
- `provider_metadata`
- `messages`
  - `index`
  - `role`
  - `timestamp` if known
  - `content_text`

System messages should be preserved in normalized sessions but excluded later from retrieval text by the benchmark loader.

## Top-Up Behavior

Top-up runs should be incremental and idempotent.

For each configured source:

1. discover candidate sessions
2. derive `source_provider + session_id`
3. compute or load `content_hash`
4. compare against `imports/chat_sources_manifest.yaml`
5. if unseen:
   - copy raw file(s) into the raw mirror
   - normalize the session
   - add manifest entry
6. if seen and unchanged:
   - do nothing
7. if seen and changed:
   - refresh raw mirror
   - rebuild normalized session
   - update manifest entry

Top-up operates on the mutable raw mirror and normalized layer. It does not mutate frozen benchmark snapshots.

## Snapshot Cutting

Benchmark snapshots should be frozen and immutable.

Recommended snapshot contents:

- `manifest.yaml`
- `normalized/sessions/...`
- `memos/...` if present

Optional:

- raw references or hashes in the manifest for provenance

The workflow becomes:

1. top up raw + normalized source material
2. manually curate point-in-time memos as needed
3. cut a frozen snapshot
4. point `prescient_os` benchmark commands at that snapshot

This keeps evaluations reproducible and prevents live import churn from changing benchmark runs.

## Commands

### In `prescient_os`

Provide reusable command functions or modules for:

- source discovery
- raw import
- normalization
- snapshot cutting

The implementation can surface these through a thin CLI later, but the code should be usable by a runner in `prescient_os_data`.

### In `prescient_os_data`

Add thin runner scripts or commands that call into `prescient_os` to:

- top up all configured sources
- cut a frozen snapshot from the current raw/normalized state

The runner layer may carry machine-local configuration such as:

- archive file paths
- live profile-space source paths
- source labels for provenance

## V1 Non-Goals

- no automatic memo generation
- no artifact-promotion workflow
- no ontology/entity mapping
- no ACL/visibility logic
- no synthesis evaluation changes
- no indexing of subagent/tool-result content into retrieval text
- no direct ingestion from live machine state inside the product repo

## Success Criteria

The slice is successful when:

- the importer can ingest Claude archive, live Claude, and live Codex sources into `prescient_os_data`
- re-running top-up does not duplicate unchanged sessions
- changed sessions are detected via `content_hash`
- normalized sessions can be rebuilt from raw sources
- a frozen snapshot can be cut for the existing private-corpus loader in `prescient_os`
- the benchmark can point at the frozen snapshot without needing access to live machine state
