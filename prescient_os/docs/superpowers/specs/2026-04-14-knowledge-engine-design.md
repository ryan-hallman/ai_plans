# Knowledge Engine — Design Spec

**Date:** 2026-04-14
**Status:** Draft
**Supersedes:** `2026-04-13-knowledge-management-design.md`
**Context:** Evolving Prescient OS from demo to MVP. The Knowledge Engine is the unified ingestion, processing, and retrieval backbone for all knowledge entering the system — user uploads, research pipeline outputs, filings, artifacts, news, and API-driven sources. The core promise: a user can throw thousands of pages at the system and it works out of the box.

## Goals

- One canonical pipeline for all knowledge entering the system, regardless of source
- Two retrieval planes over that pipeline: a canonical company-owned plane and an explicit sponsor-share projection
- Progressive availability: documents are BM25-searchable within seconds, semantically searchable within minutes
- Span-level citations: every copilot claim traces to a specific passage, page, and document version
- Processing profiles: the pipeline adapts its extraction, chunking, and enrichment strategy to the document type
- Reliable at volume: concurrent, staged, retryable, with clear failure visibility
- Real tenant isolation: RLS-enforced at the database level, not application convention
- Security-first: dirty/clean bucket pattern with AV scanning and prompt injection detection
- Trust-aware retrieval: approved internal material outranks raw external content by default

## Non-Goals (this phase)

- Shared OpenSearch indices across tenants — per-tenant indices for now, abstracted for later swap
- Approval states, supersession tracking, effective dates on source versions — versioning tracks history but no workflow yet
- Citation-claim semantic verification — the validator checks citations exist, not that they support the claim
- In-page passage highlighting in a document viewer — "View in document" links to the source page; highlighting is follow-on
- External integrations (Google Drive, SharePoint, Confluence) — architecture supports them, not built yet
- Knowledge graph traversal across entities — entities are extracted, graph queries are future work

---

## Relationship to Architecture Scale Review

This spec responds to the [Architecture Scale Review](2026-04-14-architecture-scale-review.md) conducted on the same date. The review identified 7 findings. This section maps each finding to what this spec addresses, what it defers, and why.

### Addressed in this spec

| Review Finding | What This Spec Does |
|---|---|
| **#1 Critical: KM pipeline not operationally coherent** | Replaces the entire pipeline. S3 replaces local disk (fixes API/worker file visibility). Unified index naming (fixes ingestion/retrieval mismatch). Staged tasks with progressive availability (fixes ready-on-failure). This was the most urgent finding and is the core of this spec. |
| **#2 High: Tenant isolation incomplete** | RLS policies on all KM tables keyed on denormalized `organization_id`. Session variable `app.current_org_id` set on every request and Celery task. Application-level filtering retained as defense-in-depth. |
| **#3 High: Retrieval fragmented across 3 stores** | Unified retrieval contract with single `search_knowledge` tool replacing `search_documents`, `search_knowledge_base`, and `search_artifacts`. Internally, the tool searches either the canonical company-owned plane or the sponsor-share projection depending on access context. |
| **#5 Medium-high: Ingest cost model won't scale** | Staged pipeline with streaming extraction (no full-file-in-memory), batched concurrent embedding and enrichment, prefork worker pool replacing `--pool=solo`, stage-specific queues with backpressure. |
| **#6 Medium: Grounding controls not sufficient** | Span-level citations replacing tool-result-level citations. Each `[[n]]` resolves to a specific chunk with source, version, page, section, and exact excerpt. |

### Designed but shipping incrementally

| Review Finding | What This Spec Does | What's Deferred and Why |
|---|---|---|
| **#4 High: Versioning not implemented end-to-end** | Source versioning data model is mandatory (`source_version_id` not nullable on chunks). Re-upload creates new version. Old version chunks retained for citation stability. | Approval states, supersession, effective dates, source-of-truth designation, and policy-change review workflows. **Why:** The pressure test needs version tracking for citation integrity, not document lifecycle management. Operators uploading board decks and SOPs won't need approval workflows in the first round — they need the system to handle re-uploads cleanly and keep old citations valid. |
| **#7 Medium: LLM provider inconsistency** | Acknowledged but not changed. OpenAI embeddings (`text-embedding-3-small`) remain for k-NN search; Anthropic remains for LLM reasoning and enrichment. | Provider consolidation (moving embeddings to Anthropic or another single vendor). **Why:** The dual-provider setup works and switching embedding models mid-MVP would require re-embedding all content. This is a governance and procurement concern that matters at enterprise scale, not during pressure testing with close contacts. The embedding service is already abstracted behind `EmbeddingService` — swapping providers later is a config change plus a re-index job, not an architecture change. |

### Additional design choices after review

