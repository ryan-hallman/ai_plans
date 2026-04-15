# Phase 4: Feature-Complete Operator Demo — Design Spec

**Date**: 2026-04-12
**Status**: Approved design — pending implementation planning

---

## 1. Overview

Phase 4 delivers a feature-complete, operator-first demo of Prescient OS. The demo viewer experiences the platform as Sarah Chen, CEO of Peloton Interactive, using real public company data enriched with realistic synthetic internal data.

**What this phase builds:**

1. Rich Peloton seed pipeline — real EDGAR filings, earnings transcripts, competitor profiles, LLM-generated internal scenarios
2. Copilot chat with SSE streaming — tool call visibility, inline citations, multi-turn, cross-company queries
3. Artifact & source viewer — citation click-through, document viewer, version history, provenance panel
4. Insight-to-action flow — create action items and decision records from chat and briefing with provenance links
5. Board prep deck generation — 10-section board package assembled from accumulated data, section-by-section operator curation
6. Demo script — ~9-minute operator-first walkthrough

**Architecture:** Seed-first approach. Build the data layer, then the streaming copilot, then surfaces that read from it. Every UI screen is testable against real content from day one.

**Depends on:** Phases 1-3 (Organizations, Artifacts, Auth, Intelligence, Onboarding, Monitoring, KPIs, Actions, Briefing all operational).

---

## 2. Seed Data Strategy

### 2.1 Hero Company: Peloton Interactive

**Why Peloton:** Universal brand familiarity lets demo viewers gut-check the system's analysis. Rich public filing history. Dramatic narrative arc (COVID boom → subscriber loss → cost restructuring → turnaround) creates natural demo scenarios. Mid-to-large enough for deep data but relatable as an operating challenge.

**Demo organization:** Peloton Interactive, type `independent`.
**Demo user:** Sarah Chen, CEO. Email/password auth.

### 2.2 Real Public Data (EDGAR + Web)

All tagged `source_type: "filing"`, `"transcript"`, `"news"`, or `"public"` as appropriate.

#### SEC Filings

- **10-K annual reports:** FY2022, FY2023, FY2024 (3 full years)
- **10-Q quarterly reports:** Last 8 quarters
- Each filing parsed into discrete sections: Business Overview, Risk Factors, MD&A, Financial Statements, Notes to Financial Statements
- Each section stored as a separate document — gives the copilot granular citation targets
- **Estimated volume:** 30-40 document sections, each independently citable

#### Earnings Call Transcripts

- Last 6-8 quarterly earnings calls
- Parsed into segments: CEO prepared remarks, CFO prepared remarks, Q&A (each analyst question + management answer as a separate segment)
- **Estimated volume:** 40-60 transcript segments
- Source: publicly available from investor relations / SEC filings

#### Product Catalog

Structured artifact with blocks per product line:

| Product | Details |
|---------|---------|
| Bike | Price, positioning, target customer, sales trajectory |
| Bike+ | Premium tier, differentiators from base Bike |
| Tread | Treadmill offering, history (recall, relaunch) |
| Tread+ | Premium tread, recall history, current status |
| Row | Rowing machine, launch timing, market reception |
| Guide (discontinued) | What it was, why it was killed, lessons |
| All-Access Membership | Pricing, content library, connected device requirement |
| Peloton App | Lower-tier subscription, content access, device-agnostic |
| Corporate Wellness | B2B offering, pipeline, pricing model |
| Rental/Financing Program | How it works, impact on hardware revenue recognition |

#### KPI Series

Extracted from filings and earnings transcripts. Minimum 8 quarters per metric for trend analysis.

| KPI | Granularity |
|-----|------------|
| Revenue (total) | Quarterly |
| Revenue — Connected Fitness segment | Quarterly |
| Revenue — Subscription segment | Quarterly |
| Gross margin (total) | Quarterly |
| Gross margin — Connected Fitness | Quarterly |
| Gross margin — Subscription | Quarterly |
| Connected fitness subscribers | Quarterly |
| Monthly churn rate | Quarterly |
| Average monthly subscription revenue per subscriber (ARPU) | Quarterly |
| Free cash flow | Quarterly |
| Hardware units sold | Quarterly |
| EBITDA / Adjusted EBITDA | Quarterly |

