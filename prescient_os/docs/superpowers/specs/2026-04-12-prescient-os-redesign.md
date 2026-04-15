# Prescient OS — Operator-First Platform Redesign

**Date**: 2026-04-12
**Status**: Approved design — pending implementation planning

---

## 1. Core Identity & Positioning

**Prescient OS** — an operating intelligence platform for company leadership.

**One-liner**: "Your chief of staff that never sleeps — daily intelligence, strategic context, and sponsor management in one system."

**Positioning**: The platform exists to serve operators of sponsor-backed (and independent) companies. PE firms are a deployment channel and downstream beneficiary, not the primary user. The sell to PE is: "Deploy this into your portfolio companies and they'll perform better, and you'll get transparency as a byproduct."

### Primary Users (Priority Order)

1. **Company operators** — CEO, CTO, COO, CFO. The people running the business. Daily users.
2. **Functional leaders** — VP Eng, VP Sales, VP Marketing, etc. Own KPIs, consume role-specific intelligence, contribute to reporting flow.
3. **Operating partners** — get curated context about their portfolio companies so they can contribute value instead of asking catch-up questions.
4. **PE analysts/VPs** — get transparency as a byproduct. Self-serve answers instead of emailing operators.
5. **Board members** — get pre-digested context before meetings. Replaces "paste the deck into ChatGPT" workflow.

### The Trust Principle

The system earns operator trust by protecting their time. It never shares upstream without permission (beyond agreed-upon quantitative metrics). If operators feel surveilled, they stop using it, and the system dies. The platform subtly advocates for the operator — not by blocking PE, but by demonstrating that trust = usage = better data for everyone.

### Go-to-Market

Two sales channels, same product:

- **PE channel** (partner-led): PE firm buys, deploys across portfolio. Faster adoption because PE pushes it down. PE gets a portfolio intelligence dashboard as their view, operators get the daily operating tool.
- **Direct channel**: Individual companies buy it. Works for independent companies, sponsor-backed companies that want it before their PE firm adopts, or companies with any ownership structure.

A company deployed standalone should have the full experience. When/if their PE firm later adopts, the company "connects up" to the PE parent tenant without data loss or migration.

---

## 2. Problem Statement

Operators of sponsor-backed companies waste enormous time managing up — board prep, monthly reporting, ad-hoc requests, educating sponsors — while simultaneously needing to stay on top of market intelligence, competitive dynamics, and strategic decisions.

The information asymmetry between operators and sponsors means every interaction starts with catch-up instead of value creation. PE firms, operating partners, and board members lack sufficient context to contribute strategically, so they either focus on the wrong things or generate low-value requests that further tax operator time.

Meanwhile, operators are drowning in information from the other direction — newsletters, market reports, competitive signals, AI developments — with no system to filter what's actually worth their time.

### Pain Points by Stakeholder

**Operators (CEO/CTO/COO/CFO)**:
- Board prep is a multi-week quarterly lift
- Monthly reporting to PE is a steady-state drain
- Ad-hoc requests from sponsors require context-switching to pull data
- Time spent educating sponsors on business context instead of receiving strategic help
- Information overload from external sources — newsletters, reports, market signals

**Operating Partners**:
- Stretched across multiple portfolio companies, not deep enough in any one to add real value
- Perceived as PE's mole rather than a trusted advisor
- Lack context to contribute meaningfully in weekly touchpoints

**PE Analysts/VPs**:
- Dependent on operators for information, which arrives episodically and formatted for board consumption, not analysis
- Don't know what they don't know — information asymmetry drives misaligned questions

**Board Members**:
- Receive 100-page decks, struggle to find what's meaningful
- Resort to pasting decks into ChatGPT for summaries
- Can't contribute strategic value without deep context

### The Thesis

The platform solves this by becoming the operator's first touchpoint each morning. As operators use it daily for intelligence and decision-making, the system accumulates the raw material for reporting as a byproduct. Over time, quarterly board reporting becomes irrelevant — not because anyone decided to stop, but because sponsors are already continuously informed through the platform.

---

## 3. Feature Architecture

### Module 1: Daily Briefing ("Morning Brief")

Every morning, each user sees a personalized, prioritized view of what matters to them today.