| Topic | Decision |
|---|---|
| **RLS shape** | Use denormalized row-local RLS, not join-dependent RLS on child tables. Large retrieval tables like `km.chunks` need fast, auditable policies. |
| **Operator vs sponsor retrieval** | Sponsors do not search the canonical company-owned plane directly. They search an explicit share projection built from operator-visible sharing rules. |
| **External trust model** | External-origin content is searchable but lower trust by default. Ranking and answer presentation reflect that trust tier. |

### Not addressed (and why that's fine for MVP)

| Review Recommendation | Why It's Deferred |
|---|---|
| **Shared indices with routing/filtering** (replacing per-tenant indices) | Per-tenant indices are operationally simpler and provide natural isolation. The overhead only becomes a problem at 50+ tenants. For MVP with a handful of close contacts, per-tenant indices are the right call. The unified retrieval plane abstracts the index strategy — switching to shared indices later changes the search service internals without touching anything above it. |
| **Citation-claim semantic verification** (LLM validates that the cited excerpt actually supports the claim) | The grounding validator already checks that citations exist and that factual claims have markers. Semantic verification — "does this passage actually say what the response claims it says" — is a quality improvement that adds latency and cost per response. The span-level citations already make the system auditable (the user can verify themselves). Automated verification is a v1.1 polish item. |
| **Per-tenant database isolation** | The review didn't explicitly recommend this, but Ryan flagged it as a future enterprise concern. RLS on a shared database is the right model for MVP and most customers. Per-tenant databases are an enterprise sales feature for sophisticated funds with strict data residency requirements — not relevant to the pressure test. |

---

## Architecture Overview

The Knowledge Engine is a bounded context within the existing DDD modular monolith, evolving the current `knowledge_mgmt` context. It owns all ingestion, processing, storage, retrieval, and citation logic.

There is one ingestion pipeline and two retrieval planes:

1. **Canonical plane** — company-owned, private, complete. Used by operators and internal workflows.
2. **Sponsor-share projection** — explicit, derived, auditable. Used by sponsors, operating partners, and board-facing retrieval.

External systems interact through two interfaces:

1. **Ingestion API** — `create_source()` entrypoint used by the upload UI, research pipeline, live data pipelines, and future integrations
2. **Retrieval API** — `search_knowledge()` tool used by the copilot planner, replacing the three fragmented search tools and routing to the correct retrieval plane based on access context

```
                    ┌──────────────────────────────────────┐
                    │         INGESTION SOURCES             │
                    │                                      │
                    │  User Upload   Research Pipeline      │
                    │  Live Data     API Integrations       │
                    └──────────┬───────────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   create_source()    │
                    │   (single entrypoint)│
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
    ┌───────────┐      ┌─────────────┐      ┌───────────────┐
    │  S3 Store │      │  Postgres   │      │ Canonical OS  │
    │ dirty/clean│     │  (source of │      │ index per org │
    │  buckets  │      │   truth)    │      │               │
    └───────────┘      └─────────────┘      └──────┬────────┘
                                                    │
                                                    ▼
                                          ┌──────────────────┐
                                          │ Share projection │
                                          │ + sponsor index  │
                                          └────────┬─────────┘
                                                   │
                               ┌───────────────────┴───────────────────┐
                               ▼                                       ▼
                    ┌──────────────────────┐               ┌──────────────────────┐
                    │  search_knowledge()  │               │  search_knowledge()  │
                    │  operator/internal   │               │  sponsor/projected   │
                    └──────────┬───────────┘               └──────────┬───────────┘
                               └───────────────────┬───────────────────┘
                                                   ▼
                                        ┌──────────────────────┐
                                        │   Copilot Planner    │
                                        │   (span-level        │
                                        │    citations)        │
                                        └──────────────────────┘
```

---

## Data Model

Schema: `km`

Canonical KM tables denormalize access-critical fields so row-local RLS is simple and fast. At minimum, directly queried tables carry `organization_id`; retrieval-heavy child tables also carry `company_id`, `visibility`, `min_access_role`, and `department_id` so access enforcement does not depend on join-heavy policies.

### `km.sources`

