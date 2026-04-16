# 2026-04-16 KE-First Greenfield Foundation Design

## Goal

Reset the platform around a knowledge-engine-first architecture optimized for business memory, not workflow breadth. The objective is to build a business-native knowledge engine that can ingest heterogeneous business inputs, curate durable entity-backed artifacts, update them over time, and retrieve current company context better than generic chat systems.

This design explicitly deprioritizes sponsor workflows, broad product shell concerns, and other secondary platform surfaces until the knowledge engine is strong on its own.

## Product Focus

The initial product is an API-first business knowledge engine with minimal debugging or operator tooling as needed. It is not a full platform rebuild yet.

The knowledge engine must be able to:

1. Ingest data from:
   - internal docs/files
   - chat
   - email/Outlook
   - public web/news
2. Normalize that data into durable business knowledge.
3. Curate current-state artifacts for important business entities.
4. Update those artifacts safely over time.
5. Retrieve current context intelligently, with provenance and visibility awareness.

## Core Object Model

The engine is entity-centric, not document-centric.

### Sources

Raw inputs from external and internal channels. Sources are always retained as ground truth and can be reprocessed later.

Examples:
- uploaded documents
- email messages
- chat threads
- news articles
- public web pages

### Observations

Normalized business-relevant facts, events, or signals extracted from sources.

Each observation should include:
- `primary_entity_ref`
- optional `related_entity_refs`
- `kind`
- confidence
- provenance back to source

Observations are the engine's working evidence layer. Raw sources remain canonical evidence, but observations make curation and retrieval business-native.

### Entities

The business objects or durable concerns the company actually runs on.

V1 entities should be explicit and business-specific, not generic knowledge blobs. Examples include:
- initiative
- budget
- KPI
- policy
- decision
- customer/account
- vendor
- department
- employee
- prospect
- company_context
- market_thesis
- operating_risk

Some entities are concrete business objects; some are abstract durable concerns. Both are valid as long as they are first-class business concepts.

### Primary Artifacts

The curated current-state record for an entity.

Rules for v1:
- every canonical artifact is entity-backed
- each entity has one approved primary artifact
- proposed drafts may exist, but only the approved primary artifact is the default retrieval surface

Primary artifact names should be uniform and engineer-friendly:
- `initiative_record`
- `budget_record`
- `employee_record`
- `company_context_record`

### Timeline

A separate temporal layer that records how understanding evolved.

Timeline events may include:
- extracted observations
- proposed revisions
- approvals
- priority changes
- candidate entity creation
- entity merges
- direct evidence attachment

The primary artifact holds current state. The timeline holds change history and process history.

### Packages

Assembled outputs for a moment, workflow, or audience such as operating reviews, board packs, or monthly finance packets.

Packages are outputs built from artifacts and timelines. They are not the canonical memory model.

## Artifact Model

Primary artifacts should be block-structured from day one.

Use a shared core block schema first, with entity-specific extensions only when real friction appears.

### Shared Core Blocks

- `overview`
- `status`
- `priority_assessment`
- `risks`
- `decisions`
- `open_questions`
- `sources_provenance`

### Example Extension Blocks

- initiative:
  - `milestones`
  - `dependencies`
- budget:
  - `variance`
  - `forecast`
- policy:
  - `rules`
  - `exceptions`

Blocks should remain human-readable and allow narrative where useful. The design is not a rigid database form. It is a typed business record with structured and narrative blocks.

## Priority Model

Priority is not a primary entity. It is a first-class assessment layer attached to an entity's primary artifact.

For v1:
- priority is human-authored
- priority is embedded as a structured block inside the primary artifact
- priority is versioned through artifact revision history and timeline events

The priority block should include:
- `priority_level`
- `business_impact`
- `risk`
- `cost_or_effort`
- `urgency`
- `strategic_alignment`
- `confidence`
- `rationale`

V1 stores and explains human judgment. Future versions can add system recommendations, conflict detection, and resource-aware prioritization.

## Update And Curation Model

### Default Update Path

The engine should use an append-and-revise model by default.

- primary artifacts are living current-state objects
- new observations propose revisions to specific blocks
- full regeneration is a special maintenance path, not the standard write path

This preserves continuity and trust while still allowing deep refreshes later when an artifact becomes structurally stale.

### Evidence And Contribution Model

System-driven contribution should normally flow through:

`source -> observation -> artifact revision`

Human users should also be allowed to link a source directly to an artifact as supporting evidence even if no clean observation exists yet.

This preserves machine structure without blocking useful human judgment.

### Auto-Apply Policy

Auto-apply should exist, but only for clearly bounded, low-risk, factual updates.

Examples:
- objective status from a trusted system of record
- verified title change
- objective budget actuals update

Interpretive changes should stay proposed until reviewed.

Every auto-applied change must still create:
- artifact history
- provenance
- timeline event
- user-visible change trace

### Candidate Entities

The system may propose candidate entities from evidence, but humans must confirm, merge, or discard them before they enter the canonical graph.

This is one of the highest-risk automation surfaces and should remain human-in-the-loop in v1.

### Unresolved Inbox

The system needs an unresolved inbox from day one, but it should be an inbox for meaningful ambiguity, not raw exhaust.

It should receive observations that are:
- important but unmapped
- possible new entities
- ambiguous entity matches
- conflicting with current artifact state
- requiring review before application

Raw sources are stored regardless. Only meaningful extracted observations should surface into the unresolved workflow.

## Retrieval Model

Retrieval should be layered.

### Retrieval Order

1. First pass:
   - approved primary artifacts
2. Second pass when needed:
   - supporting artifacts
   - linked evidence
3. Third pass when relevant to the conversation:
   - timeline/history

This keeps answers anchored in curated current state while still allowing drill-down for questions about why, how, or what changed.

### History Usage

Timeline/history should not dominate normal retrieval. It is a second or third pass used when:
- the user asks about change over time
- the current artifact is ambiguous or stale
- rationale or conflict matters

## Visibility And Permissions

V1 should stay simple while remaining ACL-ready.

### V1 Visibility Scopes

- `private`
- `department`
- `company`
- `executive`

Each approved primary artifact has one visibility scope in v1.

Visibility should be modeled as a first-class policy object or field, not an ad hoc flag, so richer ACLs can plug in later without redesigning artifacts or retrieval.

## Views, Derivatives, And Packages

The system should not default to making every audience-specific slice into a new durable artifact.

Recommended model:
- primary artifact = canonical current-state object
- simple audience-specific slicing = permission-aware view
- durable derived artifact = only when a projection becomes a genuinely distinct narrative worth owning
- package = assembled output built from multiple artifacts for a moment or workflow

This avoids both artifact explosion and overly complicated section-level permissions in v1.

## Explicitly Out Of Critical Path

These areas are not part of the first greenfield KE proof:

- sponsor workflows
- claim/release flows
- notification systems as a major surface
- onboarding UX as a major product surface
- broad public/private profile layering workflows
- workflow-heavy platform shell concerns

They may inform future architecture, but they should not shape the KE core at this stage.

## Initial Build Boundary

The first greenfield build should prove:

- ingestion across the four day-one channels
- entity creation and matching with human confirmation for new entities
- block-structured primary artifact curation
- evidence-backed artifact updating
- unresolved inbox handling
- approved-primary-first retrieval
- simple visibility-aware retrieval

If this core is excellent, broader platform workflows can later be rebuilt around it. If it is weak, no amount of workflow surface will compensate.