Written to BOTH `documents.kpi_values` (legacy, planner tools) AND `kpis.kpi_values` (new UI).

#### News and Market Signals

15-20 articles/signals spanning the last 12-18 months:

- CEO transitions (Foley → McCarthy → Woodworth)
- Layoff rounds (multiple)
- Amazon partnership announcement
- Rental program launch
- Lululemon Studio shutdown
- Apple Fitness+ bundling changes
- PE acquisition rumors (various)
- Quarterly earnings coverage

Each stored as a document with: source URL, published date, headline, excerpt, relevance tags.

#### Litigation & Regulatory

From 10-K risk factors, 8-K filings, and public court records:

| Case | Content |
|------|---------|
| **CPSC / Tread+ recall** | Timeline, financial impact ($400M+ recall cost), settlement, safety improvements |
| **Mad Dogg Athletics patent suit** | Spinning trademark, bike design patents, outcome and product naming implications |
| **Securities fraud class action** | Shareholder suits re: misleading subscriber growth projections, current status |
| **Music licensing (NMPA)** | Unlicensed songs in classes, $300M+ settlement, ongoing royalty structure impact on content costs |
| **FTC investigation** | Marketing claims around subscription pricing, status |

Stored as: individual documents (filing excerpts) + a synthesized "Legal & Regulatory Risk" artifact.

### 2.3 Competitor Deep Profiles

Not just names — full research profiles. Each competitor gets: a Company Profile artifact (structured blocks), monitoring watch targets, and 3-5 intelligence signals/findings.

#### Apple (Fitness+)

- Product details: bundled with Apple Watch / Apple One, workout types, instructor approach, device ecosystem lock-in
- Competitive positioning: free-with-hardware model, threat to Peloton's premium positioning
- Recent moves: feature additions (cycling with metrics), pricing changes, Apple Health integration
- Key metric comparisons where available (analyst subscriber estimates)
- **Threat narrative:** Ecosystem bundling makes Fitness+ effectively free for Apple Watch users — directly competes with Peloton's $44/month subscription

#### Lululemon (Studio / Mirror)

- Full arc: acquired Mirror for $500M → rebranded to Lululemon Studio → shut it down entirely
- Competitive lesson: hardware + subscription model didn't work for them
- Current positioning: pivoted back to pure retail/athleisure
- **Relevance:** Cautionary tale and data point for hardware-to-subscription pivot strategy

#### Echelon

- Budget alternative positioning, full product lineup (bikes, treadmills, rowers), pricing undercuts
- Walmart partnership as distribution strategy
- Quality/reliability narrative vs. Peloton premium
- **Threat narrative:** Price-sensitive segment that Peloton loses when they discount hardware

#### iFit / NordicTrack

- Parent company (Icon Health & Fitness) bankruptcy and restructuring
- Product overlap, market positioning
- **Relevance:** Cautionary tale for hardware-heavy connected fitness model

### 2.4 LLM-Generated Synthetic Internal Data

All tagged `source_type: "internal"`. Clearly distinguishable from public data in provenance views.

#### Strategic Artifacts