The canonical record for any piece of knowledge in the system.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `organization_id` | UUID | FK → organizations. RLS boundary |
| `company_id` | UUID, nullable | FK → companies. Null for org-wide docs |
| `source_type` | enum | `uploaded_document`, `research_output`, `filing`, `artifact`, `transcript`, `news`, `authored` |
| `title` | text | Document title (AI-suggested for uploads, user-confirmed) |
| `original_filename` | text, nullable | For file uploads |
| `mime_type` | text, nullable | For file uploads |
| `storage_key` | text | S3 key for raw file (replaces `storage_path`) |
| `current_version_id` | UUID, nullable | FK → source_versions. Points to latest |
| `ingestion_status` | enum | `pending`, `scanning`, `extracting`, `chunking`, `embedding`, `enriching`, `ready`, `failed`, `quarantined` |
| `processing_profile` | text, nullable | Detected profile ID (e.g., `financial_model`, `board_deck`) |
| `visibility` | enum | `org_wide`, `company_wide`, `department_only` |
| `min_access_role` | enum | References UserRole enum |
| `department_id` | UUID, nullable | FK → km.departments |
| `trust_tier` | enum | `approved_internal`, `internal_raw`, `system_generated`, `external_verified`, `external_raw` |
| `metadata` | JSONB | Auto-extracted tags, topics, error details, scan results |
| `created_by` | UUID | FK → users |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `km.source_versions`

Immutable version history. Every upload or re-ingestion creates a new version.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `organization_id` | UUID | Denormalized from source. RLS boundary |
| `company_id` | UUID, nullable | Denormalized from source for fast filtering |
| `source_id` | UUID | FK → km.sources |
| `version_number` | integer | Monotonically increasing per source |
| `storage_key` | text | S3 key for this version's raw file |
| `visibility` | enum | Snapshotted from source for historical stability |
| `min_access_role` | enum | Snapshotted from source for historical stability |
| `department_id` | UUID, nullable | Snapshotted from source for historical stability |
| `trust_tier` | enum | Snapshotted from source |
| `change_summary` | text, nullable | "Re-uploaded with Q4 actuals" |
| `created_by` | UUID | FK → users |
| `created_at` | timestamptz | |

Unique constraint: `(source_id, version_number)`.

Old version chunks remain in the index tagged with their `source_version_id` so historical citations stay valid. `source.current_version_id` always points to the latest version.

### `km.chunks`

Processed content segments with full provenance for span-level citations.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `organization_id` | UUID | Denormalized from source_version. RLS boundary |
| `company_id` | UUID, nullable | Denormalized from source_version for fast filtering |
| `source_id` | UUID | FK → km.sources |
| `source_version_id` | UUID | FK → km.source_versions. **Not nullable** |
| `chunk_index` | integer | Position within document |
| `content` | text | Raw chunk text |
| `summary` | text, nullable | AI-generated summary (populated at enrichment stage) |
| `embedding` | vector, nullable | Populated at embedding stage |
| `token_count` | integer | |
| `visibility` | enum | Denormalized from source_version for row-local access checks |
| `min_access_role` | enum | Denormalized from source_version |
| `department_id` | UUID, nullable | Denormalized from source_version |
| `trust_tier` | enum | Denormalized from source_version for ranking |
| `page_number` | integer, nullable | From extraction |
| `section_heading` | text, nullable | From extraction |
| `span_start` | integer | Character offset in extracted text (start) |
| `span_end` | integer | Character offset in extracted text (end) |
| `metadata` | JSONB | Heading hierarchy, table context, profile-specific metadata |
| `created_at` | timestamptz | |

### `km.entities`

Auto-extracted entities (people, processes, systems, policies).

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `organization_id` | UUID | FK → organizations |
| `name` | text | Entity name |
| `entity_type` | text | person, process, system, policy, etc. |
| `metadata` | JSONB | Additional context |
| `created_at` | timestamptz | |

### `km.entity_mentions`

Links entities to the chunks where they appear.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `organization_id` | UUID | Denormalized from chunk. RLS boundary |
| `entity_id` | UUID | FK → km.entities |
| `chunk_id` | UUID | FK → km.chunks |
| `context_snippet` | text | Surrounding text for context |

### `km.departments` and `km.department_memberships`

Unchanged in purpose from the existing KM design. For consistency with the row-local RLS model, both tables carry `organization_id`, and `km.departments` also carries `company_id`. Department membership remains the user-to-department join table designed for WorkOS directory sync.

### Sponsor-share projection

Sponsors and operating partners do not search canonical KM rows directly. Instead, the system builds an explicit share projection from canonical sources/chunks that are allowed to flow upstream.

The projection is driven by:

- organization portfolio links
- visibility rules / data-category sharing rules
- source-level sharing classification
- optional operator approval workflows outside this spec

Projection records are auditable and rebuildable from canonical source versions.

Minimal projection metadata:

| Field | Description |
|---|---|
| `target_org_id` | Sponsor org that can search the projection |
| `source_id` / `source_version_id` | Canonical source provenance |
| `chunk_id` | Canonical chunk provenance |
| `company_id` | Company the chunk belongs to |
| `shared_reason` | Why it is visible upstream |
| `shared_at` | When it entered the sponsor-visible plane |

### RLS Policies

Canonical KM tables get row-local RLS policies keyed on `organization_id`:

```sql
ALTER TABLE km.sources ENABLE ROW LEVEL SECURITY;
CREATE POLICY org_isolation ON km.sources
  USING (organization_id = current_setting('app.current_org_id')::uuid);
```

Applied to: `km.sources`, `km.source_versions`, `km.chunks`, `km.entities`, `km.entity_mentions`, `km.departments`, `km.department_memberships`.

This is intentionally denormalized row-local RLS, not join-dependent RLS through parent tables. The reason is scale: `km.chunks` is expected to be the largest table in the system, and access checks need to remain simple, fast, and auditable.

Session variable `app.current_org_id` is set at the start of every request and every Celery task. This replaces the existing `app.current_fund_id` for KM tables — the primary isolation boundary for canonical content is organization, not fund.

Application-level filtering in API routes and search queries remains as defense-in-depth, not the primary boundary.

---

## Ingestion Pipeline

### Pipeline Stages

Each stage is an independent Celery task. Stage N completing enqueues stage N+1. The `ingestion_status` field reflects how far the pipeline has progressed.

```
Upload / create_source()
  │
  ▼
┌─────────────────────────────────┐
│ 1. STORE                        │
│    S3 PUT to dirty bucket       │
│    Create source + version rows │
│    Status: pending              │
│    Synchronous with request     │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ 2. SCAN                         │
│    AV scan (GuardDuty / ClamAV) │
│    Prompt injection detection   │
│    Status: scanning             │
│    Pass → move to clean bucket  │
│    Fail → status: quarantined   │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ 3. DETECT + EXTRACT             │
│    Classify document type       │
│    Select processing profile    │
│    Extract text using profile   │
│    Status: extracting           │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ 4. CHUNK + INDEX                │
│    Chunk using profile strategy │
│    Write chunks to Postgres     │
│    Index into OpenSearch (text) │
│    Status: chunking             │
│    ← BM25 SEARCH AVAILABLE     │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ 5. EMBED                        │
│    Batched embedding generation │
│    Update Postgres + OpenSearch │
│    Status: embedding            │
│    ← SEMANTIC SEARCH AVAILABLE  │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ 6. ENRICH                       │
│    LLM summarization per chunk  │
│    Entity extraction            │
│    Source-level metadata        │
│    Status: enriching            │
└──────────────┬──────────────────┘
               ▼
          Status: ready
```

### Progressive Availability

The critical design property: a document becomes searchable at stage 4 (BM25) and stage 5 (semantic), even if later stages haven't completed or have failed. A document at status `embedding` is already BM25-searchable. A document at status `enriching` is both BM25 and semantically searchable. Only `pending`, `scanning`, and `extracting` documents are not yet searchable.

### Failure and Retry

- Each stage records its outcome. Failure stops the chain but does not roll back prior stages.
- `metadata.last_error` stores the failure reason and stage.
- Any stage can be retried independently without re-running prior stages.
- Sources stuck in a failed state are visible in the management UI with actionable error details.
- A source is only marked `quarantined` (terminal) by the scan stage. All other failures are retryable.

### Re-upload / New Version Flow

1. New `source_version` row created with new S3 key
2. Pipeline runs stages 2-6 for the new version
3. Old version's chunks remain in the index (tagged with old `source_version_id`)
4. `source.current_version_id` updated to point to new version
5. Retrieval defaults to current version; historical chunks remain accessible for citation stability

### Research Pipeline Integration

Research outputs enter at stage 1. The persist step in the research pipeline calls:

```python
create_source(
    source_type="research_output",
    content=synthesized_profile,
    company_id=target_company_id,
    metadata={"research_job_id": job_id, "sources_used": [...]}
)
```

The content is stored as the raw payload in S3. The same pipeline processes it from stage 2 onward.

**Security treatment by origin class:**

- **Binary malware scanning** applies to user-uploaded files and any future file-based external integrations.
- **Prompt-injection / retrieval-safety classification** applies to all external-origin content, even when fetched by trusted internal pipelines.
- `research_output` is split:
  - the synthesized internal artifact is `system_generated`
  - the external evidence it cites remains external-origin and lower trust

The trust boundary is origin of content, not whether Prescient fetched it.

### Concurrency and Scale

- **Worker pool:** Prefork pool with configurable concurrency (default: 4 workers), replacing `--pool=solo`.
- **Batched embedding:** Chunks batched at 50 per API call with concurrency limits. A 200-page doc with 400 chunks processes in ~8 batched calls, not 400 sequential ones.
- **Batched enrichment:** LLM summarization batched similarly with rate limiting.
- **Streaming extraction:** Files streamed from S3, never fully loaded into memory. Extractor yields pages/sections as processed.
- **Backpressure:** Task priorities ensure large uploads don't starve small ones. Stage-specific queues allow independent scaling.

