# Knowledge Management System — Design Spec

**Date:** 2026-04-13
**Status:** Draft
**Context:** Prescient OS operator-first platform

## Overview

A knowledge management system that lets departments upload and author institutional knowledge — policies, SOPs, training materials, vendor guides, playbooks, reference docs — and makes it queryable through the existing copilot chat interface. The primary user experience is conversational Q&A grounded in the organization's knowledge base, with citations linking back to source documents.

## Goals

- Departments can upload files or author knowledge entries directly in the app
- Heavy auto-curation: the system chunks, summarizes, extracts entities, and builds relationships automatically
- Staff ask questions in the copilot and get grounded, cited answers drawn from the knowledge base
- Role-based access control scoped by department, compatible with planned WorkOS integration
- Semantic search from day one via embeddings alongside BM25 keyword search

## Non-Goals (this phase)

- External integrations (Google Drive, SharePoint, Confluence) — architecture supports them, not built yet
- Full per-document ACLs — role-based access now, ACL expansion path designed in
- Knowledge graph relationships between entities across sources — entities are extracted but graph traversal is future work

---

## Architecture

New `knowledge_mgmt` bounded context within the existing DDD modular monolith. Follows the same patterns as `intelligence`, `artifacts`, `documents`, etc.

**Boundary:** Owns all KM-specific tables, services, and API routes. Integrates with the copilot via the existing tool registry (exposes a `search_knowledge_base` tool). Integrates with auth via the existing `UserRole` enum and a new department membership model.

**Why a new context (not extending artifacts or documents):**
- KM entries have a distinct lifecycle: upload → process → chunk → embed → query
- Artifacts are curated outputs (briefs, decision records) — different purpose and lifecycle
- Documents context is read-only (external ingestion) — adding upload and processing would fundamentally change its nature
- Clean separation allows independent evolution of the ingestion pipeline, access model, and future integrations

---

## Data Model

Schema: `km`

### `km.sources`

The uploaded file or authored entry. Unit of access control and versioning.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `tenant_id` | UUID | FK to tenants |
| `company_id` | UUID | FK to companies |
| `title` | text | Document title (AI-suggested, user-confirmed) |
| `source_type` | enum | `file_upload`, `authored`, `integration` |
| `original_filename` | text | Nullable, for file uploads |
| `mime_type` | text | Nullable, for file uploads |
| `storage_path` | text | S3/local path to raw file |
| `department_id` | UUID | FK to km.departments |
| `visibility` | enum | `company_wide`, `department_only` |
| `min_access_role` | enum | References existing `UserRole` enum |
| `status` | enum | `pending`, `processing`, `ready`, `failed` |
| `created_by` | UUID | FK to users |
| `updated_by` | UUID | FK to users |
| `meta` | JSONB | Auto-extracted tags, topics, effective date, error details |
| `created_at` | timestamp | |
| `updated_at` | timestamp | |

### `km.source_versions`

Version history when a source is updated.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `source_id` | UUID | FK to km.sources |
| `version_number` | integer | Sequential |
| `storage_path` | text | Path to this version's raw file |
| `change_summary` | text | Description of changes |
| `created_by` | UUID | FK to users |
| `created_at` | timestamp | |

### `km.chunks`

Processed content segments with embeddings.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `source_id` | UUID | FK to km.sources |
| `source_version_id` | UUID | FK to km.source_versions |
| `chunk_index` | integer | Ordering within source |
| `content` | text | Raw chunk text |
| `summary` | text | AI-generated 1-2 sentence summary |
| `token_count` | integer | |
| `embedding` | vector | For pgvector or synced to OpenSearch k-NN |
| `meta` | JSONB | Section heading, page number, heading hierarchy |
| `created_at` | timestamp | |

### `km.entities`

Auto-extracted entities (people, processes, systems, policies).

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `tenant_id` | UUID | FK to tenants |
| `name` | text | Entity name |
| `entity_type` | text | person, process, system, policy, etc. |
| `meta` | JSONB | Additional context |
| `created_at` | timestamp | |

### `km.entity_mentions`

Links entities to the chunks where they appear.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `entity_id` | UUID | FK to km.entities |
| `chunk_id` | UUID | FK to km.chunks |
| `context_snippet` | text | Surrounding text for context |

