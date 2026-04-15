# Knowledge Engine MVP Finalization Design

**Date:** 2026-04-15
**Status:** Draft
**Supersedes:** `2026-04-14-knowledge-engine-design.md`
**Context:** Prescient OS is no longer optimizing for demo constraints. The Knowledge Engine must become a real MVP-grade internal knowledge backbone: reliable ingestion, security-first object handling, provider-flexible AI execution, provenance-grade retrieval, deep citations, and production-safe operations. Sponsor-share projection is explicitly deferred to a follow-on phase.

## Goals

- Finalize the canonical internal Knowledge Engine into a production-capable MVP
- Replace provider-specific model calls in engine paths with a capability-based AI platform layer
- Support tenant-configurable chat models while keeping embeddings platform-managed
- Make ingestion operationally safe: scan, quarantine, staged retries, reprocess, reindex, backfill
- Deliver progressive availability: lexical retrieval first, full semantic/enriched retrieval after
- Upgrade retrieval from fragmented search tools to one internal retrieval plane
- Deliver chunk/span provenance and deep citations end to end
- Introduce processing profiles for the highest-value document classes
- Add an eval harness to compare Anthropic, OpenAI, and Gemini on critical paths using shared datasets

## Non-Goals

- Sponsor-share projection and sponsor-visible retrieval plane
- Self-service tenant choice of embedding provider or embedding model
- A fully generalized AI platform for every app surface in one pass
- A broad external connector marketplace in this phase
- Approval workflows, source supersession semantics, or policy-review lifecycle on source versions
- A large profile taxonomy beyond the first operationally important document classes

---

## Architectural Decision

This phase adopts a platform-shaped route 2.5:

1. Build a **shared AI platform layer** with capability-specific contracts, provider adapters, execution policy, fallbacks, and evaluation tooling
2. Keep the **Knowledge Engine layer** as the owner of all product semantics: source/version state, processing profiles, retrieval, provenance, ranking, and citations
3. Finalize only the **canonical internal retrieval plane** in this phase

This intentionally avoids two failure modes:

- a local-only Knowledge Engine that hardcodes provider logic and becomes expensive to generalize later
- a premature generalized AI platform project that delays or dilutes Knowledge Engine product correctness

---

## Layer Boundary

### Shared AI Platform Layer

The shared AI platform owns model/runtime concerns only:

- capability routing
- provider adapters
- execution policy
- retry and fallback policy
- cost and latency telemetry
- tenant chat-model policy
- evaluation harness and recorded benchmark results

It must not know about:

- KM schemas
- chunk structure
- source versions
- retrieval ranking policy
- citation payload shape beyond normalized capability outputs

### Knowledge Engine Layer

The Knowledge Engine owns:

- source and source-version lifecycle
- secure object ingress and quarantine state
- staged processing orchestration
- processing-profile selection and profile behavior
- chunk creation and provenance metadata
- lexical and semantic retrieval internals
- trust-aware ranking
- citation assembly
- reindex and backfill control

It consumes the AI platform for:

- structured extraction
- classification
- reranking
- grounded generation
- embeddings
- chat where a KM workflow naturally requires it

---

## Knowledge Engine Subsystems

The engine is decomposed into seven product-facing subsystems.

### 1. Source Registry

Owns:

- `sources`
- immutable `source_versions`
- current-version pointer
- ingest and quarantine status
- source metadata and trust tier

This is the canonical identity boundary for knowledge entering the system.

### 2. Secure Object Flow

Owns:

- dirty object storage
- scan result handling
- clean object promotion
- quarantine path
- raw object deletion policy

No downstream stage is allowed to process raw bytes until the object is cleared.

### 3. Processing Pipeline

Owns the staged ingestion flow:

`pending -> scanning -> extracting -> classifying -> chunking -> embedding -> enriching -> indexing_lexical -> indexing_semantic -> ready_lexical -> ready_full`

Each stage must be:

- idempotent
- independently retryable
- observable
- resumable

### 4. Processing Profiles

Owns profile detection and profile-specific handling strategy for:

- extraction
- chunking
- metadata capture
- enrichment depth
- retrieval weighting
- citation display preferences

### 5. Retrieval Plane

Owns the unified internal retrieval contract and retrieval pipeline:

- lexical retrieval
- semantic retrieval
- fusion
- reranking
- trust-aware scoring
- current-version bias

### 6. Provenance and Citation Layer

Owns:

- page/section/span metadata
- excerpt extraction
- stable citation ids
- source-title/version resolution
- deep citation payloads for UI and grounded generation

### 7. Operational Control

Owns:

- dead-letter behavior
- source-level error metadata
- reprocess single source/version
- lexical-only reindex
- semantic-only reindex
- profile migration backfills
- embedding-profile migration hooks
- operational visibility and run auditability

---

## AI Platform Capability Model

The AI platform is capability-based, not provider-based.

### Capabilities

- `chat`
- `structured_extract`
- `grounded_generate`
- `classify`
- `rerank`
- `embed`

