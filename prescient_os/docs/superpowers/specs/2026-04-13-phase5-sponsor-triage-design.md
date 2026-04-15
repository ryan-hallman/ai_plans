# Phase 5: Sponsor Access & Triage — Design Spec

**Goal:** Add the sponsor side of Prescient OS — a read-only portal for operating partners/board members with a full triage loop: sponsors ask questions, the system auto-answers from visible data, escalates gated items to the operator, and the operator drafts responses via brainstorm mode.

**Architecture:** New `prescient/triage/` bounded context for the question lifecycle and attention queue. A cross-cutting visibility enforcement layer gates all existing API routes based on membership scope and visibility rules. Sponsor portal is a separate Next.js route group (`/(sponsor)/`). Existing copilot chat infrastructure is reused for operator brainstorm sessions.

**Tech Stack:** Same as Phase 4 — Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Next.js 15, React 19, Tailwind, shadcn/ui, Anthropic SDK.

---

## 1. Visibility Enforcement Layer

### AccessContext

A `VisibilityFilter` FastAPI dependency resolves the current user's access at request time:

```
CurrentUser(user_id, fund_id)
    → look up Membership (role, scope, primary_org_id)
    → if sponsor role: look up PortfolioLink + VisibilityRules
    → return AccessContext
```

**AccessContext fields:**
- `user_id` — the authenticated user
- `org_id` — the user's primary organization
- `role` — UserRole enum (operator, operating_partner, pe_analyst, board_member, etc.)
- `scope` — MembershipScope (full, read_only, kpi_only)
- `is_sponsor` — derived: True if role in {operating_partner, pe_analyst, board_member}
- `visible_categories` — set of DataCategory with AUTOMATIC gate (e.g., `{kpi_metrics}`)
- `gated_categories` — set of DataCategory with OPERATOR_APPROVED gate
- `blocked_categories` — set of DataCategory with BLOCKED gate
- `company_org_ids` — list of company org IDs this user can see (from active portfolio links)

### Enforcement Pattern

Existing routes add `access: AccessContext = Depends(get_access_context)` and use it to:

1. **Filter queries** — sponsors only see data from `company_org_ids`, and only for `visible_categories` + explicitly shared items
2. **Block access** — routes that are operator-only raise HTTP 403 if `access.is_sponsor`
3. **Scope results** — briefing items, KPIs, artifacts are filtered by what the caller is allowed to see

### Demo Auth Extension

`DemoAuthMiddleware` resolves Michael Torres → operating_partner → Summit Partners → PortfolioLink to Peloton → Tier B visibility rules → AccessContext with `visible_categories={kpi_metrics}`, all other categories gated.

---

## 2. Triage Bounded Context — Domain Model

New bounded context at `prescient/triage/`.

### TriageQuestion

Represents a question asked by a person (sponsor, board member, operating partner).

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `asker_user_id` | str | Who asked |
| `asker_org_id` | str | Asker's organization |
| `target_company_org_id` | str | Which company the question is about |
| `question_text` | str | The raw question |
| `categories_detected` | list[DataCategory] | Categories the question touches (LLM-classified) |
| `status` | TriageQuestionStatus | Lifecycle state |
| `partial_answer` | str or None | Auto-generated answer from visible data |
| `partial_citations` | list[dict] or None | Citations for partial answer |
| `escalation_reason` | str or None | Why it was escalated |
| `final_answer` | str or None | Operator-approved response |
| `final_citations` | list[dict] or None | Citations for final answer |
| `brainstorm_conversation_id` | UUID or None | Link to copilot session used for drafting |
| `priority` | TriagePriority | Computed from asker's role |
| `created_at` | datetime | When asked |
| `answered_at` | datetime or None | When fully answered |

**TriageQuestionStatus lifecycle:**
`submitted → classifying → partially_answered → escalated → answered → closed`

- `submitted` — just received
- `classifying` — LLM is determining categories
- `partially_answered` — visible data answered, gated parts escalated
- `escalated` — routed to operator attention
- `answered` — operator approved final response (or fully auto-answered)
- `closed` — terminal state

### TriageItem

General-purpose attention queue entry. Not just for sponsor questions — anything needing operator attention.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `organization_id` | str | Which company org this is about |
| `item_type` | TriageItemType | What kind of item |
| `source_id` | str | FK to originating record |
| `title` | str | Display title |
| `summary` | str | One-line description |
| `priority` | TriagePriority | critical, high, medium, low |
| `priority_signals` | list[str] | Reasons for priority level |
| `status` | TriageItemStatus | pending, in_progress, resolved, dismissed |
| `assigned_to_user_id` | str or None | Optional assignment |
| `created_at` | datetime | When created |
| `resolved_at` | datetime or None | When resolved |