### `km.departments`

Department registry per company.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `tenant_id` | UUID | FK to tenants |
| `company_id` | UUID | FK to companies |
| `name` | text | Department name |
| `slug` | text | URL-safe identifier |
| `created_at` | timestamp | |

### `km.department_memberships`

User-to-department assignments. Lightweight join table, designed to be populated by WorkOS directory sync later.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `user_id` | UUID | FK to users |
| `department_id` | UUID | FK to km.departments |
| `is_lead` | boolean | Department lead flag |
| `created_at` | timestamp | |

### Future: `km.access_grants`

Not built in this phase. Expansion path for full ACL:

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `source_id` | UUID | FK to km.sources |
| `grantee_type` | enum | `user`, `group`, `role` |
| `grantee_id` | UUID | Polymorphic FK |
| `permission` | enum | `view`, `edit`, `manage` |

When present, `access_grants` overrides the simple role-based check. Search filters expand to include grant lookups — the interface stays the same.

---

## Ingestion Pipeline

Celery-backed async pipeline. Each stage is an independent task that chains to the next. Failure at any stage marks `status=failed` with error details in `meta` and allows retry from the failed stage.

### Stage 1: Intake

- **File upload:** Validate mime type, store raw file, create `km.sources` record with `status=pending`
- **Authored entry:** Store content directly, skip to Stage 2
- **Supported formats:** PDF, Word (.docx), plain text, Markdown, HTML

### Stage 2: Text Extraction

- PDF → `pdfplumber` or `pymupdf` (preserves structure, tables, headers)
- Word → `python-docx`
- Output: structured text with section headings and page boundaries in metadata

### Stage 3: Chunking

- Semantic chunking — split on section headings and paragraph boundaries, not arbitrary token counts
- Target chunk size: ~500 tokens with ~50 token overlap
- Each chunk gets positional metadata: section heading, page number, heading hierarchy
- Write `km.chunks` records

### Stage 4: Enrichment (AI-powered)

- **Per-chunk:** Generate embedding via embedding API (e.g., `text-embedding-3-small`)
- **Per-chunk:** Generate 1-2 sentence summary
- **Per-source:** Extract metadata — suggested title, department, document type, key topics, effective date
- **Per-source:** Entity extraction — identify people, processes, systems, policies mentioned
- Write embeddings to chunks, entities to `km.entities` + `km.entity_mentions`, suggested metadata to `km.sources.meta`

### Stage 5: Indexing

- Sync chunks to OpenSearch: BM25 full-text index + k-NN vector index
- Index naming: `km_chunks__{tenant_slug}` following existing slug-based sharding
- Mark source `status=ready`

### Orchestration

Celery task chain: `intake → extract_text → chunk → enrich → index`

Each stage is idempotent and retryable. Failed stages store error context in `km.sources.meta` for diagnostics. Workers can be scaled independently per stage.

---

## Search & Retrieval

### Dual search strategy

**BM25 (keyword):** OpenSearch full-text search over chunk content. Effective for exact terms — policy numbers, specific procedures, named processes. Filters by department, source type, visibility, date range.

**k-NN (semantic):** OpenSearch k-NN plugin with chunk embeddings. Effective for natural language questions — "how do we handle employee terminations" matches "Offboarding Procedure" even without keyword overlap.

### Hybrid fusion

Both searches execute in parallel. Results merged via Reciprocal Rank Fusion (RRF) — simple, effective, no tuning required. Top-k chunks returned with: content, summary, source metadata, section context.

### Access filtering

Applied at query time as OpenSearch filters (not post-retrieval). Filters on: `tenant_id`, `company_id`, user's department memberships, user's role level. Users never see results from documents they don't have access to. This preserves correct pagination and ranking.

---

## Copilot Integration

### New tool: `search_knowledge_base`

Registered in the existing tool registry alongside `search_documents`, `search_artifacts`, etc.

- **Input:** `query` (string), optional `department` filter, optional `source_type` filter
- **Output:** Top-k chunks with content, summary, source title, department, section context
- **Access:** Passes requesting user's departments and role level to the search layer

### Planner behavior

No changes to the planner loop. The existing tool selection logic picks `search_knowledge_base` when the question sounds like internal knowledge. The planner can combine it with other tools in a single turn for cross-domain questions.

