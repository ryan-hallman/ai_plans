# Research Engine & Smart Onboarding Design

**Date:** 2026-04-13
**Status:** Draft
**Scope:** Shared research pipeline, smart onboarding flow, chat-based market research tool

## Overview

Two new features that share a common backend engine:

1. **Smart Onboarding** — When a new user signs up, the platform researches their company and person, then conducts a conversational interview to understand their priorities and pain points. Replaces the current static 3-step wizard.
2. **Chat-Based Market Research** — An on-demand research tool accessible from the copilot. Users ask about companies, people, or markets in natural language. Results can be promoted to watched companies and/or saved as artifacts.

Both features are powered by a **Research Pipeline Engine** — a staged, checkpointed, source-pluggable system that orchestrates multi-source intelligence gathering and LLM synthesis.

## Research Pipeline Engine

### Research Subjects

Three subject types:

- **company** — given a name + optional website/CIK, build a company profile
- **person** — given a name + email + optional company, build a person profile
- **market** — given a company or industry, discover the competitive landscape

### Pipeline Stages

Each research request flows through five stages:

1. **Discover** — Resolve which sources are available for this subject (does the company have SEC filings? Is the website scrapable? Are there news hits?).
2. **Fetch** — Hit sources in parallel. Each source plugin returns raw findings.
3. **Extract** — LLM pass to pull structured data from raw findings (key people, products, financials, competitive positioning, recent events).
4. **Synthesize** — LLM merges extracted data across sources into a coherent profile with citations.
5. **Persist** — Store as artifacts and/or update entity profiles in the database.

### Key Properties

- **Checkpointed** — After each stage, intermediate state is saved. If fetch fails for one source, the others still proceed.
- **Progressive** — The quick onboarding pass runs Discover + Fetch (website + search only) + Extract + Synthesize in ~10-15 seconds. The deep async pass adds EDGAR, news, competitor discovery and re-synthesizes.
- **Source-pluggable** — Each source implements a common `ResearchSource` interface. Adding a paid API later means writing one new plugin.
- **Backed by Postgres** — Research tasks stored in a `research_tasks` table with status, stage, subject, findings (JSONB), and result references. No new infrastructure beyond what already exists — a simple async worker polls or gets triggered directly.

### Source Plugin Architecture

#### ResearchSource Interface

Each source implements:

- `can_handle(subject) -> bool` — Can this source research this type of subject?
- `fetch(subject) -> RawFindings` — Fetch data and return structured raw findings.
- `priority` and `timeout` — Controls execution order for quick pass vs. deep pass.

#### Initial Source Plugins (Free/Open)

| Source | Subjects | Quick Pass | Deep Pass | What It Provides |
|--------|----------|:---:|:---:|------------------|
| Website Scraper | company, person | Yes | Yes | About page, products, team, leadership bios |
| EDGAR Client | company | No | Yes | Filings, financials, risk factors (existing code) |
| News Aggregator | company, person, market | No | Yes | Recent coverage, press releases, events |
| Search Engine | company, person, market | Yes | Yes | General web results for discovery and gap-filling |
| Domain/DNS Lookup | company | Yes | No | Tech stack hints, company size signals |

**Quick pass** sources are fast enough (~5-10s) to run during onboarding Phase 1. **Deep pass** adds slower, richer sources for async research.

**Adding paid sources later** (e.g., Crunchbase, Apollo, Clearbit) means writing a new plugin that implements the same interface. The pipeline discovers available sources and runs them — no changes to orchestration logic.

#### Raw Findings Structure

Each source returns typed findings (e.g., `WebsiteFindings`, `EdgarFindings`) that conform to a base with:

- Source name
- Fetch timestamp
- Confidence level
- List of `Finding` objects (text content + URL citation + category tag)

The Extract stage knows how to pull structured fields from each finding type.

### Error Handling & Resilience

