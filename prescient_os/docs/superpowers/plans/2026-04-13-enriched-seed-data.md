# Enriched Seed Data Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add two fictional strategic initiatives (Project Pulse wearable + Peloton IQ Next) to the Peloton demo seed data, creating a richer operator experience with cross-functional tension, open decisions, and forward-looking challenges grounded in 2026 reality.

**Architecture:** New seed data lives in two initiative modules (`seed/peloton/project_pulse/`, `seed/peloton/iq_next/`) plus a thin integration module (`seed/peloton/integration.py`). Each module follows existing seed patterns: deterministic UUIDs via `uuid5`, `pg_insert` with `on_conflict_do_update`, and the same table imports used by `scenarios.py` (for artifacts/decisions/action items) and `focus.py` (for monitoring findings/observations). A new loader function in `loader.py` calls the initiative modules after the existing seed sequence.

**Tech Stack:** Python 3.12, SQLAlchemy 2.0 async, PostgreSQL, existing `prescient.*` table definitions.

**Worktree:** All work happens in `/home/rhallman/Projects/prescient_os/.worktrees/enriched-seed-data/` on branch `feature/enriched-seed-data`.

**Spec:** `docs/superpowers/specs/2026-04-13-enriched-seed-data-design.md`

---

## File Structure

```
seed/peloton/
├── __init__.py                     # (exists, empty or minimal)
├── kpi_values.yaml                 # (exists, unchanged)
├── project_pulse/
│   ├── __init__.py                 # Exports seed_project_pulse()
│   ├── artifacts.py                # 3 artifacts: product brief, manufacturing analysis, regulatory strategy
│   ├── decisions.py                # 3 decision records: manufacturing, FDA pathway, (shared) coaching beta
│   ├── action_items.py             # 5 action items
│   ├── kpis.py                     # KPI definitions + projected values YAML + seeding
│   ├── kpi_values.yaml             # Forward-looking KPI values
│   ├── findings.py                 # 4 monitoring findings (tariff, FDA, competitive)
│   └── anomalies.py                # 2 KPI anomalies
├── iq_next/
│   ├── __init__.py                 # Exports seed_iq_next()
│   ├── artifacts.py                # 3 artifacts: state of AI, brand impact, corporate wellness
│   ├── decisions.py                # 3 decision records: model strategy, AI content, wellness pilot
│   ├── action_items.py             # 8 action items
│   ├── kpis.py                     # KPI definitions + operational metrics
│   ├── kpi_values.yaml             # IQ operational KPI values
│   ├── findings.py                 # 6 monitoring findings (AI industry, member sentiment, competitive)
│   └── anomalies.py                # 2 KPI anomalies
└── integration.py                  # Cross-references between new and existing data

seed/loader.py                      # Modified: add seed_initiatives() call at end of _run()
```

---

### Task 1: Scaffold Initiative Modules

Create the directory structure and package init files so subsequent tasks can import from them.

**Files:**
- Create: `seed/peloton/project_pulse/__init__.py`
- Create: `seed/peloton/iq_next/__init__.py`
- Check: `seed/peloton/__init__.py` (create if missing)

- [ ] **Step 1: Verify seed/peloton/__init__.py exists**

Run: `ls -la /home/rhallman/Projects/prescient_os/.worktrees/enriched-seed-data/seed/peloton/__init__.py`

If it doesn't exist, create it as an empty file.

- [ ] **Step 2: Create project_pulse package**

Create `seed/peloton/project_pulse/__init__.py`:

```python
"""Project Pulse — Peloton wearable initiative seed data."""

from __future__ import annotations


async def seed_project_pulse(
    session: "AsyncSession",
    *,
    bundle: "SeedBundle",
) -> None:
    """Seed all Project Pulse entities. Called after the base seed completes."""
    from seed.peloton.project_pulse.artifacts import seed_pulse_artifacts
    from seed.peloton.project_pulse.decisions import seed_pulse_decisions
    from seed.peloton.project_pulse.action_items import seed_pulse_action_items
    from seed.peloton.project_pulse.kpis import seed_pulse_kpis
    from seed.peloton.project_pulse.findings import seed_pulse_findings
    from seed.peloton.project_pulse.anomalies import seed_pulse_anomalies

    await seed_pulse_artifacts(session, bundle=bundle)
    await seed_pulse_kpis(session, bundle=bundle)
    await seed_pulse_decisions(session, bundle=bundle)
    await seed_pulse_action_items(session, bundle=bundle)
    await seed_pulse_findings(session, bundle=bundle)
    await seed_pulse_anomalies(session, bundle=bundle)
```

- [ ] **Step 3: Create iq_next package**

Create `seed/peloton/iq_next/__init__.py`:

```python
"""Peloton IQ Next — AI scaling and expansion initiative seed data."""

from __future__ import annotations


async def seed_iq_next(
    session: "AsyncSession",
    *,
    bundle: "SeedBundle",
) -> None:
    """Seed all Peloton IQ Next entities. Called after the base seed completes."""
    from seed.peloton.iq_next.artifacts import seed_iq_artifacts
    from seed.peloton.iq_next.decisions import seed_iq_decisions
    from seed.peloton.iq_next.action_items import seed_iq_action_items
    from seed.peloton.iq_next.kpis import seed_iq_kpis
    from seed.peloton.iq_next.findings import seed_iq_findings
    from seed.peloton.iq_next.anomalies import seed_iq_anomalies

    await seed_iq_artifacts(session, bundle=bundle)
    await seed_iq_kpis(session, bundle=bundle)
    await seed_iq_decisions(session, bundle=bundle)
    await seed_iq_action_items(session, bundle=bundle)
    await seed_iq_findings(session, bundle=bundle)
    await seed_iq_anomalies(session, bundle=bundle)
```

- [ ] **Step 4: Commit**

```bash
git add seed/peloton/__init__.py seed/peloton/project_pulse/__init__.py seed/peloton/iq_next/__init__.py
git commit -m "feat(seed): scaffold project_pulse and iq_next initiative modules"
```

---

### Task 2: Project Pulse — Artifacts

Seed 3 artifacts for the wearable initiative: product brief, manufacturing & tariff analysis, and regulatory & data strategy.

**Files:**
- Create: `seed/peloton/project_pulse/artifacts.py`

**Pattern:** Follow `scenarios.py` — use `ArtifactRow`, `ArtifactVersionRow`, `ArtifactSubjectRow` from `prescient.artifacts.infrastructure.tables`. Each artifact gets a deterministic UUID via `uuid5(namespace, key)`, an artifact row, a version row with blocks, and a subject row linking to the hero company.

- [ ] **Step 1: Create artifacts.py with UUID namespace and helper**

Create `seed/peloton/project_pulse/artifacts.py`:

```python
"""Project Pulse artifacts — product brief, manufacturing analysis, regulatory strategy."""

from __future__ import annotations

import logging
from datetime import datetime, timezone
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.infrastructure.tables import (
    ArtifactRow,
    ArtifactSubjectRow,
    ArtifactVersionRow,
)
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_PULSE_ARTIFACT_NS = UUID("10a1b2c3-d4e5-4f6a-7b8c-9d0e1f2a3b4c")
_PULSE_ARTIFACT_VER_NS = UUID("20b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d")

_TODAY = datetime.now(timezone.utc)


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


async def _seed_artifact(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
    key: str,
    title: str,
    artifact_type: str,
    blocks: list[dict],
    summary: str,
) -> None:
    hero = bundle.hero()
    artifact_id = str(_uuid(_PULSE_ARTIFACT_NS, key))
    version_id = str(_uuid(_PULSE_ARTIFACT_VER_NS, key))

    stmt = pg_insert(ArtifactRow).values(
        id=artifact_id,
        organization_id=str(hero.id),
        artifact_type=artifact_type,
        title=title,
        visibility="shared",
        active_version_id=version_id,
    )
    stmt = stmt.on_conflict_do_update(
        index_elements=[ArtifactRow.id],
        set_={"title": stmt.excluded.title, "active_version_id": stmt.excluded.active_version_id},
    )
    await session.execute(stmt)

    vstmt = pg_insert(ArtifactVersionRow).values(
        id=version_id,
        artifact_id=artifact_id,
        version_number=1,
        state="published",
        confidence_label="high",
        blocks=blocks,
        citations=[],
        summary=summary,
        created_by=bundle.user.id,
    )
    vstmt = vstmt.on_conflict_do_update(
        index_elements=[ArtifactVersionRow.id],
        set_={"blocks": vstmt.excluded.blocks, "summary": vstmt.excluded.summary},
    )
    await session.execute(vstmt)

    sstmt = pg_insert(ArtifactSubjectRow).values(
        artifact_version_id=version_id,
        subject_type="company",
        subject_id=str(hero.id),
        role="subject",
        is_primary=True,
    )
    sstmt = sstmt.on_conflict_do_update(
        index_elements=[
            ArtifactSubjectRow.artifact_version_id,
            ArtifactSubjectRow.subject_type,
            ArtifactSubjectRow.subject_id,
        ],
        set_={"is_primary": sstmt.excluded.is_primary},
    )
    await session.execute(sstmt)
    logger.info("seeded pulse artifact: %s", key)
```

- [ ] **Step 2: Add the product brief artifact data**

Append to `seed/peloton/project_pulse/artifacts.py`:

```python
_PRODUCT_BRIEF_BLOCKS = [
    {
        "block_id": "vision",
        "kind": "section",
        "heading": "Vision",
        "body": (
            "Project Pulse is a fitness/wellness wristband that captures continuous "
            "biometric data — heart rate, HRV, skin temperature, SpO2, sleep stages, "
            "and activity — and feeds it directly into Peloton IQ for hyper-personalized "
            "coaching. Positioned as a 'coaching sensor' that makes the Peloton ecosystem "
            "smarter, not a smartwatch competitor. The strategic prize is first-party "
            "biometric data that Peloton currently gets secondhand from Apple Health, "
            "Garmin, and Fitbit."
        ),
    },
    {
        "block_id": "target-specs",
        "kind": "section",
        "heading": "Target Specifications",
        "body": (
            "Form factor: slim wristband, no full display (e-ink status indicator under "
            "evaluation). Sensor suite: optical HR/SpO2 (Nordic Semi nRF5340 vs. Qualcomm "
            "QCC5181 under evaluation), 3-axis accelerometer, skin temperature thermistor, "
            "bioimpedance for body composition (stretch goal). Battery: 7-day target life, "
            "wireless charging. Connectivity: BLE 5.3 to Peloton equipment + phone app. "
            "Water resistance: 5 ATM minimum. Weight target: under 30g."
        ),
    },
    {
        "block_id": "competitive-landscape",
        "kind": "section",
        "heading": "Competitive Landscape",
        "body": (
            "Apple Watch Series 10/Ultra 3: dominant ecosystem, but general-purpose — not "
            "optimized for fitness coaching integration. WHOOP 5.0: closest positioning as "
            "'coaching sensor' but received FDA Warning Letter in 2025 for medical-grade "
            "blood pressure claims — cautionary precedent. Oura Ring Gen 4: strong sleep/ "
            "recovery tracking with new AI insights, ring form factor limits workout tracking. "
            "Garmin Venu 4: deep fitness metrics but fragmented software ecosystem. "
            "Key differentiator: none of these feed data into a coaching platform as deeply "
            "integrated as Peloton IQ. Pulse's value is the closed-loop data flywheel."
        ),
    },
    {
        "block_id": "budget",
        "kind": "section",
        "heading": "Budget & Timeline",
        "body": (
            "Board-allocated budget: $45M over 18 months. Spent to date: ~$3M on feasibility "
            "(market sizing, preliminary industrial design, vendor scouting). Next gate: "
            "manufacturing partnership and component contracts (~$15-20M commitment). "
            "Target launch: Q4 2027 holiday season. Critical path: sensor vendor selection "
            "(Q2 2026), manufacturing partner commitment (Q3 2026), FDA pathway decision "
            "(Q2 2026), tooling and first prototypes (Q4 2026), certification (Q1-Q2 2027), "
            "production ramp (Q3 2027)."
        ),
    },
    {
        "block_id": "risks",
        "kind": "section",
        "heading": "Key Risks",
        "body": (
            "1. Tariff uncertainty: SCOTUS struck down IEEPA tariffs (Feb 2026) but "
            "replacement Section 301/232 tariffs are being pursued — rates and timing "
            "unknown. Manufacturing cost models have ±15-20% variance depending on outcome. "
            "2. FDA pathway: wellness-only claims are fast and clear per 2026 guidance, but "
            "health-grade claims (the real differentiator) require 12+ months of compliance "
            "work and risk a WHOOP-style enforcement action. "
            "3. Budget fragility: Peloton stock at ~$4, CFO departed, Cross Training Series "
            "underperforming. The $45M allocation is politically fragile — any overrun or "
            "delay could trigger a board review. "
            "4. Data governance: biometric data collection requires a governance framework "
            "that doesn't exist yet. Priya Sharma's team is building it but it's blocking "
            "both Pulse and IQ Next expansion."
        ),
    },
    {
        "block_id": "authors",
        "kind": "section",
        "heading": "Authors",
        "body": "Maria Santos (VP Hardware Engineering), Raj Patel (Chief Product Officer)",
    },
]

_MANUFACTURING_ANALYSIS_BLOCKS = [
    {
        "block_id": "executive-summary",
        "kind": "section",
        "heading": "Executive Summary",
        "body": (
            "This analysis evaluates four manufacturing scenarios for Project Pulse in the "
            "context of the post-SCOTUS tariff landscape. The Supreme Court's February 2026 "
            "ruling in Learning Resources v. Trump struck down IEEPA-based tariffs, but the "
            "administration is actively pursuing replacement tariffs via Section 301, Section "
            "232, and Section 122 investigations. The tariff picture is genuinely uncertain "
            "— no scenario can be modeled with confidence beyond ±15-20% on landed cost. "
            "Recommendation: Vietnam hybrid assembly with contingency planning for tariff "
            "escalation, acknowledging that neither the 'go bold' nor 'go lean' camps will "
            "be fully satisfied with this approach."
        ),
    },
    {
        "block_id": "scenario-a",
        "kind": "section",
        "heading": "Scenario A: Shenzhen Contract Manufacturing",
        "body": (
            "Established supply chain, lowest base unit cost (~$48-55 per unit at volume). "
            "Current exposure: 20% fentanyl-related tariff on Chinese electronics remains "
            "post-SCOTUS (it was not imposed under IEEPA). Potential exposure: Section 301 "
            "replacement tariffs could add 25-50% on top. US imports from China fell roughly "
            "50% in 2025 — supply chain ecosystem is thinning. Chinese manufacturers are "
            "relocating final assembly to Vietnam to dodge tariffs, which creates quality "
            "control and IP protection concerns with secondary suppliers. "
            "Risk rating: HIGH — tariff exposure is unbounded and trending worse."
        ),
    },
    {
        "block_id": "scenario-b",
        "kind": "section",
        "heading": "Scenario B: Vietnam Assembly",
        "body": (
            "Growing electronics manufacturing hub. Base unit cost ~$52-60 per unit at "
            "volume. Current tariff: 20% on most Vietnamese imports per bilateral deal, "
            "but 40% on goods flagged as Chinese transshipment. CBP has flagged 3 electronics "
            "importers in Q1 2026 for transshipment evasion — enforcement is real and "
            "increasing. Component supply chain still depends on Chinese parts (~60-70% "
            "of BOM value would originate from China). Rising labor costs and infrastructure "
            "gaps are exposing limits to Vietnam's rapid manufacturing expansion. "
            "Risk rating: MEDIUM — lower tariff exposure but transshipment enforcement "
            "and supply chain maturity are real concerns."
        ),
    },
    {
        "block_id": "scenario-c",
        "kind": "section",
        "heading": "Scenario C: Vietnam Assembly + Diversified Asian Components",
        "body": (
            "Hybrid approach: final assembly in Vietnam, with component sourcing diversified "
            "across Taiwan (chipsets), South Korea (display/sensors), and Japan (battery "
            "cells) to reduce China dependency to <30% of BOM value. Unit cost ~$58-68 "
            "per unit — higher due to supply chain complexity and smaller order volumes "
            "across multiple suppliers. Reduces transshipment risk significantly. Adds "
            "6-8 weeks to supply chain lead time. Requires dedicated supply chain management "
            "headcount (2-3 FTEs). "
            "Risk rating: MEDIUM-LOW — best tariff posture but highest operational complexity."
        ),
    },
    {
        "block_id": "scenario-d",
        "kind": "section",
        "heading": "Scenario D: US Assembly (Austin)",
        "body": (
            "Tariff-immune for domestic sales. Unit cost ~$78-95 per unit — 30-40% premium "
            "over Asian manufacturing. No existing Peloton manufacturing facility; would "
            "require lease/buildout in Austin (near Atlas Wearables' former HQ) or contract "
            "with US EMS provider. Labor availability and skills gaps are the top challenge "
            "for US electronics manufacturing per Deloitte's 2026 outlook. Only 36% of "
            "manufacturers are actively reshoring despite tariff pressure — most are passing "
            "costs to consumers instead. Lead time to production-ready: 12-18 months. "
            "Risk rating: LOW tariff risk, HIGH execution risk and cost."
        ),
    },
    {
        "block_id": "recommendation",
        "kind": "section",
        "heading": "Recommendation",
        "body": (
            "Proceed with Scenario C (Vietnam assembly + diversified components) as primary "
            "path, with Scenario D (US assembly) as a contingency if Section 301 replacement "
            "tariffs exceed 35% on Vietnamese electronics. Begin vendor qualification for "
            "both paths in parallel. Defer final manufacturing commitment until Q3 2026 when "
            "the replacement tariff landscape should be clearer. "
            "Author: Tom Brennan (Director of Supply Chain). "
            "Dissent: David Park (Interim CFO) prefers deferring all manufacturing commitments "
            "until tariff picture stabilizes. Maria Santos (VP Hardware) prefers committing "
            "to Scenario C now to protect the Q4 2027 launch timeline."
        ),
    },
]

_REGULATORY_STRATEGY_BLOCKS = [
    {
        "block_id": "fda-landscape",
        "kind": "section",
        "heading": "2026 FDA Regulatory Landscape",
        "body": (
            "In January 2026, the FDA finalized updated guidance on its General Wellness "
            "Policy, relaxing requirements for low-risk wearable devices. Noninvasive "
            "products measuring activity, recovery, sleep, pulse, or fitness-related "
            "biomarkers generally qualify as low-risk general wellness products, provided "
            "their claims avoid references to disease and measurements are appropriately "
            "validated. This is favorable for Project Pulse's core feature set."
        ),
    },
    {
        "block_id": "wellness-pathway",
        "kind": "section",
        "heading": "Pathway A: Wellness-Only Claims",
        "body": (
            "If Pulse limits claims to general wellness — heart rate tracking, activity "
            "monitoring, sleep stage estimation, HRV-based recovery scoring — it qualifies "
            "under the updated General Wellness Policy with no premarket review requirement. "
            "Timeline impact: none (no FDA submission needed). Cost: validation testing only "
            "(~$200-400K). This is the fastest path to market and eliminates regulatory risk. "
            "Limitation: cannot market SpO2 for health monitoring, blood pressure estimation, "
            "or any feature implying disease detection or medical-grade accuracy."
        ),
    },
    {
        "block_id": "health-grade-pathway",
        "kind": "section",
        "heading": "Pathway B: Health-Grade Claims",
        "body": (
            "If Pulse makes health-grade claims (SpO2 for health monitoring, blood pressure "
            "estimation, atrial fibrillation detection), it requires FDA 510(k) clearance as "
            "a Class II medical device. Timeline: 12-18 months from submission to clearance. "
            "Cost: $1.5-3M for clinical validation, regulatory submission, and QMS compliance. "
            "This is the differentiation play — it positions Pulse above consumer-grade "
            "competitors and opens the door to health insurance partnerships and B2B wellness. "
            "Risk: WHOOP received an FDA Warning Letter in mid-2025 for marketing 'Blood "
            "Pressure Insights' without clearance. The FDA is actively enforcing against "
            "wearables that blur the wellness/medical line."
        ),
    },
    {
        "block_id": "data-governance",
        "kind": "section",
        "heading": "Data Governance Requirements",
        "body": (
            "Continuous biometric data collection introduces obligations beyond typical "
            "fitness app data. Key considerations: "
            "1. HIPAA adjacency: if Pulse data is used in conjunction with health claims or "
            "shared with healthcare providers, HIPAA may apply. Wellness-only claims reduce "
            "this risk significantly. "
            "2. State privacy laws: CCPA/CPRA (California), BIPA (Illinois biometric data), "
            "and emerging state laws require explicit consent and data minimization for "
            "biometric identifiers. "
            "3. International: GDPR applies if sold in EU (likely Phase 2). "
            "4. Internal governance: a data governance framework covering collection scope, "
            "retention policy, anonymization pipeline, and member consent flow must be "
            "completed before any biometric data collection begins. This framework does not "
            "exist today. It is a blocking dependency for both Project Pulse and Peloton IQ "
            "Next's expansion plans."
        ),
    },
    {
        "block_id": "recommendation",
        "kind": "section",
        "heading": "Recommendation",
        "body": (
            "Launch Pulse v1 under the wellness-only pathway. Reserve health-grade claims "
            "for Pulse v2 after establishing market presence and completing regulatory "
            "groundwork. Begin data governance framework immediately — it is the critical "
            "path blocker shared with Peloton IQ Next. "
            "Author: Priya Sharma (Head of Data & Privacy). "
            "Dissent: Raj Patel (CPO) argues wellness-only positioning is insufficient to "
            "differentiate from WHOOP/Oura and that the health-grade path should be pursued "
            "in parallel from day one."
        ),
    },
]
```

- [ ] **Step 3: Add the seed_pulse_artifacts function**

Append to `seed/peloton/project_pulse/artifacts.py`:

```python
async def seed_pulse_artifacts(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed all Project Pulse artifacts."""
    await _seed_artifact(
        session,
        bundle=bundle,
        key="pulse-product-brief",
        title="Project Pulse — Product Brief",
        artifact_type="product_brief",
        blocks=_PRODUCT_BRIEF_BLOCKS,
        summary=(
            "Product brief for Project Pulse, Peloton's wearable fitness/wellness "
            "wristband. Covers vision, target specs, competitive landscape, budget "
            "($45M/18mo), timeline (Q4 2027 launch), and key risks."
        ),
    )
    await _seed_artifact(
        session,
        bundle=bundle,
        key="pulse-manufacturing-analysis",
        title="Project Pulse — Manufacturing & Tariff Analysis",
        artifact_type="cost_analysis",
        blocks=_MANUFACTURING_ANALYSIS_BLOCKS,
        summary=(
            "Four-scenario manufacturing analysis for Project Pulse in the post-SCOTUS "
            "tariff landscape. Recommends Vietnam hybrid assembly with contingency planning. "
            "Authored by Tom Brennan (Supply Chain)."
        ),
    )
    await _seed_artifact(
        session,
        bundle=bundle,
        key="pulse-regulatory-strategy",
        title="Project Pulse — Regulatory & Data Strategy",
        artifact_type="regulatory_analysis",
        blocks=_REGULATORY_STRATEGY_BLOCKS,
        summary=(
            "FDA regulatory pathway analysis and data governance requirements for Project "
            "Pulse. Recommends wellness-only claims for v1, health-grade for v2. Data "
            "governance framework is a blocking dependency. Authored by Priya Sharma."
        ),
    )
```

- [ ] **Step 4: Verify module imports work**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/enriched-seed-data && python -c "from seed.peloton.project_pulse.artifacts import seed_pulse_artifacts; print('OK')"`

Expected: `OK`

- [ ] **Step 5: Commit**

```bash
git add seed/peloton/project_pulse/artifacts.py
git commit -m "feat(seed): add Project Pulse artifacts — product brief, manufacturing analysis, regulatory strategy"
```

---

### Task 3: Project Pulse — Decisions

Seed 3 decision records for the wearable initiative.

**Files:**
- Create: `seed/peloton/project_pulse/decisions.py`

**Pattern:** Follow `scenarios.py:seed_decision_records` — use `ArtifactRow`, `ArtifactVersionRow`, `ArtifactSubjectRow`. Blocks use `kind: "context"`, `kind: "decision"`, `kind: "rationale"`.

- [ ] **Step 1: Create decisions.py**

Create `seed/peloton/project_pulse/decisions.py`:

```python
"""Project Pulse decision records."""

from __future__ import annotations

import logging
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.infrastructure.tables import (
    ArtifactRow,
    ArtifactSubjectRow,
    ArtifactVersionRow,
)
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_PULSE_DECISION_NS = UUID("30c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e")
_PULSE_DECISION_VER_NS = UUID("40d4e5f6-a7b8-4c9d-0e1f-2a3b4c5d6e7f")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_DECISIONS = [
    {
        "key": "pulse-manufacturing",
        "title": "Manufacturing approach for Project Pulse",
        "blocks": [
            {
                "block_id": "context",
                "kind": "context",
                "text": (
                    "Project Pulse requires a manufacturing partner and approach decision "
                    "by Q3 2026 to protect the Q4 2027 launch timeline. The tariff landscape "
                    "is in flux following the Supreme Court's February 2026 IEEPA ruling — "
                    "replacement tariffs via Section 301/232 are being pursued but rates and "
                    "timing are uncertain. Four scenarios have been modeled with unit cost "
                    "ranging from $48 (Shenzhen) to $95 (US assembly). Tom Brennan recommends "
                    "Vietnam hybrid assembly (Scenario C) with contingency planning."
                ),
            },
            {
                "block_id": "decision",
                "kind": "decision",
                "text": "OPEN — No consensus reached. Decision deferred to Q3 2026 board review.",
            },
            {
                "block_id": "positions",
                "kind": "rationale",
                "text": (
                    "Maria Santos (VP Hardware): commit to Vietnam hybrid now to protect "
                    "launch timeline. Delay risks missing the Q4 2027 window entirely. "
                    "David Park (Interim CFO): defer all manufacturing commitments until "
                    "replacement tariff rates are published. The $15-20M commitment is too "
                    "large given current financial position (stock at ~$4, CFO transition, "
                    "Cross Training underperformance). References 2022-23 cash crisis. "
                    "Tom Brennan (Supply Chain): begin vendor qualification for both Vietnam "
                    "hybrid and US assembly in parallel, defer final commitment to Q3 2026. "
                    "This preserves optionality at modest cost (~$500K for dual qualification)."
                ),
            },
        ],
    },
    {
        "key": "pulse-fda-pathway",
        "title": "FDA regulatory pathway for Project Pulse",
        "blocks": [
            {
                "block_id": "context",
                "kind": "context",
                "text": (
                    "The 2026 FDA General Wellness Policy relaxation makes wellness-only "
                    "claims straightforward — no premarket review needed for HR, HRV, sleep, "
                    "activity tracking. Health-grade claims (SpO2 monitoring, blood pressure) "
                    "require 510(k) clearance (12-18 months, $1.5-3M). WHOOP's 2025 FDA "
                    "Warning Letter for 'medical grade' blood pressure claims is a cautionary "
                    "precedent. The choice affects competitive positioning, timeline, cost, "
                    "and the data governance framework scope."
                ),
            },
            {
                "block_id": "decision",
                "kind": "decision",
                "text": "OPEN — Priya Sharma's FDA pathway recommendation is pending.",
            },
            {
                "block_id": "positions",
                "kind": "rationale",
                "text": (
                    "Priya Sharma (Data & Privacy): wellness-only for v1, health-grade for "
                    "v2. Minimizes risk, gets to market faster, avoids WHOOP-style enforcement. "
                    "Raj Patel (CPO): pursue health-grade claims from day one — wellness-only "
                    "positioning is insufficient to differentiate from WHOOP and Oura. The "
                    "whole point of Pulse is owning health data that competitors can't match. "
                    "Maria Santos (VP Hardware): leaning wellness-only for v1 to protect "
                    "timeline, but wants health-grade sensor suite in the hardware from day one "
                    "so v2 upgrade is software-only."
                ),
            },
        ],
    },
    {
        "key": "pulse-coaching-beta",
        "title": "Conversational coaching: proceed to member beta",
        "blocks": [
            {
                "block_id": "context",
                "kind": "context",
                "text": (
                    "Alex Kim (Senior ML Engineer) built a conversational coaching prototype "
                    "that lets members ask natural language questions ('what should I do today?', "
                    "'my knee hurts, modify my plan'). Demo to leadership received mixed "
                    "reception — Raj Patel and James Liu were enthusiastic, Nicole Washington "
                    "was skeptical of quality, and medical/legal flagged liability risk for "
                    "injury-related advice. A member beta would test real-world usage patterns "
                    "and member reception before broader rollout."
                ),
            },
            {
                "block_id": "decision",
                "kind": "decision",
                "text": (
                    "OPEN — Raj Patel proposing limited beta (500 members, explicit disclaimer, "
                    "injury-related queries redirected to 'consult your doctor'). Medical/legal "
                    "reviewing liability framework."
                ),
            },
            {
                "block_id": "positions",
                "kind": "rationale",
                "text": (
                    "Raj Patel (CPO): limited beta with guardrails is low risk and high signal. "
                    "We need real member data to evaluate conversational coaching, not more "
                    "internal debate. "
                    "Alex Kim (Sr ML Engineer): prototype is ready. Quality issues are solvable "
                    "with member feedback loop — we need that data. "
                    "Nicole Washington (VP Content): the prototype responses feel generic and "
                    "lack the personality that makes Peloton instructors special. Risk of "
                    "reinforcing the 'robot trainer' perception. "
                    "David Park (Interim CFO): what's the cost of the beta infrastructure? "
                    "Don't commit engineering resources without a business case."
                ),
            },
        ],
    },
]