| Artifact | Content |
|----------|---------|
| **Strategic Focus / Value Creation Plan** | 3 priorities: (1) subscription growth via App tier expansion and corporate wellness, (2) hardware margin recovery via supply chain optimization and premium positioning, (3) content differentiation as competitive moat. Each with metrics, owners, horizons. |
| **Cost Structure & Margin Drivers** | Hardware BOM breakdown (bike vs. tread vs. row), subscription unit economics, content production costs, warehouse/logistics costs, customer acquisition cost by channel |
| **Channel & Distribution Strategy** | DTC website performance, Amazon marketplace (new), retail partners (Dick's, etc.), corporate wellness pipeline, international expansion status |
| **Org & Leadership Assessment** | Current exec team strengths/gaps, recent changes (new CEO), key hires needed, board composition |
| **Technology & Platform Roadmap** | Content platform architecture, hardware R&D pipeline (next-gen bike?), app experience investment plan, AI/personalization roadmap |

#### Decision Records (8)

Each with: context, decision, rationale, spawned action items, linked artifacts.

1. Approved hardware price cuts to drive subscriber acquisition (linked to margin KPI impact)
2. Launched rental/financing program (linked to revenue recognition changes)
3. Restructured workforce — 15% reduction (linked to EBITDA improvement target)
4. Entered Amazon as distribution channel (linked to channel strategy artifact)
5. Discontinued Guide product line (linked to hardware margin analysis)
6. Shifted marketing budget from hardware to subscription value proposition
7. Accelerated corporate wellness sales program (linked to pipeline artifact)
8. Approved content licensing renegotiation strategy (linked to legal artifact)

#### Action Items (20)

Mix of statuses, owners, and origins:

| Status | Count | Examples |
|--------|-------|---------|
| `pending_review` | 5 | Analyze churn by device ecosystem, Prepare Q3 earnings narrative, Review WSJ inquiry response |
| `sent_to_owner` | 6 | VP Data: cohort analysis, CFO: margin bridge for board, CTO: app performance benchmarks |
| `completed` | 6 | Closed Amazon channel launch, Completed workforce reduction, Finalized music licensing terms |
| `deferred` | 3 | International expansion scoping, Next-gen hardware R&D kickoff, Enterprise SSO integration |

Each linked to originating decision record, briefing item, or conversation.

#### KPI Anomalies with Enrichment

| Anomaly | Enrichment Question |
|---------|-------------------|
| Q3 subscriber churn spiked to 8.2% (trailing avg: 5.1%) | "Connected fitness churn spiked following the Apple Fitness+ free bundle announcement. Is this primarily driven by Apple ecosystem users, or are you seeing elevated churn across all cohorts?" |
| Subscription revenue flat QoQ despite subscriber growth | "ARPU declined 6% QoQ. Is this driven by the lower-priced App tier mix shift, or are you seeing downgrades from All-Access?" |
| Connected Fitness gross margin turned negative (-8.2%) | "Connected Fitness segment gross margin turned negative. Is this a deliberate investment in subscriber acquisition via hardware discounting, or is there a supply chain cost issue?" |
| Free cash flow improved despite revenue decline | "FCF improved $45M QoQ while revenue declined. Is this sustainable cost discipline or one-time items (restructuring charges already behind)?" |
| Corporate wellness pipeline 3x QoQ | "Corporate wellness qualified pipeline grew from $2M to $6M ARR. What's driving the acceleration — is it the Gympass partnership or direct enterprise sales?" |

#### Monitoring Findings (15)

| Finding | Severity | Source |
|---------|----------|--------|
| Apple Fitness+ added cycling classes with real-time metrics | critical | news |
| Echelon launched a $400 rower (vs. Peloton Row at $3,195) | important | news |
| Former Peloton CTO joined Apple Health team | important | news |
| WSJ reporter inquiring about next round of layoffs | critical | internal |
| Gympass raised $300M, expanding connected fitness partnerships | notable | news |
| Lululemon confirmed Studio/Mirror fully wound down | notable | news |
| NordicTrack parent (iFit) emerged from bankruptcy with new strategy | notable | news |
| Peloton App downloads up 40% after price reduction | important | internal |
| Dick's Sporting Goods expanding Peloton floor space in 200 stores | important | news |
| TikTok fitness influencer backlash over Peloton subscription price increase | notable | social |
| Corporate wellness competitor Wellhub (ex-Gympass) signed Fortune 500 deal | notable | news |
| Patent filing detected: Peloton filed for AI-powered coaching system | notable | filing |
| Competitor Echelon facing FTC complaint over misleading comparison ads | info | news |
| Apple Watch Ultra 3 rumored to include advanced cycling power meter | important | news |
| Internal NPS survey: score dropped from 62 to 54 QoQ | important | internal |

### 2.5 Dual-Write Strategy

The seed pipeline writes to BOTH table sets:

- **Legacy tables** (`knowledge.artifacts`, `knowledge.artifact_versions`, `knowledge.artifact_citations`, `knowledge.artifact_subjects`, `documents.documents`, `documents.kpi_definitions`, `documents.kpi_values`) — so the planner's existing retrieval tools work without modification.
- **New tables** (`artifacts.artifacts`, `artifacts.artifact_versions`, `artifacts.artifact_references`, `artifacts.artifact_subjects`, `kpis.kpi_owners`, `kpis.kpi_values`, `kpis.kpi_anomalies`) — so Phase 1-3 UI surfaces work.
- **Other tables** (`action_items.*`, `monitoring.*`, `briefing.*`, `companies.*`) — single-write, these don't have a legacy/new split.

This eliminates the legacy migration problem for demo purposes. The seed script is the bridge.

### 2.6 Estimated Corpus Size

| Category | Count |
|----------|-------|
| SEC filing sections | 35-40 |
| Earnings transcript segments | 45-60 |
| News articles / signals | 15-20 |
| Litigation documents | 5-8 |
| Strategic artifacts | 5 |
| Competitor profile artifacts | 4 |
| Product catalog artifact | 1 |
| Legal & regulatory artifact | 1 |
| Decision records | 8 |
| Action items | 20 |
| KPI series (metrics x quarters) | ~120 data points |
| KPI anomalies | 5 |
| Monitoring findings | 15 |
| Competitor companies | 4 |
| **Total citable items** | **~180+** |

---

## 3. Copilot Chat Experience

### 3.1 Backend: SSE Streaming

**New endpoint:** `POST /intelligence/copilot/stream` — returns `text/event-stream`.

The existing planner loop (`run_planner`) is refactored into an async generator that yields events incrementally.

#### Event Types

```
event: tool_start
data: {"tool": "search_artifacts", "args": {"query": "subscriber churn"}}

event: tool_end
data: {"tool": "search_artifacts", "summary": "3 artifacts matched 'subscriber churn'"}

event: delta
data: {"text": "The subscriber churn increase in Q3 "}

event: citation
data: {"marker": 1, "tool_name": "get_kpi_series", "source_type": "kpi", "excerpt": "Churn: 8.2% Q3 vs 5.1% trailing avg"}

event: done
data: {"conversation_id": "...", "grounding_status": "grounded", "citations": [...], "tool_calls": [...], "iterations": 4}
```

#### Streaming Planner Loop

The planner becomes an async generator:

```
async for event in run_planner_stream(...):
    yield f"event: {event.type}\ndata: {event.json()}\n\n"
```

- Tool call → yields `tool_start`, executes tool, yields `tool_end`, appends result, calls LLM again
- LLM text response → yields `delta` events as tokens arrive from the Anthropic streaming API
- Citation markers `[[n]]` detected in the stream → yields `citation` event with resolved metadata
- Grounding validation runs on complete text at stream end
- If regrounding needed → stream continues transparently (user sees the retry)
- **Hard cap:** 15 iterations, same as existing planner

#### Conversation Persistence

- Each conversation gets a UUID `conversation_id`
- At stream completion: messages, tool calls, and tool results persisted in one transaction
- Multi-turn supported: follow-up questions include prior conversation messages
- Company scope attached to conversation: `company_slug` determines which company's data is accessible
- Cross-company queries work via the existing `ConversationScope.visible_company_ids` (competitors in the companies graph)

### 3.2 Next.js BFF Route

`src/app/api/chat/route.ts`:

- Proxies SSE from FastAPI to the browser
- Adds auth headers from session
- Preserves event framing end-to-end
- Handles upstream connection errors gracefully

### 3.3 Chat UI

**Route:** Accessible from company pages and as a standalone `/chat` route.

**Layout:** Full-width. Two panels when a citation is expanded.

#### Left Panel (Message Thread)

- User messages styled as sent bubbles
- Assistant messages stream in token-by-token from `delta` events
- Tool call activity renders above the answer as compact status chips: `Searching artifacts... → 3 found` → `Fetching KPI series... → 8 quarters`. Chips collapse/minimize once the answer starts streaming.
- Input at bottom: textarea with send button, `Cmd+Enter` to submit
- Company context shown at top of chat (scoped to the active company)

#### Inline Citations

- `[[n]]` markers in the stream → rendered as clickable pills with source type icon and short label
- Hover tooltip shows excerpt from the cited source
- Click opens the right panel with the full source content

#### Right Panel (Source Viewer)

Slides in on citation click:

- **For artifacts:** Shows the artifact content with the relevant block highlighted
- **For documents (filings, transcripts):** Shows the document section with the cited excerpt highlighted, plus metadata (filing date, source URL linking to EDGAR)
- **For KPI data:** Shows the KPI chart/table with the cited data point highlighted
- Close button to dismiss, or click another citation to swap content

#### Provenance Panel

Toggle "Show reasoning" on any assistant message:

- Every tool call the planner made, in chronological order
- What each tool returned (summary + record count)
- Which citations map to which tool results
- Total iterations and token count

#### Error Handling

| Scenario | Behavior |
|----------|----------|
| Stream interruption (network) | "Connection lost" message with "Retry" button that re-sends the last question |
| Grounding failure (couldn't ground after retries) | Response shown with warning banner: "This response could not be fully verified against sources." Ungrounded claims visually flagged. |
| Planner loop exceeded | "This question required too many retrieval steps. Try a more specific question." |
| No relevant data found | "I couldn't find relevant information about that in the available sources. Try rephrasing or ask about a specific metric or filing." |

---

## 4. Artifact & Source Viewer

### 4.1 Artifact Detail View (`/companies/[slug]/artifacts/[id]`)

- **Header:** Title, type badge (Company Profile, Cost Structure, etc.), confidence label, last updated
- **Content:** Blocks rendered in order — heading + body. Inline citations within blocks are clickable pills (same as chat)
- **Version sidebar:** Version list with state badges (draft, active, approved, archived). Click to view previous versions. Shows approver and timestamp.
- **Cross-references:** "Related artifacts" section showing linked artifacts via `artifact_references` (INFORMS, SPAWNED, SUMMARIZES, RELATES_TO relationships). Clickable.

### 4.2 Document Viewer

When a citation points to a raw document (SEC filing section, transcript segment, news article):

- Document title, type badge, filing/publication date
- Source URL (links to EDGAR for SEC filings, original source for news)
- The relevant excerpt highlighted within the full section text
- "View full filing on EDGAR" external link where applicable

### 4.3 Company Overview Enrichment (`/companies/[slug]`)

The existing page gets richer:

- **KPI strip** — last 4 quarters of key metrics with sparkline trends (already partially built)
- **Artifacts grid** — grouped by function. Cards show: title, type, confidence, last updated. Click through to detail view.
- **Recent activity timeline** — intelligence signals, decisions, action items touching this company. Chronological, filterable.
- **"Ask about this company"** — button that opens chat pre-scoped to this company
- **Competitor sidebar** — watched competitors with latest finding per competitor

---

## 5. Insight-to-Action Flow

### 5.1 From Chat

After any assistant message, a subtle action bar:

- **"Create action item"** — inline form pre-filled from conversation context. System proposes: title (extracted from insight), owner (suggested by topic — KPI → CFO, competitive → CRO), priority, due date. Operator edits and confirms.
- **"Log decision"** — decision record form. Context block auto-populated from the conversation. Operator adds: decision made, rationale. Can optionally spawn action items.

Both link back to the conversation via `source_artifact_id` for full provenance.

### 5.2 From Briefing

"Decide" action-type briefing items get a "Take action" button → same action item / decision record flow. Originating briefing item ID linked.

### 5.3 Actions Page Enrichment

Existing page additions:

- **Provenance column** — shows origin (chat, briefing, decision record) with clickable link to source
- **Filter by owner and status**
- **Status transitions** with appropriate confirmation UX

---

## 6. Board Prep & Deck Generation

### 6.1 Board Deck Structure

The system generates a full board deck, section by section. Not a brief — the full package a board expects.

| # | Section | Primary Data Sources |
|---|---------|---------------------|
| 1 | **Executive Summary** | Synthesized from all below |
| 2 | **Financial Performance** | KPI snapshots (revenue, margins, EBITDA, cash flow), anomaly enrichments, filing data. Multiple sub-pages: P&L summary, revenue by segment, margin bridge, cash flow. |
| 3 | **Subscriber / Customer Metrics** | Subscriber count, churn, ARPU, cohort trends, LTV. KPI series + anomaly commentary. |
| 4 | **Pipeline & Go-Get** | Corporate wellness pipeline, retail expansion, Amazon channel, subscriber acquisition funnel. Action items + internal artifacts. |
| 5 | **Key Initiative Updates** | Status per strategic priority from Value Creation Plan. Committed vs. delivered vs. slipped. RAG status. Action items by initiative. |
| 6 | **Competitive Landscape** | Monitoring findings, competitive profile artifacts, intelligence signals. Per-competitor summary. |
| 7 | **Risk & Legal** | Litigation updates, regulatory exposure, operational risks. Legal artifact + risk-tagged signals. |
| 8 | **People & Organization** | Key hires/departures, org changes, headcount vs. plan. Decision records + action items tagged to org/people. |
| 9 | **What Not to Worry About** | Signals that look alarming but aren't strategic, scored against Strategic Focus artifact. The differentiator. |
| 10 | **Decisions Needed** | Open items requiring board input, framed with context and recommendation. |

### 6.2 Generation Approach

**Section-by-section, not monolithic:**

- Each section has its own prompt template specifying what data to gather and how to structure the output
- A `BoardPrepAssembler` service gathers relevant artifacts, KPIs, action items, and signals per section
- Each section is grounded independently — citations scoped per section
- Sections can be regenerated individually

**Assembly service flow:**

1. Query time-range scoped data: KPIs, anomalies, decision records, action items, monitoring findings, intelligence signals for the quarter
2. For each section template: filter relevant data, call LLM with section-specific prompt + gathered data
3. Return structured result: list of sections, each with title, generated content (markdown with `[[n]]` citations), and citation metadata

### 6.3 Operator Workflow

1. **Trigger:** "Prepare board deck" → select quarter, confirm company
2. **Generation:** Progress indicator per section as each generates
3. **Review:** Deck rendered section by section. Each section editable:
   - Edit text directly (rich text)
   - Reorder sections via drag
   - Remove a section
   - "Regenerate this section" with optional guidance ("focus more on the margin bridge")
   - Add a manual section
4. **Approve:** Saved as a Board Narrative artifact with version history
5. **Export:** Deferred post-demo. In-platform view is sufficient for demo.

### 6.4 Deferred: Template Support

Future phases will add:
- Configurable board deck templates (define which sections, in what order, with what content guidance)
- Template library (different templates for different board audiences or reporting cadences)
- Per-section configuration (data sources, depth, formatting preferences)
- Template sharing across organizations

Not in scope for Phase 4.

---

## 7. Demo Script

### Persona

**Sarah Chen, CEO of Peloton Interactive.** The demo viewer experiences the platform through her daily workflow.

### Flow (~9 minutes)

#### 0:00 — 1:00 | Morning Brief

Land on the Morning Brief. 6 prioritized items:
- KPI anomaly: subscriber churn spiked to 8.2%
- Competitive signal: Apple Fitness+ added cycling with metrics
- Action item due: Q3 earnings narrative draft (assigned to CFO)
- Monitoring finding: WSJ reporter inquiry about layoffs
- Completed milestone: Amazon channel launch finalized
- Action item: Corporate wellness pipeline review Thursday

Monday variant shows week-ahead calendar with the above as a living task list.

#### 1:00 — 1:30 | Drill into Anomaly

Click the churn anomaly → KPI page. Historical churn trend visible: steady at ~5%, then the Q3 spike to 8.2%. System's enrichment question displayed: "Is this driven by Apple ecosystem users?"

#### 1:30 — 3:30 | Ask the Copilot

Open chat from company page. Type: *"What's driving the subscriber churn increase, and is it related to the Apple Fitness+ bundling announcement?"*

Watch: tool call chips appear (Searching artifacts → 3 found, Fetching KPI series → 8 quarters, Searching documents → CFO earnings remarks). Answer streams with 5-6 inline citations to real sources.

#### 3:30 — 4:30 | Cross-Company Query

Follow up: *"How does our churn compare to what happened when Lululemon shut down Studio? Is there a pattern in how fitness subscription services lose users to ecosystem bundles?"*

Copilot pulls from Lululemon competitor profile and Apple competitive analysis. Demonstrates cross-company reasoning with citations to both profiles.

#### 4:30 — 5:30 | Citation & Provenance

Click a citation from the CFO's earnings call remarks. Right panel shows the actual transcript excerpt highlighted. Toggle "Show reasoning" to see the full tool call chain — what the planner searched, what it found, how it assembled the answer.

#### 5:30 — 6:30 | Insight to Action

Click "Create action item" on the assistant message. System proposes: *"Analyze subscriber churn by device ecosystem — quantify Apple Watch vs. Android exposure"*, assigned to VP Data, due Friday. Approve.

Click "Log decision": *"Decided to fast-track Android app experience investment based on ecosystem churn analysis."* Decision record created with link back to chat.

#### 6:30 — 8:30 | Board Prep

Open board prep. Select Q3. System generates the full deck — progress indicators per section. Financial pages with real numbers, subscriber metrics with churn narrative, competitive landscape with Apple/Echelon/Lululemon analysis, initiative updates with RAG statuses, risk/legal with litigation summary, "What Not to Worry About" section deprioritizing the TikTok backlash.

Scroll through. Every claim cites a real source. Edit one section to add CEO commentary. Approve the deck.

#### 8:30 — 9:00 | Close

"Everything you saw was grounded in real SEC filings, real earnings transcripts, and structured operational data. The board deck that takes your exec team three weeks wrote itself from the work you were already doing in the system."

---

## 8. Technical Considerations

### Streaming Architecture

- Anthropic SDK supports streaming via `client.messages.stream()`. The planner loop wraps this to yield events at both the tool-call and text-generation levels.
- SSE from FastAPI: use `StreamingResponse` with `media_type="text/event-stream"`.
- Next.js BFF: proxy the SSE stream through an API route to add auth headers. Use `ReadableStream` to pipe events to the browser.
- Browser: `EventSource` API or `fetch` with `ReadableStream` for typed event handling.

### Prompt Caching

The existing Anthropic adapter already uses prompt caching on the system prompt. With a stable system prompt + tool catalog, cache hits reduce latency and cost significantly for multi-turn conversations within the same company context.

### Grounding at Scale

With 180+ citable items in the corpus, the grounding validator needs to handle:
- Answers that cite 8-10 sources (higher than the 3-5 we tested with Olaplex)
- Cross-company citations (citing Peloton KPIs alongside Apple competitive analysis)
- Mixed source types in a single answer (filing data + transcript excerpt + synthetic artifact)

This is a deliberate stress test. If grounding fails on a complex query, we want to discover that now, not during a demo.

### Board Prep Token Budget

A 10-section board deck involves 10 LLM calls (one per section). Each call receives the section-relevant data (not the full corpus). Estimated token budget per section: 2-4K input (gathered data + prompt template), 500-1500 output. Total for full deck: ~30-50K tokens. Should complete in under 60 seconds with parallel section generation where data doesn't overlap.

---

## 9. What's NOT in Phase 4

- Board deck template support (configurable sections, ordering, content rules)
- Board deck export (PDF/PPTX)
- Sponsor access / triage layer (Module 6 from spec)
- Notification delivery (push, email briefing)
- Real-time monitoring (continuous scan) — demo uses pre-seeded findings
- Multi-user / role-scoped views — demo is single-user (Sarah Chen)
- Conversation history browser (past conversations list) — multi-turn works within a session but there's no saved history view
- Mobile responsiveness
