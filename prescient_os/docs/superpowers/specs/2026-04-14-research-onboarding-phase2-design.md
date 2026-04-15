# Research Engine & Smart Onboarding — Phase 2 Design Spec

> Extends the Phase 1 spec (`2026-04-13-research-engine-and-smart-onboarding-design.md`). Phase 1 delivered the research pipeline engine, domain entities, source plugins, copilot tools (`research_company`, `watch_company`, `save_artifact`), interview system prompt, and `UserContext` entity. This phase covers async infrastructure, remaining tools, the onboarding interview UI, and the standalone research page.

## Overview

Five workstreams, built infrastructure-first so every feature gets async research, notifications, and live updates from day one:

1. **Async research infrastructure** — Celery-based parallel source fetching, research completion events, notifications, live page updates
2. **`research_market` and `research_person` copilot tools** — quick inline results + offer to go deeper
3. **Onboarding interview UI** — concurrent chat interview + background research, dual path with existing wizard
4. **Standalone `/research` page** — split-view with user-controlled depth and source toggles
5. **Cross-cutting data model changes** — "competitors" → "watched companies", user intent as first-class concept

---

## 1. Async Research Infrastructure

### 1.1 Deep Research Celery Tasks

New module: `prescient.research.infrastructure.tasks.deep_research`

**`run_deep_research(task_id, subject_data, mode, tenant_id, created_by, enabled_sources)`**

- Follows existing `@shared_task` + `asyncio.run()` + `async with SessionLocal()` pattern (see `monitoring/infrastructure/tasks/sweep.py`)
- Fans out source fetches as parallel Celery subtasks via `chord` or `group`
- Each source runs as its own subtask with per-source timeout
- Coordinator waits for all subtasks or hits a mode-based timeout ceiling:
  - `QUICK`: 25 seconds
  - `STANDARD`: 60 seconds
  - `DEEP`: 5 minutes
- After fetch phase, runs `ExtractAndSynthesize` LLM call, persists results via `ResearchTaskRepository`
- Soft/hard time limits: `soft_time_limit=360, time_limit=420`

**`upgrade_to_deep(quick_task_id)`**

- Loads the completed quick `ResearchTask`, clones its subject
- Dispatches `run_deep_research` with `mode=DEEP` and all sources enabled
- Returns new task ID for tracking

**Registration:**

- Add `"prescient.research.infrastructure.tasks"` to `celery_app.py` autodiscover list
- No beat schedule entry needed — these are on-demand tasks, not periodic

### 1.2 Research Mode Expansion

`ResearchMode` enum expands:

```python
class ResearchMode(StrEnum):
    QUICK = "quick"
    STANDARD = "standard"
    DEEP = "deep"
```

Each `ResearchSource` gains a `tier` attribute (replaces `quick_pass: bool`):

```python
class SourceTier(StrEnum):
    QUICK = "quick"        # website_scraper, search_engine, dns_lookup
    STANDARD = "standard"  # news_aggregator
    DEEP = "deep"          # edgar
```

`SourceRegistry.sources_for()` filters: a source is included if its tier ≤ the requested mode.

### 1.3 User-Controlled Source Selection

`RunResearchCommand` gains:

```python
enabled_sources: list[str] | None = None  # e.g. ["website_scraper", "edgar"]
```

When present, `SourceRegistry.sources_for()` additionally filters to only named sources (intersection of tier-eligible and user-selected). When `None`, falls back to mode-based filtering.

**Source discovery endpoint:**

```
GET /research/sources
```

Returns:

```json
[
  {"name": "website_scraper", "label": "Website", "tier": "quick", "description": "Scrapes company website for about pages and key content"},
  {"name": "search_engine", "label": "Web Search", "tier": "quick", "description": "DuckDuckGo search results"},
  {"name": "dns_lookup", "label": "DNS/Domain", "tier": "quick", "description": "Domain infrastructure signals"},
  {"name": "news_aggregator", "label": "News", "tier": "standard", "description": "Recent news coverage via Google News"},
  {"name": "edgar", "label": "SEC Filings", "tier": "deep", "description": "SEC EDGAR financial filings"}
]
```

