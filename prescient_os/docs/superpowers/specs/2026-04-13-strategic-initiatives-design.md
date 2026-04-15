# Strategic Initiatives & Unified Notifications — Design Spec

**Date:** 2026-04-13
**Status:** Draft
**Context:** Prescient OS operator-first platform

## Overview

A strategic initiative management system driven by value creation plans and other strategy documents, with proactive agent follow-ups, dependency and resource tracking, and a timeline/roadmap visualization. Includes a new unified notifications system that consolidates attention items across the platform, with strategy as its first consumer.

Informed by COO feedback validating the "chief of staff" positioning: operators need VCP-driven strategic management with minimal setup burden, proactive nudges, and clear visualization of dependencies and risks.

## Goals

- Operators can create initiatives manually or extract them from any uploaded strategy document (VCPs, board decks, annual plans, OKR docs)
- Guided review flow walks operators through gaps without requiring everything upfront — drafts stay incomplete until ready
- Timeline/roadmap view shows initiative health, milestones, dependencies, and resource conflicts at a glance
- Cadence engine proactively detects staleness, propagates dependency risks, and surfaces resource conflicts
- Copilot integration surfaces strategy items non-intrusively — passive indicators + end-of-task nudge, never mid-conversation interruption
- Unified notification widget consolidates attention items across the platform in a single header dropdown
- Strategy items surface in the daily briefing alongside KPIs, action items, and monitoring findings

## Non-Goals (this phase)

- Kanban view for initiatives
- Resource auto-leveling / optimization
- Saved or custom timeline views
- Outbound communication channels (email, Slack) — in-platform only
- Migrating existing contexts (action items, triage, monitoring) to publish into the notification system — architecture supports it, not built yet

---

## Architecture

Two new bounded contexts:

1. **`strategy/`** — owns initiatives, milestones, dependencies, resource assignments, evidence, and the cadence engine. Follows the existing hexagonal layering: domain, application, infrastructure, api.
2. **`notifications/`** — cross-cutting context owning the unified notification model and widget. Any context can publish into it via a `NotificationPublisher` port. Strategy is the first consumer.

**Boundary decisions:**
- Soft references for cross-context links — no hard foreign keys to KPIs, action items, or artifacts
- No cross-context domain imports — briefing reads strategy tables through its own source adapter
- LLM never publishes — document extraction produces drafts, agent follow-ups produce approval requests, humans decide
- Notifications are published through a port, not direct imports — producing contexts don't depend on the notification context

---

## Domain Model

### Core Entities

**Initiative** — the central entity representing a strategic initiative (e.g., "Margin Expansion Program"):
- `id: UUID`
- `owner_tenant_id: UUID` — soft FK to tenant/fund
- `title: str`, `description: str`
- `owner_id: UUID` — person responsible
- `status: InitiativeStatus` — `draft | active | at_risk | on_hold | completed | cancelled`
- `priority: InitiativePriority` — `critical | high | medium | low`
- `time_horizon: TimeHorizon` — `q1 | q2 | q3 | q4 | annual | multi_year`
- `start_date: date`, `target_end_date: date`
- `health_score: Decimal(0-1)` — computed from milestone progress, staleness, dependency risk
- `completion_pct: Decimal(0-1)` — derived from milestone completion
- `source_artifact_id: UUID | None` — soft reference to the document it was extracted from
- `update_cadence_days: int` — expected update frequency, default 7
- `milestones: tuple[Milestone, ...]`
- `subjects: tuple[InitiativeSubject, ...]` — polymorphic subject pattern (company, product, market, person, topic)
- `created_at`, `updated_at`, `last_updated_at` (tracks substantive updates)

**Milestone** — progress markers within an initiative:
- `id: UUID`, `initiative_id: UUID`
- `title: str`, `description: str | None`
- `assignee_id: UUID | None` — can differ from initiative owner for delegated work
- `status: MilestoneStatus` — `pending | in_progress | completed | missed | deferred`
- `target_date: date`, `completed_date: date | None`
- `sort_order: int`