async def seed_pulse_decisions(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Project Pulse decision records."""
    hero = bundle.hero()
    for d in _DECISIONS:
        artifact_id = str(_uuid(_PULSE_DECISION_NS, d["key"]))
        version_id = str(_uuid(_PULSE_DECISION_VER_NS, d["key"]))

        stmt = pg_insert(ArtifactRow).values(
            id=artifact_id,
            organization_id=str(hero.id),
            artifact_type="decision_record",
            title=d["title"],
            visibility="shared",
            active_version_id=version_id,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[ArtifactRow.id],
            set_={"title": stmt.excluded.title, "active_version_id": stmt.excluded.active_version_id},
        )
        await session.execute(stmt)

        vstmt = pg_insert(ArtifactVersionRow).values(
            id=version_id,
            artifact_id=artifact_id,
            version_number=1,
            state="published",
            confidence_label="high",
            blocks=d["blocks"],
            citations=[],
            summary=d["title"],
            created_by=bundle.user.id,
        )
        vstmt = vstmt.on_conflict_do_update(
            index_elements=[ArtifactVersionRow.id],
            set_={"blocks": vstmt.excluded.blocks, "summary": vstmt.excluded.summary},
        )
        await session.execute(vstmt)

        sstmt = pg_insert(ArtifactSubjectRow).values(
            artifact_version_id=version_id,
            subject_type="company",
            subject_id=str(hero.id),
            role="subject",
            is_primary=True,
        )
        sstmt = sstmt.on_conflict_do_update(
            index_elements=[
                ArtifactSubjectRow.artifact_version_id,
                ArtifactSubjectRow.subject_type,
                ArtifactSubjectRow.subject_id,
            ],
            set_={"is_primary": sstmt.excluded.is_primary},
        )
        await session.execute(sstmt)
        logger.info("seeded pulse decision: %s", d["key"])
```

- [ ] **Step 2: Commit**

```bash
git add seed/peloton/project_pulse/decisions.py
git commit -m "feat(seed): add Project Pulse decision records — manufacturing, FDA pathway, coaching beta"
```

---

### Task 4: Project Pulse — Action Items

Seed 5 action items for the wearable initiative.

**Files:**
- Create: `seed/peloton/project_pulse/action_items.py`

**Pattern:** Follow `scenarios.py:seed_action_items` — use `ActionItemTable`, `_TODAY + timedelta(days=offset)` for due dates, optional citations linking to decision record artifact IDs.

- [ ] **Step 1: Create action_items.py**

Create `seed/peloton/project_pulse/action_items.py`:

```python
"""Project Pulse action items."""

from __future__ import annotations

import logging
from datetime import date, datetime, timedelta, timezone
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.action_items.infrastructure.tables import ActionItemTable
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_PULSE_ACTION_NS = UUID("50e5f6a7-b8c9-4d0e-1f2a-3b4c5d6e7f8a")
_PULSE_DECISION_NS = UUID("30c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e")

_TODAY = date.today()
_NOW = datetime.now(timezone.utc)


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_ACTION_ITEMS = [
    {
        "key": "pulse-sensor-shortlist",
        "title": "Finalize sensor suite vendor shortlist for Project Pulse",
        "description": (
            "Complete technical evaluation of Nordic Semi nRF5340 vs. Qualcomm QCC5181 "
            "chipsets for Pulse's optical HR/SpO2 sensor suite. Deliverable: comparison "
            "matrix covering power consumption, sensor accuracy (validated against clinical "
            "reference), BLE 5.3 stack maturity, supply chain lead times, and unit cost at "
            "10K/100K/500K volumes. Include recommendation with rationale."
        ),
        "owner_name": "Maria Santos",
        "owner_role": "VP Hardware Engineering",
        "priority": "high",
        "status": "pending_review",
        "due_offset_days": 14,
        "decision_key": None,
    },
    {
        "key": "pulse-tariff-update",
        "title": "Complete post-SCOTUS tariff scenario update with Section 301/232 projections",
        "description": (
            "Update the four-scenario manufacturing cost model with latest intelligence on "
            "replacement tariff rates being pursued via Section 301 and Section 232 "
            "investigations. Include CBP transshipment enforcement trends for Vietnam "
            "corridor. Incorporate the 3 recent CBP enforcement actions against electronics "
            "importers. Deliver updated landed-cost ranges for each scenario with confidence "
            "intervals."
        ),
        "owner_name": "Tom Brennan",
        "owner_role": "Director of Supply Chain",
        "priority": "high",
        "status": "sent_to_owner",
        "due_offset_days": 10,
        "decision_key": "pulse-manufacturing",
    },
    {
        "key": "pulse-budget-gate",
        "title": "Present Project Pulse budget gate review to board",
        "description": (
            "Prepare board presentation for Pulse budget gate review. Must include: updated "
            "manufacturing cost scenarios, FDA pathway recommendation, competitive landscape "
            "update (Apple Watch, WHOOP, Oura), revised timeline and risk register. David Park "
            "to present — note he is skeptical of proceeding and may recommend deferral or "
            "scope reduction pending tariff clarity and Cross Training Series performance "
            "improvement."
        ),
        "owner_name": "David Park",
        "owner_role": "Interim CFO",
        "priority": "high",
        "status": "pending_review",
        "due_offset_days": 30,
        "decision_key": "pulse-manufacturing",
    },
    {
        "key": "pulse-fda-recommendation",
        "title": "Deliver FDA pathway recommendation — wellness vs. health-grade claims",
        "description": (
            "Complete analysis of wellness-only vs. health-grade regulatory pathways for "
            "Pulse v1. Deliverable: written recommendation with timeline impact, cost "
            "estimate, risk assessment, and precedent analysis (including WHOOP Warning "
            "Letter implications). Must address Maria Santos' proposal to include health-grade "
            "sensor hardware in v1 even if software claims are wellness-only."
        ),
        "owner_name": "Priya Sharma",
        "owner_role": "Head of Data & Privacy",
        "priority": "medium",
        "status": "sent_to_owner",
        "due_offset_days": 21,
        "decision_key": "pulse-fda-pathway",
    },
    {
        "key": "pulse-austin-feasibility",
        "title": "Evaluate Austin facility feasibility for US assembly",
        "description": (
            "Assess feasibility of US-based assembly for Project Pulse at an Austin, TX "
            "facility (near Atlas Wearables' former HQ). Include: available EMS partners, "
            "lease/buildout costs, labor market analysis, timeline to production-ready, and "
            "comparison against Deloitte's 2026 manufacturing outlook findings on reshoring "
            "barriers. Note: deprioritized after initial cost analysis showed 30-40% unit "
            "cost premium — maintain as contingency if tariff escalation warrants."
        ),
        "owner_name": "Maria Santos",
        "owner_role": "VP Hardware Engineering",
        "priority": "low",
        "status": "deferred",
        "due_offset_days": 60,
        "decision_key": "pulse-manufacturing",
    },
]


async def seed_pulse_action_items(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Project Pulse action items."""
    for item in _ACTION_ITEMS:
        item_id = _uuid(_PULSE_ACTION_NS, item["key"])
        due = _TODAY + timedelta(days=item["due_offset_days"])

        citations: list[dict] = []
        if item["decision_key"]:
            decision_artifact_id = str(_uuid(_PULSE_DECISION_NS, item["decision_key"]))
            citations.append({
                "type": "decision_record",
                "artifact_id": decision_artifact_id,
                "label": item["decision_key"],
            })

        deferred_until = due if item["status"] == "deferred" else None
        deferred_reason = (
            "Deprioritized — US assembly cost premium (30-40%) makes this a contingency "
            "path only. Revisit if Section 301 replacement tariffs exceed 35% on Vietnam."
            if item["status"] == "deferred"
            else None
        )

        stmt = pg_insert(ActionItemTable).values(
            id=item_id,
            owner_tenant_id=bundle.fund.id,
            conversation_id=None,
            owner_name=item["owner_name"],
            owner_role=item["owner_role"],
            title=item["title"],
            description=item["description"],
            rationale=None,
            priority=item["priority"],
            priority_score=None,
            priority_rationale=None,
            linked_focus_priorities=None,
            status=item["status"],
            due_date=due,
            deferred_until=deferred_until,
            deferred_reason=deferred_reason,
            citations=citations,
            created_by=bundle.user.id,
            approved_by=bundle.user.id if item["status"] == "completed" else None,
            approved_at=_NOW if item["status"] == "completed" else None,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[ActionItemTable.id],
            set_={
                "title": stmt.excluded.title,
                "description": stmt.excluded.description,
                "status": stmt.excluded.status,
                "priority": stmt.excluded.priority,
                "due_date": stmt.excluded.due_date,
                "citations": stmt.excluded.citations,
            },
        )
        await session.execute(stmt)
        logger.info("seeded pulse action item: %s", item["key"])
```

- [ ] **Step 2: Commit**

```bash
git add seed/peloton/project_pulse/action_items.py
git commit -m "feat(seed): add Project Pulse action items — sensor shortlist, tariff update, budget gate, FDA, Austin"
```

---

### Task 5: Project Pulse — KPIs, Findings, Anomalies

Seed forward-looking KPI values, monitoring findings, and KPI anomalies for Project Pulse.

**Files:**
- Create: `seed/peloton/project_pulse/kpis.py`
- Create: `seed/peloton/project_pulse/kpi_values.yaml`
- Create: `seed/peloton/project_pulse/findings.py`
- Create: `seed/peloton/project_pulse/anomalies.py`

- [ ] **Step 1: Create kpi_values.yaml**

Create `seed/peloton/project_pulse/kpi_values.yaml`:

```yaml
# Project Pulse — forward-looking KPI values.
# These are internal projections, NOT historical EDGAR data.

definitions:
  - id: pulse_feasibility_spend
    label: "Pulse: feasibility spend (actual)"
    unit: usd_millions
    description: Actual spend on Project Pulse feasibility phase.

  - id: pulse_budget_total
    label: "Pulse: total program budget"
    unit: usd_millions
    description: Board-allocated total budget for Project Pulse.

  - id: pulse_budget_committed
    label: "Pulse: committed spend"
    unit: usd_millions
    description: Contractually committed spend on Project Pulse.

  - id: pulse_unit_cost_shenzhen
    label: "Pulse: projected unit cost — Shenzhen"
    unit: usd
    description: Projected per-unit manufacturing cost, Shenzhen assembly scenario.

  - id: pulse_unit_cost_vietnam
    label: "Pulse: projected unit cost — Vietnam hybrid"
    unit: usd
    description: Projected per-unit manufacturing cost, Vietnam hybrid assembly scenario.

  - id: pulse_unit_cost_us
    label: "Pulse: projected unit cost — US assembly"
    unit: usd
    description: Projected per-unit manufacturing cost, US assembly scenario.

periods:
  - label: "FY2026 Q1"
    start: 2025-07-01
    end: 2025-09-30
  - label: "FY2026 Q2"
    start: 2025-10-01
    end: 2025-12-31
  - label: "FY2026 Q3 (proj)"
    start: 2026-01-01
    end: 2026-03-31

values:
  pulse_feasibility_spend:
    "FY2026 Q1": 0.8
    "FY2026 Q2": 1.2
    "FY2026 Q3 (proj)": 1.0
  pulse_budget_total:
    "FY2026 Q1": 45.0
    "FY2026 Q2": 45.0
    "FY2026 Q3 (proj)": 45.0
  pulse_budget_committed:
    "FY2026 Q1": 0.8
    "FY2026 Q2": 2.0
    "FY2026 Q3 (proj)": 3.0
  pulse_unit_cost_shenzhen:
    "FY2026 Q3 (proj)": 51.50
  pulse_unit_cost_vietnam:
    "FY2026 Q3 (proj)": 63.00
  pulse_unit_cost_us:
    "FY2026 Q3 (proj)": 86.50
```

- [ ] **Step 2: Create kpis.py**

Create `seed/peloton/project_pulse/kpis.py`:

```python
"""Project Pulse KPI definitions and values."""

from __future__ import annotations

import logging
from decimal import Decimal
from pathlib import Path
from uuid import UUID, uuid5

import yaml
from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.documents.infrastructure.tables import (
    KpiDefinitionTable,
    KpiValueTable,
)
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_PULSE_KPI_NS = UUID("60f6a7b8-c9d0-4e1f-2a3b-4c5d6e7f8a9b")
_YAML_PATH = Path(__file__).resolve().parent / "kpi_values.yaml"


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


async def seed_pulse_kpis(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Project Pulse KPI definitions and forward-looking values."""
    data = yaml.safe_load(_YAML_PATH.read_text(encoding="utf-8"))
    hero = bundle.hero()

    # Upsert definitions
    for d in data["definitions"]:
        stmt = pg_insert(KpiDefinitionTable).values(
            id=d["id"],
            label=d["label"],
            unit=d["unit"],
            description=d.get("description"),
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[KpiDefinitionTable.id],
            set_={
                "label": stmt.excluded.label,
                "unit": stmt.excluded.unit,
                "description": stmt.excluded.description,
            },
        )
        await session.execute(stmt)

    # Build period lookup
    periods = {
        row["label"]: (row["start"], row["end"]) for row in data["periods"]
    }

    # Upsert values
    for kpi_id, period_values in data["values"].items():
        for period_label, value in period_values.items():
            start, end = periods[period_label]
            row_id = _uuid(_PULSE_KPI_NS, f"{hero.id}:{kpi_id}:{period_label}")
            stmt = pg_insert(KpiValueTable).values(
                id=row_id,
                company_id=hero.id,
                kpi_id=kpi_id,
                period_start=start,
                period_end=end,
                period_label=period_label,
                value=Decimal(str(value)),
            )
            stmt = stmt.on_conflict_do_update(
                index_elements=[KpiValueTable.id],
                set_={"value": stmt.excluded.value},
            )
            await session.execute(stmt)

    logger.info("seeded pulse KPIs: %d definitions, %d values",
                len(data["definitions"]),
                sum(len(v) for v in data["values"].values()))
```

- [ ] **Step 3: Create findings.py**

Create `seed/peloton/project_pulse/findings.py`:

```python
"""Project Pulse monitoring findings."""

from __future__ import annotations

import logging
from datetime import datetime, timedelta, timezone
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.monitoring.infrastructure.tables import MonitorFindingTable
from seed.companies import SeedBundle
from seed.focus import _MONITOR_TARGET_NS, _FINDING_NS

logger = logging.getLogger(__name__)

# Use a distinct namespace so Pulse findings don't collide with existing ones.
_PULSE_FINDING_NS = UUID("70a7b8c9-d0e1-4f2a-3b4c-5d6e7f8a9b0c")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_FINDINGS = [
    {
        "key": "scotus-ieepa-ruling",
        "target_slug": "peloton",
        "finding_type": "regulatory",
        "title": "SCOTUS strikes down IEEPA tariffs in 6-3 ruling (Learning Resources v. Trump)",
        "summary": (
            "The Supreme Court ruled 6-3 that President Trump's IEEPA-based tariffs exceed "
            "presidential authority. Chief Justice Roberts wrote that IEEPA's 'regulate' "
            "language does not include the power to impose tariffs. The ruling invalidates "
            "the 145% China tariffs and 46% Vietnam tariffs imposed under IEEPA, but does "
            "not affect Section 301 tariffs (20% fentanyl-related tariff on China remains). "
            "The administration is pursuing replacement tariffs via Section 301, 232, and 122 "
            "investigations — rates and timeline uncertain. Critical implications for Project "
            "Pulse manufacturing cost models."
        ),
        "severity": "critical",
        "hours_ago": 72,
    },
    {
        "key": "section-301-replacement",
        "target_slug": "peloton",
        "finding_type": "regulatory",
        "title": "Administration pursuing Section 301/232 replacement tariffs — timeline uncertain",
        "summary": (
            "Following the SCOTUS IEEPA ruling, USTR has initiated accelerated Section 301 "
            "and Section 232 investigations to reimpose tariffs on a legal footing. Industry "
            "sources suggest consumer electronics tariffs of 15-30% on China and 10-20% on "
            "Vietnam are being considered, but formal proposals have not been published. "
            "Tom Brennan's manufacturing analysis must be updated with these projections. "
            "Tariff uncertainty is the primary blocker for Pulse manufacturing commitment."
        ),
        "severity": "important",
        "hours_ago": 48,
    },
    {
        "key": "cbp-transshipment-crackdown",
        "target_slug": "peloton",
        "finding_type": "regulatory",
        "title": "CBP flags 3 electronics importers for Vietnam transshipment tariff evasion",
        "summary": (
            "US Customs and Border Protection flagged three consumer electronics importers "
            "for suspected tariff evasion via Vietnamese transshipment — final assembly in "
            "Vietnam with Chinese-origin components adding less than 8% of export value. "
            "Penalties include retroactive application of China tariff rates plus fines. "
            "Directly relevant to Pulse Scenario B (Vietnam assembly) where 60-70% of BOM "
            "value would originate from China. Tom Brennan flagging as risk factor in "
            "updated manufacturing analysis."
        ),
        "severity": "important",
        "hours_ago": 24,
    },
    {
        "key": "whoop-fda-warning",
        "target_slug": "peloton",
        "finding_type": "regulatory",
        "title": "WHOOP received FDA Warning Letter for 'medical grade' blood pressure claims",
        "summary": (
            "The FDA issued a Warning Letter to WHOOP, Inc. for marketing its 'Blood "
            "Pressure Insights' feature as 'medical grade' without required 510(k) clearance. "
            "WHOOP was required to cease marketing health-grade claims or obtain clearance. "
            "Directly relevant to Pulse's FDA pathway decision — validates Priya Sharma's "
            "recommendation to stay in wellness-only claims for v1. Raj Patel's push for "
            "health-grade claims from day one carries real enforcement risk."
        ),
        "severity": "important",
        "hours_ago": 168,
    },
]


async def seed_pulse_findings(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Project Pulse monitoring findings."""
    fund = bundle.fund
    now = datetime.now(timezone.utc)

    # Reuse the existing Peloton monitor target (created by focus.py).
    peloton_target_id = uuid5(_MONITOR_TARGET_NS, "peloton:headspace")

    for finding in _FINDINGS:
        fid = _uuid(_PULSE_FINDING_NS, finding["key"])
        discovered = now - timedelta(hours=finding["hours_ago"])

        stmt = pg_insert(MonitorFindingTable).values(
            id=fid,
            run_id=None,
            target_id=peloton_target_id,
            owner_tenant_id=fund.id,
            finding_type=finding["finding_type"],
            title=finding["title"],
            summary=finding["summary"],
            severity=finding["severity"],
            occurred_at=discovered,
            discovered_at=discovered,
            source_url=None,
            raw_payload=None,
            processed=False,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[MonitorFindingTable.id],
            set_={
                "title": stmt.excluded.title,
                "summary": stmt.excluded.summary,
                "severity": stmt.excluded.severity,
                "discovered_at": stmt.excluded.discovered_at,
            },
        )
        await session.execute(stmt)
        logger.info("seeded pulse finding: %s", finding["key"])
```

- [ ] **Step 4: Create anomalies.py**

Create `seed/peloton/project_pulse/anomalies.py`:

```python
"""Project Pulse KPI anomalies."""

from __future__ import annotations

import logging
from datetime import date
from decimal import Decimal
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.kpis.infrastructure.tables import KpiAnomalyRow, KpiValueRow
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_PULSE_ANOMALY_NS = UUID("80b8c9d0-e1f2-4a3b-4c5d-6e7f8a9b0c1d")
_PULSE_ANOMALY_VALUE_NS = UUID("90c9d0e1-f2a3-4b4c-5d6e-7f8a9b0c1d2e")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_ANOMALIES = [
    {
        "key": "cross-training-attach",
        "kpi_id": "connected_fitness_subscribers",
        "anomaly_type": "deviation_from_trend",
        "severity": "important",
        "description": (
            "Cross Training Series attach rate is 18% below launch projections after "
            "two quarters. The IQ-powered upgrade value proposition is not compelling "
            "enough to drive hardware purchases at the new price points. Connected "
            "Fitness subscriber decline of 7% YoY persists despite the product refresh."
        ),
        "enrichment_question": (
            "Is the Cross Training underperformance driven by price resistance, weak "
            "IQ value proposition, macro headwinds, or competitive alternatives? What "
            "does this mean for Project Pulse's pricing strategy and market sizing?"
        ),
        "period_label": "FY2026 Q2",
        "value": Decimal("2661000"),
        "unit": "count",
    },
    {
        "key": "pulse-feasibility-burn",
        "kpi_id": "pulse_feasibility_spend",
        "anomaly_type": "deviation_from_trend",
        "severity": "notable",
        "description": (
            "Pulse feasibility spend is tracking to $3M against a $45M total budget — "
            "on plan, but the burn rate is accelerating as vendor evaluations intensify. "
            "Q3 projected spend ($1M) includes travel for Vietnam and Austin site visits "
            "plus sensor vendor technical evaluations. David Park flagging that uncommitted "
            "budget ($42M) represents significant exposure given current financial position."
        ),
        "enrichment_question": (
            "Is the feasibility phase delivering sufficient signal to justify the next "
            "gate ($15-20M manufacturing commitment)? Should scope be narrowed to reduce "
            "the commitment required?"
        ),
        "period_label": "FY2026 Q3 (proj)",
        "value": Decimal("1.0"),
        "unit": "usd_millions",
    },
]


async def seed_pulse_anomalies(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Project Pulse KPI anomalies."""
    hero = bundle.hero()
    org_id = str(hero.id)

    for a in _ANOMALIES:
        # Ensure KPI value row exists for FK
        kpi_value_id = str(_uuid(
            _PULSE_ANOMALY_VALUE_NS,
            f"{org_id}:{a['kpi_id']}:{a['period_label']}",
        ))

        vstmt = pg_insert(KpiValueRow).values(
            id=kpi_value_id,
            organization_id=org_id,
            kpi_id=a["kpi_id"],
            period_label=a["period_label"],
            period_start=date(2025, 10, 1),
            period_end=date(2025, 12, 31),
            value=a["value"],
            unit=a["unit"],
            commentary=None,
            reported_by="seed",
        )
        vstmt = vstmt.on_conflict_do_update(
            index_elements=[KpiValueRow.id],
            set_={"value": vstmt.excluded.value, "unit": vstmt.excluded.unit},
        )
        await session.execute(vstmt)

        anomaly_id = str(_uuid(_PULSE_ANOMALY_NS, a["key"]))
        astmt = pg_insert(KpiAnomalyRow).values(
            id=anomaly_id,
            kpi_value_id=kpi_value_id,
            organization_id=org_id,
            anomaly_type=a["anomaly_type"],
            description=a["description"],
            severity=a["severity"],
            enrichment_question=a["enrichment_question"],
            enrichment_response=None,
        )
        astmt = astmt.on_conflict_do_update(
            index_elements=[KpiAnomalyRow.id],
            set_={
                "description": astmt.excluded.description,
                "severity": astmt.excluded.severity,
                "enrichment_question": astmt.excluded.enrichment_question,
            },
        )
        await session.execute(astmt)
        logger.info("seeded pulse anomaly: %s", a["key"])
```

- [ ] **Step 5: Commit**

```bash
git add seed/peloton/project_pulse/kpi_values.yaml seed/peloton/project_pulse/kpis.py seed/peloton/project_pulse/findings.py seed/peloton/project_pulse/anomalies.py
git commit -m "feat(seed): add Project Pulse KPIs, monitoring findings, and anomalies"
```

---

### Task 6: Peloton IQ Next — Artifacts

Seed 3 artifacts for the AI initiative: state of AI report, content & brand impact assessment, and corporate wellness expansion brief.

**Files:**
- Create: `seed/peloton/iq_next/artifacts.py`

**Pattern:** Same as Task 2 — use `ArtifactRow`, `ArtifactVersionRow`, `ArtifactSubjectRow` from `prescient.artifacts.infrastructure.tables`.

- [ ] **Step 1: Create artifacts.py**

Create `seed/peloton/iq_next/artifacts.py`:

```python
"""Peloton IQ Next artifacts — state of AI, brand impact, corporate wellness."""

from __future__ import annotations

import logging
from datetime import datetime, timezone
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.infrastructure.tables import (
    ArtifactRow,
    ArtifactSubjectRow,
    ArtifactVersionRow,
)
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_IQ_ARTIFACT_NS = UUID("11b1c2d3-e4f5-4a6b-7c8d-9e0f1a2b3c4d")
_IQ_ARTIFACT_VER_NS = UUID("21c2d3e4-f5a6-4b7c-8d9e-0f1a2b3c4d5e")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


async def _seed_artifact(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
    key: str,
    title: str,
    artifact_type: str,
    blocks: list[dict],
    summary: str,
) -> None:
    hero = bundle.hero()
    artifact_id = str(_uuid(_IQ_ARTIFACT_NS, key))
    version_id = str(_uuid(_IQ_ARTIFACT_VER_NS, key))

    stmt = pg_insert(ArtifactRow).values(
        id=artifact_id,
        organization_id=str(hero.id),
        artifact_type=artifact_type,
        title=title,
        visibility="shared",
        active_version_id=version_id,
    )
    stmt = stmt.on_conflict_do_update(
        index_elements=[ArtifactRow.id],
        set_={"title": stmt.excluded.title, "active_version_id": stmt.excluded.active_version_id},
    )
    await session.execute(stmt)

    vstmt = pg_insert(ArtifactVersionRow).values(
        id=version_id,
        artifact_id=artifact_id,
        version_number=1,
        state="published",
        confidence_label="high",
        blocks=blocks,
        citations=[],
        summary=summary,
        created_by=bundle.user.id,
    )
    vstmt = vstmt.on_conflict_do_update(
        index_elements=[ArtifactVersionRow.id],
        set_={"blocks": vstmt.excluded.blocks, "summary": vstmt.excluded.summary},
    )
    await session.execute(vstmt)

    sstmt = pg_insert(ArtifactSubjectRow).values(
        artifact_version_id=version_id,
        subject_type="company",
        subject_id=str(hero.id),
        role="subject",
        is_primary=True,
    )
    sstmt = sstmt.on_conflict_do_update(
        index_elements=[
            ArtifactSubjectRow.artifact_version_id,
            ArtifactSubjectRow.subject_type,
            ArtifactSubjectRow.subject_id,
        ],
        set_={"is_primary": sstmt.excluded.is_primary},
    )
    await session.execute(sstmt)
    logger.info("seeded iq_next artifact: %s", key)


_STATE_OF_AI_BLOCKS = [
    {
        "block_id": "executive-summary",
        "kind": "section",
        "heading": "Executive Summary",
        "body": (
            "Peloton IQ launched October 2025 on Amazon Bedrock and is generating millions "
            "of personalized fitness insights weekly. Six months in, the platform shows "
            "promise but faces three compounding challenges: inference costs growing faster "
            "than revenue uplift, mixed member satisfaction that may be contributing to "
            "subscriber decline, and a data quality ceiling that limits recommendation "
            "accuracy. This report assesses the current state and recommends a path forward."
        ),
    },
    {
        "block_id": "cost-trajectory",
        "kind": "section",
        "heading": "Bedrock Cost Trajectory",
        "body": (
            "Monthly Bedrock inference costs: $340K (Oct 2025) → $380K (Jan 2026) → $418K "
            "(Mar 2026). 23% increase QoQ while connected fitness subscribers declined 7% "
            "YoY. Cost per member per month: $0.13 → $0.16 — trending in the wrong direction. "
            "Primary cost drivers: GPT model calls for complex reasoning (60% of spend), "
            "Llama 4 Scout for real-time insights (30%), embedding and retrieval (10%). "
            "The build-vs-buy decision directly impacts this trajectory — fine-tuned models "
            "could reduce per-inference cost by 40-60% but require $2-3M upfront investment "
            "in training infrastructure and data pipeline."
        ),
    },
    {
        "block_id": "member-satisfaction",
        "kind": "section",
        "heading": "Member Satisfaction",
        "body": (
            "IQ user NPS: +32 (vs. +38 for non-IQ users — a negative delta, not positive). "
            "Recommendation engagement rate: 34% in October, now 22% — members are ignoring "
            "IQ suggestions at an accelerating rate. Qualitative feedback clusters: "
            "1) 'Recommendations feel generic — it doesn't know me' (42% of negative feedback). "
            "2) 'I miss the human instructor energy' (28%). "
            "3) 'The AI suggestions conflict with what my instructor says in class' (18%). "
            "4) 'Form tracking is impressive but recommendations are basic' (12%). "
            "Power users (5+ workouts/week) rate IQ significantly higher — the system needs "
            "more data per member to become useful, creating a cold-start problem for "
            "casual users who are the majority."
        ),
    },
    {
        "block_id": "data-quality-gap",
        "kind": "section",
        "heading": "Data Quality Gap",
        "body": (
            "IQ currently ingests: workout history, class performance metrics, and opt-in "
            "wearable data from Apple Health/Garmin/Fitbit (~28% of members share this). "
            "IQ does NOT have: sleep data, recovery metrics, nutrition, stress indicators, "
            "injury history, or continuous heart rate outside workouts. Recommendation "
            "quality plateaus after ~15 workouts without these signals. This is the core "
            "business case for Project Pulse — first-party continuous biometric data would "
            "break the ceiling. However, Pulse is 18+ months from launch. Bridge option: "
            "deeper Apple Health/Garmin integration could surface sleep and recovery data "
            "for the ~28% who already share, buying time while Pulse develops."
        ),
    },
    {
        "block_id": "build-vs-buy",
        "kind": "section",
        "heading": "Build vs. Buy Analysis",
        "body": (
            "Current stack: off-the-shelf models on Bedrock (GPT for reasoning, Llama 4 "
            "Scout for real-time). Proposed: fine-tune proprietary models on Peloton's "
            "workout corpus (500M+ workout records, anonymized). "
            "Fine-tuning benefits: 40-60% inference cost reduction, significantly better "
            "recommendation relevance in internal benchmarks (+18% engagement in A/B test "
            "on 5K member sample), defensible IP, reduced dependency on third-party model "
            "providers. "
            "Fine-tuning costs: $2-3M compute for initial training, 3-4 months to production "
            "quality, 2 additional ML engineers (headcount request pending). "
            "Recommendation: approve 6-month fine-tuning pilot. The engagement uplift in "
            "A/B testing justifies the investment, and per-inference cost savings would "
            "recoup training costs within 12-18 months at current volume. "
            "Author: James Liu (Head of AI/ML)."
        ),
    },
]

_BRAND_IMPACT_BLOCKS = [
    {
        "block_id": "executive-summary",
        "kind": "section",
        "heading": "Executive Summary",
        "body": (
            "This assessment compiles member sentiment data, instructor feedback, and "
            "industry context on AI-generated content quality to evaluate the brand risk "
            "of Peloton's AI-forward strategy. The findings suggest that while Peloton IQ "
            "has technical merit, its current execution is damaging the brand's core value "
            "proposition of human connection and world-class instruction. We recommend "
            "slowing AI content expansion and investing in instructor-AI collaboration "
            "rather than replacement."
        ),
    },
    {
        "block_id": "member-sentiment",
        "kind": "section",
        "heading": "Member Sentiment Analysis",
        "body": (
            "Member feedback analysis (Jan-Mar 2026, n=2,847 survey responses + 12K social "
            "media mentions): "
            "Negative themes: 'robot trainer' criticism (31% of negative mentions), "
            "'recommendations feel like they come from an algorithm, not a coach' (24%), "
            "'I'm paying more for a worse experience' (22% — tied to subscription price "
            "increase that funded IQ development), 'the AI doesn't understand my body' (15%). "
            "A viral member forum thread ('I cancelled because the AI recommendations are "
            "garbage') reached 4,200 upvotes and was picked up by The Verge. "
            "Positive themes: form tracking accuracy (48% of positive mentions), workout "
            "generation flexibility (32%), personalized plan structure (20%). "
            "Net: the hardware/camera features land well, but the recommendation engine and "
            "AI coaching persona are not meeting member expectations."
        ),
    },
    {
        "block_id": "instructor-survey",
        "kind": "section",
        "heading": "Instructor Sentiment Survey",
        "body": (
            "Internal survey of 45 Peloton instructors (87% response rate): "
            "62% are 'concerned' or 'very concerned' about AI replacing instructor roles. "
            "71% report members asking them about AI recommendations that conflict with "
            "their coaching. 44% feel 'less valued' since the IQ launch. "
            "Verbatim: 'Members used to come to my classes because of me. Now they come "
            "because the algorithm told them to. It's not the same.' — Senior cycling instructor. "
            "'The AI doesn't understand that fitness is emotional, not just physiological. "
            "It can't read a room.' — Strength instructor. "
            "Instructor attrition risk: 3 high-profile instructors have explored competing "
            "offers. Losing marquee instructors would compound the brand damage."
        ),
    },
    {
        "block_id": "industry-context",
        "kind": "section",
        "heading": "Industry Context: The AI Quality Crisis",
        "body": (
            "Peloton is not alone. The enterprise AI quality problem is industry-wide in 2026: "
            "Microsoft reorganized its Copilot division over quality concerns and 'AI slop' "
            "proliferation. PwC's 2026 Global CEO Survey: 56% report getting 'nothing' from "
            "AI investments. Deloitte's State of AI report: 79% of enterprises face adoption "
            "challenges, up significantly from 2025. 54% of C-suite executives say AI adoption "
            "is 'tearing their company apart.' "
            "The 'AI slop' label — originally applied to low-quality AI-generated web content "
            "— is now being applied to enterprise AI outputs. Members don't distinguish "
            "between Peloton's AI and the broader AI backlash. Positioning Peloton as an "
            "'AI-first' company carries brand risk in the current climate."
        ),
    },
    {
        "block_id": "recommendation",
        "kind": "section",
        "heading": "Recommendation",
        "body": (
            "1. Pause AI-generated content expansion. Do not approve the supplemental "
            "workout content pilot until recommendation quality improves measurably. "
            "2. Invest in instructor-AI collaboration, not replacement. Position IQ as a "
            "tool that makes instructors better, not a substitute for them. Prototype: "
            "AI-generated class structure suggestions that instructors customize. "
            "3. Fix recommendation quality before expanding scope. The cold-start problem "
            "for casual users is real — address it with better onboarding, not more AI. "
            "4. Monitor instructor retention closely. Losing marquee instructors to "
            "competitors would be more damaging than any AI feature could offset. "
            "Authors: Nicole Washington (VP Content & Programming), Rachel Torres (Brand "
            "Marketing Director)."
        ),
    },
]

_CORPORATE_WELLNESS_BLOCKS = [
    {
        "block_id": "opportunity",
        "kind": "section",
        "heading": "Market Opportunity",
        "body": (
            "The corporate wellness market is projected at $85B by 2028. Peloton's brand "
            "recognition, content library, and IQ personalization capabilities position it "
            "for a premium B2B offering. Inbound inquiry volume has been 4x expectations "
            "in Q1 2026 — 23 enterprise accounts have expressed interest without active "
            "outbound sales. Key verticals: tech (employee retention perk), finance "
            "(high-stress/sedentary workforce), healthcare (practice-what-you-preach). "
            "Competitive landscape: ClassPass Corporate, Gympass, Virgin Pulse — none "
            "offer the depth of content + AI personalization that Peloton can deliver."
        ),
    },
    {
        "block_id": "revenue-case",
        "kind": "section",
        "heading": "Revenue Diversification Case",
        "body": (
            "Consumer hardware revenue is declining (Q2 FY2026 missed guidance, Connected "
            "Fitness subscribers down 7% YoY). B2B wellness offers a diversification path: "
            "higher contract values ($50-200K/year per enterprise), lower churn (annual "
            "contracts vs. monthly consumer), and no hardware dependency (app-only delivery). "
            "Conservative model: 10 enterprise accounts in Year 1 = $1-2M ARR. Aggressive "
            "model: 30 accounts = $4-6M ARR. Neither moves the needle on total revenue "
            "($2.6B annualized) but establishes a beachhead for a segment that could reach "
            "$50-100M within 3 years."
        ),
    },
    {
        "block_id": "requirements",
        "kind": "section",
        "heading": "Technical & Privacy Requirements",
        "body": (
            "Enterprise deployment requires capabilities that don't exist today: "
            "1. Multi-tenant data isolation — enterprise employee data must be segregated "
            "from consumer data and from other enterprise customers. "
            "2. SSO integration — SAML/OIDC for enterprise identity providers. "
            "3. Admin dashboards — aggregate participation and engagement metrics for HR "
            "teams without exposing individual employee workout data. "
            "4. HIPAA compliance — if positioned as wellness benefit, employer may require "
            "BAA (Business Associate Agreement). Priya Sharma's data governance framework "
            "is a prerequisite. "
            "5. Content licensing — some instructor content licenses may not cover B2B "
            "distribution. Legal review required."
        ),
    },
    {
        "block_id": "pilot-proposal",
        "kind": "section",
        "heading": "Pilot Proposal",
        "body": (
            "Proposed: 90-day pilot with 3 enterprise accounts (1 tech, 1 finance, 1 "
            "healthcare). App-only delivery (no hardware subsidy). Standard consumer IQ "
            "features plus basic admin dashboard (aggregate metrics only). "
            "Success metrics: >30% monthly active rate among enrolled employees, NPS >40, "
            "conversion to annual contract for all 3 accounts. "
            "Budget: $150K for pilot infrastructure, admin dashboard MVP, and dedicated "
            "account management. Approved by David Park with condition: Priya Sharma's data "
            "governance framework must be complete before employee data collection begins. "
            "Author: Raj Patel (Chief Product Officer)."
        ),
    },
]


async def seed_iq_artifacts(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed all Peloton IQ Next artifacts."""
    await _seed_artifact(
        session,
        bundle=bundle,
        key="iq-state-of-ai-report",
        title="Peloton IQ — State of AI Report",
        artifact_type="strategic_analysis",
        blocks=_STATE_OF_AI_BLOCKS,
        summary=(
            "Six-month assessment of Peloton IQ performance: Bedrock costs rising 23% QoQ, "
            "member engagement declining, data quality gap limiting recommendations. "
            "Recommends fine-tuning pilot and deeper wearable integration. Authored by James Liu."
        ),
    )
    await _seed_artifact(
        session,
        bundle=bundle,
        key="iq-brand-impact-assessment",
        title="Peloton IQ — Content & Brand Impact Assessment",
        artifact_type="brand_analysis",
        blocks=_BRAND_IMPACT_BLOCKS,
        summary=(
            "Member sentiment, instructor feedback, and industry AI quality context. "
            "Recommends slowing AI content expansion and investing in instructor-AI "
            "collaboration. Authored by Nicole Washington and Rachel Torres."
        ),
    )
    await _seed_artifact(
        session,
        bundle=bundle,
        key="iq-corporate-wellness-brief",
        title="AI Strategy — Corporate Wellness Expansion Brief",
        artifact_type="business_case",
        blocks=_CORPORATE_WELLNESS_BLOCKS,
        summary=(
            "B2B corporate wellness opportunity sizing and pilot proposal. 90-day pilot "
            "with 3 enterprise accounts, $150K budget. Approved with data governance "
            "condition. Authored by Raj Patel."
        ),
    )
```

- [ ] **Step 2: Verify module imports work**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/enriched-seed-data && python -c "from seed.peloton.iq_next.artifacts import seed_iq_artifacts; print('OK')"`

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add seed/peloton/iq_next/artifacts.py
git commit -m "feat(seed): add Peloton IQ Next artifacts — state of AI, brand impact, corporate wellness"
```

---

### Task 7: Peloton IQ Next — Decisions

Seed 3 decision records for the AI initiative.

**Files:**
- Create: `seed/peloton/iq_next/decisions.py`

- [ ] **Step 1: Create decisions.py**

Create `seed/peloton/iq_next/decisions.py`:

```python
"""Peloton IQ Next decision records."""

from __future__ import annotations

import logging
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.infrastructure.tables import (
    ArtifactRow,
    ArtifactSubjectRow,
    ArtifactVersionRow,
)
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_IQ_DECISION_NS = UUID("31d3e4f5-a6b7-4c8d-9e0f-1a2b3c4d5e6f")
_IQ_DECISION_VER_NS = UUID("41e4f5a6-b7c8-4d9e-0f1a-2b3c4d5e6f7a")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_DECISIONS = [
    {
        "key": "iq-model-strategy",
        "title": "IQ model strategy: continue Bedrock vs. invest in fine-tuning",
        "blocks": [
            {
                "block_id": "context",
                "kind": "context",
                "text": (
                    "Peloton IQ runs on Amazon Bedrock using GPT models for complex reasoning "
                    "and Llama 4 Scout for real-time insights. Inference costs are growing 23% "
                    "QoQ while subscriber count declines. James Liu's team ran an A/B test "
                    "showing +18% recommendation engagement with a fine-tuned prototype model "
                    "trained on Peloton's workout corpus. Fine-tuning would cost $2-3M upfront "
                    "plus 2 additional ML engineer headcount, but could reduce per-inference "
                    "cost by 40-60% and improve recommendation quality."
                ),
            },
            {
                "block_id": "decision",
                "kind": "decision",
                "text": "OPEN — James Liu's build-vs-buy analysis is under review.",
            },
            {
                "block_id": "positions",
                "kind": "rationale",
                "text": (
                    "James Liu (Head AI/ML): approve 6-month fine-tuning pilot. A/B results "
                    "prove the quality gap is solvable with better models. Cost savings recoup "
                    "investment within 12-18 months. This is the path to defensible AI. "
                    "Raj Patel (CPO): strongly supports. Fine-tuned models are the only way "
                    "to escape the 'generic AI' perception that's driving member complaints. "
                    "David Park (Interim CFO): need to see unit economics. $2-3M is real money "
                    "when stock is at $4. Show me the payback model with sensitivity analysis "
                    "on subscriber count scenarios. "
                    "Nicole Washington (VP Content): better models don't solve the 'robot "
                    "trainer' problem. The issue is that AI coaching lacks personality and "
                    "emotional connection, not that the recommendations are wrong."
                ),
            },
        ],
    },
    {
        "key": "iq-ai-content",
        "title": "AI-generated supplemental content: approve pilot",
        "blocks": [
            {
                "block_id": "context",
                "kind": "context",
                "text": (
                    "Proposal to use AI to generate supplemental workout content — warm-ups, "
                    "cool-downs, meditation scripts, stretching routines — at scale. This "
                    "would increase content library volume by 40-60% without proportional "
                    "instructor cost. The content team and instructors are unanimously opposed, "
                    "citing brand risk and quality concerns."
                ),
            },
            {
                "block_id": "decision",
                "kind": "decision",
                "text": (
                    "DECIDED: NO. Pilot not approved. The content team's unanimous opposition "
                    "and instructor retention risk outweigh the cost savings. Revisit only "
                    "after recommendation quality improves measurably and instructor-AI "
                    "collaboration model is tested."
                ),
            },
            {
                "block_id": "positions",
                "kind": "rationale",
                "text": (
                    "Nicole Washington (VP Content): blocked. AI-generated content is brand "
                    "poison. Our instructors ARE the brand. Industry context (Microsoft Copilot "
                    "reorg, 'AI slop' backlash) validates this concern. 3 marquee instructors "
                    "are already exploring competing offers — this would accelerate attrition. "
                    "Rachel Torres (Brand Marketing): blocked. Member sentiment data is clear — "
                    "'robot trainer' is already the #1 negative theme. Adding AI-generated "
                    "content doubles down on the wrong strategy. "
                    "Raj Patel (CPO): dissented. The content team's protectionism is preventing "
                    "us from exploring a legitimate efficiency gain. A limited pilot with "
                    "quality gates would have been low risk."
                ),
            },
        ],
    },
    {
        "key": "iq-wellness-pilot",
        "title": "Corporate wellness pilot: approve 3 enterprise accounts",
        "blocks": [
            {
                "block_id": "context",
                "kind": "context",
                "text": (
                    "Inbound corporate wellness inquiry volume is 4x expectations — 23 "
                    "enterprise accounts expressed interest in Q1 2026. Raj Patel proposes a "
                    "90-day pilot with 3 enterprise accounts (1 tech, 1 finance, 1 healthcare), "
                    "app-only delivery, $150K budget. Revenue diversification from declining "
                    "consumer hardware is the strategic driver."
                ),
            },
            {
                "block_id": "decision",
                "kind": "decision",
                "text": (
                    "DECIDED: YES, with conditions. Pilot approved at $150K budget. Condition: "
                    "Priya Sharma's data governance framework must be complete before any "
                    "enterprise employee data collection begins. SSO integration required for "
                    "all pilot accounts. Content licensing review must confirm B2B distribution "
                    "rights."
                ),
            },
            {
                "block_id": "positions",
                "kind": "rationale",
                "text": (
                    "Raj Patel (CPO): this is the most politically viable growth path. Demand "
                    "signal is strong, budget is modest, and it proves out the B2B model without "
                    "major infrastructure investment. "
                    "David Park (Interim CFO): approved with tight budget cap. $150K is "
                    "acceptable for a 90-day experiment. Do not exceed without board review. "
                    "Priya Sharma (Data & Privacy): support with conditions. Enterprise data "
                    "handling introduces HIPAA-adjacent obligations. Data governance framework "
                    "is non-negotiable prerequisite — it's the same framework blocking Pulse."
                ),
            },
        ],
    },
]


async def seed_iq_decisions(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Peloton IQ Next decision records."""
    hero = bundle.hero()
    for d in _DECISIONS:
        artifact_id = str(_uuid(_IQ_DECISION_NS, d["key"]))
        version_id = str(_uuid(_IQ_DECISION_VER_NS, d["key"]))

        stmt = pg_insert(ArtifactRow).values(
            id=artifact_id,
            organization_id=str(hero.id),
            artifact_type="decision_record",
            title=d["title"],
            visibility="shared",
            active_version_id=version_id,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[ArtifactRow.id],
            set_={"title": stmt.excluded.title, "active_version_id": stmt.excluded.active_version_id},
        )
        await session.execute(stmt)

        vstmt = pg_insert(ArtifactVersionRow).values(
            id=version_id,
            artifact_id=artifact_id,
            version_number=1,
            state="published",
            confidence_label="high",
            blocks=d["blocks"],
            citations=[],
            summary=d["title"],
            created_by=bundle.user.id,
        )
        vstmt = vstmt.on_conflict_do_update(
            index_elements=[ArtifactVersionRow.id],
            set_={"blocks": vstmt.excluded.blocks, "summary": vstmt.excluded.summary},
        )
        await session.execute(vstmt)

        sstmt = pg_insert(ArtifactSubjectRow).values(
            artifact_version_id=version_id,
            subject_type="company",
            subject_id=str(hero.id),
            role="subject",
            is_primary=True,
        )
        sstmt = sstmt.on_conflict_do_update(
            index_elements=[
                ArtifactSubjectRow.artifact_version_id,
                ArtifactSubjectRow.subject_type,
                ArtifactSubjectRow.subject_id,
            ],
            set_={"is_primary": sstmt.excluded.is_primary},
        )
        await session.execute(sstmt)
        logger.info("seeded iq_next decision: %s", d["key"])
```

- [ ] **Step 2: Commit**

```bash
git add seed/peloton/iq_next/decisions.py
git commit -m "feat(seed): add Peloton IQ Next decision records — model strategy, AI content, wellness pilot"
```

---

### Task 8: Peloton IQ Next — Action Items

Seed 8 action items for the AI initiative.

**Files:**
- Create: `seed/peloton/iq_next/action_items.py`

- [ ] **Step 1: Create action_items.py**

Create `seed/peloton/iq_next/action_items.py`:

```python
"""Peloton IQ Next action items."""

from __future__ import annotations

import logging
from datetime import date, datetime, timedelta, timezone
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.action_items.infrastructure.tables import ActionItemTable
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_IQ_ACTION_NS = UUID("51f5a6b7-c8d9-4e0f-1a2b-3c4d5e6f7a8b")
_IQ_DECISION_NS = UUID("31d3e4f5-a6b7-4c8d-9e0f-1a2b3c4d5e6f")

_TODAY = date.today()
_NOW = datetime.now(timezone.utc)


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_ACTION_ITEMS = [
    {
        "key": "iq-build-vs-buy",
        "title": "Deliver build-vs-buy model comparison with cost projections",
        "description": (
            "Complete the build-vs-buy analysis for Peloton IQ's model strategy. Must "
            "include: A/B test results from fine-tuned prototype (n=5K, +18% engagement), "
            "Bedrock cost trajectory with 12-month projection, fine-tuning compute cost "
            "estimate ($2-3M), headcount requirements (2 ML engineers), payback timeline "
            "with sensitivity analysis on subscriber count scenarios (as requested by "
            "David Park), and risk assessment for third-party model dependency."
        ),
        "owner_name": "James Liu",
        "owner_role": "Head of AI/ML",
        "priority": "high",
        "status": "pending_review",
        "due_offset_days": 14,
        "decision_key": "iq-model-strategy",
    },
    {
        "key": "iq-coaching-demo",
        "title": "Demo conversational coaching prototype to leadership",
        "description": (
            "Present the conversational coaching prototype to the leadership team. "
            "Prototype allows members to ask natural language questions about workout "
            "planning and receive personalized responses. Demo completed — reception was "
            "mixed: Raj Patel and James Liu enthusiastic, Nicole Washington skeptical of "
            "response quality, medical/legal flagged liability risk for injury-related "
            "advice queries."
        ),
        "owner_name": "Alex Kim",
        "owner_role": "Senior ML Engineer",
        "priority": "medium",
        "status": "completed",
        "due_offset_days": -7,
        "decision_key": None,
    },
    {
        "key": "iq-instructor-survey",
        "title": "Complete instructor sentiment survey on AI coaching",
        "description": (
            "Design and execute internal survey of Peloton instructors on AI coaching "
            "impact. Survey completed (45 instructors, 87% response rate). Key findings: "
            "62% concerned about AI replacing their roles, 71% report members asking about "
            "conflicting AI recommendations, 44% feel less valued since IQ launch. "
            "3 high-profile instructors exploring competing offers. Results included in "
            "Brand Impact Assessment artifact."
        ),
        "owner_name": "Nicole Washington",
        "owner_role": "VP Content & Programming",
        "priority": "medium",
        "status": "completed",
        "due_offset_days": -14,
        "decision_key": None,
    },
    {
        "key": "iq-member-feedback",
        "title": "Compile member feedback report on IQ recommendation quality",
        "description": (
            "Aggregate and analyze member feedback on Peloton IQ recommendation quality "
            "from all channels: in-app surveys (n=2,847), social media mentions (12K), "
            "community forum posts, and support tickets. Segment by member type (power "
            "user vs. casual) and identify actionable patterns. Include the viral forum "
            "thread ('I cancelled because the AI recommendations are garbage' — 4,200 "
            "upvotes, picked up by The Verge) as a case study."
        ),
        "owner_name": "Rachel Torres",
        "owner_role": "Brand Marketing Director",
        "priority": "high",
        "status": "sent_to_owner",
        "due_offset_days": 7,
        "decision_key": None,
    },
    {
        "key": "iq-wellness-pilot",
        "title": "Draft corporate wellness pilot proposal for 3 enterprise accounts",
        "description": (
            "Prepare detailed pilot proposal for 3 enterprise accounts (1 tech, 1 finance, "
            "1 healthcare vertical). Include: account selection criteria, app-only delivery "
            "spec, admin dashboard requirements, SSO integration scope, success metrics "
            "(>30% MAU, NPS >40, conversion to annual), content licensing review findings, "
            "and 90-day timeline. Budget capped at $150K per David Park."
        ),
        "owner_name": "Raj Patel",
        "owner_role": "Chief Product Officer",
        "priority": "medium",
        "status": "pending_review",
        "due_offset_days": 21,
        "decision_key": "iq-wellness-pilot",
    },
    {
        "key": "iq-data-governance",
        "title": "Publish data governance framework for biometric data collection",
        "description": (
            "Complete and publish the data governance framework covering: biometric data "
            "collection scope and consent requirements, retention and deletion policies, "
            "anonymization pipeline for ML training, HIPAA-adjacency assessment, state "
            "privacy law compliance (CCPA/CPRA, BIPA), enterprise data isolation model, "
            "and incident response procedures. This framework is a blocking dependency for "
            "both Project Pulse (wearable biometric data) and the corporate wellness pilot "
            "(enterprise employee data). Cannot proceed on either initiative without it."
        ),
        "owner_name": "Priya Sharma",
        "owner_role": "Head of Data & Privacy",
        "priority": "high",
        "status": "sent_to_owner",
        "due_offset_days": 28,
        "decision_key": None,
    },
    {
        "key": "iq-finetune-scope",
        "title": "Scope proprietary model fine-tuning — data requirements, compute cost, timeline",
        "description": (
            "Prepare detailed technical scoping document for proprietary model fine-tuning. "
            "Cover: training data requirements (500M+ workout records, anonymization "
            "pipeline), compute infrastructure (GPU cluster sizing for training runs), "
            "model architecture selection (base model for fine-tuning), evaluation framework "
            "(recommendation relevance benchmarks against Bedrock baseline), timeline to "
            "production quality (estimated 3-4 months), and ongoing retraining cadence."
        ),
        "owner_name": "James Liu",
        "owner_role": "Head of AI/ML",
        "priority": "medium",
        "status": "pending_review",
        "due_offset_days": 21,
        "decision_key": "iq-model-strategy",
    },
    {
        "key": "iq-bridge-integration",
        "title": "Evaluate deeper Apple Health / Garmin integration as data bridge",
        "description": (
            "Assess feasibility and scope of deepening Apple Health and Garmin Connect "
            "integrations to surface sleep, recovery, and activity data for IQ recommendations. "
            "Currently ~28% of members share wearable data. Evaluate: what additional data "
            "fields are available, API rate limits, member consent UX for expanded data "
            "sharing, expected recommendation quality improvement, and timeline. This bridges "
            "the data quality gap while Project Pulse develops (18+ months away). Note: the "
            "Pulse team (Maria Santos) has flagged that making third-party integration 'too "
            "good' may undermine the business case for the wearable."
        ),
        "owner_name": "Alex Kim",
        "owner_role": "Senior ML Engineer",
        "priority": "medium",
        "status": "sent_to_owner",
        "due_offset_days": 14,
        "decision_key": None,
    },
]


async def seed_iq_action_items(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Peloton IQ Next action items."""
    for item in _ACTION_ITEMS:
        item_id = _uuid(_IQ_ACTION_NS, item["key"])
        due = _TODAY + timedelta(days=item["due_offset_days"])

        citations: list[dict] = []
        if item["decision_key"]:
            decision_artifact_id = str(_uuid(_IQ_DECISION_NS, item["decision_key"]))
            citations.append({
                "type": "decision_record",
                "artifact_id": decision_artifact_id,
                "label": item["decision_key"],
            })

        stmt = pg_insert(ActionItemTable).values(
            id=item_id,
            owner_tenant_id=bundle.fund.id,
            conversation_id=None,
            owner_name=item["owner_name"],
            owner_role=item["owner_role"],
            title=item["title"],
            description=item["description"],
            rationale=None,
            priority=item["priority"],
            priority_score=None,
            priority_rationale=None,
            linked_focus_priorities=None,
            status=item["status"],
            due_date=due,
            deferred_until=None,
            deferred_reason=None,
            citations=citations,
            created_by=bundle.user.id,
            approved_by=bundle.user.id if item["status"] == "completed" else None,
            approved_at=_NOW if item["status"] == "completed" else None,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[ActionItemTable.id],
            set_={
                "title": stmt.excluded.title,
                "description": stmt.excluded.description,
                "status": stmt.excluded.status,
                "priority": stmt.excluded.priority,
                "due_date": stmt.excluded.due_date,
                "citations": stmt.excluded.citations,
            },
        )
        await session.execute(stmt)
        logger.info("seeded iq_next action item: %s", item["key"])
```

- [ ] **Step 2: Commit**

```bash
git add seed/peloton/iq_next/action_items.py
git commit -m "feat(seed): add Peloton IQ Next action items — 8 items across build/buy, content, wellness, governance"
```

---

### Task 9: Peloton IQ Next — KPIs, Findings, Anomalies

Seed operational KPI values, monitoring findings, and KPI anomalies for the AI initiative.

**Files:**
- Create: `seed/peloton/iq_next/kpis.py`
- Create: `seed/peloton/iq_next/kpi_values.yaml`
- Create: `seed/peloton/iq_next/findings.py`
- Create: `seed/peloton/iq_next/anomalies.py`

- [ ] **Step 1: Create kpi_values.yaml**

Create `seed/peloton/iq_next/kpi_values.yaml`:

```yaml
# Peloton IQ Next — operational KPI values.
# These are internal operational metrics, NOT historical EDGAR data.

definitions:
  - id: iq_bedrock_monthly_cost
    label: "IQ: Bedrock inference cost (monthly)"
    unit: usd_thousands
    description: Monthly Amazon Bedrock inference cost for Peloton IQ.

  - id: iq_recommendation_engagement
    label: "IQ: recommendation engagement rate"
    unit: percent
    description: Percentage of IQ-suggested workouts actually completed by members.

  - id: iq_member_nps_delta
    label: "IQ: member NPS delta (IQ vs. non-IQ)"
    unit: points
    description: NPS difference between IQ-active members and non-IQ members.

  - id: iq_ai_content_volume
    label: "IQ: AI-generated content volume (monthly)"
    unit: count
    description: Number of AI-generated supplemental content pieces per month.

  - id: iq_human_content_volume
    label: "IQ: human-produced content volume (monthly)"
    unit: count
    description: Number of human instructor-produced content pieces per month.

  - id: wellness_pipeline_value
    label: "Corporate wellness: pipeline value"
    unit: usd_thousands
    description: Total value of corporate wellness opportunities in sales pipeline.

  - id: wellness_pilot_conversions
    label: "Corporate wellness: pilot conversions"
    unit: count
    description: Number of corporate wellness pilot accounts converted.

periods:
  - label: "Oct 2025"
    start: 2025-10-01
    end: 2025-10-31
  - label: "Nov 2025"
    start: 2025-11-01
    end: 2025-11-30
  - label: "Dec 2025"
    start: 2025-12-01
    end: 2025-12-31
  - label: "Jan 2026"
    start: 2026-01-01
    end: 2026-01-31
  - label: "Feb 2026"
    start: 2026-02-01
    end: 2026-02-28
  - label: "Mar 2026"
    start: 2026-03-01
    end: 2026-03-31

values:
  iq_bedrock_monthly_cost:
    "Oct 2025": 340
    "Nov 2025": 355
    "Dec 2025": 368
    "Jan 2026": 380
    "Feb 2026": 398
    "Mar 2026": 418
  iq_recommendation_engagement:
    "Oct 2025": 34.2
    "Nov 2025": 31.8
    "Dec 2025": 29.1
    "Jan 2026": 26.5
    "Feb 2026": 24.8
    "Mar 2026": 22.0
  iq_member_nps_delta:
    "Oct 2025": 2
    "Nov 2025": 0
    "Dec 2025": -2
    "Jan 2026": -4
    "Feb 2026": -5
    "Mar 2026": -6
  iq_ai_content_volume:
    "Jan 2026": 45
    "Feb 2026": 62
    "Mar 2026": 78
  iq_human_content_volume:
    "Jan 2026": 320
    "Feb 2026": 315
    "Mar 2026": 308
  wellness_pipeline_value:
    "Jan 2026": 280
    "Feb 2026": 520
    "Mar 2026": 1150
  wellness_pilot_conversions:
    "Mar 2026": 0
```

- [ ] **Step 2: Create kpis.py**

Create `seed/peloton/iq_next/kpis.py`:

```python
"""Peloton IQ Next KPI definitions and values."""

from __future__ import annotations

import logging
from decimal import Decimal
from pathlib import Path
from uuid import UUID, uuid5

import yaml
from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.documents.infrastructure.tables import (
    KpiDefinitionTable,
    KpiValueTable,
)
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_IQ_KPI_NS = UUID("61a6b7c8-d9e0-4f1a-2b3c-4d5e6f7a8b9c")
_YAML_PATH = Path(__file__).resolve().parent / "kpi_values.yaml"


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


async def seed_iq_kpis(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Peloton IQ Next KPI definitions and operational values."""
    data = yaml.safe_load(_YAML_PATH.read_text(encoding="utf-8"))
    hero = bundle.hero()

    for d in data["definitions"]:
        stmt = pg_insert(KpiDefinitionTable).values(
            id=d["id"],
            label=d["label"],
            unit=d["unit"],
            description=d.get("description"),
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[KpiDefinitionTable.id],
            set_={
                "label": stmt.excluded.label,
                "unit": stmt.excluded.unit,
                "description": stmt.excluded.description,
            },
        )
        await session.execute(stmt)

    periods = {
        row["label"]: (row["start"], row["end"]) for row in data["periods"]
    }

    for kpi_id, period_values in data["values"].items():
        for period_label, value in period_values.items():
            start, end = periods[period_label]
            row_id = _uuid(_IQ_KPI_NS, f"{hero.id}:{kpi_id}:{period_label}")
            stmt = pg_insert(KpiValueTable).values(
                id=row_id,
                company_id=hero.id,
                kpi_id=kpi_id,
                period_start=start,
                period_end=end,
                period_label=period_label,
                value=Decimal(str(value)),
            )
            stmt = stmt.on_conflict_do_update(
                index_elements=[KpiValueTable.id],
                set_={"value": stmt.excluded.value},
            )
            await session.execute(stmt)

    logger.info("seeded iq_next KPIs: %d definitions, %d values",
                len(data["definitions"]),
                sum(len(v) for v in data["values"].values()))
```

- [ ] **Step 3: Create findings.py**

Create `seed/peloton/iq_next/findings.py`:

```python
"""Peloton IQ Next monitoring findings."""

from __future__ import annotations

import logging
from datetime import datetime, timedelta, timezone
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.monitoring.infrastructure.tables import MonitorFindingTable
from seed.companies import SeedBundle
from seed.focus import _MONITOR_TARGET_NS

logger = logging.getLogger(__name__)

_IQ_FINDING_NS = UUID("71b7c8d9-e0f1-4a2b-3c4d-5e6f7a8b9c0d")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_FINDINGS = [
    {
        "key": "apple-watch-hrv",
        "target_slug": "apple_fitness",
        "finding_type": "product_launch",
        "title": "Apple Watch adds continuous HRV and recovery scoring in watchOS 13",
        "summary": (
            "Apple announced continuous HRV monitoring and AI-powered recovery scoring "
            "in watchOS 13, available on Apple Watch Series 10 and Ultra 3. Recovery "
            "scores are integrated with Apple Fitness+ workout recommendations. This "
            "narrows Project Pulse's differentiation — Apple is offering the same "
            "biometric-to-coaching pipeline that Pulse aims to build, but within an "
            "ecosystem that 28% of Peloton members already use."
        ),
        "severity": "important",
        "hours_ago": 96,
    },
    {
        "key": "oura-gen4-launch",
        "target_slug": "peloton",
        "finding_type": "product_launch",
        "title": "Oura Ring Gen 4 launches with AI-powered health insights",
        "summary": (
            "Oura launched Gen 4 ring with expanded AI health insights including "
            "stress detection, illness prediction, and personalized recovery "
            "recommendations. The ring form factor limits real-time workout tracking "
            "but excels at passive health monitoring — exactly the 'always-on wellness "
            "data' positioning that Project Pulse targets."
        ),
        "severity": "notable",
        "hours_ago": 120,
    },
    {
        "key": "msft-copilot-reorg",
        "target_slug": "peloton",
        "finding_type": "industry_trend",
        "title": "Microsoft reorganizes Copilot division over AI quality and 'slop' concerns",
        "summary": (
            "Microsoft announced a major reorganization of its Copilot division, citing "
            "uneven adoption rates, escalating AI infrastructure costs, and growing public "
            "concern over AI-generated content quality — the 'AI slop' phenomenon. The "
            "reorganization signals that even the largest AI investors are struggling with "
            "quality and ROI. Directly relevant to Peloton IQ: validates Nicole Washington's "
            "concerns about AI content quality and Rachel Torres' brand risk assessment."
        ),
        "severity": "notable",
        "hours_ago": 48,
    },
    {
        "key": "pwc-ceo-survey-ai",
        "target_slug": "peloton",
        "finding_type": "industry_trend",
        "title": "PwC CEO Survey: 56% report getting 'nothing' from AI investments",
        "summary": (
            "PwC's 2026 Global CEO Survey reports that 56% of CEOs say they are getting "
            "'nothing' from their AI adoption efforts, despite significant investment. "
            "Only 29% see meaningful ROI from generative AI. David Park has cited this "
            "finding in questioning the IQ expansion budget. James Liu counters that "
            "Peloton has unique advantages (proprietary workout data, direct member "
            "relationship) that most enterprises lack."
        ),
        "severity": "notable",
        "hours_ago": 72,
    },
    {
        "key": "deloitte-ai-adoption",
        "target_slug": "peloton",
        "finding_type": "industry_trend",
        "title": "Deloitte: 79% of enterprises face AI adoption challenges, up from 2025",
        "summary": (
            "Deloitte's 2026 State of AI report finds 79% of organizations face AI adoption "
            "challenges — a double-digit increase from 2025. 54% of C-suite executives say "
            "AI adoption is 'tearing their company apart.' The talent gap is cited as the "
            "#1 barrier. Peloton's internal AI debate mirrors the industry pattern: "
            "champions vs. skeptics, cost pressure, quality concerns, organizational tension."
        ),
        "severity": "notable",
        "hours_ago": 96,
    },
    {
        "key": "member-forum-viral",
        "target_slug": "peloton",
        "finding_type": "customer_signal",
        "title": "Peloton member forum thread goes viral: 'I cancelled because the AI recommendations are garbage'",
        "summary": (
            "A member forum thread criticizing Peloton IQ recommendation quality reached "
            "4,200 upvotes and was picked up by The Verge. The post details a long-time "
            "member's frustration with generic workout recommendations that 'don't know me "
            "after 3 years of data.' Responses include 200+ members sharing similar "
            "experiences. Rachel Torres has escalated this to the leadership team as "
            "evidence of brand damage from the AI-forward strategy."
        ),
        "severity": "important",
        "hours_ago": 12,
    },
]


async def seed_iq_findings(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Peloton IQ Next monitoring findings."""
    fund = bundle.fund
    now = datetime.now(timezone.utc)

    # Build target ID lookup — reuse existing monitor targets from focus.py
    target_ids = {
        "peloton": uuid5(_MONITOR_TARGET_NS, "peloton:headspace"),
        "apple_fitness": uuid5(_MONITOR_TARGET_NS, "apple_fitness:headspace"),
    }

    for finding in _FINDINGS:
        fid = _uuid(_IQ_FINDING_NS, finding["key"])
        target_id = target_ids.get(finding["target_slug"], target_ids["peloton"])
        discovered = now - timedelta(hours=finding["hours_ago"])

        stmt = pg_insert(MonitorFindingTable).values(
            id=fid,
            run_id=None,
            target_id=target_id,
            owner_tenant_id=fund.id,
            finding_type=finding["finding_type"],
            title=finding["title"],
            summary=finding["summary"],
            severity=finding["severity"],
            occurred_at=discovered,
            discovered_at=discovered,
            source_url=None,
            raw_payload=None,
            processed=False,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[MonitorFindingTable.id],
            set_={
                "title": stmt.excluded.title,
                "summary": stmt.excluded.summary,
                "severity": stmt.excluded.severity,
                "discovered_at": stmt.excluded.discovered_at,
            },
        )
        await session.execute(stmt)
        logger.info("seeded iq_next finding: %s", finding["key"])
```

- [ ] **Step 4: Create anomalies.py**

Create `seed/peloton/iq_next/anomalies.py`:

```python
"""Peloton IQ Next KPI anomalies."""

from __future__ import annotations

import logging
from datetime import date
from decimal import Decimal
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.kpis.infrastructure.tables import KpiAnomalyRow, KpiValueRow
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_IQ_ANOMALY_NS = UUID("81c8d9e0-f1a2-4b3c-4d5e-6f7a8b9c0d1e")
_IQ_ANOMALY_VALUE_NS = UUID("91d9e0f1-a2b3-4c4d-5e6f-7a8b9c0d1e2f")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_ANOMALIES = [
    {
        "key": "iq-engagement-drop",
        "kpi_id": "iq_recommendation_engagement",
        "anomaly_type": "sudden_change",
        "severity": "important",
        "description": (
            "IQ recommendation engagement rate dropped from 26.5% to 22.0% in March — "
            "a 12% MoM decline and the steepest monthly drop since launch. The rate has "
            "fallen continuously from 34.2% at launch (Oct 2025) to 22.0% now. Members "
            "are increasingly ignoring IQ-suggested workouts. Casual users (1-2 workouts/ "
            "week) show the steepest decline — the cold-start problem means IQ never "
            "becomes useful for them."
        ),
        "enrichment_question": (
            "Is the engagement decline driven by recommendation quality degradation, "
            "feature fatigue, seasonal patterns, or member self-selection (engaged members "
            "leaving)? What is the correlation between engagement drop and the subscriber "
            "churn trend?"
        ),
        "period_label": "Mar 2026",
        "value": Decimal("22.0"),
        "unit": "percent",
    },
    {
        "key": "wellness-pipeline-spike",
        "kpi_id": "wellness_pipeline_value",
        "anomaly_type": "sudden_change",
        "severity": "notable",
        "description": (
            "Corporate wellness pipeline value surged from $520K to $1,150K in March — "
            "a 121% MoM increase and 4x the January baseline. 23 enterprise accounts have "
            "expressed interest without active outbound sales. The demand signal is "
            "significantly stronger than the business case projected."
        ),
        "enrichment_question": (
            "Is the pipeline spike driven by organic inbound or a specific trigger (press "
            "coverage, referral, event)? What is the breakdown by vertical (tech, finance, "
            "healthcare)? Are these serious buyers or tire-kickers?"
        ),
        "period_label": "Mar 2026",
        "value": Decimal("1150"),
        "unit": "usd_thousands",
    },
]


async def seed_iq_anomalies(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed Peloton IQ Next KPI anomalies."""
    hero = bundle.hero()
    org_id = str(hero.id)

    for a in _ANOMALIES:
        kpi_value_id = str(_uuid(
            _IQ_ANOMALY_VALUE_NS,
            f"{org_id}:{a['kpi_id']}:{a['period_label']}",
        ))

        vstmt = pg_insert(KpiValueRow).values(
            id=kpi_value_id,
            organization_id=org_id,
            kpi_id=a["kpi_id"],
            period_label=a["period_label"],
            period_start=date(2026, 3, 1),
            period_end=date(2026, 3, 31),
            value=a["value"],
            unit=a["unit"],
            commentary=None,
            reported_by="seed",
        )
        vstmt = vstmt.on_conflict_do_update(
            index_elements=[KpiValueRow.id],
            set_={"value": vstmt.excluded.value, "unit": vstmt.excluded.unit},
        )
        await session.execute(vstmt)

        anomaly_id = str(_uuid(_IQ_ANOMALY_NS, a["key"]))
        astmt = pg_insert(KpiAnomalyRow).values(
            id=anomaly_id,
            kpi_value_id=kpi_value_id,
            organization_id=org_id,
            anomaly_type=a["anomaly_type"],
            description=a["description"],
            severity=a["severity"],
            enrichment_question=a["enrichment_question"],
            enrichment_response=None,
        )
        astmt = astmt.on_conflict_do_update(
            index_elements=[KpiAnomalyRow.id],
            set_={
                "description": astmt.excluded.description,
                "severity": astmt.excluded.severity,
                "enrichment_question": astmt.excluded.enrichment_question,
            },
        )
        await session.execute(astmt)
        logger.info("seeded iq_next anomaly: %s", a["key"])
```

- [ ] **Step 5: Commit**

```bash
git add seed/peloton/iq_next/kpi_values.yaml seed/peloton/iq_next/kpis.py seed/peloton/iq_next/findings.py seed/peloton/iq_next/anomalies.py
git commit -m "feat(seed): add Peloton IQ Next KPIs, monitoring findings, and anomalies"
```

---

### Task 10: Integration Module

Create the cross-reference module that connects new initiative data to existing seed entities.

**Files:**
- Create: `seed/peloton/integration.py`

- [ ] **Step 1: Create integration.py**

Create `seed/peloton/integration.py`:

```python
"""Cross-references between initiative data and existing seed entities.

Seeds observation records that add initiative context to existing KPI anomalies
and decisions, connecting the new strategic work to the existing data without
modifying any existing seed files.
"""

from __future__ import annotations

import logging
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.intelligence.infrastructure.tables import ObservationTable
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_INTEGRATION_NS = UUID("a0b1c2d3-e4f5-4a6b-7c8d-9e0f1a2b3c4d")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


_OBSERVATIONS = [
    {
        "key": "cf-gross-margin-pulse-context",
        "title": "Connected Fitness gross margin decline includes Pulse feasibility costs",
        "summary": (
            "The negative Connected Fitness gross margin (-8.2%) includes ~$1.2M in "
            "Project Pulse feasibility costs allocated to the hardware division in Q2. "
            "While small relative to total hardware COGS, this is a new and growing R&D "
            "line item. As Pulse progresses toward manufacturing commitment ($15-20M), "
            "this margin pressure will intensify unless the budget is reclassified to "
            "a dedicated R&D line."
        ),
        "severity": "notable",
        "priority_score": 0.65,
        "priority_rationale": (
            "Contextualizes existing CF gross margin anomaly with forward-looking "
            "Project Pulse cost implications."
        ),
        "linked_priorities": [
            "Drive subscription growth via App tier and corporate wellness",
        ],
        "deprioritized": False,
        "trend_direction": "negative",
        "requires_decision": False,
    },
    {
        "key": "fcf-pulse-budget-tension",
        "title": "FCF improvement at risk if Pulse manufacturing commitment proceeds",
        "summary": (
            "Peloton's FCF improvement (driven by cost discipline and workforce reduction) "
            "is the one bright spot in Q2 FY2026 results. However, the Project Pulse $45M "
            "budget allocation — with $15-20M in manufacturing commitments ahead — threatens "
            "to reverse this progress. David Park (Interim CFO) is using this tension to "
            "argue for deferring Pulse commitments. Maria Santos counters that the launch "
            "window is fixed and delay means cancellation."
        ),
        "severity": "important",
        "priority_score": 0.80,
        "priority_rationale": (
            "Connects existing FCF improvement trend to upcoming Pulse budget decision. "
            "The board will weigh this trade-off at the Q3 gate review."
        ),
        "linked_priorities": [
            "Drive subscription growth via App tier and corporate wellness",
        ],
        "deprioritized": False,
        "trend_direction": "negative",
        "requires_decision": True,
    },
    {
        "key": "churn-iq-quality-link",
        "title": "Subscriber churn may be amplified by IQ recommendation quality issues",
        "summary": (
            "The subscriber churn trend (2.66M connected fitness subscribers, down 7% YoY) "
            "coincides with declining IQ recommendation engagement (34% → 22% since launch). "
            "While churn has multiple drivers (price increase, macro headwinds, competitive "
            "alternatives), the member feedback data suggests IQ quality issues are a "
            "contributing factor — the viral forum thread ('I cancelled because of AI') and "
            "the negative NPS delta for IQ users (-6 points vs. non-IQ) point to causation, "
            "not just correlation. Nicole Washington argues this validates slowing the AI "
            "push; James Liu argues it validates investing in better models."
        ),
        "severity": "important",
        "priority_score": 0.85,
        "priority_rationale": (
            "Links existing subscriber churn anomaly to IQ quality data. Both sides of the "
            "AI debate use this data to support opposite conclusions — an operator needs to "
            "see both interpretations."
        ),
        "linked_priorities": [
            "Drive subscription growth via App tier and corporate wellness",
        ],
        "deprioritized": False,
        "trend_direction": "negative",
        "requires_decision": True,
    },
    {
        "key": "workforce-reduction-funded-initiatives",
        "title": "2024 workforce reduction freed budget now funding Pulse and IQ Next",
        "summary": (
            "The 15% workforce reduction (completed 2024) generated ~$120M in annualized "
            "savings, contributing to the 39% adjusted EBITDA improvement. A portion of "
            "these savings is funding Project Pulse ($45M allocation) and IQ Next expansion "
            "(Bedrock costs, ML team). The operational efficiency gains are being reinvested "
            "in strategic bets — whether this is smart capital allocation or spending the "
            "cushion depends on which bets pay off."
        ),
        "severity": "notable",
        "priority_score": 0.60,
        "priority_rationale": (
            "Contextualizes existing workforce reduction decision with forward-looking "
            "budget allocation decisions."
        ),
        "linked_priorities": [],
        "deprioritized": False,
        "trend_direction": "flat",
        "requires_decision": False,
    },
]


async def seed_initiative_integration(
    session: AsyncSession,
    *,
    bundle: SeedBundle,
) -> None:
    """Seed cross-reference observations linking initiative data to existing entities."""
    hero = bundle.hero()
    fund = bundle.fund
    user = bundle.user

    for obs in _OBSERVATIONS:
        stmt = pg_insert(ObservationTable).values(
            id=_uuid(_INTEGRATION_NS, f"{hero.slug}:{obs['key']}"),
            owner_tenant_id=fund.id,
            subject_company_id=hero.id,
            title=obs["title"],
            summary=obs["summary"],
            severity=obs["severity"],
            status="approved",
            priority_score=obs["priority_score"],
            priority_rationale=obs["priority_rationale"],
            linked_focus_priorities=obs["linked_priorities"] or None,
            deprioritized=obs["deprioritized"],
            trend_direction=obs["trend_direction"],
            requires_decision=obs["requires_decision"],
            citations=None,
            created_by=user.id,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[ObservationTable.id],
            set_={
                "title": stmt.excluded.title,
                "summary": stmt.excluded.summary,
                "severity": stmt.excluded.severity,
                "priority_score": stmt.excluded.priority_score,
                "priority_rationale": stmt.excluded.priority_rationale,
                "linked_focus_priorities": stmt.excluded.linked_focus_priorities,
                "deprioritized": stmt.excluded.deprioritized,
                "trend_direction": stmt.excluded.trend_direction,
                "requires_decision": stmt.excluded.requires_decision,
            },
        )
        await session.execute(stmt)
        logger.info("seeded integration observation: %s", obs["key"])
```

- [ ] **Step 2: Commit**

```bash
git add seed/peloton/integration.py
git commit -m "feat(seed): add initiative integration — cross-reference observations linking new and existing data"
```

---

### Task 11: Wire Into Loader

Add the initiative seed functions to the main loader so they execute after the base seed completes.

**Files:**
- Modify: `seed/loader.py` — add `seed_initiatives()` call at end of `_run()`

- [ ] **Step 1: Read the end of _run() to find the insertion point**

Run: `grep -n "seed_kpi_anomalies\|sync_legacy\|dual_write\|return\|initiative" /home/rhallman/Projects/prescient_os/.worktrees/enriched-seed-data/seed/loader.py | tail -20`

Identify the exact line after the final existing seed call and before the return statement.

- [ ] **Step 2: Add initiative seed imports**

Add to the imports section of `seed/loader.py` (near the other seed imports):

```python
from seed.peloton.project_pulse import seed_project_pulse
from seed.peloton.iq_next import seed_iq_next
from seed.peloton.integration import seed_initiative_integration
```

- [ ] **Step 3: Add seed_initiatives() call**

Add after the existing `seed_kpi_anomalies` call and before the dual-write bridge (or at the end of the focus discipline block), gated by an environment variable:

```python
        # ── Initiative seed data (Project Pulse + IQ Next) ───────────
        if not os.environ.get("SEED_SKIP_INITIATIVES"):
            logger.info("seeding initiative data …")
            await seed_project_pulse(session, bundle=bundle)
            await seed_iq_next(session, bundle=bundle)
            await seed_initiative_integration(session, bundle=bundle)
            logger.info("initiative seed complete")
```

Also add `import os` at the top of the file if not already present.

- [ ] **Step 4: Verify the seed still loads without errors**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/enriched-seed-data && python -c "from seed.loader import main; print('loader imports OK')"`

Expected: `loader imports OK` (or similar — just verifying imports resolve)

- [ ] **Step 5: Commit**

```bash
git add seed/loader.py
git commit -m "feat(seed): wire initiative modules into loader — Project Pulse, IQ Next, integration"
```

---

### Task 12: Integration Test — Full Seed Run

Run the full seed against a live database to verify everything works.

**Files:** None (verification only)

- [ ] **Step 1: Start infrastructure**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/enriched-seed-data && make up`

Wait for Postgres, OpenSearch, and Redis to be healthy.

- [ ] **Step 2: Run migrations**

Run: `make migrate`

Expected: migrations complete successfully.

- [ ] **Step 3: Run full seed**

Run: `make docker-seed-full`

Expected: seed completes without errors. Look for log lines confirming initiative data was seeded:
- `seeded pulse artifact: pulse-product-brief`
- `seeded pulse decision: pulse-manufacturing`
- `seeded pulse action item: pulse-sensor-shortlist`
- `seeded iq_next artifact: iq-state-of-ai-report`
- `seeded iq_next decision: iq-model-strategy`
- `seeded iq_next action item: iq-build-vs-buy`
- `seeded integration observation: cf-gross-margin-pulse-context`

- [ ] **Step 4: Run seed verification**

Run: `make seed-verify`

Expected: verification passes with increased entity counts.

- [ ] **Step 5: Test idempotency — run seed again**

Run: `make docker-seed-full`

Expected: completes without errors, no duplicate key violations. All upserts should resolve cleanly.

- [ ] **Step 6: Test skip flag**

Run: `SEED_SKIP_INITIATIVES=1 make docker-seed-full`

Expected: seed completes without initiative data log lines. Base seed still works independently.

- [ ] **Step 7: Commit any fixes**

If any issues were discovered and fixed during testing:

```bash
git add -u
git commit -m "fix(seed): resolve issues found during integration testing"
```