Derived from the source registry at runtime — as new sources are added, they automatically appear.

### 1.4 Research Completion Events

When `run_deep_research` completes, the worker emits a domain event:

```python
DomainEvent(
    aggregate_id=task_id,
    aggregate_type="research.task",
    event_type="research.task.completed",
    payload={
        "task_id": task_id,
        "subject_type": subject.subject_type.value,
        "subject_name": subject.name,  # for notification title
        "status": task.status.value,
        "artifact_id": task.result_artifact_id,
        "tenant_id": tenant_id,
        "created_by": created_by,
    },
)
```

Persisted via outbox, picked up by `drain_outbox` (runs every 30s).

### 1.5 Research Completion Notification Subscriber

New event subscriber: `prescient.events.subscribers.research_to_notification`

Registered in the subscriber registry:

```python
"research.task.completed": [notify_research_complete],
```

The handler:

```python
@shared_task(name="prescient.events.subscribers.research_to_notification.notify_research_complete",
             max_retries=3, default_retry_delay=10)
def notify_research_complete(payload: dict) -> dict:
    # Publishes notification via PublishNotification use case
    # title: "Research complete: {subject_name}"
    # body: "Deep research on {subject_name} has finished. View the full profile."
    # action_url: "/companies/{artifact_id}" or "/research?task={task_id}"
    # category: "research"
    # priority: "normal"
```

Frontend notification bell picks this up automatically — no frontend changes needed for basic notifications.

### 1.6 Live Page Updates via SSE

The copilot SSE stream gains a new event type:

```
event: research_complete
data: {"task_id": "...", "subject_name": "Acme Corp", "artifact_id": "..."}
```

Frontend handling: when the copilot panel or `/research` page receives this event, it invalidates relevant React Query cache keys, triggering re-fetch of company data or research results.

### 1.7 Copilot Nudge

When a user opens a copilot conversation, the backend checks for recently completed research tasks (completed since last copilot interaction, for this user/tenant). If found, the system prompt is augmented:

```
Note: Research on {subject_name} completed recently. If relevant, mention this to the user.
```

The LLM naturally incorporates this — "By the way, I finished that deep dive on Acme Corp. Want me to walk you through the findings?"

---

## 2. Research Tools: `research_market` & `research_person`

### 2.1 `research_market`

**File:** `apps/api/src/prescient/intelligence/infrastructure/tools/research_market.py`

**Arguments:**

```python
class ResearchMarketArguments(BaseModel):
    seed_company_name: str  # Required — anchor company for market discovery
    industry: str | None = None
    sources: list[str] | None = None  # Optional source override
```

**Behavior:**

- Builds a `MarketSubject(seed_company_name=..., industry=...)`
- Runs `RunResearchCommand` with `subject_type="market"`, `mode="quick"`
- LLM synthesis uses the market-specific system prompt (already in `ExtractAndSynthesize`) producing blocks: Market Overview, Key Competitors, Market Trends, Competitive Dynamics
- Returns `ToolExecution.ok()` with summary and structured records
- After returning results, the copilot can offer: "Want me to do a deeper analysis? I can pull in news coverage and more detailed research."
- If user accepts, calls `upgrade_to_deep` helper

**Description for LLM:**

```
"Analyze the competitive landscape around a company. Discovers competitors, market trends, and
competitive dynamics. Use when the user asks about a market, competitive landscape, or wants
to understand who the key players are in an industry."
```

### 2.2 `research_person`

**File:** `apps/api/src/prescient/intelligence/infrastructure/tools/research_person.py`

**Arguments:**

```python
class ResearchPersonArguments(BaseModel):
    name: str  # Required
    company_name: str | None = None
    role: str | None = None
    email: str | None = None
    sources: list[str] | None = None
```

**Behavior:**

- Builds a `PersonSubject(name=..., company_name=..., role=..., email=...)`
- Runs quick pass (website scraper + search engine)
- LLM synthesis uses person-specific system prompt producing blocks: Overview, Career History, Areas of Expertise, Public Presence
- Same quick→deep offer pattern