**For operators (CEO/CTO/COO/CFO)**:
- Top 3-5 items ranked by urgency and relevance to their role
- Each item tagged with action type: Read, Decide, FYI
- Sources: external intelligence (competitor news, market signals), internal activity (action items due, KPIs that moved, sponsor questions pending), and forward-looking items (board meeting in 12 days, operating partner call Thursday)

**For functional leaders (VP Eng, VP Sales)**:
- Scoped to their domain: VP Eng sees tech signals, engineering KPIs, their action items
- Does not surface CEO-level strategic noise unless explicitly tagged

**For operating partners**:
- Cross-company view: across their portfolio companies, what's noteworthy today
- Scoped to what operators have made visible

**Monday variant**:
- Weekly delta: what changed since Friday
- Forward calendar: what's coming this week, as a living task list
- Task list updates through the week — items cross off as completed, at-risk items flagged past due

**Quality principle**: Err toward fewer, higher-confidence items rather than a firehose. An empty briefing that says "nothing urgent today" is better than 10 low-value items. If the briefing surfaces noise, operators stop checking, and the platform loses its position as the first daily touchpoint.

### Module 2: Intelligence Engine

**Layer 1 — Monitored Sources (MVP)**:
- Company-level setup: watch named competitors, how they compete, monitoring scope (SEC filings, news, job postings, social media, events), monitoring frequency
- User-level interests: Pinterest-style onboarding — topics, technologies, markets the user cares about
- Continuous scan, filter by relevance to company + user role, surface items that pass threshold into Daily Briefing

**Layer 2 — Ad-Hoc Company Research (MVP)**:
- "Tell me everything about Company X" — kicks off an intel-gathering workflow
- Pulls public data: SEC filings, news, job postings, social media presence, product announcements, funding history, leadership team
- Structures into a Company Profile artifact
- User can then "keep watching" to add to monitored sources
- Also used during onboarding: when a company names competitors, the research module runs automatically so the user sees it work in real-time and curates the output

**Layer 3 — Contextual Intelligence (Post-MVP)**:
- Because the system knows what the user is working on (from actions, decisions, briefing interactions), it surfaces signals in context
- Example: "You're evaluating a new distribution channel — here's what Competitor X just announced about theirs"

### Module 3: Action & Decision Log

Lightweight capture of decisions made and actions taken. Not a project management tool — the connective tissue between intelligence and reporting.

**How it works**:
- Operator reads a briefing item, takes an action, logs it: "Decided to accelerate Q3 hiring plan based on competitor expansion signal. Assigned to VP Eng."
- Actions have: owner, status, due date, originating context (which briefing item or intelligence signal triggered it)
- KPI owners assigned by leadership — responsible for updating their metrics on cadence

**Why it matters**: Every logged action and decision becomes raw material for the Board Prep view. The operator isn't "doing reporting" — they're working in the system. Reporting accumulates as a byproduct.

### Module 4: KPI Management

KPI reporting is an intelligence interaction, not data entry.

**Ownership model**: Leadership assigns KPI owners. Each owner is responsible for updating their metrics on a defined cadence (weekly/monthly).

**Anomaly detection and enrichment**: When an owner reports a metric, the system compares against historical trends, targets, and related signals. If it detects an anomaly, deviation, or noteworthy pattern, it initiates a conversation:
- "Revenue came in 12% below forecast this month. I see that your largest customer delayed their renewal (from the intelligence feed). Is that the primary driver, or is something else going on?"
- The owner responds, adding context attached to the KPI Snapshot as structured commentary
- If the deviation triggers an action, the system helps log it as a Decision Record + Action Item

Every KPI entry has the potential to become a rich artifact — not just a number, but a number with *why* and *what we're doing about it*. This is exactly the context that makes board narratives write themselves.

### Module 5: Board Prep & Reporting View

A "lens" on everything that's accumulated — not a separate workflow.

**How it works**:
- "Show me what's happened this quarter" — system drafts a narrative from: intelligence signals acted on, decisions logged, KPI movements, action item progress, key risks surfaced
- Operator reviews, edits, approves. This becomes board deck content.
- Over time, as more activity flows through the system, the draft improves and the operator edits less.