- **Sources fail independently.** If the website is unreachable, EDGAR and news still run. The synthesize stage works with whatever findings are available and notes gaps: "Limited information available — no website data could be retrieved."
- **Per-source timeouts.** Quick pass: 10s per source. Deep pass: 30s per source. Timeouts are partial failures, not blockers.
- **Per-source status tracking.** `ResearchTask` records whether each source succeeded, failed, timed out, or was skipped — enables targeted retries.
- **Graceful degradation.** If the quick pass fails entirely (all sources down), onboarding proceeds without pre-populated context. The interview asks a couple more questions to compensate.

## Smart Onboarding Flow

Replaces the current 3-step wizard (company setup → competitors form → interests tags) with a research-powered conversational experience.

### Signup Form

User provides: **email, company name, website, role.** This is the only form. Everything else happens through conversation.

### Phase 1: Quick Research (~10-15 seconds)

Triggered immediately after signup. While the user sees a welcome/loading screen, the pipeline runs a fast pass:

- Scrape the company website (about, products, team pages)
- Search engine lookup on the person (email + name, public info only)
- Resolve industry and basic company profile

This gives the platform enough context to have an informed conversation.

### Phase 2: The Interview (Chat-Based)

The user lands in a copilot conversation. The AI already knows their company and role from Phase 1 and interviews them adaptively.

**Target information to gather:**

1. **Confirm/correct research** — "Here's what I found about your company. Anything I got wrong?" Builds trust and fixes errors early.
2. **Role and focus** — "What does your day-to-day look like?" / "What's your primary focus right now?"
3. **Pain points** — "What keeps you up at night?" / "Where are you spending the most time that you wish you weren't?"
4. **Desired outcomes** — "If this platform could solve one thing for you, what would it be?"
5. **Competitive awareness** — "Who are the competitors you watch most closely? Anyone emerging that concerns you?"

**Interview properties:**

- **Conversational, not form-like.** The AI responds naturally, asks follow-ups, acknowledges what the user says.
- **5-8 questions max.** Respects the user's time. Adapts based on how much the research already answered.
- **Structured extraction.** After the interview, an LLM pass parses the conversation into structured `UserContext` fields.
- **Summary and handoff.** Ends with: "Here's what I've set up for you based on our conversation: [company profile, watched competitors, your priorities]. I'm running deeper research in the background — I'll let you know when it's ready."
- **Immediate value.** After the interview, the user lands in the main app with a personalized experience from minute one.

### Phase 3: Deep Research (Async, Background)

After the interview completes, the pipeline kicks off full research:

- EDGAR filings, news coverage, competitive landscape discovery
- Person background (career history, expertise areas)
- Full profiles on each competitor the user named

Results surface later in-app: "Your company profile is ready for review" / "We found 4 competitors worth watching."

### What Changes from Current Onboarding

- The static competitor form goes away — competitors are either discovered by research or named in conversation.
- The interests step goes away — replaced by the interview capturing richer context.
- The company setup step becomes just the signup form fields.

## Chat-Based Market Research Tool

An on-demand research capability accessible from the copilot. Replaces the current clunky manual competitor-add flow with natural conversation.

### Two Entry Points

1. **Standalone Research Copilot** — Accessible from a top-level route (e.g., `/research`), not scoped to any company. Uses the research pipeline tools but without company context. For open-ended exploration: "What can you tell me about the industrial packaging market?" or "Research Acme Corp for me."

2. **Company-Scoped Copilot** — The existing company chat at `/companies/{slug}/chat` gains research tools alongside its current internal tools. "Who are our main competitors in the Southeast?" or "Research this company I heard about — Packright Industries." Research results are framed relative to the scoped company.

### New Copilot Tools

| Tool | Description |
|------|-------------|
| `research_company` | Triggers the full pipeline for a company. Returns a synthesized profile. Copilot streams progress as findings arrive. |
| `research_market` | Given an industry or company, discovers the competitive landscape. Returns relevant companies with brief profiles. |
| `research_person` | Researches an individual (leadership, board member, etc.). |
| `watch_company` | Promotes a researched company to a watched entity with ongoing monitoring. |
| `save_artifact` | Saves any research output as a persistent artifact linked to the user's company. |

### Conversation Flow Example