### Grounding and citations

Existing grounding validator works as-is. Chunks become citable sources. Citations reference source document title + section/page when available. Example: *"According to the Employee Onboarding SOP (Section 3: Equipment Setup)..."*

### Context scoping

- Company context: copilot scopes KM queries to the active company automatically
- Department context: if viewing a department-specific page, default the department filter (user can override)

---

## Access Control

### Current phase: role-based with department scoping

Access is determined by two properties on each source:

- **`visibility`**: `company_wide` or `department_only`
- **`min_access_role`**: references existing `UserRole` enum

Role hierarchy (least to most privileged): `OPERATOR` → `PE_ANALYST` → `BOARD_MEMBER` → `OPERATING_PARTNER`

Rules:
- `company_wide` + `OPERATOR` → all users at the company can see it
- `company_wide` + `OPERATING_PARTNER` → only operating partners
- `department_only` + any role → only users who are members of that department and meet the role minimum
- Users with a role at or above `min_access_role` in the hierarchy pass the check
- Admins (determined by platform-level flag, not `UserRole`) see everything within the tenant

Note: The existing `UserRole` enum is persona-based. If the hierarchy above doesn't match the intended access semantics for KM, this enum may need to be extended with KM-specific roles (e.g., `KM_ADMIN`, `KM_CONTRIBUTOR`, `KM_VIEWER`) or replaced with a dedicated `km.access_level` enum. Finalize during implementation.

### Department membership

Users are assigned to departments via `km.department_memberships`. A user can belong to multiple departments. Department leads (flagged via `is_lead`) can manage sources within their department.

### Enforcement points

1. **Search time** — OpenSearch query filters enforce access before results return
2. **Direct access** — API endpoints check permissions before returning source content
3. **Copilot** — `search_knowledge_base` tool passes user context to the search layer

### WorkOS compatibility

- `department_memberships` is a lightweight join table that WorkOS directory sync can populate later
- All permission checks go through a single `KMAccessPolicy` service — backing logic can swap from "check DB" to "check WorkOS" without changing callers
- The future `km.access_grants` table maps to WorkOS's authorization model

---

## Management UI

### Navigation

New "Knowledge" section in the main nav alongside Brief, Intelligence, Triage, etc.

### Source list (landing page)

- Table of sources for the active company
- Filterable by department, source type, status
- Sortable by date, title, department
- Quick stats: source count, processing status

### Upload flow

- Drag-and-drop or file picker for file uploads (PDF, Word, text, Markdown, HTML)
- "Create entry" button for authored content — Markdown-based rich text editor
- After upload: display AI-suggested metadata (title, department, tags, document type) for user to confirm/edit
- Set visibility and access role
- Submit → processing indicator → status updates as pipeline stages complete

### Source detail view

- View original document or authored content
- See extracted metadata, entities, chunk count
- Edit metadata, visibility, access settings
- Version history — upload new version, view change log
- Processing status and error details if failed

### Department management (admin only)

- Create/edit departments
- Assign department leads
- Knowledge base stats per department: source count, last updated, most queried

---

## Omni Search Bar

Global search bar in the top navigation, available on all pages.

### Behavior

- Always visible in top nav, activated via `Cmd/Ctrl+K` keyboard shortcut
- Hybrid results in a dropdown:
  - **Quick results:** Top 3-5 matching sources by title/tag (BM25, fast). Click to open source detail.
  - **"Ask the copilot":** Always shown at the bottom of the dropdown. Click or press Enter on a natural language question to open the copilot chat pre-filled with the query.
- Same access control filtering as all other surfaces
- Scoped to active company context

### Growth path

As more content types become searchable (artifacts, action items, briefings), the omni bar becomes the universal search entry point across all of Prescient OS, not just KM.

---

## Technical Dependencies

- **Celery + Redis:** Task queue for ingestion pipeline
- **OpenSearch k-NN plugin:** Vector search (already on the cluster, needs k-NN plugin enabled)
- **Embedding API:** `text-embedding-3-small` or equivalent for chunk embeddings
- **Text extraction libraries:** `pdfplumber`/`pymupdf` for PDF, `python-docx` for Word
- **File storage:** S3 or local volume for raw uploads