**TriageItemType values:**
- `sponsor_question` — from TriageQuestion
- `kpi_anomaly` — from KPI anomaly detection
- `monitoring_finding` — from monitor findings
- `action_review` — action item needing review
- `system_alert` — system-generated

### Priority Computation

Priority is derived from signals, not set manually:

| Source | Default Priority |
|--------|-----------------|
| Board member question | high |
| Operating partner question | high |
| PE analyst question | medium |
| KPI anomaly (critical severity) | high |
| KPI anomaly (important severity) | medium |
| Monitoring finding (critical) | high |
| Monitoring finding (important) | medium |
| Action item past due (> 7 days) | high |
| Action item past due (1-7 days) | medium |
| Action item pending review | low |
| System alert | low |

### SharedItem

Tracks what operators have explicitly shared with sponsors.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `artifact_id` | str | What was shared |
| `sponsor_org_id` | str | Who it was shared with |
| `shared_by_user_id` | str | Who approved sharing |
| `shared_at` | datetime | When shared |
| `triage_question_id` | UUID or None | If shared as part of answering a question |

---

## 3. Auto-Answer & Escalation Flow

When a sponsor submits a question, the system runs a three-step orchestration:

### Step 1: Category Classification

LLM call with the question text and the list of DataCategory values. Returns which categories the question touches. Small, fast prompt.

**Input:** question text + category definitions
**Output:** list of DataCategory (e.g., `[kpi_metrics, narrative, decisions]`)

### Step 2: Visibility Check

Compare detected categories against the sponsor's visibility rules:

- **All AUTOMATIC** → full auto-answer, no escalation
- **Mix of AUTOMATIC and OPERATOR_APPROVED** → partial answer from visible data + escalate gated parts
- **All BLOCKED** → escalate immediately, no partial answer
- **All OPERATOR_APPROVED** → escalate, attempt partial answer from any AUTOMATIC data tangentially relevant

### Step 3: Answer Generation

**Auto-answer path:** Run the copilot planner scoped to only artifacts/data the sponsor can see (AUTOMATIC categories). Generate answer with citations. If well-grounded, deliver as `answered`. If partial, deliver as `partial_answer` and proceed to escalation.

**Escalation path:** Create escalation with summary for operator:
- What the sponsor asked
- What categories are gated
- What was already answered (partial answer)
- What the sponsor likely wants to know (inferred from question)

Create/update TriageItem with priority based on asker's role.

### Step 4: Operator Brainstorm (on escalation)

Operator opens brainstorm from triage queue. This launches a copilot chat session with a specialized system prompt:

```
You are helping draft a response to a sponsor question.

SPONSOR QUESTION: {question_text}
PARTIAL ANSWER ALREADY SHARED: {partial_answer}
SPONSOR CAN SEE: {visible_categories}
INTERNAL CONTEXT: {full internal artifacts, including gated data}
VISIBILITY CONSTRAINTS: {what can and cannot be shared}

Help the operator draft a response that:
1. Answers the question as fully as possible
2. Only uses information that can be shared
3. Is informed by internal context without leaking it
4. Suggests what artifacts to attach/share
```

The conversation is saved and linked to the TriageQuestion via `brainstorm_conversation_id`.

### Step 5: Response Delivery

Operator approves the drafted response. TriageQuestion status → `answered`. TriageItem status → `resolved`. Sponsor sees the complete answer with citations in their Q&A tab.

---

## 4. Triage Queue (Operator View)

New page at `/triage` in the operator's `/(main)/` layout.

### Layout

**Left panel — Queue list:**
- Ranked list of TriageItems sorted by priority (critical > high > medium > low), then recency
- Each item shows:
  - Priority badge (color-coded)
  - Item type icon (question bubble, chart, shield, checklist)
  - Title and one-line summary
  - Source label ("Michael Torres, Summit Partners" or "KPI Anomaly Detection")
  - Time since created (relative)
  - Status chip (pending, in progress, resolved)
- Filters: by type, by priority, by status
- Default view: pending + in_progress items

**Right panel — Detail view (varies by type):**

**Sponsor question:**
- Question text
- Asker info (name, role, org)
- Partial answer with citations (if any)
- Escalation reason
- "Open Brainstorm" button → launches copilot chat with triage context
- After drafting: "Approve & Send Response" button
- Option to attach/share artifacts with the response

**KPI anomaly:**
- Anomaly detail, severity, metric context
- Link to KPI page
- "Acknowledge" or "Create Action Item" buttons