**Graduation path**:
- Phase 1: System drafts, operator heavily edits. Still saves time vs. starting from scratch.
- Phase 2: System drafts, operator does light edits. Board deck is 80% automated.
- Phase 3: Sponsors use the platform directly. The "deck" is a snapshot export. Quarterly reporting becomes a formality.

### Module 6: Sponsor Access & Triage Layer

Sponsors (PE, board members, operating partners) get a gated view and can ask questions the system handles.

**Default visibility (Tier B)**:
- Quantitative metrics (KPIs, financials) flow automatically on agreed cadence
- Narrative content, strategic decisions, sensitive items — operator-gated

**Configurable visibility (Tier C)**:
- PE and operator negotiate boundaries at onboarding
- Platform enforces boundaries
- PE can request full transparency, but the system makes clear that heavy-handed access suppresses operator engagement

**Triage flow**:
1. Sponsor asks a question through the platform
2. System checks if it can answer from information visible to the sponsor
3. If yes → answers with citations
4. If no → assesses importance
5. Only escalates to operator if validated as important AND unanswerable from shared information
6. When escalated, the system checks the operator's full context (including firewalled information) and either:
   - Proposes a draft response that threads the needle — answers using only shareable information, informed by the full picture
   - If nuanced, enters a brainstorm dialogue: "Here's what they're really asking, here's what we know internally, here's what we can and can't share — how do you want to handle this?"
7. Operator sees all sponsor questions, even when auto-handled (transparency goes both ways)

**Operating partner view**:
- Cross-portfolio summary of their companies
- Pre-call briefing: "Your Thursday call with Acme Corp — here's what's been happening, open action items, CTO's flagged priorities"
- Questions triaged through the same flow

### Module 7: Onboarding Wizard

**Company-level setup**:
- Company profile: industry, size, ownership structure, key metrics
- Competitive landscape: name competitors → research module runs immediately against each, user watches it work and curates the output
- How competitors compete: user annotates each competitor profile with competitive dimensions
- Monitoring scope and frequency per competitor
- Reporting relationships: PE firm, board composition, current reporting cadence
- Visibility defaults: what flows automatically vs. what's operator-gated

**User-level setup**:
- Role selection → determines intelligence filter baseline
- Interest profile (Pinterest-style): topics, technologies, markets they care about
- Notification preferences: daily briefing delivery, alert thresholds

---

## 4. Artifact Model

Every meaningful interaction with the system produces or enriches a durable artifact. These compound over time into institutional knowledge.

### Artifact Types

| Artifact | What it captures | How it's created |
|----------|-----------------|------------------|
| **Company Profile** | Structured intelligence about a company — financials, leadership, products, market position, recent activity | Research module on first run, continuously enriched by monitoring |
| **Intelligence Signal** | A discrete piece of external intelligence — news event, filing, competitor move, market shift | Automatically surfaced by monitoring, filtered by relevance |
| **Decision Record** | A decision made, the context that informed it, who made it, what actions followed | Operator logs from briefing interaction or ad-hoc |
| **Action Item** | A tracked task with owner, status, due date, originating context | Created from decisions, briefing items, or sponsor triage |
| **KPI Snapshot** | A point-in-time metric value with owner, commentary, and enrichment conversation | KPI owners report on cadence or ingested from exports; anomaly detection triggers enrichment |
| **Briefing** | The daily/weekly briefing as delivered — record of what the system surfaced | Auto-generated daily, becomes historical record |
| **Board Narrative** | Drafted quarterly narrative assembled from other artifacts | System drafts from accumulated artifacts, operator curates |
| **Triage Record** | A sponsor question, how it was handled, the outcome | Created by triage flow |

### Cross-Referencing

Artifacts reference each other. A Decision Record links to the Intelligence Signals that informed it. An Action Item links to the Decision that spawned it. A Board Narrative references the KPI Snapshots, Decisions, and Actions it summarizes. A Triage Record links to the artifacts the system used to answer.

This web of references is what makes the system compound. When the Board Prep view drafts a narrative, it assembles from a graph of connected artifacts with full provenance.

### Data Freshness