---

## Security Scanning

### Dirty/Clean Bucket Pattern

All user-uploaded files land in the **dirty bucket** first. No other system reads from the dirty bucket. Only after passing security scanning does the file move to the **clean bucket**, which is the only bucket the extraction stage reads from.

```
Upload → S3 dirty bucket → scan → S3 clean bucket → extract → ...
```

This ensures no unscanned content ever reaches the retrieval plane.

### AV Scanning

- **Production (AWS):** GuardDuty Malware Protection for S3. Triggers on object creation, scans automatically, publishes results via EventBridge.
- **Local dev (LocalStack):** ClamAV behind a thin service container.
- **Abstraction:** `ScanService` interface with `GuardDutyScanService` and `ClamAVScanService` implementations.

### Prompt Injection Detection

Runs alongside AV scanning for uploaded files and runs as a retrieval-safety classification stage for all external-origin text:

- Pattern matching for known injection techniques (role overrides, system prompt extraction, hidden instructions)
- Optional LLM-based classification for sophisticated injection attempts
- Flagged documents are quarantined with a human-readable reason, not silently dropped
- The operator sees "Document flagged for review" in the management UI

External-origin content that is not quarantined still enters the system with a lower `trust_tier`; passing safety checks does not upgrade it to internal-trust material.

### Ingestion Status Extension

```
pending → scanning → extracting → chunking → embedding → enriching → ready
                 ↘ quarantined (terminal, requires manual review/override)
```

---

## Processing Profiles

The pipeline adapts its extraction, chunking, and enrichment strategy based on the detected document type. This is the extensibility mechanism — new profiles can be added without changing the pipeline core.

### Profile Model

```python
ProcessingProfile {
    id: str                      # "financial_model", "board_deck", "legal_contract"
    display_name: str            # "Financial Model (Excel)"
    detector: DocumentDetector   # decides if this profile applies
    extractor: ExtractorStrategy # how to pull text/structure from the raw file
    chunker: ChunkerStrategy     # how to split into meaningful chunks
    enricher: EnricherStrategy   # what LLM enrichment to apply
    metadata_schema: JSONSchema  # what structured metadata to extract
}
```

### Detection Chain

Before extraction begins, a detection step classifies the document beyond its mime type. Detectors run in order from cheap to expensive:

1. **Mime type + filename pattern** — `*.xlsx` named "SAP_Export_Q4.xlsx" → likely ERP export
2. **Structural heuristics** — Excel with 50+ columns and headers like "GL Account", "Cost Center" → ERP export. Excel with tabs named "P&L", "BS", "CF" → financial model.
3. **LLM classification** — for ambiguous cases, send first page/sheet to a fast model for classification

The detected profile is stored in `source.processing_profile` and displayed to the user as "Detected as: Financial Model (Excel)".

### Profile Registry

```python
class ProfileRegistry:
    profiles: list[ProcessingProfile]  # ordered by specificity

    def detect(self, source: Source, raw_sample: bytes) -> ProcessingProfile:
        """Run detectors in order, return first match or general fallback."""
        for profile in self.profiles:
            if profile.detector.matches(source, raw_sample):
                return profile
        return self.general_profile
```

Adding a new profile means writing a detector and composing extractor/chunker/enricher strategies. No pipeline code changes.

### V1 Profiles

**`general` (fallback)**
- Today's pipeline behavior. Mime-based extraction, semantic chunking by token window, generic enrichment.
- Handles: plain text, Markdown, HTML, unrecognized formats.

**`board_deck` (PDF/PPTX)**
- **Detector:** PDF/PPTX + slide-like structure, executive summary patterns, agenda pages.
- **Extractor:** Page/slide-level extraction preserving visual hierarchy. Header/body distinction. Chart descriptions from alt-text.
- **Chunker:** One chunk per slide/page with context from surrounding slides. Section grouping from agenda/TOC detection.
- **Enricher:** Extract action items, decisions, owners, dates. Tag slides by function (financial, operational, strategic).

**`financial_model` (Excel)**
- **Detector:** XLSX + multiple sheets with formula relationships or financial statement headers (P&L, BS, CF).
- **Extractor:** Sheet-aware extraction preserving table structure. Each sheet becomes a logical section. Formulas rendered as computed values.
- **Chunker:** Chunk by logical table/section, not by token count. A P&L statement stays as one chunk even at 800 tokens. Cross-references captured in metadata.
- **Enricher:** Extract KPIs, time periods, currency, growth rates as structured metadata.

### Near-term Profile Additions