**Dependency** — edges between initiatives:
- `id: UUID`
- `upstream_initiative_id: UUID`, `downstream_initiative_id: UUID`
- `dependency_type: DependencyType` — `blocks | informs | shares_resources`
- `description: str | None`

**ResourceAssignment** — who or what is allocated to an initiative:
- `id: UUID`, `initiative_id: UUID`
- `resource_type: ResourceType` — `person | team | budget`
- `resource_id: str` — person UUID, team name, or budget line identifier
- `resource_label: str` — display name
- `allocation_pct: Decimal(0-100)`
- `start_date: date | None`, `end_date: date | None`

**Evidence** — soft cross-context references linking initiatives to other entities:
- `id: UUID`, `initiative_id: UUID`
- `evidence_type: EvidenceType` — `kpi | action_item | artifact | observation`
- `evidence_id: UUID`
- `description: str | None`

**InitiativeSubject** — same composite pattern as `ActionItemSubject` / `ArtifactSubject`:
- `initiative_id: UUID`, `subject_type: SubjectType`, `subject_id: UUID`, `role: str`, `is_primary: bool`

**ApprovalRequest** — a pending change request that requires initiative owner sign-off:
- `id: UUID`, `initiative_id: UUID`
- `requester_id: UUID` — the assignee requesting the change
- `request_type: ApprovalRequestType` — `date_change | status_change | scope_change`
- `description: str` — what's being requested and why (structured by copilot from conversation)
- `target_entity_type: str` — `milestone` or `initiative`
- `target_entity_id: UUID`
- `proposed_changes: dict` — JSON of field→new_value pairs (e.g., `{"target_date": "2026-06-15"}`)
- `status: ApprovalStatus` — `pending | approved | rejected`
- `resolved_by: UUID | None`, `resolved_at: datetime | None`
- `rejection_reason: str | None`
- `created_at: datetime`

### Enums

All as `StrEnum`: `InitiativeStatus`, `InitiativePriority`, `TimeHorizon`, `MilestoneStatus`, `DependencyType`, `ResourceType`, `EvidenceType`.

### Health Score Computation

Derived (cached and recomputed on relevant changes):
- **Milestone progress (40%):** completed milestones / total milestones
- **Staleness penalty (30%):** days since last update vs. expected cadence. No update within 2x cadence → score degrades linearly
- **Dependency risk (20%):** upstream initiative `at_risk` or `on_hold` → penalty proportional to dependency type (`blocks` > `informs` > `shares_resources`)
- **Overdue penalty (10%):** any milestone past `target_date` without completion

Health thresholds: >= 0.7 green, 0.4-0.7 yellow, < 0.4 red.

---

## Unified Notifications Model

### Notification Entity

- `id: UUID`
- `recipient_id: UUID` — user who should see it
- `organization_id: UUID`
- `category: NotificationCategory` — `strategy | action_item | triage | monitoring | system`
- `priority: NotificationPriority` — `urgent | high | normal | low`
- `title: str`, `body: str`
- `source_type: str` — entity type that triggered it (e.g., `initiative`, `milestone`, `action_item`)
- `source_id: UUID` — the triggering entity
- `action_url: str | None` — deep link (e.g., `/strategy?initiative=<id>`)
- `read: bool`, `dismissed: bool`
- `created_at: datetime`, `read_at: datetime | None`

### Publishing Pattern

Contexts publish through a `NotificationPublisher` port — no direct imports of the notifications module:

```
NotificationPublisher(Protocol):
    async def publish(notification: NotificationCommand) -> None
```

### Widget Behavior

- Header dropdown with bell icon and unread count badge
- Click opens panel grouped by category, sorted by priority then recency
- Each item: icon (by category), title, relative timestamp, source link
- Per-item actions: mark read, dismiss, click-through to source
- Bulk actions: mark all read, dismiss all for a category
- Shows latest 50; "View all" link for a full page if needed later

### Strategy Notification Types