| Source | Refresh cadence | Method |
|--------|----------------|--------|
| Monitored competitor news/social | Continuous (hours) | Automated scan + filter |
| SEC filings | As published | Automated ingest + parse |
| KPIs | Owner-defined (weekly/monthly) | Manual entry or financial export; anomaly-triggered enrichment |
| Action item status | As updated by owners | Manual in-platform |
| Briefing generation | Daily (morning) | Automated assembly |
| Board narrative draft | On-demand or quarterly trigger | Automated assembly + operator curation |

---

## 5. User Experience by Role

### Operator (CEO / CTO / COO / CFO)

**Daily**:
- Opens Morning Brief — 3-5 prioritized items, action-typed (Read/Decide/FYI)
- Scans intelligence feed for surfaced signals
- Acts on items: logs decisions, creates action items, dismisses noise
- Reviews triage queue: sponsor questions the system couldn't auto-handle, each with a proposed response or brainstorm prompt

**Weekly**:
- Monday brief adds delta view + forward calendar
- Reviews action items in flight — updates status, flags at-risk
- Checks KPI submissions from functional leaders — system flags anomalies for conversational follow-up

**Quarterly (diminishing over time)**:
- Opens Board Prep view — system drafts narrative from accumulated artifacts
- Reviews, edits, approves. Exports to deck format if needed.
- Matures from 20-hour creation process to 30-minute review

### Functional Leader (VP Eng, VP Sales, etc.)

**Daily**: Opens domain-scoped Morning Brief. Intelligence feed filtered to interests and role.

**On cadence (weekly/monthly)**: Reports owned KPIs — system engages on anomalies. Updates action items.

**As needed**: Runs ad-hoc company research. Curates competitive monitoring within domain.

### Operating Partner

**Daily/weekly**: Cross-portfolio Morning Brief scoped to visible information.

**Before touchpoints**: Pre-call briefing with context, open items, operator-flagged priorities.

**As needed**: Asks questions through platform — system answers or triages. Sees triage history.

### PE Analyst / VP

**On cadence**: Portfolio dashboard with automatically flowing KPIs per visibility rules. Drill into shared metrics.

**As needed**: Asks questions — system triages. Gets notifications when operators share narrative content.

### Board Member

**Pre-meeting**: Curated briefing — key metrics, strategic context, decisions since last meeting, recommended discussion topics.

**Post-meeting**: Action items from meeting logged into system, decisions captured with context.

---

## 6. Technical Architecture

### Tech Stack

- **API**: Python / FastAPI / SQLAlchemy / Pydantic
- **Frontend**: Next.js App Router / React Server Components / BFF pattern
- **Primary store**: Aurora Postgres with RLS for tenant isolation
- **Search**: OpenSearch with BM25 (no vector embeddings at MVP)
- **LLM**: Anthropic only, hand-rolled planner loop (no agent frameworks)
- **Events**: Transactional outbox pattern
- **Auth**: Email/password with JWT for MVP (WorkOS SSO/SAML/SCIM deferred)

### Bounded Contexts

| Context | Purpose |
|---------|---------|
| **Intelligence Engine** | Monitoring, research workflows, signal scoring, source management |
| **Briefing** | Daily/weekly briefing assembly, personalization, delivery |
| **Artifact Store** | All artifact types, cross-references, provenance graph |
| **KPI Management** | KPI definitions, ownership, cadence, anomaly detection, enrichment conversations |
| **Board & Reporting** | Board prep view, narrative drafting, quarterly cycle management |
| **Triage** | Sponsor question handling, auto-answer, brainstorm sessions, escalation routing |
| **Action Management** | Decision records, action items, status tracking, owner assignment |
| **Organization & Access** | Companies, users, roles, PE relationships, visibility rules, billing entity modeling |
| **Onboarding** | Company setup wizard, user interest profiling, initial research runs |
| **Identity & Access** | Authentication, authorization, session management |
| **Notifications** | Briefing delivery, alert thresholds, push notifications |
| **Integrations** | External source connectors, financial export ingestion |

### Multi-Tenancy Model

- **PE-deployed**: PE firm is top-level tenant. Each portfolio company is a sub-tenant. Visibility rules govern information flow between them.
- **Direct-deployed**: Company is its own tenant. Can later "link" to a PE parent tenant without data migration.
- **RLS on Postgres**: Policies account for operator-gated visibility layer, not just simple tenant isolation.