**Description for LLM:**

```
"Research a person's professional background. Finds career history, expertise, and public
presence. Use when the user asks about a specific person, wants background on an executive,
or needs to prepare for a meeting with someone."
```

### 2.3 Quick→Deep Upgrade Helper

Shared utility (not a tool):

**File:** `apps/api/src/prescient/research/application/use_cases/upgrade_research.py`

```python
@dataclass(frozen=True)
class UpgradeResearchCommand:
    quick_task_id: str

@dataclass(frozen=True)
class UpgradeResearchResult:
    deep_task_id: str
    message: str  # "Deep research started — you'll get a notification when it's ready."

class UpgradeResearch:
    async def execute(self, command: UpgradeResearchCommand) -> UpgradeResearchResult:
        # Load quick task, clone subject, dispatch upgrade_to_deep.delay()
```

Any tool can call this when the user asks for a deeper dive. Keeps the upgrade logic in one place.

### 2.4 Registration

Both tools registered in `build_tool_registry()` in `tools/__init__.py`, same wiring pattern as `research_company`. Factory functions for dependency injection where needed.

---

## 3. Onboarding Interview UI

### 3.1 Dual Path Entry

The onboarding landing page (`/onboarding/page.tsx`) presents two options:

- **"Set up manually"** → existing 3-step wizard (company → watched companies → interests). Step 2 renamed from "Competitors" to "Companies to Watch."
- **"Let us learn about you"** → interview path (new)

### 3.2 Concurrent Interview + Research Flow

```
┌─────────────────────────────────────────────────────────┐
│  1. User submits: company name, website, role           │
│                                                          │
│  2. Two things happen simultaneously:                    │
│     ┌──────────────────┐  ┌───────────────────────────┐ │
│     │ Quick research   │  │ Chat interview starts     │ │
│     │ kicks off async  │  │ with intent-focused Qs    │ │
│     │ (20-30s)         │  │                           │ │
│     └──────┬───────────┘  └───────────────────────────┘ │
│            │                        │                    │
│  3. As findings arrive, injected    │                    │
│     into conversation context ──────┘                    │
│     Later questions become research-informed             │
│                                                          │
│  4. Interview wraps (~2-3 min)                           │
│     → Extract UserContext                                │
│     → Dispatch deep research async                       │
│     → Redirect to main app                               │
└─────────────────────────────────────────────────────────┘
```

### 3.3 Interview System Prompt Updates

The existing `build_interview_system_prompt()` in `run_interview.py` is updated:

- Initial prompt has a placeholder for research context: `"Research is in progress — ask general questions first."`
- As quick research findings arrive, the system prompt is re-built with actual research data
- The copilot planner re-reads context each turn, so mid-conversation updates take effect naturally

**Question priority:**

1. Intent-focused (asked immediately, no research needed):
   - "What's your role day-to-day?"
   - "What would make this platform invaluable to you?"
   - "What keeps you up at night in your work?"
2. Research-informed (asked once findings arrive):
   - "I see {company} is in {industry} — which companies in this space do you watch?"
   - "Based on what I found, it looks like {insight}. Is that accurate?"
3. Goal-oriented (wrapping up):
   - "What's the first thing you'd want to check when you open this platform each morning?"

### 3.4 Frontend Components

**New page:** `/onboarding/interview/page.tsx`

Orchestrates the full flow:

1. **Initial form** — company name, website URL, your name, your role. Minimal, one screen.
2. **Chat interface** — full-page chat (not side panel). Reuses `streamChat` helper and SSE event parsing from copilot infrastructure. Simple message bubbles — no tool call UI, the interview prompt handles the flow.
3. **Subtle progress indicator** — small indicator showing research status ("Researching your company...") that disappears when quick research completes. Not a blocking loading screen.
4. **Completion transition** — final message summarizes what was learned, brief animation, redirect to main app.

**Backend endpoint changes:**