**`erp_export` (SAP, QuickBooks, NetSuite)**
- **Detector:** XLSX/CSV + high column count + domain-specific column headers (GL codes, cost centers, vendor IDs).
- **Extractor:** Column mapping with source system detection. Parse date formats, currency columns, categoricals.
- **Chunker:** Group by logical entity (vendor, cost center, time period). Summary statistics per group.
- **Enricher:** Detect anomalies, extract aggregates, flag outliers. Metadata includes source system, report type, date range.

**`legal_contract` (PDF/DOCX)**
- **Detector:** Numbered clauses, defined terms, signature blocks, legal boilerplate patterns.
- **Extractor:** Clause-level parsing, defined terms extraction, cross-reference mapping.
- **Chunker:** One chunk per clause/sub-clause with defined terms inlined.
- **Enricher:** Extract parties, dates, obligations, termination conditions as structured metadata.

**`sop_playbook` (PDF/DOCX/Markdown)**
- **Detector:** Step-numbered procedures, role/responsibility sections, flowchart references.
- **Extractor:** Hierarchical section extraction preserving step ordering.
- **Chunker:** One chunk per procedure/section with full heading hierarchy.
- **Enricher:** Extract roles, tools, dependencies, frequency as structured metadata.

### Pipeline Stage Integration

```
Stage 3 becomes: DETECT + EXTRACT

3a. DETECT — classify document, select profile from registry
3b. EXTRACT — using selected profile's extractor strategy

Stage 4: CHUNK — using selected profile's chunker strategy
Stage 5: EMBED — same for all profiles (embedding is content-agnostic)
Stage 6: ENRICH — using selected profile's enricher strategy
```

---

## Unified Retrieval Plane

### One Search Interface

Three current search tools collapse into one:

```python
# Before: planner picks between these
search_documents(query, company_slug, ...)      # BM25 over filings
search_knowledge_base(query, company_slug, ...) # Hybrid over KM
search_artifacts(query, company_slug, ...)      # ILIKE over artifacts

# After: one tool with optional filters
search_knowledge(query, company_id?, source_types?, date_range?)
```

The planner can filter by `source_type` if the question is clearly about filings or research, but the default is search everything within the caller's retrieval plane. This eliminates recall misses where the model searches the wrong store without letting sponsors query the canonical company-owned plane.

Deterministic tools (`get_artifact`, `get_kpi_series`, `get_filing_section`) remain as-is — they're direct lookups, not search.

### Retrieval planes

`search_knowledge()` is one tool, but it can search two different planes:

1. **Canonical plane**
   - used by operators and internal company users
   - index name: `km_chunks__{organization_id}` (UUID-based, no slugs)
   - contains complete company-owned knowledge

2. **Sponsor-share projection**
   - used by sponsor, board, and operating-partner contexts
   - index name: `km_shared_chunks__{sponsor_org_id}` (UUID-based)
   - contains only projected, explicitly shareable chunks

This preserves one retrieval contract for the planner while keeping the operator-first trust boundary intact.

### Canonical OpenSearch index schema

One canonical index per org: `km_chunks__{organization_id}` (UUID-based — aligns with slug-to-UUID migration)

Every chunk from stage 4 of the pipeline goes into this index, regardless of `source_type`.

| Field | Type | Notes |
|---|---|---|
| `chunk_id` | keyword | |
| `source_id` | keyword | |
| `source_version_id` | keyword | |
| `source_type` | keyword | Filterable |
| `organization_id` | keyword | RLS boundary |
| `company_id` | keyword, nullable | Filterable |
| `content` | text | BM25 searchable |
| `summary` | text, nullable | BM25 searchable (when available) |
| `source_title` | text | BM25 searchable |
| `embedding` | knn_vector (1536, cosine) | Nullable until stage 5 |
| `section_heading` | text | |
| `page_number` | integer, nullable | |
| `span_start` | integer | |
| `span_end` | integer | |
| `chunk_index` | integer | |
| `visibility` | keyword | |
| `min_access_role` | keyword | |
| `department_id` | keyword, nullable | |
| `trust_tier` | keyword | Ranking signal |
| `processing_profile` | keyword, nullable | |
| `created_at` | date | |
| `metadata` | object | Flexible per source_type |

### Sponsor-share OpenSearch projection schema

The sponsor projection index carries the same provenance fields plus:

| Field | Type | Notes |
|---|---|---|
| `target_org_id` | keyword | Sponsor org visibility boundary |
| `shared_at` | date | |
| `shared_reason` | keyword | Why the chunk is visible upstream |

### Hybrid Search

Same approach as current KM search, applied to the unified index:

- **BM25** on `content`, `summary`, `source_title`
- **k-NN** on `embedding` (skipped for chunks without embeddings — graceful degradation)
- **RRF merge** with K=60

### Trust-aware ranking

Retrieval ranking is not trust-neutral. By default:

1. `approved_internal`
2. `internal_raw`
3. `system_generated`
4. `external_verified`
5. `external_raw`