```
User: "Tell me about Packright Industries"
→ Copilot calls research_company("Packright Industries")
→ Streams findings as they come in (website → filings → news)
→ Presents a synthesized profile with citations
→ Suggests: "Want me to add Packright as a watched company?
   I can also save this profile as an artifact."

User: "Yeah, watch them. And how do they compare to us?"
→ Copilot calls watch_company → creates monitoring target
→ Uses internal data (user's company) + new research for comparison
→ May suggest: "I could research the broader competitive landscape
   if you want to see who else is in this space."
```

### Promotion Behavior

- **Proactive suggestions** — The copilot suggests watching a company or saving an artifact when it seems appropriate (e.g., after a thorough research result).
- **User-initiated** — The user can ask directly anytime: "watch this company" or "save this as an artifact."
- **Not pushy** — Suggestions are offered once, not repeated if ignored.

### Company-Scoped Context

When research is triggered from within a company copilot, results are automatically framed in terms of relevance to the user's business. The copilot has access to the company's existing profile, KPIs, and watched companies for comparison.

## Data Model

### New Entities

#### ResearchTask

Tracks a pipeline execution. Serves as both work queue and audit trail.

- `id` — UUID
- `subject_type` — enum: company, person, market
- `subject` — JSONB (name, website, email, cik, industry — varies by type)
- `mode` — enum: quick, deep
- `status` — enum: pending, discovering, fetching, extracting, synthesizing, persisting, completed, failed
- `source_results` — JSONB (per-source status: succeeded/failed/timed_out/skipped + raw findings)
- `extracted_data` — JSONB (structured data from extract stage)
- `synthesized_result` — JSONB (final merged profile)
- `result_artifact_id` — FK to artifact (once persisted)
- `triggered_by` — enum: onboarding, copilot, system
- `tenant_id`, `created_by`, `created_at`, `updated_at`

#### UserContext

Captures what the onboarding interview learns about a user. Richer than `InterestProfile` (which is just tags).

- `id` — UUID
- `user_id` — FK to user
- `company_id` — FK to company
- `role_description` — text (what they actually do, beyond job title)
- `primary_focus` — text (current priorities)
- `pain_points` — text[] (what keeps them up at night)
- `desired_outcomes` — text[] (what they want the platform to solve)
- `competitors_mentioned` — JSONB (names + context from conversation)
- `raw_interview_summary` — text (LLM-generated summary of the full interview)
- `created_at`, `updated_at`

#### PersonProfile (Artifact Type)

A new artifact type for person-level research.

- Stored as an `Artifact` with `artifact_type = 'person_profile'`
- Blocks: overview, career history, expertise areas, public appearances/quotes, role context
- Linked to a user entity when the person is a platform user

### Changes to Existing Models

- **CompanySetup** — Simplified. No longer stores a competitors JSONB array. Onboarding creates the company entity and kicks off research tasks. Competitors come from interview or research discovery.
- **InterestProfile** — Still exists for tag-based personalization. `UserContext` becomes the primary source of user intent. Interest tags can be auto-derived from the interview.
- **MonitorTarget** (watched companies) — No structural change, but creation shifts from form-based to copilot-driven via `watch_company` tool.

### Integration with Existing Systems

- Research pipeline produces artifacts through the existing `Artifact` + `ArtifactVersion` system. Profiles, competitive analyses, and market overviews all become artifacts with proper versioning and citations.
- Copilot tools call the pipeline service directly for structured research. Ad-hoc conversational follow-ups use LLM-driven tool execution for questions that don't need the full pipeline.
- Knowledge base and OpenSearch indexing pick up new artifacts automatically — research results become searchable within the platform.

## Scope Summary

| Component | Description |
|-----------|-------------|
| Research Pipeline Engine | Staged, checkpointed, source-pluggable, Postgres-backed |
| Smart Onboarding | Quick research → chat interview → deep async research |
| Chat Research Tool | Standalone + company-scoped, with promote-to-watched/artifact |
| Data Model | ResearchTask, UserContext, PersonProfile, extended CompanyProfile |
| Source Plugins | 5 initial free sources, interface ready for paid upgrades |
| Error Handling | Graceful degradation, per-source resilience, adaptive fallback |