Each capability has:

- a normalized request contract
- a normalized response contract
- provider adapters
- execution-policy config
- telemetry hooks

### Provider Support

Adapters are implemented for:

- Anthropic
- OpenAI
- Gemini
- OpenAI-compatible providers where useful in later extensions

### Production Routing Policy

Initial production defaults:

- `structured_extract`: Anthropic primary, OpenAI fallback
- `grounded_generate`: Anthropic primary, OpenAI fallback
- `classify`: Anthropic primary, OpenAI fallback
- `rerank`: platform-selected implementation, defaulting to the best evaluated provider or internal scoring path
- `chat`: tenant-selected primary if allowed, platform fallback if configured
- `embed`: platform-managed embedding profile only

Gemini remains in:

- provider adapters
- evaluation harness
- non-default comparison paths

but is not in the initial default production fallback chain.

### Tenant Configurability

Tenants may configure:

- preferred chat model
- allowed chat providers
- fallback behavior for chat if supported by account policy

Tenants may not configure in MVP:

- embedding provider
- embedding model
- reranker provider
- extraction provider
- classification provider

This boundary is intentional. Chat model choice is a tenant-facing preference. Embedding choice is an index-governing infrastructure decision.

---

## Provider Evaluation Harness

Provider selection must be evidence-based, not taste-based.

### Critical Path Eval Tracks

Initial tracks:

- `structured_extraction`
- `grounded_answer_generation`
- `chat_reasoning`
- `classification`
- `reranking` when the rerank interface is ready for direct comparison

### Eval Design

Each track has:

- a fixed dataset of representative cases
- shared prompts/instructions per capability
- normalized output parsing
- automated scoring where possible
- optional human review for subjective quality cases

### Recorded Metrics

Per provider/model pair record:

- task success rate
- schema adherence rate
- unsupported-claim rate
- citation validity rate
- latency p50 and p95
- cost per successful task
- retry sensitivity
- variance across repeated runs

### Decision Rule

Production defaults should be chosen from recorded eval outcomes. Provider policy changes should be explainable from the benchmark record.

---

## Data Model

Schema remains `km`.

### `km.sources`

The stable logical identity for a knowledge object.

Key fields:

- `id`
- `organization_id`
- `company_id`
- `source_type`
- `title`
- `original_filename`
- `mime_type`
- `storage_key`
- `current_version_id`
- `ingestion_status`
- `processing_profile`
- `visibility`
- `min_access_role`
- `department_id`
- `trust_tier`
- `metadata`
- `created_by`
- `created_at`
- `updated_at`

### `km.source_versions`

Immutable history of a source. Every upload or re-upload creates a new version.

Key fields:

- `id`
- `organization_id`
- `company_id`
- `source_id`
- `version_number`
- `storage_key`
- `visibility`
- `min_access_role`
- `department_id`
- `trust_tier`
- `change_summary`
- `created_by`
- `created_at`

Rules:

- old versions remain addressable for citation stability
- `sources.current_version_id` points to the latest version
- retrieval defaults to current versions unless historical access is explicitly required

### `km.chunks`

The provenance-grade retrieval unit.

Key fields:

- `id`
- `organization_id`
- `company_id`
- `source_id`
- `source_version_id`
- `chunk_index`
- `content`
- `summary`
- `token_count`
- `visibility`
- `min_access_role`
- `department_id`
- `trust_tier`
- `page_number`
- `section_heading`
- `span_start`
- `span_end`
- `metadata`
- `created_at`

The provenance fields are mandatory for MVP:

- `source_version_id`
- `page_number` where extractable
- `section_heading` where extractable
- `span_start`
- `span_end`

### Processing Status Model

Source status must be stage-based, not binary:

- `pending`
- `scanning`
- `extracting`
- `classifying`
- `chunking`
- `embedding`
- `enriching`
- `indexing_lexical`
- `indexing_semantic`
- `ready_lexical`
- `ready_full`
- `quarantined`
- `failed`

This allows both progressive availability and precise failure visibility.

---

## Secure Object Flow

Raw object handling follows a dirty/clean/quarantine model.

### Flow

1. upload enters dirty storage
2. scan service checks object
3. if clean, object is promoted to clean storage
4. if unsafe or indeterminate by policy, object is quarantined
5. only clean objects proceed to extraction

### Scan Abstraction

The system must use a scan-service abstraction, not inline ad hoc logic.

Implementations:

- local/dev-safe implementation for development
- production implementation suitable for real malware scanning infrastructure

The abstraction returns:

- `clean`
- `quarantined`
- `failed`
- scan metadata for auditability

### Operational Behavior

- quarantined sources remain visible in operational status but are never processed further
- clean promotion and quarantine are explicit events in source metadata
- scan failures are visible distinctly from extraction failures

---

## Ingestion Lifecycle

### Entry Point

All knowledge enters through a single logical entrypoint:

- `create_source()`

This is used by:

- user uploads
- authored notes
- research pipeline outputs
- future live-data or connector paths

### Stage Flow

1. source created
2. object stored and scanned
3. text extracted into normalized representation
4. document classified and processing profile selected
5. chunks created with provenance metadata
6. lexical index written
7. source marked `ready_lexical`
8. embeddings and enrichment completed
9. semantic index written
10. source marked `ready_full`

### Product Behavior

- documents become searchable lexically before semantic enrichment completes
- failure in a late stage does not destroy earlier successful outputs
- stage reruns are targeted and idempotent

---

## Processing Profiles

The first production profile registry should include:

- `generic_document`
- `board_deck`
- `financial_export`
- `research_memo`
- `filing_or_transcript`

### Profile Responsibilities

Each profile controls:

- extraction strategy
- chunking strategy
- metadata capture
- enrichment depth
- retrieval weighting
- citation rendering preferences

### Expected Behavior

`board_deck`
- preserve slide/page boundaries
- keep terse heading hierarchy
- prefer slide-local chunks

`financial_export`
- preserve table semantics where possible
- bias extraction toward structured columns and headings
- support stronger structured extraction prompts

`research_memo`
- preserve argument blocks and executive-summary structure
- emphasize summary and conclusion extraction

`filing_or_transcript`
- preserve section hierarchy
- preserve speaker/segment metadata where applicable
- bias citations toward exact section anchors

---

## Unified Internal Retrieval Plane

There is one canonical retrieval API for internal users and workflows.

The planner and internal QA flows do not choose between multiple search tools or stores.

### Internal Retrieval Pipeline

1. lexical retrieval
2. semantic retrieval
3. fusion/merge
4. reranking
5. trust-aware scoring
6. citation packaging

### Ranking Inputs

Final ranking combines:

- lexical relevance
- semantic similarity
- rerank score
- trust-tier boost
- recency
- processing-profile weighting
- current-version boost

### Trust Order

Default ranking preference:

`approved_internal > internal_raw > system_generated > external_verified > external_raw`

This order ensures operator-trusted internal material outranks noisy external text by default.

---

## Provenance and Citation Model

All grounded-answer paths must operate on chunk-level citations, not tool-result-level citations.

### Retrieval Result Payload

Every grounded retrieval result should include:

- `source_id`
- `source_version_id`
- `chunk_id`
- `title`
- `excerpt`
- `page_number` if available
- `section_heading` if available
- `span_start`
- `span_end`
- `trust_tier`

### Grounded Generation Contract

The grounded generator receives a curated chunk set with stable citation ids.

Rules:

- factual claims must map to one or more chunk citation ids
- citation resolution is deterministic
- answer rendering resolves citations into source title, excerpt, version context, and page/section context

### Frontend Citation UX

The frontend must move from generic source pills to deep citations:

- exact excerpt
- source title
- version context
- page/section display where available
- jump-to-document context
- subtle trust-tier display
- grouping of repeated citations from the same source

This is a product requirement, not just a backend nicety. Operator trust depends on it.

---

## Operational Model

MVP readiness requires explicit production control surfaces.

### Required Controls

- stage-specific retries
- idempotent stage execution
- failed-stage visibility
- source-level error metadata
- single-source reprocess
- single-version reprocess
- lexical-only reindex
- semantic-only reindex
- backfill jobs for profile or chunking changes
- provider-eval result visibility
- ingestion progress visibility in API and UI

### Future-Safe Hook

Embedding profile changes are not self-service in MVP, but the operational model must preserve a hook for managed migration:

- profile version recorded
- full re-embed and reindex workflow available later
- rollback strategy based on index/profile versioning

---

## Concrete MVP Cut

This phase includes:

- shared AI platform capability layer
- Anthropic primary / OpenAI fallback production policy
- Gemini adapters and eval coverage without default fallback status
- tenant-configurable chat model preference
- platform-managed embedding profile
- immutable source-version model
- dirty/clean/quarantine object flow
- staged ingestion pipeline with progressive availability
- unified internal retrieval API
- rerank and trust-aware ranking
- provenance-grade chunks
- deep citation payloads and frontend rendering
- initial processing-profile registry
- reprocess, reindex, and backfill controls
- provider eval harness for critical paths

This phase excludes:

- sponsor-share projection
- sponsor-visible retrieval plane
- self-service embedding changes
- a broad external connector suite
- a large processing-profile taxonomy
- complete platform-wide AI migration for every app surface

---

## Why This Is The Right MVP

This design makes the Knowledge Engine a real product backbone rather than a hardened demo.

It is strong enough to:

- ingest and process real company knowledge safely
- support provider choice where it matters
- keep retrieval quality controlled where model differences would otherwise fragment the system
- produce trustworthy answers with deep provenance
- operate safely under retries, reprocesses, and provider changes

It also leaves clean expansion seams for:

- sponsor-share projection
- broader AI platform adoption outside the engine
- managed embedding-profile migrations
- additional document profiles and connectors