**Monitoring finding:**
- Finding detail, severity, source
- "Mark Reviewed" or "Escalate" buttons

**Action item review:**
- Action item detail
- "Approve", "Reject", or "Defer" buttons

### Morning Brief Integration

High-priority triage items (priority = high or critical) generate BriefingItems with `action_type: DECIDE` and `source: ItemSource.TRIAGE`. The briefing item links to the triage queue filtered to that item.

### Relationship to Actions Page

The triage queue is the "what needs my attention now" view. The existing Actions page (`/actions`) remains for tracking action item lifecycle (assigned, completed, deferred). They are complementary, not overlapping:
- Triage queue: attention management, ranked by urgency
- Actions page: commitment tracking, filtered by status

---

## 5. Sponsor Portal

### Routing

After login, the app checks the user's role:
- `operator`, `functional_leader`, `admin` → redirect to `/(main)/`
- `operating_partner`, `pe_analyst`, `board_member` → redirect to `/(sponsor)/`

The `/(sponsor)/` route group has its own layout with sponsor-appropriate navigation.

### Portfolio Dashboard (`/(sponsor)/`)

- Header: sponsor org name + user name
- Card grid of linked companies (from active PortfolioLinks)
- Each company card shows:
  - Company name
  - Key KPI sparklines (from AUTOMATIC visibility data — subscriber count, churn, revenue)
  - Count of pending/recent Q&A items
  - "Last updated" timestamp
- Click card → company detail

### Company Detail (`/(sponsor)/companies/[slug]`)

Three tabs:

**KPI Dashboard tab:**
- Charts for all KPIs visible under AUTOMATIC rules
- Same visualizations as operator KPI page, but read-only (no edit, no configure)
- Time range selector (quarterly)

**Shared Artifacts tab:**
- List of artifacts the operator has shared (from SharedItem records)
- Board narratives, intelligence signals, decision records that were explicitly shared
- Each shows: title, type badge, date, confidence label
- Click to view artifact detail (read-only)

**Q&A tab:**
- Text input to ask a new question
- History of past questions, newest first
- Each question shows:
  - Question text
  - Status badge (answered, partially answered, pending)
  - Timestamp
- Click to expand:
  - Partial answer (if available) with citations and note: "We've shared what's available. The team has been flagged for the rest."
  - Final answer (when available) with citations
  - Attached/shared artifacts (if any)

### Pre-Call Briefing (`/(sponsor)/companies/[slug]/briefing`)

Synthesized view for the sponsor's next interaction with the company. Pulls from:
- Recent KPI movements (AUTOMATIC data)
- Recently shared artifacts
- Answered triage questions
- Open action items visible to sponsor

Rendered as a focused summary: "Here's what's changed since your last interaction, open items to discuss, and suggested questions to ask." Generated via LLM call with visible data as context.

---

## 6. Operator Approval Flow for Gated Content

### Share Action

On existing operator pages (artifact detail, board prep, action items), add a "Share with Sponsor" button. When clicked:
- Modal shows which sponsors have portfolio links to this company
- Operator selects sponsor org(s)
- Creates SharedItem record
- Artifact appears in sponsor's "Shared Artifacts" tab

### Board Prep Integration

After operator approves a board deck, prompt: "Share this deck with Summit Partners?" Options:
- Share entire deck (creates SharedItem for the board_narrative artifact)
- Share individual sections (future — not in Phase 5 scope)

### Triage Response Sharing

When responding to a sponsor question via brainstorm, the operator can attach artifacts. Those artifacts are auto-shared (SharedItem created) as part of the response.

---

## 7. API Endpoints (New)

### Triage Context

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/triage/questions` | Sponsor submits a question | sponsor |
| GET | `/triage/questions` | List sponsor's own questions | sponsor |
| GET | `/triage/questions/{id}` | Get question detail + answers | sponsor or operator |
| POST | `/triage/questions/{id}/respond` | Operator submits final answer | operator |
| GET | `/triage/items` | List triage queue items | operator |
| GET | `/triage/items/{id}` | Get triage item detail | operator |
| PATCH | `/triage/items/{id}` | Update status (resolve, dismiss, assign) | operator |
| POST | `/triage/items/{id}/brainstorm` | Start brainstorm session for item | operator |

### Sharing

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/sharing/items` | Share an artifact with a sponsor org | operator |
| GET | `/sharing/items` | List shared items (filtered by sponsor org) | sponsor or operator |
| DELETE | `/sharing/items/{id}` | Revoke sharing | operator |