### Billing Model

Billing entity is decoupled from tenant structure.

| Model | How it works |
|-------|-------------|
| **PE-funded** | PE firm pays for all portfolio companies. Single invoice at parent tenant. |
| **Port co-funded** | Each company pays independently. PE deploys but doesn't pay. |
| **Hybrid** | PE pays platform fee + base coverage. Port cos pay for usage above that, or PE selectively funds specific companies. |
| **Direct** | Company is its own billing entity. No PE parent. |

Billing entity modeled in Organization & Access from day one. Billing infrastructure (Stripe, invoicing) deferred post-MVP.

### AI Workflows

| Workflow | What the LLM Does |
|----------|-------------------|
| **Research synthesis** | Takes raw data from multiple sources, structures into Company Profile artifact |
| **Briefing generation** | Scores signals by relevance to role + company, selects top items, generates actionable summaries |
| **KPI enrichment** | Detects anomalies, generates conversational follow-up questions, attaches context to snapshots |
| **Board narrative drafting** | Assembles quarterly narrative from artifact graph |
| **Triage reasoning** | Assesses sponsor questions — can it answer, is it important, should it escalate. Drafts responses using full context including firewalled info for operator review. |
| **Brainstorm dialogue** | Interactive session with operator for nuanced triage situations or strategic questions |

### Load-Bearing Invariants

- **Grounding validation**: Every AI output cites its source artifact. No uncited claims.
- **Human-in-the-loop**: The system drafts, the human approves. The LLM never publishes directly.
- **Tenant isolation**: Defense-in-depth — RLS + application-level checks + visibility rules.
- **Operator trust**: Nothing flows upstream without operator permission (beyond agreed quantitative metrics).
- **Pure domain layer**: No infrastructure dependencies in domain logic.
- **No cross-context DB queries**: Bounded contexts communicate through defined interfaces.
- **Transactional outbox**: Events propagate reliably; no dual-write.

---

## 7. MVP Scope

### In Scope

1. Onboarding wizard — company setup with live research, user interest profiling, competitor monitoring config, visibility defaults
2. Intelligence Engine (Layers 1 + 2) — monitored source scanning + ad-hoc company research module
3. Daily Briefing — personalized morning brief with Monday weekly variant, forward-looking task list updating through the week
4. Action & Decision logging — lightweight capture from briefing interactions, with owner assignment
5. KPI Management — owner assignment, cadence-based reporting, anomaly detection with conversational enrichment
6. Artifact Store — Company Profiles, Intelligence Signals, Decision Records, Action Items, KPI Snapshots, Briefings, Board Narratives, Triage Records. Cross-referencing between artifacts.
7. Board Prep view — "show me what's accumulated" with system-drafted narrative, operator curation
8. Sponsor Access (read-only + triage) — gated dashboard for PE/operating partners/board, question submission with auto-answer and escalation
9. Triage flow — including brainstorm mode for questions involving firewalled context
10. Multi-tenancy — PE-parent and direct-deploy models, RLS, visibility rules, billing entity modeling (not billing infrastructure)

### Deferred (Post-MVP)

- Contextual intelligence (Layer 3) — signals surfaced in context of current work
- Full billing infrastructure — Stripe, usage metering, invoicing
- Board narrative auto-export to deck format (PDF/PPTX)
- Meeting integration — transcription, action item capture from board meetings
- Cross-portfolio pattern recognition — learning across companies on the platform
- Port co financial system integrations — direct ERP/accounting connectors
- Mobile app — push notifications, on-the-go triage
- WorkOS SSO/SAML/SCIM — enterprise auth

---

## 8. Success Criteria

The platform succeeds when:

- Operators open it as their first touchpoint of the day
- Board prep time drops from weeks to hours
- Sponsors stop emailing operators for information and use the platform instead
- KPI reporting generates insight (via enrichment conversations), not just numbers
- Durable artifacts compound — each quarter's board narrative is easier to produce than the last
- Operating partners show up to calls with context and contribute value
- PE firms deploy it across portfolio companies because the first one demonstrably performed better
