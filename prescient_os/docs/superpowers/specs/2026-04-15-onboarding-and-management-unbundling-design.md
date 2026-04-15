# Onboarding & Management Unbundling Design

**Date:** 2026-04-15
**Status:** Draft
**Scope:** Pipeline-driven onboarding, management unbundling product framework, strategic initiative generation from onboarding, stakeholder viral adoption
**Builds on:** [Research Engine & Smart Onboarding](2026-04-13-research-engine-and-smart-onboarding-design.md), [Strategic Initiatives](2026-04-13-strategic-initiatives-design.md)

## Vision

Prescient OS is a management unbundling platform. PE functionality is the go-to-market wedge — due diligence, data rooms, and reporting create natural data intake channels. The real product is the compounding context layer: every task an operator completes feeds the system, making the next task faster and the platform more intelligent.

The core competitive insight: other platforms ask users to front-load data entry on faith. Prescient OS lets users walk in with a real job to do, gets it done faster, and the platform gets smarter as a side effect.

### The Management Unbundling Framework

Three core management functions (from classical management theory, validated by how companies like Kimi, Block, and Meta are restructuring today):

1. **Routing** — Information logistics. Who needs to know what, when. AI handles this well.
2. **Sensemaking** — Interpreting signal from noise, understanding what data actually means for the business. Deeply human, AI assists.
3. **Accountability** — Ownership, follow-through, feedback. Human function, AI creates structure around it.

The platform maps to all three:
- **Routing**: Briefings, KPI snapshots, board narratives, monitoring alerts — the right information to the right person at the right time.
- **Sensemaking**: Copilot with persistent company context, intelligence signals, triage records — AI helps interpret, humans decide.
- **Accountability**: Strategic initiatives with owners, milestones, proactive agent follow-ups — the system holds people to their commitments.

### The Flywheel

```
User completes a task (board deck, initiative update, triage decision)
  -> Data enters the system (routing map, context, accountability record)
  -> Platform gets smarter (better routing, sharper sensemaking, tighter accountability)
  -> Next task is faster and more informed
  -> User does more in the platform
  -> More data enters...
```

## Onboarding Pipeline

The onboarding experience is the first value delivery. Instead of "fill out your profile so we can help you later," it's "we already started researching your company — here's what we found."

### Pre-condition: Account Creation

Account creation happens before Stage 1 begins. The user signs up (email + password or SSO), their account and company record are created, and only then does the research pipeline kick off. No anonymous access to the onboarding pipeline. Every response, assessment, and artifact is tied to an authenticated user and company from the start.

### Pipeline Stages

Six stages, managed as a state machine:

```
RESEARCH -> DISCOVERY_QA -> AREA_SELECTION -> DEEP_DIVE -> SYNTHESIS -> INITIATIVE_GENERATION
```

**Stage 1 — RESEARCH** (automated, runs on signup)
- Triggers the existing research pipeline for user's company + competitors in parallel.
- Pulls EDGAR, web, news, DNS via existing source adapters (quick pass first, deep pass async).
- Writes results to persisted `OnboardingResearch` records as they arrive.
- Exit condition: research complete, or timeout with partial results. Does not block the user — Stage 2 runs in parallel.

**Stage 2 — DISCOVERY Q&A** (interactive, runs in parallel with Stage 1)
- Lightweight conversational questions, one at a time:
  1. "What are your top priorities right now?"
  2. "Where are you struggling most?"
  3. "Are you using AI tools today? Where do they fall short?"
  4. "What feels like it's going well?"
- Each answer stored as a `DiscoveryResponse` with extracted themes.
- LLM extracts themes/keywords from each answer in the background for matching to research results later.
- The user can skip remaining questions at any time.
- Exit condition: user completes or skips remaining questions.

**Stage 3 — AREA SELECTION** (interactive, waits for Stage 1 to have enough data)
- System merges research findings with discovery themes to present identified business areas (products, business lines, functions).
- User selects which areas to dive into.
- User can add areas the system missed.
- User can bail — skip straight to synthesis with whatever data exists.
- Exit condition: user confirms selection or bails.