### Sponsor Views

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/sponsor/portfolio` | Portfolio dashboard data | sponsor |
| GET | `/sponsor/companies/{slug}/kpis` | KPIs visible to sponsor | sponsor |
| GET | `/sponsor/companies/{slug}/briefing` | Pre-call briefing | sponsor |

### Visibility

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/visibility/rules` | Get visibility rules for a portfolio link | operator or sponsor |

---

## 8. Database Tables (New)

All in the `triage` schema:

**`triage.triage_questions`**
- `id` UUID PK
- `asker_user_id` VARCHAR FK → organizations.users
- `asker_org_id` VARCHAR FK → organizations.organizations
- `target_company_org_id` VARCHAR FK → organizations.organizations
- `question_text` TEXT
- `categories_detected` JSONB (list of category strings)
- `status` VARCHAR
- `partial_answer` TEXT nullable
- `partial_citations` JSONB nullable
- `escalation_reason` TEXT nullable
- `final_answer` TEXT nullable
- `final_citations` JSONB nullable
- `brainstorm_conversation_id` UUID nullable
- `priority` VARCHAR
- `created_at` TIMESTAMPTZ
- `answered_at` TIMESTAMPTZ nullable

**`triage.triage_items`**
- `id` UUID PK
- `organization_id` VARCHAR FK → organizations.organizations
- `item_type` VARCHAR
- `source_id` VARCHAR
- `title` VARCHAR
- `summary` TEXT
- `priority` VARCHAR
- `priority_signals` JSONB (list of strings)
- `status` VARCHAR default 'pending'
- `assigned_to_user_id` VARCHAR nullable FK → organizations.users
- `created_at` TIMESTAMPTZ
- `resolved_at` TIMESTAMPTZ nullable

**`triage.shared_items`**
- `id` UUID PK
- `artifact_id` VARCHAR FK → artifacts.artifacts
- `sponsor_org_id` VARCHAR FK → organizations.organizations
- `shared_by_user_id` VARCHAR FK → organizations.users
- `shared_at` TIMESTAMPTZ
- `triage_question_id` UUID nullable FK → triage.triage_questions

---

## 9. Seed Data Additions

Extend the existing seed pipeline to populate Phase 5 demo data:

- **2-3 pre-existing TriageQuestions** from Michael Torres with answered status, showing Q&A history
- **5-8 TriageItems** across types: 2 sponsor questions, 2 KPI anomalies (from existing seed), 2 monitoring findings, 1 action review, 1 system alert
- **3-4 SharedItems** — operator has already shared a board deck and two intelligence signals with Summit Partners
- **1 pending TriageQuestion** — Michael's "What's driving subscriber churn?" question, status `escalated`, with partial answer already generated. This is the live demo item.

---

## 10. Demo Script

### Act 1: Sarah Chen, CEO of Peloton (~9 minutes)
(Existing Phase 4 demo — Morning Brief → Copilot → Citations → Actions → Board Prep)

At the end of Act 1, Sarah approves the board deck and shares it with Summit Partners.

### Act 2: Michael Torres, Operating Partner at Summit Partners (~5 minutes)

1. **Login as Michael** → portfolio dashboard shows Peloton card with KPI sparklines
2. **Click into Peloton** → KPI dashboard tab shows subscriber metrics, churn, revenue (AUTOMATIC data)
3. **Check Shared Artifacts tab** → sees the board deck Sarah just shared, plus previously shared intelligence signals
4. **Go to Q&A tab** → sees history of past questions and answers
5. **Ask a new question:** "What's driving the subscriber churn increase, and what's the team's plan to address it?"
6. **Immediate partial answer:** "Churn increased from 1.1% to 1.4% over the last two quarters. Connected Fitness subscribers declined by 8% YoY." (from AUTOMATIC KPI data)
7. **Status note:** "We've shared what's available from the data. The strategic details have been flagged for the Peloton team."

### Cut back to Sarah:

8. **Sarah opens triage queue** → Michael's question appears as high-priority item (operating partner)
9. **Clicks the item** → sees question, partial answer already shared, escalation reason ("touches narrative + decisions categories")
10. **Clicks "Open Brainstorm"** → copilot session pre-loaded with full context
11. **Drafts response** with AI assistance, attaches a strategic artifact about the retention initiative
12. **Clicks "Approve & Send"** → response delivered to Michael

### Back to Michael:

13. **Michael's Q&A tab updates** → full answer with citations, plus the shared strategic artifact

**Total demo: ~14 minutes.** Shows the complete two-sided loop: operator prepares, sponsor consumes and asks, system mediates, operator responds with AI assistance.