- Copilot stream endpoint gains `mode: "interview"` body field → swaps in interview system prompt
- New `POST /onboarding/interview/complete`:
  - Receives `conversation_id`
  - Triggers LLM extraction of `UserContext` from conversation (extraction prompt already exists)
  - Persists `UserContext`
  - Dispatches `run_deep_research.delay()` for the deep pass
  - Marks onboarding as complete
  - Returns `{status: "complete", redirect_url: "/brief"}`

### 3.5 Research Context Injection

How findings arrive mid-interview:

- The interview page polls `GET /research/{task_id}` every 3 seconds during the conversation
- The response includes `status`, `source_results` (which sources completed), and `synthesized_result` (if synthesis has run)
- When the poll detects new source completions, the frontend includes a `research_context` field in the next chat message body: `{research_context: {sources_completed: [...], summary: "..."}}`
- The backend's interview handler checks for this field and rebuilds the system prompt with the new research data before the next LLM turn
- If quick research completes fully before the user sends their next message, the frontend sends a synthetic "context update" message (not displayed in chat) to trigger prompt refresh
- This avoids any complex server-push mechanism — simple polling + piggybacking on the existing chat stream

---

## 4. Standalone `/research` Page

### 4.1 Layout

Split view:

```
┌────────────────────────────────────────────────────────┐
│  Research Controls (collapsible)                        │
│  [Quick ▾] [Standard] [Deep]                           │
│  ☑ Website  ☑ Web Search  ☑ DNS  ☐ News  ☐ SEC       │
├───────────────────────────────┬─────────────────────────┤
│                               │                         │
│  Research Results (60%)       │  Chat (40%)             │
│                               │                         │
│  ┌─────────────────────────┐  │  Research-focused       │
│  │ Company: Acme Corp      │  │  copilot with           │
│  │ Manufacturing            │  │  research tools         │
│  │ ───────────────          │  │  pre-loaded             │
│  │ Overview: ...            │  │                         │
│  │ Competitive Position:... │  │  Suggested prompts:     │
│  │                          │  │  "Research [company]"   │
│  │ [Watch] [Save] [Deep ↻] │  │  "Analyze market for.." │
│  └─────────────────────────┘  │  "Who is [person]?"     │
│                               │                         │
│  ┌─────────────────────────┐  │                         │
│  │ Market: Manufacturing   │  │                         │
│  │ [In Progress ████░░ ]   │  │                         │
│  └─────────────────────────┘  │                         │
│                               │                         │
├───────────────────────────────┴─────────────────────────┤
│  Empty state: "Ask me to research a company, person,    │
│  or market — or try one of the suggestions above."      │
└────────────────────────────────────────────────────────┘
```

### 4.2 Research Controls

Collapsible settings bar at the top:

- **Depth preset buttons:** Quick / Standard / Deep — selecting one pre-populates source toggles
- **Source toggles:** Checkboxes for each source, fetched from `GET /research/sources`. User can override after selecting a preset.
- Controls state is passed to the copilot as page context, so research tools use the user's preferred settings

### 4.3 Results Panel

- Renders `ResearchResultCard` components from research task data
- Cards display the structured blocks from `synthesized_result` (already standardized as `{block_id, heading, body, order}`)
- **Card actions:**
  - "Watch" → triggers `watch_company` tool via the chat
  - "Save" → triggers `save_artifact` tool via the chat
  - "Go deeper" → triggers `upgrade_to_deep` via the chat
- **Progress cards:** For in-progress async research, shows a progress indicator with source completion status
- **Live updates:** SSE `research_complete` event triggers React Query cache invalidation → card re-renders with full results

### 4.4 Chat Panel

- Reuses copilot streaming infrastructure
- `pageContext: { scope: "research", depth: "standard", enabled_sources: [...] }` sent with each message
- Backend uses this context to configure research tools with user's preferred settings
- System prompt emphasizes research capabilities
- Suggested prompts shown in empty state

### 4.5 Frontend Components