| Trigger | Recipient | Priority |
|---------|-----------|----------|
| Milestone approaching (within 7 days) | Assignee | high |
| Milestone overdue | Assignee + owner | urgent |
| Staleness warning (no update in 2x cadence) | Owner | high |
| Dependency risk (upstream slipped) | Downstream owner | high |
| Resource conflict (>100% allocation) | Both initiative owners | normal |
| Approval request (assignee requests change) | Initiative owner | urgent |

---

## Document-to-Initiatives Extraction

### Flow

Operator uploads any strategy document or starts a conversation with no document. The system extracts draft initiatives and walks the operator through gaps.

### Extraction Pipeline

**Port:** `InitiativeExtractionPort(Protocol)` with method:
```
async def extract_initiatives(document_text: str, context: ExtractionContext) -> ExtractionResult
```

**ExtractionContext:** organization info, existing initiatives (deduplication), operator's interest profile.

**ExtractionResult:**
- `draft_initiatives: list[DraftInitiative]` — each field has a `confidence: float`
- `gaps: list[GapQuestion]` — questions the system couldn't resolve from the document
- `metadata: dict` — token usage, model, etc.

### Guided Review

1. Draft list — extracted initiatives as cards. High-confidence fields (>0.8) highlighted green, gaps highlighted amber.
2. One-at-a-time gap resolution — system asks simple questions for missing/low-confidence fields.
3. Skip/defer — any question can be skipped. Initiative stays `draft`.
4. Promote to active — once core fields complete (title, description, owner, at least one milestone, time horizon). Bulk promote available.
5. Discard — operator can discard any draft.

### Conversational Intake

No document — copilot asks operator to describe priorities. Structures responses into `DraftInitiative` objects using the same port (different prompt, same output shape). Results appear in the same review screen.

### Provenance

The uploaded document is stored as an artifact using existing artifact infrastructure. The initiative's `source_artifact_id` links back to it. No new artifact type needed — provenance tracked through the link.

---

## Cadence Engine

Scheduled background process, same pattern as the existing anomaly detector. Runs on a configurable interval (default: daily).

### Detection Rules

**Staleness detection:**
- No milestone status change or manual update within 1.5x cadence → `warning` notification to owner
- No update within 2x cadence → health score degrades, initiative auto-transitions to `at_risk`

**Milestone approaching:**
- `target_date` within 7 days and status `pending` → notification to assignee (or owner if no assignee)
- Past `target_date` with status not `completed` → auto-transition to `missed`, notification to assignee + owner

**Dependency risk propagation:**
- Initiative transitions to `at_risk` / `on_hold` or has milestone slip → find downstream dependencies
- `blocks`: heavy health penalty + notification to downstream owner
- `informs` / `shares_resources`: lighter penalty + informational notification

**Resource conflict detection:**
- Sum `allocation_pct` per `resource_id` across active initiatives with overlapping date ranges
- Total > 100% → notification to both initiative owners
- Re-evaluated on initiative creation, assignment changes, and timeline shifts (not just scheduled runs)

### Health Score Recomputation

Recomputes all active initiative health scores on each run. Publishes a notification when health crosses a threshold boundary (green→yellow, yellow→red).

### Approval Request Flow

1. Copilot collects status update from assignee implying a change (e.g., "need 2 more weeks")
2. Packages as structured `ApprovalRequest` — what changed, who requested, why
3. Published as `urgent` notification to initiative owner
4. Owner approves/rejects through notification action or initiative detail page
5. Approval → system updates milestone, recomputes health, propagates downstream impacts
6. Rejection → assignee notified, milestone unchanged

### Autonomy Boundary

**Agent acts autonomously:** sending reminders, collecting responses, flagging staleness, auto-transitioning overdue milestones to `missed`.

**Agent requires human approval:** changing dates, statuses (except `missed`), scope, or ownership.

---

## Copilot Integration

### Strategy Tools

New tools registered with the copilot tool system:

- **`get_strategy_summary`** — active initiatives with health scores, upcoming milestones, open follow-ups for the current org
- **`update_milestone_status`** — allows assignee to report progress through conversation. Structures into status update or approval request. Never modifies directly.
- **`get_initiative_detail`** — full detail on a specific initiative including milestones, dependencies, resources, evidence

### Passive Attention Indicators

When a chat session opens, copilot checks for pending strategy notifications:
- Small icon cluster at top of chat panel showing count by priority (e.g., "2 urgent, 1 normal")
- Icons clickable — loads notification context into chat
- Non-intrusive: no banner, no modal, just subtle indicators in the chat header

### End-of-Task Nudge

Simple heuristic for natural conversation pauses:
1. **Gratitude signals** — "thanks", "got it", "perfect", etc.
2. **Closure signals** — "that's all", "I'm good", etc.
3. **Idle timeout** — 60+ seconds after copilot's last response

When triggered and user has pending strategy items:
> "By the way, you have 2 strategy items that could use your input. Want to take a look?"

- Yes → presents items one at a time, highest priority first
- No → drops it, doesn't ask again in that session
- **Never fires more than once per session**
- **Never fires if strategy items already surfaced** during the conversation

### Planner Context

Strategy context block injected into planner system prompt (same pattern as strategic focus):
- Count of pending notifications
- Top 3 items needing attention (title + priority only)
- Full details fetched via tools only when needed

---

## Briefing Integration

### Source Adapter

New method on `DatabaseSourceAdapter`:
```
async def get_strategy_items(*, organization_id: str) -> list[BriefingItem]
```

### What Gets Surfaced

| Condition | action_type | Base relevance_score |
|-----------|-------------|---------------------|
| Initiative health crossed to red | DECIDE | 0.9 |
| Milestone due within 3 days | DECIDE | 0.85 |
| Milestone overdue | DECIDE | 0.8 |
| Dependency risk (upstream slipped) | READ | 0.75 |
| Resource conflict detected | DECIDE | 0.7 |
| Initiative stale (no update in 2x cadence) | READ | 0.65 |
| Milestone completed yesterday | FYI | 0.5 |
| Initiative promoted to active | FYI | 0.4 |

### Integration Points

- Add `STRATEGY` to `ItemSource` enum
- Add `get_strategy_items` to `BriefingSourcePort` protocol
- Assembler calls it alongside existing sources, merges into ranked list
- `ItemScorer` boosts relevance when user interest profile matches initiative subjects

---

## Database Schema

### Schema: `strategy`

**`strategy.initiatives`**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| owner_tenant_id | UUID | indexed, soft FK to tenants.funds |
| title | VARCHAR(256) | not null |
| description | TEXT | not null |
| owner_id | UUID | not null |
| status | VARCHAR(32) | not null, indexed, default 'draft' |
| priority | VARCHAR(16) | not null, default 'medium' |
| time_horizon | VARCHAR(16) | not null |
| start_date | DATE | not null |
| target_end_date | DATE | not null |
| health_score | DECIMAL(3,2) | nullable, 0.00-1.00 |
| completion_pct | DECIMAL(3,2) | nullable, 0.00-1.00 |
| source_artifact_id | UUID | nullable, soft FK to artifacts |
| update_cadence_days | INT | not null, default 7 |
| last_updated_at | TIMESTAMP(tz) | not null |
| created_at | TIMESTAMP(tz) | not null, server default now() |
| updated_at | TIMESTAMP(tz) | not null, server default now() |

**`strategy.milestones`**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| initiative_id | UUID | FK to initiatives, indexed |
| title | VARCHAR(256) | not null |
| description | TEXT | nullable |
| assignee_id | UUID | nullable |
| status | VARCHAR(32) | not null, default 'pending' |
| target_date | DATE | not null |
| completed_date | DATE | nullable |
| sort_order | INT | not null, default 0 |
| created_at | TIMESTAMP(tz) | not null |