This is implemented as a ranking prior, not a hard filter. Lower-trust content remains searchable, but it loses ties and near-ties to higher-trust internal material.

Examples:

- An approved board narrative should outrank a scraped news article on the same topic.
- An uploaded internal SOP should outrank a system-generated research summary that references it.
- A filing excerpt can still win if it is the only directly relevant evidence.

### Access Control in Queries

Canonical-plane queries are filtered by:

1. `organization_id` — hard boundary, always applied
2. `company_id` — scoped when user is on a company page
3. `min_access_role` — user's role must meet or exceed the chunk's threshold
4. `visibility` + `department_id` — department-only chunks filtered by user membership

Sponsor-plane queries are filtered by:

1. `target_org_id` — hard boundary, always applied
2. `company_id` — optional scope to one visible portfolio company
3. any projection-specific share category filters

Sponsors do not query canonical chunks directly. Triage and operator workflows may consult the canonical plane privately and then draft sponsor-safe answers from the projected plane.

Filters are built from the authenticated user's context, not route parameters. RLS in Postgres is primary enforcement for canonical data; OpenSearch filters are performance optimization and projection scoping.

### Migration Strategy

Legacy indices (`documents__{slug}`, old `km_chunks__{tenant_id}`) stay readable during migration. New ingestion writes to the canonical unified index only. A backfill job re-indexes existing filings and artifacts into the new schema. Sponsor-facing retrieval continues to use existing share paths until the sponsor projection is backfilled. Old search tools remain registered but deprecated — the planner prefers `search_knowledge`, falls back to legacy tools for content not yet migrated.

---

## Span-Level Citations

### Citation Data Model

Every citation resolves to a specific passage in a specific version of a specific document.

```
Citation {
    marker: int                     # [[1]], [[2]], etc.
    source_id: UUID
    source_version_id: UUID
    source_title: string            # "Q4 2025 Board Deck"
    source_type: string             # "uploaded_document"
    trust_tier: string              # "approved_internal"
    chunk_id: UUID
    chunk_index: int                # position in document
    page_number: int | null         # page 14
    section_heading: string | null  # "Financial Highlights"
    span_start: int                 # character offset in extracted text
    span_end: int
    excerpt: string                 # exact passage, ~200 chars
    score: float                    # retrieval relevance score
}
```

### How It Works End-to-End

**1. Retrieval returns chunk-level provenance.**

Each chunk from `search_knowledge` carries its full provenance: `source_id`, `source_version_id`, `chunk_id`, `page_number`, `section_heading`, `span_start/end`, and excerpt.

**2. Citation map is chunk-granular.**

Each chunk from each search result gets its own marker number:

```
# Today (tool-result level):
[[1]] → tool_result (search_documents, 10 chunks)

# After (chunk-level):
[[1]] → chunk (source: "Q4 Board Deck", page 14, "Revenue grew 15.2%...")
[[2]] → chunk (source: "Q4 Board Deck", page 22, "Headcount target: 340...")
[[3]] → chunk (source: "Competitive Analysis", section "Pricing", "Acme reduced...")
```

**3. Planner prompt teaches chunk-level citation.**

```
When making a factual claim, cite it with [[n]] where n corresponds to a specific
chunk from your search results. Each chunk has a unique marker. Do not combine
multiple chunks into one citation. If a claim draws from two passages, cite both:
[[1]][[3]].
```

**4. Grounding validator is span-aware.**

- Each `[[n]]` must map to a specific chunk with full provenance
- Uncited factual claims still trigger regrounding
- Validator checks markers reference real chunks (not just "some tool was called")

**5. Frontend renders deep citations.**

Clicking `[[1]]` in a copilot response shows:

```
┌─────────────────────────────────────────────┐
│ Source [1]                                  │
│ Q4 2025 Board Deck (v2)                    │
│ Page 14 · Financial Highlights              │
│                                             │
│ "Consolidated revenue increased 15.2% YoY,  │
│  driven primarily by expansion in the mid-   │
│  market segment which grew 23% relative to   │
│  the prior quarter."                         │
│                                             │
│ ┌───────────────────┐                       │
│ │ View in document  │                       │
│ └───────────────────┘                       │
└─────────────────────────────────────────────┘
```

"View in document" links to `/knowledge/{source_id}` — in-page passage highlighting is follow-on work.

### Changes from Current Implementation

| Component | Today | After |
|---|---|---|
| Citation granularity | Per tool result (~10 chunks) | Per chunk |
| Citation map | `marker → tool_result_id` | `marker → chunk_id + full provenance` |
| `_build_citation_map` in planner | Maps markers to tool results in order | Maps markers to individual chunks |
| `_extract_citations` in planner | Grabs first record's metadata | Uses each chunk's provenance fields |
| Grounding validator | Checks markers exist | Checks markers map to real chunks |
| Frontend citation detail | Source name + generic excerpt | Source + version + page + section + exact excerpt |

