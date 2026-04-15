# Enriched Seed Data: Project Pulse & Peloton IQ Next

**Date:** 2026-04-13
**Status:** Draft
**Branch:** `feature/operator-first-demo-build-foundation`

## Summary

Enrich the Peloton demo seed data with two fictional strategic initiatives — a wearable device program ("Project Pulse") and an AI scaling/expansion effort ("Peloton IQ Next") — that make the demo feel like a living, operated company with real cross-functional tension, open decisions, and forward-looking challenges grounded in 2026 reality.

## Goals

1. Make the demo environment feel "lived in" with in-flight strategic work, not just historical data
2. Create natural operator tension: open decisions, stakeholder friction, competing priorities
3. Stay grounded in real 2026 events (SCOTUS tariff ruling, FDA wellness policy, Peloton's actual financial state, enterprise AI adoption challenges)
4. Preserve the integrity of real EDGAR financial data — fictional content is clearly internal/projected, never modifying historical actuals
5. Exercise more of the system's features: action items across statuses, decision records with dissent, monitoring findings with urgency, KPI anomalies that demand attention

## Non-Goals

- Adding new companies or portfolio entities
- Modifying existing seed data files (integration happens through new cross-references, not edits to existing YAMLs)
- Creating new auth system users for every persona (stakeholders are string names on action items and artifact blocks, consistent with existing patterns)
- Building new backend features — all data fits existing entity schemas

## Design Constraints

- **Financial separation:** Fictional KPIs are forward-looking projections and internal metrics, tagged distinctly from EDGAR-sourced historical financials. No modification to existing `peloton/kpi_values.yaml` periods or values.
- **Existing patterns:** All new seed data uses the same formats, schemas, and loading patterns as existing seed code (block-based artifacts, string-based persona references, deterministic UUIDs, upsert-on-conflict).
- **Seed module structure:** New data lives in dedicated directories (`seed/peloton/project_pulse/`, `seed/peloton/iq_next/`) with a thin integration layer, per Approach 3. Existing seed files are not modified.

---

## Initiative 1: Project Pulse (Wearable)

### Concept

A fitness/wellness wristband that captures continuous biometric data (heart rate, HRV, skin temperature, SpO2, sleep stages, activity) and feeds it into Peloton IQ for hyper-personalized coaching. Positioned as a "coaching sensor," not a smartwatch competitor.

### Stage: Late Planning / Early Greenlight

- Board approved the initiative ~6 weeks ago, allocated $45M over 18 months
- Small feasibility team has spent ~$3M on market sizing, preliminary industrial design, vendor scouting
- Big spend decisions (tooling, manufacturing partnerships, component contracts) are the next gate
- Target launch: Q4 2027 holiday season
- The project is politically fragile — Peloton's stock is at ~$4, the CFO just left, and the Cross Training Series underperformed. This is a diversification bet, not a growth-from-strength play.

### 2026 Context Grounding

**Tariffs:**
- SCOTUS struck down IEEPA tariffs in February 2026 (Learning Resources v. Trump, 6-3)
- Administration reimposing via Section 301/232/122 — rates and timing uncertain
- China still faces 20% fentanyl-related tariff on electronics even post-SCOTUS
- Vietnam: 20% base tariff, 40% on goods flagged as Chinese transshipment; CBP actively enforcing
- US assembly: feasible but adds 30-40% to unit cost, labor/skills gaps, no existing Peloton facility
- Net: maximum uncertainty — nobody knows what tariff rates will be in 6 months

**FDA:**
- January 2026: FDA relaxed General Wellness Policy for low-risk wearables
- Noninvasive fitness tracking (HR, HRV, sleep, activity) — no premarket review needed
- Health-grade claims (blood pressure, SpO2 for medical use) — full FDA oversight required
- WHOOP received FDA Warning Letter in mid-2025 for "medical grade" blood pressure claims
- Net: wellness-only path is fast and clear; health-grade is differentiating but adds 12+ months of compliance

**Peloton financials:**
- Q2 FY2026: Revenue $657M, down 3% YoY, missed guidance
- Connected Fitness subscribers: 2.66M, down 7% YoY
- Stock at ~$4, down 25% post-earnings
- CFO Liz Coddington departed; interim/new CFO in place
- Bright spot: adjusted EBITDA up 39%, net debt down 52%
- Cross Training Series attach rate below projections

### Internal Tension

| Stakeholder | Position |
|-------------|----------|
| Maria Santos (VP Hardware) | Go bold: health-grade sensors, Vietnam assembly. This is Peloton's chance to own the hardware stack. |
| David Park (Interim CFO) | Minimize exposure. Shenzhen is cheapest, but defer big commitments until tariff clarity. References 2022-23 cash crisis. |
| Raj Patel (CPO) | The wearable is existential — it's the data source that makes IQ dominant. Fund it. |
| Tom Brennan (Supply Chain) | Modeling 4 scenarios post-SCOTUS. Advocates Vietnam hybrid but warns about transshipment enforcement. Neither camp likes his answer. |
| Priya Sharma (Data & Privacy) | Stay in wellness lane per new FDA guidance. Health-grade claims would be a compliance nightmare (cites WHOOP letter). Data governance framework must exist before biometric collection begins. |
| James Liu (Head AI/ML) | Impatient — the wearable solves the data quality problem holding IQ back, but it's 18+ months away. Pushing for deeper Apple Health/Garmin integration as a bridge. |

---

## Initiative 2: Peloton IQ Next (AI)

### Concept

Peloton IQ launched October 2025 on Amazon Bedrock. The fictional layer covers the internal operational challenges of scaling it and the strategic debates about what comes next. Two tracks: scaling pains (operational) and strategic expansion (forward-looking).

### 2026 Context Grounding

**Enterprise AI reality:**
- 79% of organizations face AI adoption challenges (Deloitte State of AI 2026) — up double digits from 2025
- 56% of CEOs report getting "nothing" from AI investments (PwC Global CEO Survey 2026)
- Only 29% see significant ROI from generative AI
- 54% of C-suite executives say AI adoption is "tearing their company apart"
- Microsoft reorganized Copilot over quality concerns and "AI slop" — the problem is industry-wide
- The trust gap: employees trust AI-driven data but lack literacy to question it

**Peloton IQ specifics:**
- Runs on Amazon Bedrock using GPT models (complex reasoning) and Llama 4 Scout (real-time insights)
- Generates millions of personalized insights weekly
- Member reception is mixed — power users love it, casual users find recommendations generic
- Cross Training Series (which IQ was supposed to drive) underperformed sales expectations
- CEO Peter Stern has framed AI as "the new era" — the company is publicly committed

### Track A: Scaling Pains

- **Cost pressure:** Bedrock inference costs growing faster than subscription revenue uplift. Interim CFO wants ROI proof before approving next phase.
- **Quality concerns:** Member feedback is mixed. Casual users report generic recommendations. Content team calls it "AI slop" and worries it's diluting the instructor-led brand. Instructors feel threatened.
- **Build vs. buy:** ML team wants to fine-tune proprietary models on Peloton's workout data for quality improvement. Leadership split on cost/defensibility trade-off.
- **Data gap:** IQ only sees workout data plus whatever members share from third-party wearables. Recommendations plateau without richer signals (sleep, recovery, nutrition). This is the business case for Project Pulse — but the wearable is 18+ months away.

### Track B: Strategic Expansion

- **AI-generated content:** Can AI produce supplemental workouts (warm-ups, cool-downs, meditation scripts) at scale? Content team fiercely opposed — brand poison. Legal flagging IP issues with AI-generated content using instructor styles.
- **Conversational coaching:** Chatbot prototype exists (built by Alex Kim). Members could ask "what should I do today?" or "my knee hurts." Medical/legal terrified of liability.
- **B2B corporate wellness:** Packaging IQ for enterprise wellness programs. Revenue diversification lifeline. Requires different data model and privacy framework.
- **Wearable data bridge:** AI team can't wait 18 months for Pulse. Pushing for deeper Apple Health/Garmin integration now, which the Pulse team sees as undermining their business case.

### Internal Tension

| Stakeholder | Position |
|-------------|----------|
| Raj Patel (CPO) | AI is survival. The subscriber decline proves we need better personalization, not less. Double down. |
| James Liu (Head AI/ML) | The quality problems are solvable with better data and fine-tuned models. Give me budget and Pulse data. |
| Nicole Washington (VP Content) | Coined "robot trainer problem." Points to subscriber decline as evidence AI is pushing people away. AI-generated content is a slippery slope. |
| Rachel Torres (Brand Marketing) | Armed with member complaint data and industry AI backlash. Voice of the customer. Aligned with Nicole. |
| David Park (Interim CFO) | If 56% of CEOs say AI delivers nothing, why are we spending more? Show me unit economics. |
| Alex Kim (Sr ML Engineer) | Built the coaching prototype. Pragmatic — knows AI limits are real but they're engineering problems, not fundamental. Frustrated that non-technical leaders make technical judgments. |
| Priya Sharma (Data & Privacy) | Data governance framework must exist before expanding biometric data collection. Blocking both Pulse and IQ expansion until it's done. |

---

## Personas (10 Stakeholders)

All personas are represented as string names in `owner_name` / `owner_role` fields on action items, and as attributed text within artifact blocks. This is consistent with the existing seed pattern — personas are not auth system users (except Sarah Chen who is the demo login).

| # | Name | Title | Key Trait |
|---|------|-------|-----------|
| 1 | Sarah Chen | CEO | Existing demo user. Needs the team to converge. Her briefings surface the tensions. |
| 2 | David Park | Interim CFO | New to the role (post-Coddington). Skeptical of capital commitments. Cites 2022-23 cash crisis. |
| 3 | Raj Patel | Chief Product Officer | AI champion. Believes wearable + IQ flywheel is the survival path. |
| 4 | Maria Santos | VP Hardware Engineering | Owns Project Pulse. Wants bold path: health-grade sensors, Vietnam assembly. |
| 5 | James Liu | Head of AI/ML | Built IQ pipeline. Wants proprietary models. Says data quality is the bottleneck Pulse solves. |
| 6 | Nicole Washington | VP Content & Programming | AI skeptic. Coined "robot trainer problem." Points to subscriber decline as evidence. |
| 7 | Tom Brennan | Director of Supply Chain | Modeling post-SCOTUS tariff scenarios. Advocates Vietnam hybrid with caveats. |
| 8 | Priya Sharma | Head of Data & Privacy | Pushing data governance framework. FDA wellness path is favorable but warns against health claims. |
| 9 | Alex Kim | Senior ML Engineer | Built conversational coaching prototype. Pragmatic about AI limits. Frustrated by non-technical skepticism. |
| 10 | Rachel Torres | Brand Marketing Director | Member complaint data, industry AI backlash. Voice of the customer in internal debates. |

---

## Seed Data Entities

### Seed Module Structure

```
seed/peloton/project_pulse/
├── __init__.py
├── artifacts.py          # Product brief, manufacturing analysis, regulatory strategy
├── action_items.py       # Pulse-related action items
├── decisions.py          # Open and decided decision records
├── kpis.py               # Forward-looking projected KPIs
└── findings.py           # Tariff, FDA, competitive monitoring findings

seed/peloton/iq_next/
├── __init__.py
├── artifacts.py          # State of AI report, content impact assessment, B2B brief
├── action_items.py       # IQ-related action items
├── decisions.py          # AI content, coaching beta, model strategy decisions
├── kpis.py               # IQ operational metrics (Bedrock cost, engagement, NPS)
└── findings.py           # AI industry, member sentiment monitoring findings

seed/peloton/integration.py  # Cross-references between new and existing data
```

A new loader function in `loader.py` calls the initiative modules after the existing seed sequence completes. This keeps the existing seed path untouched.

### Artifacts (6 new)

All artifacts follow the existing pattern: `artifacts.artifacts` + `artifacts.artifact_versions` with block-based content, linked to the hero company via `artifacts.artifact_subjects`.

**Project Pulse artifacts:**

1. **Project Pulse — Product Brief** (type: `product_brief`)
   - Key: `pulse-product-brief`
   - Authors attributed: Maria Santos, Raj Patel
   - Blocks: vision, target_specs (sensor suite, battery life, form factor options), competitive_landscape (Apple Watch, WHOOP, Oura Ring, Garmin Venu), target_market, launch_timeline, budget_summary, risks
   - Tone: confident but acknowledging constraints. This is the document Maria uses to rally support.

2. **Project Pulse — Manufacturing & Tariff Analysis** (type: `cost_analysis`)
   - Key: `pulse-manufacturing-analysis`
   - Author attributed: Tom Brennan
   - Blocks: executive_summary, scenario_a (Shenzhen — 20% fentanyl tariff, lowest cost, highest tariff risk), scenario_b (Vietnam assembly — 20-40% tariff, transshipment enforcement risk, CBP flagging cases), scenario_c (Vietnam assembly + diversified Asian components — hedged but complex), scenario_d (Austin US assembly — 30-40% cost premium, labor gaps, no existing facility), post_scotus_context (IEEPA struck down, Section 301/232 replacement timeline uncertain), recommendation (Vietnam hybrid with contingency planning, neither camp fully satisfied)
   - Tone: analytical, scenario-based, deliberately non-committal because the tariff picture is genuinely uncertain.

3. **Project Pulse — Regulatory & Data Strategy** (type: `regulatory_analysis`)
   - Key: `pulse-regulatory-strategy`
   - Author attributed: Priya Sharma
   - Blocks: fda_landscape (2026 General Wellness Policy relaxation), wellness_pathway (no premarket review for HR/HRV/sleep/activity — fast, low risk), health_grade_pathway (SpO2/blood pressure claims trigger full FDA oversight — differentiating but 12+ month delay), whoop_precedent (Warning Letter for "medical grade" claims — cautionary tale), data_governance (biometric collection requirements, consent framework, HIPAA-adjacency analysis), recommendation (wellness-only for v1, health-grade as v2 stretch goal after regulatory groundwork)
   - Tone: cautious, risk-aware, blocking until governance framework exists.

**Peloton IQ Next artifacts:**

4. **Peloton IQ — State of AI Report** (type: `strategic_analysis`)
   - Key: `iq-state-of-ai-report`
   - Author attributed: James Liu
   - Blocks: executive_summary, bedrock_cost_trajectory (inference costs up 23% QoQ, cost per member rising as subscriber count declines), model_performance (off-the-shelf Bedrock vs. fine-tuned prototype benchmarks on recommendation relevance), member_satisfaction (NPS delta for IQ users vs. non-users — positive but narrowing), data_quality_gap (IQ only sees workout data + opt-in wearable data — recommendations plateau without sleep/recovery/nutrition signals), build_vs_buy (cost/timeline/defensibility analysis of proprietary fine-tuning vs. Bedrock), recommendation (invest in fine-tuning with 6-month pilot, bridge data gap with deeper Apple Health/Garmin integration while Pulse develops)
   - Tone: technical, data-driven, advocacy for investment. James is making his case.

5. **Peloton IQ — Content & Brand Impact Assessment** (type: `brand_analysis`)
   - Key: `iq-brand-impact-assessment`
   - Authors attributed: Nicole Washington, Rachel Torres
   - Blocks: executive_summary, member_sentiment (curated member feedback — "robot trainer" complaints, forum posts, social media), instructor_survey (results from internal instructor sentiment poll — majority concerned about AI replacing their role), recommendation_quality_audit (examples of generic vs. good IQ recommendations, pattern analysis of when IQ works and when it doesn't), industry_context (Microsoft Copilot reorganization, PwC CEO survey, Deloitte enterprise AI report — this isn't just a Peloton problem), brand_risk (Peloton's brand was built on human connection and world-class instructors — AI-forward messaging risks alienating the core community), recommendation (slow AI content expansion, invest in instructor-AI collaboration rather than replacement, fix recommendation quality before expanding scope)
   - Tone: passionate, data-backed, protective of the brand. This is the counterargument to Raj and James.

6. **AI Strategy — Corporate Wellness Expansion Brief** (type: `business_case`)
   - Key: `iq-corporate-wellness-brief`
   - Author attributed: Raj Patel
   - Blocks: opportunity (B2B corporate wellness market sizing, pilot demand signal), revenue_case (diversification away from declining consumer hardware, subscription revenue uplift), data_model_requirements (multi-tenant enterprise data isolation, SSO, admin dashboards), privacy_framework (enterprise data handling, HIPAA considerations for employer wellness programs), pilot_proposal (3 target enterprise accounts, 90-day pilot, success metrics), dependencies (Priya's data governance framework must be complete, interim CFO must approve limited budget)
   - Tone: optimistic but constrained. This is the most politically viable expansion path.

### KPIs (new definitions + projected values)

New KPI definitions added to `kpi_definitions.yaml` (or a parallel file in the initiative module):

**Project Pulse (all forward-looking/internal):**

| KPI ID | Label | Unit | Source |
|--------|-------|------|--------|
| `pulse_feasibility_spend` | Pulse feasibility spend (actual) | USD_millions | Internal finance |
| `pulse_budget_total` | Pulse total program budget | USD_millions | Board allocation |
| `pulse_budget_committed` | Pulse committed spend | USD_millions | Internal finance |
| `pulse_unit_cost_shenzhen` | Pulse projected unit cost — Shenzhen | USD | Manufacturing analysis |
| `pulse_unit_cost_vietnam` | Pulse projected unit cost — Vietnam | USD | Manufacturing analysis |
| `pulse_unit_cost_us` | Pulse projected unit cost — US assembly | USD | Manufacturing analysis |

**Peloton IQ (operational metrics):**

| KPI ID | Label | Unit | Source |
|--------|-------|------|--------|
| `iq_bedrock_monthly_cost` | IQ Bedrock inference cost (monthly) | USD_thousands | AWS billing |
| `iq_recommendation_engagement` | IQ recommendation engagement rate | percent | Product analytics |
| `iq_member_nps_delta` | IQ member NPS delta (IQ users vs. non) | points | Member surveys |
| `iq_ai_content_volume` | AI-generated content volume (monthly) | count | Content ops |
| `iq_human_content_volume` | Human-produced content volume (monthly) | count | Content ops |
| `wellness_pipeline_value` | Corporate wellness pipeline value | USD_thousands | CRM |
| `wellness_pilot_conversions` | Corporate wellness pilot conversions | count | Sales |

Values are seeded for 2-3 recent periods to show trends (e.g., Bedrock cost rising, engagement rate declining).

### Action Items (13 new)

All action items follow the existing schema: `action_items.action_items` with `owner_name`, `owner_role`, `title`, `description`, `priority`, `status`, `due_date`, and optional `linked_focus_priorities`.

**Project Pulse (5):**

| Key | Title | Owner | Priority | Status |
|-----|-------|-------|----------|--------|
| `pulse-sensor-shortlist` | Finalize sensor suite vendor shortlist | Maria Santos, VP Hardware | high | pending_review |
| `pulse-tariff-update` | Complete post-SCOTUS tariff scenario update | Tom Brennan, Dir Supply Chain | high | sent_to_owner |
| `pulse-budget-gate` | Present Pulse budget gate review to board | David Park, Interim CFO | high | pending_review |
| `pulse-fda-recommendation` | Deliver FDA pathway recommendation | Priya Sharma, Head Data & Privacy | medium | sent_to_owner |
| `pulse-austin-feasibility` | Evaluate Austin facility for US assembly | Maria Santos, VP Hardware | low | deferred |

**Peloton IQ Next (8):**

| Key | Title | Owner | Priority | Status |
|-----|-------|-------|----------|--------|
| `iq-build-vs-buy` | Deliver build-vs-buy model comparison | James Liu, Head AI/ML | high | pending_review |
| `iq-coaching-demo` | Demo conversational coaching prototype | Alex Kim, Sr ML Engineer | medium | completed |
| `iq-instructor-survey` | Complete instructor sentiment survey on AI | Nicole Washington, VP Content | medium | completed |
| `iq-member-feedback` | Compile member feedback report on IQ quality | Rachel Torres, Brand Marketing Dir | high | sent_to_owner |
| `iq-wellness-pilot` | Draft corporate wellness pilot proposal | Raj Patel, CPO | medium | pending_review |
| `iq-data-governance` | Publish data governance framework for biometrics | Priya Sharma, Head Data & Privacy | high | sent_to_owner |
| `iq-finetune-scope` | Scope proprietary model fine-tuning requirements | James Liu, Head AI/ML | medium | pending_review |
| `iq-bridge-integration` | Evaluate deeper Apple Health/Garmin integration | Alex Kim, Sr ML Engineer | medium | sent_to_owner |

### Decision Records (6 new)

All decision records follow the existing pattern: `artifacts.artifacts` (type=`decision_record`) with blocks for context, decision, rationale, and dissent.

| Key | Title | Status | Advocates | Dissenters |
|-----|-------|--------|-----------|------------|
| `pulse-manufacturing` | Manufacturing approach for Pulse | OPEN | Maria Santos (Vietnam hybrid) | David Park (defer until clarity) |
| `pulse-fda-pathway` | FDA regulatory pathway for Pulse | OPEN | Raj Patel (health-grade), Priya Sharma (wellness-only) | — |
| `iq-model-strategy` | IQ model strategy: Bedrock vs. fine-tuning | OPEN | James Liu, Raj Patel (fine-tune) | David Park (ROI first), Nicole Washington (wrong problem) |
| `iq-ai-content` | AI-generated supplemental content: approve pilot | DECIDED: NO | Raj Patel (dissented) | Nicole Washington, Rachel Torres (blocked) |
| `iq-wellness-pilot` | Corporate wellness pilot: approve 3 accounts | DECIDED: YES (conditional) | Raj Patel, David Park (limited budget) | Priya Sharma (governance first — condition) |
| `iq-coaching-beta` | Conversational coaching: proceed to member beta | OPEN | Alex Kim (prototype ready), Raj Patel | Medical/legal (liability), David Park (budget) |

### Monitoring Findings (10 new)

All findings follow the existing pattern: `monitoring.monitor_findings` linked to existing monitor sources and targets.

**Tariff & Manufacturing:**

| Key | Type | Title | Severity |
|-----|------|-------|----------|
| `scotus-ieepa-ruling` | regulatory | SCOTUS strikes down IEEPA tariffs in 6-3 ruling | critical |
| `section-301-replacement` | regulatory | Administration pursuing Section 301/232 replacement tariffs — timeline uncertain | important |
| `cbp-transshipment-crackdown` | regulatory | CBP flags 3 electronics importers for Vietnam transshipment tariff evasion | important |
| `whoop-fda-warning` | regulatory | WHOOP receives FDA Warning Letter for "medical grade" blood pressure claims | important |

**Competitive:**

| Key | Type | Title | Severity |
|-----|------|-------|----------|
| `apple-watch-hrv` | product_launch | Apple Watch adds continuous HRV and recovery scoring in watchOS 13 | important |
| `oura-gen4-launch` | product_launch | Oura Ring Gen 4 launches with AI-powered health insights | notable |

**AI Industry:**

| Key | Type | Title | Severity |
|-----|------|-------|----------|
| `msft-copilot-reorg` | industry_trend | Microsoft reorganizes Copilot division over AI quality and "slop" concerns | notable |
| `pwc-ceo-survey-ai` | industry_trend | PwC CEO Survey: 56% report "nothing" from AI investments | notable |
| `deloitte-ai-adoption` | industry_trend | Deloitte: 79% of enterprises face AI adoption challenges, up from 2025 | notable |

**Member Sentiment:**

| Key | Type | Title | Severity |
|-----|------|-------|----------|
| `member-forum-viral` | customer_signal | Peloton member forum thread goes viral: "I cancelled because the AI recommendations are garbage" | important |

### KPI Anomalies (4 new)

All anomalies follow the existing pattern: `kpis.kpi_anomalies` linked to specific KPI values with enrichment questions.

| Key | KPI | Signal | Enrichment Questions |
|-----|-----|--------|---------------------|
| `iq-engagement-drop` | `iq_recommendation_engagement` | Dropped 12% MoM | Seasonal? Quality regression? Feature fatigue? |
| `iq-cost-per-member` | `iq_bedrock_monthly_cost` | Up 23% QoQ while subscribers declined | Cost per member rising — sustainable? Build-vs-buy implications? |
| `cross-training-attach` | (existing HW metric) | Cross Training Series attach rate below projections | Is the IQ value prop not compelling enough to drive hardware upgrades? |
| `wellness-pipeline-spike` | `wellness_pipeline_value` | Pipeline 4x quarter-over-quarter | Is B2B the real growth path? Inbound vs. outbound breakdown? |

### Integration Touchpoints

These are cross-references seeded by `seed/peloton/integration.py` that connect new data to existing entities without modifying existing seed files:

1. **Existing anomaly: negative Connected Fitness gross margin** → New context: early Pulse feasibility costs (~$3M) are a contributing factor alongside existing hardware margin pressure.

2. **Existing anomaly: FCF improvement despite revenue decline** → New context: connects to the Pulse budget question — can they fund a $45M program while maintaining cash discipline that drove FCF improvement?

3. **Existing decision: workforce reduction** → New context: the layoffs freed budget that partly funds Pulse and IQ Next R&D.

4. **Existing anomaly: subscriber churn spike** → New context: ties to IQ quality concerns — did the AI-forward push and subscription price increase (to fund IQ) accelerate churn?

5. **Existing decision: hardware price cuts** → New context: Pulse pricing strategy must account for lessons learned from prior hardware pricing mistakes.

Integration is implemented as:
- **New observation records** in `intelligence.observations` that reference existing KPI anomaly keys and add initiative context (e.g., an observation noting that the negative CF gross margin is partly attributable to Pulse feasibility costs)
- **New artifact blocks** in a dedicated "cross-reference" artifact (type: `initiative_context`) that maps existing decisions/anomalies to new initiative implications
- **`linked_focus_priorities`** on new action items that reference existing strategic focus priority titles, connecting new work to the existing strategic framework

Existing seed files and database rows are not modified.

---

## Seed Execution

### Load Order

New initiative modules execute after the existing seed sequence completes:

```
[existing seed sequence unchanged]
  ↓
seed_project_pulse()
  1. Artifacts (product brief, manufacturing analysis, regulatory strategy)
  2. KPI definitions + projected values
  3. Action items
  4. Decision records
  5. Monitoring findings
  6. KPI anomalies
  ↓
seed_iq_next()
  1. Artifacts (state of AI, brand impact, corporate wellness)
  2. KPI definitions + operational metrics
  3. Action items
  4. Decision records
  5. Monitoring findings
  6. KPI anomalies
  ↓
seed_initiative_integration()
  1. Cross-reference observations linking new → existing entities
  2. Additional artifact references
```

### Idempotency

All new entities use deterministic UUIDs via `uuid5(NAMESPACE, key)` with keys prefixed by initiative (e.g., `pulse-product-brief`, `iq-state-of-ai-report`). All inserts use `on_conflict_do_nothing()` or `on_conflict_do_update()` for safe re-runs.

### Toggleability

The initiative modules are called from a new function in `loader.py` that can be toggled via a `--skip-initiatives` flag or `SEED_SKIP_INITIATIVES=1` env var. This allows seeding the base demo without the enriched data if needed.

---

## Content Guidelines

### Voice & Tone

Artifact content should read like real internal documents:
- Product briefs are confident but acknowledge constraints
- Cost analyses are scenario-based and deliberately non-committal where uncertainty exists
- The AI skeptic artifacts (Nicole, Rachel) are passionate and data-backed, not luddite
- The AI advocate artifacts (James, Raj) are technically rigorous, not hype-driven
- Decision records capture genuine disagreement with attributed positions
- Action items have specific, actionable descriptions — not vague "look into X"

### Realism Checks

All seed content should be validated against:
- Peloton's actual 2026 financial state (Q2 FY2026 earnings)
- Real tariff landscape post-SCOTUS ruling
- FDA 2026 General Wellness Policy
- Enterprise AI adoption data (Deloitte, PwC, industry reports)
- Competitive landscape (Apple Watch, WHOOP, Oura, Garmin)

### What's Fictional vs. Real

| Fictional | Real |
|-----------|------|
| Project Pulse (wearable product) | Peloton's financial results and stock price |
| Named personas (except Sarah Chen) | SCOTUS IEEPA ruling (Feb 2026) |
| Internal debates and positions | FDA 2026 General Wellness Policy |
| Projected KPI values | WHOOP FDA Warning Letter |
| Specific Bedrock cost figures | Enterprise AI adoption statistics |
| Member forum posts | Apple Watch / Oura / Garmin product features |
| Decision record outcomes | Microsoft Copilot reorganization |
| Action item statuses | Vietnam/China manufacturing dynamics |

---

## Entity Counts

| Entity Type | Existing | New | Total |
|-------------|----------|-----|-------|
| Personas (string-based) | ~5 generic roles | 10 named | 10 named + existing |
| Artifacts | 8 | 6 | 14 |
| KPI definitions | ~12 | ~13 | ~25 |
| Action items | 20 | 13 | 33 |
| Decision records | 8 | 6 | 14 |
| Monitoring findings | 15 | 10 | 25 |
| KPI anomalies | 5 | 4 | 9 |

---

## Sources Referenced

- [SCOTUS IEEPA ruling (Learning Resources v. Trump)](https://www.scotusblog.com/2026/02/supreme-court-strikes-down-tariffs/)
- [Tax Foundation Tariff Tracker 2026](https://taxfoundation.org/research/all/federal/trump-tariffs-trade-war/)
- [Peloton Q2 FY2026 Earnings](https://investor.onepeloton.com/news-releases/news-release-details/peloton-announces-q2-fy2026-financial-results/)
- [Peloton Stock / CFO Departure](https://www.indexbox.io/blog/peloton-stock-falls-below-5-cfo-departs-amid-q2-2026-earnings-miss/)
- [FDA 2026 General Wellness Policy](https://natlawreview.com/article/fdas-2026-general-wellness-policy-and-what-it-means-manufacturers-wearable-devices)
- [WHOOP FDA Warning Letter](https://hackaday.com/2026/02/16/what-the-fdas-2026-wellness-device-update-means-for-wearables/)
- [Enterprise AI Adoption 2026 (Deloitte)](https://www.deloitte.com/us/en/what-we-do/capabilities/applied-artificial-intelligence/content/state-of-ai-in-the-enterprise.html)
- [Enterprise AI Adoption 2026 (Writer/79%)](https://writer.com/blog/enterprise-ai-adoption-2026/)
- [Peloton IQ on Amazon Bedrock](https://aws.amazon.com/blogs/industries/peloton-iq-how-peloton-generates-millions-of-personalized-fitness-insights-weekly-using-amazon-bedrock/)
- [Peloton AI Principles](https://www.onepeloton.com/ai-principles)
- [Vietnam Manufacturing in Trade War 2.0](https://www.bloomberg.com/graphics/2026-vietnam-trump-tariffs-supply-chain/)
- [Peloton Cross Training Series Launch](https://investor.onepeloton.com/news-releases/news-release-details/peloton-enters-new-era-ai-powered-peloton-iq-and-new-product/)