**`strategy.dependencies`**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| upstream_initiative_id | UUID | FK to initiatives, indexed |
| downstream_initiative_id | UUID | FK to initiatives, indexed |
| dependency_type | VARCHAR(32) | not null |
| description | TEXT | nullable |
| created_at | TIMESTAMP(tz) | not null |
| | | unique(upstream_initiative_id, downstream_initiative_id) |

**`strategy.resource_assignments`**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| initiative_id | UUID | FK to initiatives, indexed |
| resource_type | VARCHAR(16) | not null |
| resource_id | VARCHAR(64) | not null, indexed |
| resource_label | VARCHAR(128) | not null |
| allocation_pct | DECIMAL(5,2) | not null |
| start_date | DATE | nullable |
| end_date | DATE | nullable |
| created_at | TIMESTAMP(tz) | not null |

**`strategy.evidence`**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| initiative_id | UUID | FK to initiatives, indexed |
| evidence_type | VARCHAR(32) | not null |
| evidence_id | UUID | not null |
| description | TEXT | nullable |
| created_at | TIMESTAMP(tz) | not null |

**`strategy.initiative_subjects`**

| Column | Type | Notes |
|--------|------|-------|
| initiative_id | UUID | composite PK |
| subject_type | VARCHAR(16) | composite PK |
| subject_id | UUID | composite PK |
| role | VARCHAR(32) | default 'subject' |
| is_primary | BOOLEAN | default false |

**`strategy.approval_requests`**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| initiative_id | UUID | FK to initiatives, indexed |
| requester_id | UUID | not null |
| request_type | VARCHAR(32) | not null |
| description | TEXT | not null |
| target_entity_type | VARCHAR(32) | not null |
| target_entity_id | UUID | not null |
| proposed_changes | JSONB | not null |
| status | VARCHAR(16) | not null, default 'pending', indexed |
| resolved_by | UUID | nullable |
| resolved_at | TIMESTAMP(tz) | nullable |
| rejection_reason | TEXT | nullable |
| created_at | TIMESTAMP(tz) | not null |

### Schema: `notifications`

**`notifications.notifications`**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| recipient_id | UUID | not null, indexed |
| organization_id | UUID | not null, indexed |
| category | VARCHAR(32) | not null, indexed |
| priority | VARCHAR(16) | not null |
| title | VARCHAR(256) | not null |
| body | TEXT | not null |
| source_type | VARCHAR(32) | not null |
| source_id | UUID | not null |
| action_url | VARCHAR(512) | nullable |
| read | BOOLEAN | not null, default false |
| dismissed | BOOLEAN | not null, default false |
| read_at | TIMESTAMP(tz) | nullable |
| created_at | TIMESTAMP(tz) | not null, indexed |

Composite index: `(recipient_id, read, created_at DESC)` for widget queries.

### Migrations

Two Alembic migrations:
- `YYYYMMDD_strategy_schema.py` — creates `strategy` schema and all tables
- `YYYYMMDD_notifications_schema.py` — creates `notifications` schema and table

---

## API Endpoints

### Strategy Routes (`/strategy`)

**Initiatives CRUD:**
- `GET /strategy/initiatives` — list for org, filterable by status, priority, owner. Summary shape.
- `POST /strategy/initiatives` — create manually
- `GET /strategy/initiatives/{id}` — full detail with milestones, dependencies, resources, evidence
- `PATCH /strategy/initiatives/{id}` — update fields
- `DELETE /strategy/initiatives/{id}` — soft delete (status → cancelled)

**Milestones:**
- `POST /strategy/initiatives/{id}/milestones` — add
- `PATCH /strategy/milestones/{id}` — update
- `DELETE /strategy/milestones/{id}` — remove

**Dependencies:**
- `POST /strategy/dependencies` — create (upstream_id, downstream_id, type)
- `DELETE /strategy/dependencies/{id}` — remove
- `GET /strategy/initiatives/{id}/dependencies` — all upstream and downstream

**Resource Assignments:**
- `POST /strategy/initiatives/{id}/resources` — assign
- `PATCH /strategy/resources/{id}` — update allocation
- `DELETE /strategy/resources/{id}` — remove
- `GET /strategy/resources/conflicts` — active resource conflicts for org