**Stage 4 — DEEP DIVE** (interactive, per selected area)
- For each selected area:
  - "What's going well with [area]?"
  - "What needs improvement with [area]?"
  - "Who owns this area?"
  - "Why do you think performance is below expectations?" (if they flagged a weakness)
- User can bail early at any point — completed assessments are kept, remaining areas are skipped.
- Exit condition: user finishes all selected areas or bails.

**Stage 5 — SYNTHESIS** (automated)
- Merges all collected data: research results + discovery responses + area assessments.
- LLM generates a structured company context: strengths, weaknesses, priorities, ownership map, gaps.
- Produces `SynthesisResult` — the foundation for initiative generation and ongoing platform intelligence.
- Exit condition: synthesis complete.

**Stage 6 — INITIATIVE GENERATION** (interactive)
- System proposes strategic initiatives based on synthesis, weighted toward weakness areas and stated priorities.
- Each proposed initiative includes: title, rationale (tied to the user's own words), suggested owner, initial milestones.
- User approves, edits, or rejects each.
- Approved initiatives become real `Initiative` entities in the strategy context.
- Assigned owners receive invitations to the platform.
- Exit condition: user approves at least one initiative or defers.

## Domain Model

### OnboardingSession

The root entity. One active session per company.

- `id` (UUID)
- `company_id` (FK)
- `user_id` (FK)
- `current_stage` (enum: research, discovery_qa, area_selection, deep_dive, synthesis, initiative_generation, completed, abandoned)
- `status` (enum: in_progress, completed, abandoned)
- `started_at` (timestamp)
- `completed_at` (timestamp, nullable)

### OnboardingResearch

One record per source per target company. Populated by the existing research pipeline.

- `id` (UUID)
- `session_id` (FK to OnboardingSession)
- `source` (enum: edgar, web, news, dns, website)
- `company_target` (enum: own_company, competitor)
- `raw_data` (JSONB)
- `status` (enum: pending, complete, failed)
- `completed_at` (timestamp, nullable)

### DiscoveryResponse

Up to 4 per session (one per question).

- `id` (UUID)
- `session_id` (FK to OnboardingSession)
- `question_key` (enum: priorities, struggles, ai_tools, going_well)
- `answer_text` (text)
- `extracted_themes` (JSONB array)
- `answered_at` (timestamp)

### FocusArea

Business areas identified by research or added by the user.

- `id` (UUID)
- `session_id` (FK to OnboardingSession)
- `name` (text)
- `source` (enum: discovered, user_added)
- `selected` (bool)
- `matched_research_ids` (JSONB array of OnboardingResearch IDs)

### AreaAssessment

One per selected FocusArea.

- `id` (UUID)
- `focus_area_id` (FK to FocusArea)
- `strengths` (text)
- `weaknesses` (text)
- `owner_name` (text)
- `owner_email` (text, nullable — for stakeholder invitation)
- `underperformance_reason` (text, nullable)
- `completed` (bool)

### SynthesisResult

One per session. The seed for persistent company context.

- `id` (UUID)
- `session_id` (FK to OnboardingSession)
- `company_context` (JSONB — structured strengths, weaknesses, priorities, ownership map, gaps)
- `generated_at` (timestamp)

### ProposedInitiative

Generated from synthesis. User decides on each.

- `id` (UUID)
- `session_id` (FK to OnboardingSession)
- `synthesis_id` (FK to SynthesisResult)
- `title` (text)
- `rationale` (text)
- `suggested_owner_email` (text)
- `milestones` (JSONB)
- `user_decision` (enum: pending, approved, edited, rejected)

### Entity Relationships

```
User -> OnboardingSession -> OnboardingResearch (many)
                          -> DiscoveryResponse (up to 4)
                          -> FocusArea (many) -> AreaAssessment (one per)
                          -> SynthesisResult (one)
                          -> ProposedInitiative (many)
```

## Service Layer

### OnboardingOrchestrator

Core application service. Manages stage transitions.

- `start_session(user_id, company_id)` — Creates session, dispatches research, returns session.
- `get_current_state(session_id)` — Returns current stage, progress summary, available actions.
- `advance_stage(session_id)` — Validates exit conditions, transitions to next stage.
- `abandon_session(session_id)` — Marks abandoned, keeps all collected data.

### Stage-Specific Services

**ResearchStageService**
- `launch_research(session_id, company_id)` — Dispatches to existing research pipeline via `ResearchPort`. Registers callbacks to update `OnboardingResearch` records as sources complete.
- `get_research_progress(session_id)` — Returns which sources have completed (for frontend progress display).
- `is_sufficient(session_id)` — Returns true when enough data exists to proceed to area selection.

**DiscoveryQAService**
- `get_next_question(session_id)` — Returns next unanswered question or null.
- `record_answer(session_id, question_key, answer_text)` — Stores response, triggers background theme extraction via `ThemeExtractionPort`.
- `skip_remaining(session_id)` — Marks remaining questions as skipped.

**AreaSelectionService**
- `generate_suggested_areas(session_id)` — Merges research findings with discovery themes via `AreaMatchingPort`, produces candidate FocusArea records.
- `select_areas(session_id, selected_ids)` — Marks chosen areas.
- `add_custom_area(session_id, name)` — Creates user_added FocusArea.
- `confirm_selection(session_id)` — Locks in choices, advances stage.

**DeepDiveService**
- `get_next_assessment(session_id)` — Returns next incomplete FocusArea to assess.
- `record_assessment(focus_area_id, strengths, weaknesses, owner_name, owner_email, underperformance_reason)` — Stores AreaAssessment.
- `bail_early(session_id)` — Marks remaining areas as skipped, advances with collected data.

**SynthesisService**
- `generate_synthesis(session_id)` — Gathers all research + responses + assessments, calls `SynthesisPort`, produces SynthesisResult.

**InitiativeGenerationService**
- `propose_initiatives(session_id)` — Reads SynthesisResult, calls `InitiativeProposalPort`, creates ProposedInitiative records.
- `decide_on_initiative(initiative_id, decision, edits)` — Records user decision.
- `promote_approved(session_id)` — Converts approved ProposedInitiatives into real Initiative entities via `StrategyPort`, triggers stakeholder invitations via `InvitationPort`.

### Ports

All LLM calls and cross-context communication go through ports:

| Port | Purpose |
|------|---------|
| `ResearchPort` | Dispatches to existing research pipeline |
| `ThemeExtractionPort` | LLM extracts themes/keywords from free-text answers |
| `AreaMatchingPort` | LLM matches research findings to discovery themes, suggests focus areas |
| `SynthesisPort` | LLM generates structured company context from all collected data |
| `InitiativeProposalPort` | LLM proposes initiatives from synthesis |
| `StrategyPort` | Writes to strategy bounded context (Initiative creation) |
| `InvitationPort` | Triggers user invitations for assigned stakeholders |
| `MonitoringPort` | Registers competitors as watch targets |

## API Layer

All endpoints scoped under `/api/v1/onboarding/`:

```
POST   /sessions                          -> start_session
GET    /sessions/{id}                     -> get_current_state

# Research (Stage 1)
GET    /sessions/{id}/research            -> research progress

# Discovery Q&A (Stage 2)
GET    /sessions/{id}/discovery/next      -> next question
POST   /sessions/{id}/discovery           -> record_answer
POST   /sessions/{id}/discovery/skip      -> skip remaining

# Area Selection (Stage 3)
GET    /sessions/{id}/areas               -> suggested areas
POST   /sessions/{id}/areas/select        -> select areas
POST   /sessions/{id}/areas/add           -> add custom area
POST   /sessions/{id}/areas/confirm       -> confirm selection

# Deep Dive (Stage 4)
GET    /sessions/{id}/assessments/next    -> next area to assess
POST   /sessions/{id}/assessments         -> record assessment
POST   /sessions/{id}/assessments/bail    -> bail early

# Synthesis (Stage 5)
GET    /sessions/{id}/synthesis           -> get synthesis result

# Initiative Generation (Stage 6)
GET    /sessions/{id}/initiatives         -> proposed initiatives
POST   /sessions/{id}/initiatives/{id}    -> approve/edit/reject
POST   /sessions/{id}/initiatives/promote -> promote approved
```

## Frontend Flow

A single onboarding page that transitions through views:

**View 1 — Research + Discovery (Stages 1 & 2 in parallel)**
- Split layout: research progress on one side ("Analyzing SEC filings... Scanning news... Building company profile..."), conversational Q&A on the other.
- Research progress updates via polling.
- User answers discovery questions while research runs — no waiting.
- Transitions when both are done or user skips Q&A.

**View 2 — Area Selection (Stage 3)**
- Cards or list of discovered areas, each with a brief summary from research.
- Checkboxes to select, "Add your own" input at the bottom.
- Visual indicator of which areas matched discovery responses.
- "These look good" / "Skip this" buttons.

**View 3 — Deep Dive (Stage 4)**
- One area at a time, focused view.
- Structured inputs with conversational framing: "Tell us about [Area]."
- Strength/weakness text inputs, owner name/email fields.
- Progress indicator: "Area 2 of 4" with "I'm done, skip the rest" option.
- Each submission saves immediately.

**View 4 — Synthesis Loading (Stage 5)**
- Brief loading state: "Connecting the dots between what we found and what you told us..."
- Transitions automatically when ready.

**View 5 — Initiatives & Next Steps (Stage 6)**
- Cards for each proposed initiative: title, rationale tied to the user's words ("You said GTM was a struggle with AngeionAffirm..."), suggested owner, initial milestones.
- Approve / Edit / Reject per card.
- "Launch these" button promotes approved initiatives and triggers stakeholder invitations.
- Exit from onboarding — user lands on main dashboard with real work in flight.

**Resumability:** `GET /sessions/{id}` returns current stage and progress. Frontend reads this on load and drops the user back into the right view with prior answers intact.

## Cross-Context Integration

All communication through ports. No direct imports.

**Onboarding -> Research (existing)**
- `ResearchPort` dispatches company + competitor research using the existing multi-source pipeline.
- No changes needed to the research pipeline — onboarding is another consumer.
- Research results copied into `OnboardingResearch` records (onboarding owns its own read model).

**Onboarding -> Strategy (from strategic initiatives spec)**
- `StrategyPort` creates `Initiative` entities from approved `ProposedInitiative` records.
- Carries over: title, rationale, owner, milestones, linked FocusArea context.
- Strategy context doesn't know about onboarding.

**Onboarding -> Organizations (existing)**
- `InvitationPort` triggers user invitations for stakeholders identified during deep dive.
- Uses existing org/user infrastructure for invite emails and account provisioning.

**Onboarding -> Intelligence/Copilot (existing)**
- `SynthesisResult` gets published as an extended `COMPANY_PROFILE` artifact. The existing `CompanyProfileBuilder` produces profiles from research data alone; the onboarding synthesis enriches this with the operator's subjective assessment (strengths, weaknesses, ownership map, priorities). Same artifact type, richer content.
- This is the critical handoff — the copilot reads this artifact, not onboarding tables.
- Everything the copilot does after onboarding is informed by this context.

**Onboarding -> Monitoring (existing)**
- Competitors identified during research get registered as watch targets via `MonitoringPort`.
- SEC filing monitoring, news monitoring, URL change detection start automatically.

### Data Lifecycle

```
Onboarding Session (temporary orchestration state)
  -> CompanyContext artifact (permanent, used by copilot)
  -> Initiatives (permanent, live in strategy context)
  -> Watch Targets (permanent, live in monitoring context)
  -> User Invitations (permanent, live in organizations context)
```

The onboarding session can be archived after completion. Its value has been transferred into permanent entities. Keeping it is useful for analytics (drop-off points, question signal quality).

## Stakeholder Onboarding

When a stakeholder accepts an invitation, they do not go through the full 6-stage pipeline. They get a scoped experience:

1. Account creation (same gate — no anonymous access).
2. Context presentation: "Your [COO name] set up an initiative around [AngeionAffirm GTM] and assigned you as owner. Here's the context, here's your first milestone."
3. Validation prompt: "Does this look right? Anything you'd adjust?"
4. Their edits feed back into the initiative and the company context.
5. They're in the platform with real work to do from minute one.

This is the viral adoption mechanism. Each onboarded operator spawns 2-5 new users through task assignments. Each stakeholder who enters through an initiative has immediate context and purpose.

## Management Unbundling — Platform Capability Mapping

### Routing

The platform handles information routing progressively:
- **Onboarding seeds the routing map.** Discovery Q&A + deep dive identifies who owns what and who cares about what.
- **Initiatives define routing paths.** Milestone updates route to initiative owners. Dependency changes propagate downstream.
- **Briefings synthesize and distribute.** Existing briefing system pulls from KPIs, initiatives, monitoring signals and delivers the right summary to the right person.
- **Monitoring catches external signals.** SEC filings, news, website changes get routed to the people who own the affected areas.
- The system improves routing as it learns who acts on what information.

### Sensemaking

The platform assists but does not replace human judgment:
- **Research pipeline provides raw signal.** EDGAR data, news, competitor moves.
- **Copilot helps interpret.** "Your competitor just filed an 8-K about an acquisition. Here's how that might affect your GTM positioning for AngeionAffirm, given what you told us about your market education challenge."
- **SynthesisResult is the persistent context.** The copilot doesn't start from scratch. It knows the company's strengths, weaknesses, who owns what, and what matters.
- **Triage records capture sensemaking decisions.** When an operator decides "this signal matters, we need to act on it," that decision is recorded and feeds future signal prioritization.

### Accountability

The platform creates structure around human ownership:
- **Initiatives have owners and milestones with dates.** The system knows who committed to what by when.
- **Proactive agent follows up.** As due dates approach, the system reaches out to assignees for status updates.
- **Health scores degrade on staleness.** No update as a deadline approaches — the initiative goes yellow, then red.
- **Dependency propagation.** Upstream slip automatically flags downstream risk.
- **Humans make all decisions.** The agent collects status, packages it as an approval request, and the owner decides. No automated scope changes or AI-driven reassignments.

### Compounding Across Functions

Each management function reinforces the others:
- Better routing means the right person sees the right signal.
- Better sensemaking means initiatives are pointed at real problems.
- Better accountability means those initiatives actually get executed.
- Every interaction feeds all three.

## PE Go-to-Market Considerations

For PE-sold deployments, two adjustments:

1. **"Add watched company"** replaces "add competitor" — watched companies may be portfolio companies, deal targets, or competitors. The PE user's relationship to watched companies varies.
2. **Board deck workflow** is a viable starting point when PE mandates adoption. Data flows through compliance, not choice. The onboarding can be streamlined when the PE firm has already provided context about the portfolio company.

For direct-to-company sales, the full onboarding pipeline described here is the primary path. The "aha moment" comes from the system knowing about the company before the user tells it anything, and the transition from research to actionable initiatives in a single session.

## Relationship to Existing Specs

This spec extends and builds on:

- **Research Engine & Smart Onboarding (2026-04-13)**: The research pipeline, source plugins, ResearchTask model, and quick/deep pass architecture are defined there. This spec adds the 6-stage onboarding pipeline on top of that research infrastructure.
- **Strategic Initiatives (2026-04-13)**: The Initiative, Milestone, Dependency, and Evidence models are defined there. This spec adds the onboarding-to-initiative generation flow that feeds into that system.

No conflicts — the research spec defines the engine, this spec defines how onboarding orchestrates it and what it produces.