### Token Cost Trade-off

More granular citations mean more tokens in the planner's context. Each chunk is labeled with its marker and provenance. For a search returning 15 chunks, approximately 200-300 extra tokens. Worth it for the audit trail.

### Trust presentation in citations

Citation cards also display the source trust tier. Lower-trust sources are visually labeled so the user can distinguish:

- approved internal guidance
- raw internal material
- system-generated synthesis
- external evidence

This supports auditability without hiding relevant evidence.

---

## Component Ownership

| Component | Location | Status |
|---|---|---|
| S3 storage service | `knowledge_mgmt/infrastructure/services/file_storage.py` | Replace `LocalFileStorage` with S3 client |
| Scan service | `knowledge_mgmt/infrastructure/services/scan.py` | New |
| Processing profile registry | `knowledge_mgmt/infrastructure/services/profiles/` | New |
| Text extractor (per profile) | `knowledge_mgmt/infrastructure/services/text_extractor.py` | Modified — streaming, page-aware, profile-driven |
| Chunker (per profile) | `knowledge_mgmt/infrastructure/services/chunker.py` | Modified — span offsets, page numbers, profile-driven |
| Embedding service | `knowledge_mgmt/infrastructure/services/embedding.py` | Modified — batched, concurrent |
| Enrichment service (per profile) | `knowledge_mgmt/infrastructure/services/enrichment.py` | Modified — batched, concurrent, profile-driven |
| Pipeline tasks | `knowledge_mgmt/infrastructure/tasks/` | Replace monolith with chained staged tasks |
| Unified search service | `knowledge_mgmt/infrastructure/services/search.py` | Modified — unified index schema |
| Sponsor-share projection builder | `knowledge_mgmt/infrastructure/services/share_projection.py` | New — builds sponsor-visible search docs from canonical chunks |
| Citation builder | `intelligence/application/planner.py` | Modified — chunk-granular citation map |
| Grounding validator | `intelligence/domain/grounding.py` | Modified — span-aware validation |
| Unified search tool | `intelligence/infrastructure/tools/search_knowledge.py` | New — replaces 3 search tools |
| RLS migration | `alembic/versions/` | New |
| Access-context routing | `auth/access_context.py` + `intelligence/infrastructure/tools/search_knowledge.py` | Modified — choose canonical vs sponsor plane at query time |
| Frontend citation UI | `components/copilot/` | Modified — deep citation rendering |

---

## Technical Dependencies

- **S3 / LocalStack:** Object storage for dirty and clean buckets
- **GuardDuty Malware Protection / ClamAV:** AV scanning
- **Celery + Redis:** Task queue for staged pipeline
- **OpenSearch k-NN plugin:** Hybrid BM25 + vector search
- **Embedding API:** OpenAI `text-embedding-3-small` (1536 dimensions)
- **Anthropic Claude:** LLM enrichment, entity extraction, document classification
- **Text extraction:** `pdfplumber`/`pymupdf` (PDF), `python-docx` (Word), `openpyxl` (Excel)
- **Postgres 16:** Source of truth with RLS and JSONB

---

## Priority Order

### Immediate (pipeline reliability + security)

- S3 storage replacing LocalFileStorage (dirty/clean bucket pattern)
- Fix index naming mismatch between ingestion and retrieval
- Staged pipeline tasks replacing monolithic ingestion
- RLS policies on all KM tables
- Scan service abstraction (ClamAV for dev, GuardDuty for prod)
- Trust-tier classification on sources and chunks
- Progressive availability (BM25 at stage 4, semantic at stage 5)

### Next (retrieval + citations)

- Unified search tool replacing three fragmented tools
- Sponsor-share projection and sponsor-visible search index
- Access-context routing between canonical and sponsor planes
- Trust-aware ranking in unified retrieval
- Span-level citation data model and planner changes
- Chunk-granular citation map in planner
- Frontend deep citation rendering
- Legacy index migration / backfill

### After that (processing intelligence + scale)

- Processing profile registry and detection chain
- `board_deck` and `financial_model` profiles
- Batched concurrent embedding and enrichment
- Prompt injection detection layer
- Worker pool scaling (prefork, stage-specific queues)
- Source versioning flow (re-upload creates new version)

### Future

- `erp_export`, `legal_contract`, `sop_playbook` profiles
- Shared indices across tenants
- Citation-claim semantic verification
- In-page passage highlighting
- External integrations (Google Drive, SharePoint)
- Source system connectors (SAP, QuickBooks, NetSuite)