**Evidence:**
- `POST /strategy/initiatives/{id}/evidence` — link entity
- `DELETE /strategy/evidence/{id}` — unlink

**Document Extraction:**
- `POST /strategy/initiatives/extract` — upload document, returns extraction result with drafts
- `GET /strategy/extractions/{id}/drafts` — list drafts from extraction
- `PATCH /strategy/extractions/{id}/drafts/{draft_id}` — update draft during review
- `POST /strategy/extractions/{id}/drafts/{draft_id}/promote` — promote to active
- `DELETE /strategy/extractions/{id}/drafts/{draft_id}` — discard

**Timeline:**
- `GET /strategy/timeline` — optimized query returning all active/at_risk initiatives with milestones, dependencies, and resource conflicts in a single response

**Approvals:**
- `GET /strategy/approvals` — pending for current user
- `POST /strategy/approvals/{id}/approve` — approve
- `POST /strategy/approvals/{id}/reject` — reject with optional reason

### Notification Routes (`/notifications`)

- `GET /notifications` — list for current user, filterable by category, read/unread. Limit 50.
- `GET /notifications/count` — unread count by category
- `PATCH /notifications/{id}` — mark read or dismissed
- `POST /notifications/mark-all-read` — bulk, optional category filter

---

## Timeline / Roadmap UI

### Page: `/strategy`

**Layout:**
- Top summary bar: initiative counts by status, upcoming milestones this week, unresolved notification count
- Toolbar: grouping toggle (by priority | by owner), time scale toggle (months | quarters), "New Initiative" button
- Main area: Gantt-style timeline chart (library-based, refactor to custom SVG if limitations arise)
- Right panel: initiative detail drill-down (opens on click)

### Timeline Chart

- Initiatives as horizontal bars colored by health: green (>=0.7), yellow (0.4-0.7), red (<0.4), gray (on_hold/cancelled)
- Milestones as diamond markers at target date
- Bar length from `start_date` to `target_end_date`
- Completion shown as filled portion matching `completion_pct`
- Labels: initiative title, owner avatar

### Overlays (Toggleable)

- Dependency arrows: subtle gray by default, highlighted on hover/select. Solid = `blocks`, dashed = `informs`, dotted = `shares_resources`
- Resource conflict icons: warning badge on overlapping over-allocated bars
- Today line: vertical marker for current date

### Interaction

- Hover bar → tooltip: title, owner, health, completion %, next milestone
- Click bar → detail panel opens, dependency chain highlighted, rest dimmed
- Click milestone diamond → milestone detail inline in panel

### Detail Panel

- Title, status badge, health indicator
- Owner + assignees
- Milestone list with inline editing (status, date, assignee)
- Evidence links as clickable chips (KPIs, action items, artifacts)
- Resource assignments with allocation %
- Recent activity timeline
- Quick actions: update status, add milestone, add evidence, request update

### VCP / Document Entry

- "New Initiative" offers: "Create manually" or "Extract from document"
- "Extract from document" → upload + guided review flow
- Drafts appear as dashed/ghosted bars on the timeline

### Responsive

- Narrow screens: detail panel becomes full-screen overlay
- Timeline horizontally scrollable with pinch/zoom on touch
- Summary bar collapses to icon-only on small viewports

---

## Implementation Slices (Vertical)

| Slice | Scope |
|-------|-------|
| **1** | Initiative + milestone domain model, schema, CRUD API, `/strategy` page with basic timeline |
| **2** | Document extraction — upload, extract via LLM, guided review, promote drafts |
| **3** | Dependencies + resource assignments + conflict detection, timeline overlays |
| **4** | Notifications schema + model + widget, cadence engine |
| **5** | Copilot integration — strategy tools, passive indicators, end-of-task nudge |
| **6** | Briefing integration + evidence linking |

Each slice delivers a usable, demoable feature. Slices build on each other but are independently valuable.