- **New page:** `apps/web/src/app/(main)/research/page.tsx` — split layout container
- **New components:**
  - `ResearchControls` — depth preset + source toggles
  - `ResearchResultsPanel` — list of result cards
  - `ResearchResultCard` — renders a single research result with blocks and actions
  - `ResearchProgressCard` — in-progress indicator with per-source status
- **Reused:** `streamChat`, SSE event parsing, `CopilotMessage` types, message bubble components

---

## 5. Cross-Cutting Data Model Changes

### 5.1 "Competitors" → "Watched Companies"

**Backend:**

- `UserContext.competitors_mentioned` field renamed to `watched_companies`
- Type changes from `list[dict]` to `list[WatchedCompanyEntry]`:

```python
@dataclass(frozen=True)
class WatchedCompanyEntry:
    name: str
    relationship: str  # "competitor" | "partner" | "customer" | "prospect" | "other"
    context: str | None = None  # Why they mentioned this company
```

- Stored as JSONB (same column, new shape)
- Migration adds column alias or renames column
- Interview extraction prompt updated to capture `relationship` field

**Frontend:**

- Onboarding wizard step 2: "Competitors" → "Companies to Watch"
- Competitor form fields updated to include a "relationship" dropdown
- `watch_company` tool description broadened: "consultants, investors, and operators" framing

### 5.2 User Intent as First-Class

**New field on `UserContext`:**

```python
platform_goals: tuple[str, ...] = ()
```

Structured, machine-readable goals. Examples:

- `"track_competitor_moves"`
- `"prepare_board_meetings"`
- `"monitor_market_trends"`
- `"source_deals"`
- `"manage_portfolio_risk"`
- `"stay_informed_industry"`

**Interview extraction prompt** updated to extract these from conversation.

**Downstream consumers** (documented for future implementation, not built in this phase):

- Brief page curation: prioritize content aligned with stated goals
- Research suggestions: copilot proactively suggests research relevant to goals
- Monitoring config: auto-configure sweep priorities based on user goals
- Notification filtering: higher priority for goal-aligned notifications

The `platform_goals` field is persisted now; downstream consumers are wired up incrementally as those features are built.

### 5.3 Database Migration

New migration extending `20260413_research_engine`:

- Rename `user_contexts.competitors_mentioned` → `watched_companies` (or add new column + backfill if safer)
- Add `user_contexts.platform_goals` column (JSONB, default `[]`)
- No changes to `research_tasks` table

---

## Dependency Graph

```
Layer 1: Infrastructure
  ├── 1a. ResearchMode expansion + source tiers + enabled_sources
  ├── 1b. Deep research Celery tasks (run_deep_research, upgrade_to_deep)
  ├── 1c. Research completion domain event + notification subscriber
  └── 1d. SSE research_complete event + copilot nudge

Layer 2: Research Tools (depends on 1a, 1b)
  ├── 2a. research_market tool
  ├── 2b. research_person tool
  └── 2c. UpgradeResearch use case (quick→deep helper)

Layer 3: Data Model Changes (independent)
  ├── 3a. competitors_mentioned → watched_companies rename
  ├── 3b. platform_goals field
  └── 3c. Migration

Layer 4: Onboarding Interview UI (depends on 1b, 1c, 3a, 3b)
  ├── 4a. Backend: interview mode on copilot stream + complete endpoint
  ├── 4b. Frontend: interview page with concurrent research + chat
  └── 4c. Dual path entry on onboarding landing page

Layer 5: /research Page (depends on 1a-1d, 2a-2c)
  ├── 5a. Backend: GET /research/sources endpoint
  ├── 5b. Frontend: split-view page with controls, results, chat
  └── 5c. Research result cards with live updates
```

Layers 1-3 can be built in parallel. Layer 4 and 5 depend on infrastructure being in place.

---

## Out of Scope

- Paid source plugins (Crunchbase, Apollo, Clearbit) — same `ResearchSource` protocol, added later
- Downstream goal-based curation (brief page, monitoring priorities) — `platform_goals` is persisted now, wiring comes later
- Research history / search across past research tasks
- Sharing research results between users
- Research scheduling (periodic re-research of watched companies)
