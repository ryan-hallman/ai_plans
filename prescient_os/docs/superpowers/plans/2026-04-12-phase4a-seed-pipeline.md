# Phase 4a: Peloton Seed Pipeline

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Olaplex demo seed with a deeply seeded Peloton Interactive dataset — real EDGAR filings, earnings transcript segments, competitor profiles, LLM-generated strategic artifacts, decision records, action items, KPI anomalies, and monitoring findings. Dual-write to both legacy and new tables.

**Architecture:** Extend the existing seed pipeline (`seed/` directory) following established patterns: deterministic UUIDs, SQLAlchemy Core upserts, per-company EDGAR caching, OpenSearch BM25 indexing. Add a new transcript fetcher module. LLM-generated content uses the Anthropic adapter at seed time. Dual-write bridge ensures both legacy (`knowledge.*`, `documents.*`) and new (`artifacts.*`, `kpis.*`) tables are populated.

**Tech Stack:** Python 3.12, SQLAlchemy 2.0 (async), httpx, Anthropic SDK, PyYAML, existing seed infrastructure (EdgarClient, parser, indexer).

**Spec:** `docs/superpowers/specs/2026-04-12-phase4-demo-design.md` — Section 2

---

## File Structure

### New Files

```
seed/
├── peloton/
│   ├── kpi_values.yaml              # 12 KPIs × 8-12 quarters of real Peloton data
│   ├── transcripts/                  # Cached earnings call transcript segments
│   │   └── (fetched at seed time, cached locally)
│   └── artifacts.yaml               # Strategic artifact content (blocks, summaries)
├── competitors/
│   ├── apple_fitness.yaml            # Apple Fitness+ competitor profile data
│   ├── lululemon_studio.yaml         # Lululemon Studio competitor profile data
│   ├── echelon.yaml                  # Echelon competitor profile data
│   └── ifit.yaml                     # iFit/NordicTrack competitor profile data
├── transcripts.py                    # Earnings transcript fetcher + parser
├── scenarios.py                      # Decision records, action items, KPI anomalies
└── dual_write.py                     # Bridge: write to both legacy and new tables
```

### Modified Files

```
seed/
├── companies.yaml                    # Replace Olaplex/Meridian with Peloton + competitors
├── kpi_definitions.yaml              # Add subscriber metrics (churn, ARPU, etc.)
├── config.py                         # Peloton CIK, updated filing counts
├── companies.py                      # Update SeedBundle for new company structure
├── loader.py                         # Wire new seeders, update orchestration
├── focus.py                          # Replace Olaplex artifacts with Peloton
├── seed_demo_user.py                 # Peloton org + Sarah Chen user
```

---

## Task 1: Update Company Graph Configuration

**Files:**
- Modify: `seed/companies.yaml`
- Modify: `seed/config.py`
- Modify: `seed/companies.py`

- [ ] **Step 1: Replace companies.yaml with Peloton ecosystem**

Replace the entire contents of `seed/companies.yaml`:

```yaml
# Seed data for the Prescient OS demo — Peloton Interactive.
#
# The demo runs as Peloton Interactive (independent company) with Sarah Chen
# as CEO. Peloton is the hero company (complete dataset: filings + transcripts
# + KPIs + artifacts + scenarios). Four competitors are seeded with profiles
# and monitoring findings.
#
# CIKs come from SEC EDGAR. Zero-padding is preserved as a string to match
# the /submissions/CIK{10-digit}.json URL format.

# No fund in operator-first mode — Peloton is independent.
# The fund field is retained for backward compat with the tenants schema
# but represents the company itself for tenancy purposes.
fund:
  slug: peloton
  name: Peloton Interactive
  fund_type: equity  # reused as org type placeholder

user:
  id: sarah
  display_name: Sarah Chen
  email: sarah@onepeloton.com
  role: operator

companies:
  # ------- hero company -------
  - slug: peloton
    name: Peloton Interactive
    legal_name: Peloton Interactive, Inc.
    ticker: PTON
    cik: "0001639825"
    domain: onepeloton.com
    sector: Connected Fitness / Consumer Technology
    entity_type: public
    engagement_role: portco
    hero: true

  # ------- competitors -------
  - slug: apple_fitness
    name: Apple Fitness+
    legal_name: Apple Inc.
    ticker: AAPL
    cik: "0000320193"
    domain: apple.com/apple-fitness-plus
    sector: Technology / Digital Fitness
    entity_type: public
    engagement_role: null
    hero: false

  - slug: lululemon_studio
    name: Lululemon Studio
    legal_name: Lululemon Athletica Inc.
    ticker: LULU
    cik: "0001397187"
    domain: lululemon.com
    sector: Athletic Apparel / Connected Fitness
    entity_type: public
    engagement_role: null
    hero: false

  - slug: echelon
    name: Echelon Fitness
    legal_name: Echelon Fit US LLC
    ticker: null
    cik: null
    domain: echelonfit.com
    sector: Budget Connected Fitness
    entity_type: private
    engagement_role: null
    hero: false

  - slug: ifit
    name: iFit / NordicTrack
    legal_name: iFit Inc.
    ticker: null
    cik: null
    domain: ifit.com
    sector: Connected Fitness
    entity_type: private
    engagement_role: null
    hero: false

relationship_types:
  - slug: competitor
    label: Competitor
    category: market
    reverse_label: Competitor of
  - slug: ecosystem_threat
    label: Ecosystem Threat
    category: market
    reverse_label: Ecosystem threat to
  - slug: cautionary_parallel
    label: Cautionary Parallel
    category: market
    reverse_label: Cautionary parallel of

relationships:
  - source: peloton
    target: apple_fitness
    type: ecosystem_threat
  - source: peloton
    target: lululemon_studio
    type: cautionary_parallel
  - source: peloton
    target: echelon
    type: competitor
  - source: peloton
    target: ifit
    type: competitor
```

- [ ] **Step 2: Update config.py with Peloton CIK and expanded filing counts**

In `seed/config.py`, update the filing counts to pull more data:

```python
# Filing ingestion limits for the demo.
# Hero gets 3 years of annuals + 8 quarters + key 8-Ks.
HERO_COUNTS = {"10-K": 3, "10-Q": 8, "8-K": 6}
# Competitors with CIKs get minimal filings for profile context.
COMPETITOR_COUNTS = {"10-K": 1, "10-Q": 2}
```

- [ ] **Step 3: Update companies.py for new company structure**

The `SeedBundle` class needs minor updates to handle the new YAML structure. The `hero()` method should return Peloton. Verify the `CompanyConfig` dataclass handles `engagement_role: null` companies.

- [ ] **Step 4: Run the company graph seeder to verify YAML loads**

```bash
make docker-seed
```

Check that Peloton and 4 competitors appear in the `companies.companies` table.

- [ ] **Step 5: Commit**

```bash
git add seed/companies.yaml seed/config.py seed/companies.py
git commit -m "feat(seed): replace Olaplex with Peloton company graph and competitors"
```

---

## Task 2: Update Demo User and Organization

**Files:**
- Modify: `seed/seed_demo_user.py`

- [ ] **Step 1: Update seed_demo_user.py for Peloton + Sarah Chen**

Replace the Olaplex/Meridian demo user setup with:

- **Organization:** "Peloton Interactive", type `independent`, slug `peloton`
- **User:** Sarah Chen, `sarah@onepeloton.com`, role `operator`
- **Credentials:** password `demo` (hashed via `hash_password`)
- **Membership:** Sarah → Peloton, scope `full`

Follow the existing pattern: async SQLAlchemy ORM, idempotent upserts, deterministic UUIDs.

Also seed a second organization + user for the PE angle if needed later:
- **Organization:** "Summit Partners", type `pe_firm`
- **User:** Michael Torres, `michael@summit.com`, role `operating_partner`
- **Portfolio link:** Summit → Peloton, billing model `pe_funded`, visibility Tier B

- [ ] **Step 2: Run and verify**

```bash
make docker-seed
```

Verify login works: `curl -X POST http://localhost:8000/auth/login -H "Content-Type: application/json" -d '{"email":"sarah@onepeloton.com","password":"demo"}'`

- [ ] **Step 3: Commit**

```bash
git add seed/seed_demo_user.py
git commit -m "feat(seed): add Peloton org and Sarah Chen demo user"
```

---

## Task 3: Expand KPI Definitions and Seed Peloton KPI Values

**Files:**
- Modify: `seed/kpi_definitions.yaml`
- Create: `seed/peloton/kpi_values.yaml`

- [ ] **Step 1: Add subscriber and fitness-specific KPI definitions**

Add to `seed/kpi_definitions.yaml`:

```yaml
  - id: connected_fitness_subscribers
    label: Connected fitness subscribers
    unit: thousands
    description: Total paid connected fitness subscribers at period end.

  - id: monthly_churn_rate
    label: Monthly churn rate
    unit: percent
    description: Average monthly connected fitness subscription churn rate.

  - id: subscription_arpu
    label: Subscription ARPU
    unit: USD
    description: Average monthly net subscription revenue per connected fitness subscriber.

  - id: revenue_connected_fitness
    label: Revenue — Connected Fitness
    unit: USD_millions
    description: Revenue from connected fitness product sales (hardware).

  - id: revenue_subscription
    label: Revenue — Subscription
    unit: USD_millions
    description: Revenue from subscription services.

  - id: gross_margin_connected_fitness
    label: Gross margin % — Connected Fitness
    unit: percent
    description: Gross margin on connected fitness hardware segment.

  - id: gross_margin_subscription
    label: Gross margin % — Subscription
    unit: percent
    description: Gross margin on subscription segment.

  - id: hardware_units_sold
    label: Hardware units sold
    unit: thousands
    description: Connected fitness products delivered in the period.
```

- [ ] **Step 2: Create Peloton KPI values from real filings**

Create `seed/peloton/kpi_values.yaml` with real data extracted from Peloton's 10-K and 10-Q filings. Structure follows the existing pattern (see `seed/olaplex/kpi_values.yaml`):

```yaml
# Peloton KPI values — extracted from SEC filings (10-K, 10-Q).
# FY ends June 30. Quarters: Q1=Jul-Sep, Q2=Oct-Dec, Q3=Jan-Mar, Q4=Apr-Jun.
#
# Source: Peloton Interactive 10-K (FY2023, FY2024) and 10-Q filings.

values:
  # ---- Revenue (total) ----
  - kpi_id: revenue
    period_label: "FY2023 Q1"
    period_start: "2022-07-01"
    period_end: "2022-09-30"
    value: 616.5

  - kpi_id: revenue
    period_label: "FY2023 Q2"
    period_start: "2022-10-01"
    period_end: "2022-12-31"
    value: 792.7

  # ... continue for all KPIs × 8-12 quarters
  # The implementer must extract actual values from Peloton's SEC filings.
  # CIK 0001639825. Use the EDGAR client to fetch filings, then extract
  # financial data from the MD&A and financial statements sections.

  # ---- Subscriber churn (synthetic for recent quarters) ----
  # Peloton reports "Average Net Monthly Connected Fitness Churn" in earnings.
  # For the demo anomaly scenario, Q3 FY2024 shows a spike:
  - kpi_id: monthly_churn_rate
    period_label: "FY2024 Q2"
    period_start: "2023-10-01"
    period_end: "2023-12-31"
    value: 1.4

  - kpi_id: monthly_churn_rate
    period_label: "FY2024 Q3"
    period_start: "2024-01-01"
    period_end: "2024-03-31"
    value: 2.1  # Spike — triggers anomaly detection
```

The implementer should populate all 12 KPI IDs × 8+ quarters. Real numbers from filings where available; realistic synthetic numbers for internal metrics (churn detail, ARPU breakdown).

- [ ] **Step 3: Delete old company KPI directories**

```bash
rm -rf seed/olaplex seed/latham seed/mister_car_wash seed/european_wax_center seed/revolve_group
```

- [ ] **Step 4: Update kpis.py to load from the new path**

Update the `load_kpi_values` function in `seed/kpis.py` to look for `seed/{slug}/kpi_values.yaml` using the new company slugs.

- [ ] **Step 5: Run seed and verify KPI data loads**

```bash
make docker-seed-full
```

Verify: `curl http://localhost:8000/companies/peloton/kpis` returns populated KPI series.

- [ ] **Step 6: Commit**

```bash
git add seed/kpi_definitions.yaml seed/peloton/ seed/kpis.py
git add -u  # pick up deleted old company dirs
git commit -m "feat(seed): add Peloton KPI definitions and values from SEC filings"
```

---

## Task 4: Earnings Transcript Fetcher

**Files:**
- Create: `seed/transcripts.py`

- [ ] **Step 1: Create the transcript fetcher module**

Peloton earnings call transcripts are available from public sources (SEC filing attachments, investor relations pages). Create `seed/transcripts.py` that:

1. Fetches raw transcript text (or reads from a local cache file if already fetched)
2. Parses into segments: `ceo_remarks`, `cfo_remarks`, `qa_segment` (each analyst Q + management A)
3. Returns a list of `TranscriptSegment` dataclass objects

```python
"""Earnings call transcript fetcher and parser.

Fetches Peloton quarterly earnings call transcripts and parses them into
citable segments: CEO prepared remarks, CFO prepared remarks, and
individual Q&A exchanges.

Transcripts are cached locally under seed/cache/{cik}/transcripts/.
"""

from __future__ import annotations

import logging
from dataclasses import dataclass
from datetime import date
from pathlib import Path

from seed.config import CACHE_ROOT

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class TranscriptSegment:
    """One citable segment of an earnings call transcript."""
    quarter: str          # e.g. "FY2024 Q2"
    segment_type: str     # "ceo_remarks" | "cfo_remarks" | "qa"
    speaker: str          # e.g. "Barry McCarthy, CEO" or "Liz Coddington, CFO"
    title: str            # e.g. "CEO Prepared Remarks — Q2 FY2024 Earnings Call"
    content: str          # The actual text
    call_date: date       # Date the earnings call occurred


def _transcript_cache_dir(cik: str) -> Path:
    """Return the local cache directory for transcript files."""
    path = CACHE_ROOT / cik / "transcripts"
    path.mkdir(parents=True, exist_ok=True)
    return path


async def fetch_transcript_segments(
    cik: str,
    company_name: str,
    quarters: list[dict],
) -> list[TranscriptSegment]:
    """Fetch and parse earnings call transcripts for the given quarters.

    Each entry in `quarters` is a dict with:
      - quarter: str ("FY2024 Q2")
      - call_date: str (ISO date)
      - source_path: str (path to cached transcript text file)

    If the transcript is not cached, the implementer should populate the
    cache from public sources (investor relations page, SEC filings).

    Returns a flat list of TranscriptSegment objects — typically 3-8 per call
    (CEO remarks, CFO remarks, 2-5 Q&A exchanges).
    """
    segments: list[TranscriptSegment] = []
    cache_dir = _transcript_cache_dir(cik)

    for q in quarters:
        txt_path = cache_dir / f"{q['quarter'].replace(' ', '_')}.txt"
        if not txt_path.exists():
            logger.warning("transcript not cached: %s — skipping", txt_path)
            continue

        raw = txt_path.read_text(encoding="utf-8")
        parsed = _parse_transcript(raw, q["quarter"], date.fromisoformat(q["call_date"]))
        segments.extend(parsed)
        logger.info("parsed %d segments from %s", len(parsed), q["quarter"])

    return segments


def _parse_transcript(
    raw: str,
    quarter: str,
    call_date: date,
) -> list[TranscriptSegment]:
    """Parse a raw transcript into segments.

    Heuristic: split on speaker headers (lines matching "Name, Title" pattern
    or "Operator" / "Question" markers). Group consecutive paragraphs under
    each speaker into one segment.

    For prepared remarks, combine all of a speaker's paragraphs.
    For Q&A, each analyst question + management answer is one segment.
    """
    # Implementation: regex-based speaker detection, accumulate paragraphs,
    # emit segments. Follow the same pattern as parser.py's extract_sections.
    segments: list[TranscriptSegment] = []

    # Split into prepared remarks and Q&A sections
    # Look for markers like "Prepared Remarks", "Questions and Answers",
    # "Operator" as section delimiters.
    # Within prepared remarks: CEO block, CFO block
    # Within Q&A: each analyst question + management response pair

    # The actual parsing logic depends on transcript format.
    # A robust implementation uses line-by-line scanning with speaker
    # name detection (capitalized name followed by title/company).

    return segments
```

The parsing implementation depends on the actual transcript format. The implementer should:
1. Cache 6-8 Peloton earnings call transcripts as `.txt` files in `seed/cache/0001639825/transcripts/`
2. Implement `_parse_transcript` to handle the specific format
3. Each segment becomes a document in the corpus (same as filing sections)

- [ ] **Step 2: Wire transcript ingestion into the loader**

In `seed/loader.py`, after EDGAR filing ingestion, add transcript ingestion:

```python
# In _run(), after ingest_company():
from seed.transcripts import fetch_transcript_segments

transcript_quarters = [
    {"quarter": "FY2024 Q1", "call_date": "2023-11-01"},
    {"quarter": "FY2024 Q2", "call_date": "2024-02-01"},
    {"quarter": "FY2024 Q3", "call_date": "2024-05-01"},
    {"quarter": "FY2024 Q4", "call_date": "2024-08-22"},
    {"quarter": "FY2025 Q1", "call_date": "2024-11-06"},
    {"quarter": "FY2025 Q2", "call_date": "2025-02-06"},
]

segments = await fetch_transcript_segments(
    cik=hero.cik,
    company_name=hero.name,
    quarters=transcript_quarters,
)

# Insert each segment as a document in documents.documents
# and index in OpenSearch (same pattern as filing sections)
for seg in segments:
    doc_id = uuid5(TRANSCRIPT_NS, f"{hero.slug}:{seg.quarter}:{seg.segment_type}:{seg.speaker}")
    # pg_insert into documents.documents with document_type="transcript"
    # Index content into OpenSearch documents__{hero.slug} index
```

- [ ] **Step 3: Test transcript parsing**

Create a sample transcript text file and verify parsing produces expected segments.

- [ ] **Step 4: Commit**

```bash
git add seed/transcripts.py seed/loader.py
git commit -m "feat(seed): add earnings transcript fetcher and parser"
```

---

## Task 5: Competitor Profile Artifacts

**Files:**
- Create: `seed/competitors/apple_fitness.yaml`
- Create: `seed/competitors/lululemon_studio.yaml`
- Create: `seed/competitors/echelon.yaml`
- Create: `seed/competitors/ifit.yaml`
- Modify: `seed/focus.py` (add competitor profile seeder)

- [ ] **Step 1: Create competitor profile YAML files**

Each file contains structured profile data for one competitor. Example for Apple Fitness+:

```yaml
# Apple Fitness+ competitor profile — structured data for artifact generation.
slug: apple_fitness
artifact_title: "Competitor Profile: Apple Fitness+"
function: competitive

blocks:
  - block_id: overview
    heading: Overview
    body: >
      Apple Fitness+ is Apple's subscription fitness service, launched December 2020.
      Bundled with Apple Watch and available through Apple One subscription bundles.
      As of late 2024, available in 21 countries. Apple does not disclose subscriber
      numbers but analyst estimates range from 15-25M globally.

  - block_id: product_offering
    heading: Product & Pricing
    body: >
      $9.99/month or $79.99/year standalone. Included in Apple One Premier ($34.95/month).
      Effectively free for Apple Watch buyers (3-month trial, then bundled in Apple One).
      Workout types: cycling, running, HIIT, yoga, strength, rowing, meditation, and more.
      Recent additions: custom workout plans, real-time cycling metrics with Apple Watch.

  - block_id: competitive_positioning
    heading: How They Compete With Peloton
    body: >
      Ecosystem bundling is the primary threat. Apple Fitness+ doesn't need to win on
      content quality — it wins by being included in a bundle users already pay for.
      The addition of real-time cycling metrics (power, cadence, resistance) directly
      addresses the feature gap that kept serious cyclists on Peloton. Apple's installed
      base of 100M+ Apple Watch users is the distribution moat.

  - block_id: recent_moves
    heading: Recent Strategic Moves
    body: >
      - Added cycling with real-time metrics (direct Peloton feature parity)
      - Expanded to Android via Apple TV app (breaking ecosystem exclusivity)
      - Hired former Peloton CTO for Apple Health team
      - Integrated with third-party fitness equipment (Concept2, Life Fitness)

  - block_id: threat_assessment
    heading: Threat Assessment
    body: >
      CRITICAL. Apple's bundling strategy makes Fitness+ effectively free for the
      90M+ Apple Watch installed base. Peloton's $44/month All-Access subscription
      competes against a service that costs nothing incremental. The churn spike
      among Apple ecosystem users in Q3 is likely a leading indicator, not an anomaly.

signals:
  - title: "Apple Fitness+ added cycling classes with real-time metrics"
    summary: "Direct feature parity with Peloton's core differentiator — power zones, cadence tracking, resistance guidance. Removes the last technical barrier for Apple Watch cyclists to switch."
    severity: critical
    finding_type: product_launch
    hours_ago: 48

  - title: "Former Peloton CTO joined Apple Health team"
    summary: "John Foley-era CTO Tom Cortese moved to Apple's Health division. Likely working on hardware+software fitness integration."
    severity: important
    finding_type: exec_move
    hours_ago: 168

  - title: "Apple Watch Ultra 3 rumored to include advanced cycling power meter"
    summary: "If confirmed, this removes the need for a Peloton bike's built-in power measurement entirely. Users could get equivalent metrics from their watch on any stationary bike."
    severity: important
    finding_type: product_launch
    hours_ago: 72
```

Create similar YAML files for Lululemon Studio, Echelon, and iFit with appropriate content. Each should have 4-6 blocks and 2-4 monitoring signals.

- [ ] **Step 2: Add competitor profile seeder to focus.py**

Add a `seed_competitor_profiles()` function that:
1. Reads each competitor YAML file
2. Creates a `knowledge.artifacts` row (type `company_profile`, function `competitive`)
3. Creates an `artifact_version` with blocks from the YAML
4. Creates `artifact_subjects` linking to the competitor company and to Peloton (as secondary subject)
5. Creates monitoring findings from the `signals` list in each YAML

Follow the exact same `pg_insert().on_conflict_do_update()` pattern used in `seed_strategic_focus()`.

- [ ] **Step 3: Wire into loader.py**

Call `seed_competitor_profiles()` in the focus discipline phase of `_run()`.

- [ ] **Step 4: Run seed and verify competitor profiles are queryable**

```bash
make docker-seed-full
```

Verify artifacts appear: `curl http://localhost:8000/companies/peloton/artifacts` should include competitor profiles.

- [ ] **Step 5: Commit**

```bash
git add seed/competitors/ seed/focus.py seed/loader.py
git commit -m "feat(seed): add competitor profile artifacts for Apple, Lululemon, Echelon, iFit"
```

---

## Task 6: Peloton Strategic Artifacts

**Files:**
- Modify: `seed/focus.py`

- [ ] **Step 1: Replace Olaplex strategic focus with Peloton**

Replace `OLAPLEX_STRATEGIC_FOCUS_BLOCKS` with Peloton-specific content:

```python
PELOTON_STRATEGIC_FOCUS_BLOCKS = [
    {
        "block_id": "priority-1",
        "kind": "priority",
        "priority_number": 1,
        "title": "Drive subscription growth via App tier and corporate wellness",
        "rationale": (
            "Connected fitness subscriber growth has stalled at ~3M. The lower-priced "
            "App tier ($12.99/month) and corporate wellness channel represent the two "
            "highest-leverage growth vectors. Target: 4M subscribers by FY2025 end."
        ),
        "horizon": "12 months",
        "owner": "CRO + VP Corporate Wellness",
        "linked_metrics": ["connected_fitness_subscribers", "monthly_churn_rate", "subscription_arpu"],
    },
    {
        "block_id": "priority-2",
        "kind": "priority",
        "priority_number": 2,
        "title": "Recover hardware margins through supply chain optimization",
        "rationale": (
            "Connected Fitness segment gross margin is negative (-8.2% in Q3) due to "
            "aggressive hardware discounting and excess inventory. Must reach breakeven "
            "by Q2 FY2025 through BOM cost reduction, SKU rationalization, and "
            "reduced warehousing footprint."
        ),
        "horizon": "9 months",
        "owner": "COO + CFO",
        "linked_metrics": ["gross_margin_connected_fitness", "hardware_units_sold"],
    },
    {
        "block_id": "priority-3",
        "kind": "priority",
        "priority_number": 3,
        "title": "Differentiate on content as competitive moat",
        "rationale": (
            "Apple and other ecosystem players can replicate hardware metrics but cannot "
            "replicate Peloton's instructor brand, community, and content library. "
            "Content quality and exclusivity is the defensible moat. Invest in "
            "AI-powered personalized workout programming."
        ),
        "horizon": "18 months",
        "owner": "CTO + VP Content",
        "linked_metrics": ["monthly_churn_rate", "connected_fitness_subscribers"],
    },
    {
        "block_id": "narrative",
        "kind": "narrative",
        "body": (
            "The turnaround thesis: grow the subscriber base through lower-priced tiers "
            "and enterprise, stop losing money on hardware, and invest in the one thing "
            "competitors can't copy — the content and community experience. Hardware "
            "becomes the distribution vehicle for subscriptions, not a profit center."
        ),
    },
    {
        "block_id": "out-of-scope-1",
        "kind": "out_of_scope",
        "title": "TikTok fitness influencer pricing backlash",
        "rationale": (
            "Social media criticism of subscription price is a sentiment issue, not a "
            "strategic one. NPS dropped but subscriber behavior hasn't materially changed. "
            "Marketing team managing. Not a VCP priority."
        ),
    },
    {
        "block_id": "out-of-scope-2",
        "kind": "out_of_scope",
        "title": "International market expansion",
        "rationale": (
            "Multiple board members pushing for expansion into new markets. Deferred until "
            "domestic unit economics are healthy. Expanding while hardware margins are "
            "negative would accelerate losses."
        ),
    },
]
```

- [ ] **Step 2: Add additional strategic artifacts**

Add seeders for the remaining Peloton artifacts from the spec — Cost Structure, Channel & Distribution, Org & Leadership, Technology Roadmap. Each follows the same pattern as `seed_strategic_focus()`: create artifact + version + subject rows.

Create these as separate functions (e.g., `seed_cost_structure()`, `seed_channel_strategy()`, etc.) called from the loader. Each artifact has:
- 4-6 content blocks with realistic Peloton-specific information
- `function` field: "finance", "strategy", "competitive", "people", "technology"
- Primary subject: Peloton company

Also seed a **Legal & Regulatory Risk** artifact covering the 5 litigation items from the spec.

- [ ] **Step 3: Update observations for Peloton**

Replace the Olaplex observations in `_OBSERVATIONS` with Peloton-specific ones matching the spec's KPI anomalies and monitoring findings.

- [ ] **Step 4: Update headspace for Sarah Chen**

Replace `seed_headspace()` to create Sarah's headspace watching Peloton + 4 competitors.

- [ ] **Step 5: Run and verify all artifacts seed correctly**

```bash
make docker-seed-full
```

Check: each artifact type appears in the API, with correct blocks and citations.

- [ ] **Step 6: Commit**

```bash
git add seed/focus.py
git commit -m "feat(seed): add Peloton strategic artifacts, observations, and headspace"
```

---

## Task 7: Seed Decision Records, Action Items, and KPI Anomalies

**Files:**
- Create: `seed/scenarios.py`
- Modify: `seed/loader.py`

- [ ] **Step 1: Create scenarios.py**

This module seeds the operational scenario data: 8 decision records, 20 action items, and 5 KPI anomalies with enrichment questions. All as specified in the design spec Section 2.4.

```python
"""Seed operational scenarios — decisions, actions, KPI anomalies.

Creates the realistic operational state that makes the demo feel lived-in:
decision records linked to strategic priorities, action items in various
stages across multiple owners, and KPI anomalies with enrichment questions.

All UUIDs are deterministic. Uses SQLAlchemy Core upserts.
"""

from __future__ import annotations

import logging
from datetime import datetime, timedelta, timezone
from decimal import Decimal
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.action_items.infrastructure.tables import (
    ActionItemTable,
    DecisionRecordTable,
)
from prescient.artifacts.infrastructure.tables import (
    ArtifactRow,
    ArtifactVersionRow,
)
from prescient.kpis.infrastructure.tables import (
    KpiAnomalyRow,
    KpiValueRow,
)
from seed.companies import SeedBundle

logger = logging.getLogger(__name__)

_SCENARIO_NS = UUID("e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b")
_DECISION_NS = UUID("a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d")
_ACTION_NS = UUID("b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e")
_ANOMALY_NS = UUID("c3d4e5f6-a7b8-4c9d-0e1f-2a3b4c5d6e7f")


def _uuid(ns: UUID, key: str) -> UUID:
    return uuid5(ns, key)


DECISION_RECORDS = [
    {
        "key": "hardware-price-cuts",
        "title": "Approved hardware price cuts to drive subscriber acquisition",
        "context": "Connected Fitness segment revenue declining 30%+ YoY. Hardware inventory building. Competitive pressure from Echelon and used market.",
        "decision": "Cut Bike price to $1,445 (from $1,745), Bike+ to $2,495 (from $2,495), Tread to $2,695 (from $3,495). Accept negative hardware margins to drive subscriber flywheel.",
        "rationale": "Each hardware sale converts to $44/month subscription. LTV of a subscriber ($1,200+) exceeds hardware margin sacrifice. Priority 1 alignment.",
        "made_by": "CEO + CFO",
    },
    {
        "key": "rental-program",
        "title": "Launched Bike rental program at $89/month all-inclusive",
        "context": "High upfront hardware cost is the #1 barrier to subscriber acquisition. Peloton Rental eliminates the capital expenditure barrier.",
        "decision": "Launch rental program: $89/month includes Bike + All-Access Membership. 12-month minimum commitment. Option to buy at end of term.",
        "rationale": "Reduces acquisition friction. Monthly revenue is lower per-unit but subscriber conversion rate expected to increase 3-5x for price-sensitive segment.",
        "made_by": "CRO + CFO",
    },
    {
        "key": "workforce-reduction",
        "title": "Restructured workforce — 15% reduction (approx. 800 positions)",
        "context": "Operating expenses remain too high for current revenue base. Previous reductions insufficient. Path to profitability requires lower fixed cost base.",
        "decision": "Reduce workforce by approximately 800 positions across all functions. Close several retail showrooms. Consolidate warehousing to 2 facilities (from 4).",
        "rationale": "Expected $200M+ annualized savings. Painful but necessary for runway extension and path to positive FCF by FY2025.",
        "made_by": "CEO + Board",
    },
    {
        "key": "amazon-channel",
        "title": "Entered Amazon as distribution channel",
        "context": "DTC website traffic declining. Need additional customer acquisition channels. Amazon reaches customers who would never visit onepeloton.com.",
        "decision": "List Bike, Guide, and accessories on Amazon. Maintain DTC pricing parity. Amazon handles fulfillment for standard accessories; Peloton handles Bike delivery.",
        "rationale": "Incremental reach to Amazon's fitness equipment shoppers. Risks: brand dilution, margin compression from Amazon fees. Mitigated by pricing parity.",
        "made_by": "CRO + CEO",
    },
    {
        "key": "guide-discontinuation",
        "title": "Discontinued the Peloton Guide (camera-based strength product)",
        "context": "Guide launched in 2022 at $295. Adoption below expectations. Camera-based form tracking technology not meeting quality bar. Competes for engineering resources.",
        "decision": "Discontinue Guide hardware. Redirect engineering resources to core Bike/Tread/App experience. Write down remaining inventory.",
        "rationale": "Guide consumed disproportionate engineering resources relative to revenue contribution (<2% of hardware sales). Better to focus on winning products.",
        "made_by": "CTO + CEO",
    },
    {
        "key": "marketing-pivot",
        "title": "Shifted marketing budget from hardware to subscription value proposition",
        "context": "Legacy marketing focused on hardware aspiration ('the Peloton lifestyle'). New reality: hardware is a vehicle for subscriptions, not the product itself.",
        "decision": "Reallocate 60% of marketing budget from hardware-focused campaigns to subscription value messaging. Emphasize content library, instructor community, and App tier.",
        "rationale": "Customer research shows subscription value (not hardware) drives retention. New messaging aligns with Priority 1.",
        "made_by": "VP Marketing + CRO",
    },
    {
        "key": "corporate-wellness",
        "title": "Accelerated corporate wellness sales program",
        "context": "Corporate wellness is a $60B+ market. Peloton has early traction with Fortune 500 companies. B2B channel has lower CAC and higher retention than consumer.",
        "decision": "Hire 10 enterprise sales reps. Build self-service corporate portal. Target 50 enterprise accounts by EOY. Partner with Gympass/Wellhub for distribution.",
        "rationale": "Corporate subscribers churn at half the rate of consumer. Pipeline already at $6M ARR qualified. Highest-ROI growth lever.",
        "made_by": "CRO + CEO",
    },
    {
        "key": "content-licensing",
        "title": "Approved content licensing renegotiation strategy",
        "context": "Music licensing costs are $50M+/year following NMPA settlement. Current deals expire in 18 months. Need to renegotiate from a position of platform scale.",
        "decision": "Engage major labels directly (bypass publishers where possible). Offer data-sharing partnerships. Build original music program as negotiating leverage.",
        "rationale": "Music is essential to the experience but current costs are unsustainable. Direct deals could save 20-30% vs. publisher intermediaries.",
        "made_by": "CFO + VP Content",
    },
]

# Action items: 20 total, mix of statuses and owners
ACTION_ITEMS = [
    # pending_review (5)
    {"key": "churn-ecosystem-analysis", "title": "Analyze subscriber churn by device ecosystem — quantify Apple Watch vs. Android exposure", "owner": "VP Data", "priority": "high", "status": "pending_review", "days_until_due": 3, "decision_key": None},
    {"key": "q3-earnings-narrative", "title": "Prepare Q3 earnings narrative draft for CEO review", "owner": "CFO", "priority": "high", "status": "pending_review", "days_until_due": 5, "decision_key": None},
    {"key": "wsj-response", "title": "Review and approve WSJ inquiry response re: layoffs", "owner": "CEO", "priority": "high", "status": "pending_review", "days_until_due": 1, "decision_key": None},
    {"key": "board-deck-q3", "title": "Assemble Q3 board deck — financial pages and initiative updates", "owner": "CFO", "priority": "medium", "status": "pending_review", "days_until_due": 10, "decision_key": None},
    {"key": "nps-deep-dive", "title": "Deep dive on NPS decline — identify top 3 drivers from verbatim analysis", "owner": "VP Product", "priority": "medium", "status": "pending_review", "days_until_due": 7, "decision_key": None},

    # sent_to_owner (6)
    {"key": "cohort-analysis", "title": "Cohort retention analysis — 30/60/90 day retention by acquisition channel", "owner": "VP Data", "priority": "high", "status": "sent_to_owner", "days_until_due": 5, "decision_key": "hardware-price-cuts"},
    {"key": "margin-bridge", "title": "Build margin bridge for board — hardware vs. subscription waterfall", "owner": "CFO", "priority": "high", "status": "sent_to_owner", "days_until_due": 8, "decision_key": "workforce-reduction"},
    {"key": "app-benchmarks", "title": "App performance benchmarks — latency, crash rate, feature adoption by tier", "owner": "CTO", "priority": "medium", "status": "sent_to_owner", "days_until_due": 10, "decision_key": "guide-discontinuation"},
    {"key": "wellness-pipeline", "title": "Corporate wellness pipeline review — stage each deal, identify blockers", "owner": "VP Corporate Wellness", "priority": "high", "status": "sent_to_owner", "days_until_due": 2, "decision_key": "corporate-wellness"},
    {"key": "amazon-metrics", "title": "Amazon channel first-month metrics — units, conversion, return rate, reviews", "owner": "CRO", "priority": "medium", "status": "sent_to_owner", "days_until_due": 4, "decision_key": "amazon-channel"},
    {"key": "content-cost-model", "title": "Model content licensing cost scenarios for renegotiation", "owner": "CFO", "priority": "medium", "status": "sent_to_owner", "days_until_due": 14, "decision_key": "content-licensing"},

    # completed (6)
    {"key": "amazon-launch-done", "title": "Amazon channel launch — listing live, fulfillment tested, monitoring active", "owner": "CRO", "priority": "high", "status": "completed", "days_until_due": -5, "decision_key": "amazon-channel"},
    {"key": "reduction-complete", "title": "Workforce reduction execution complete — 800 positions, 4 showrooms closed", "owner": "COO", "priority": "high", "status": "completed", "days_until_due": -14, "decision_key": "workforce-reduction"},
    {"key": "music-terms", "title": "Music licensing interim renewal terms finalized with NMPA", "owner": "CFO", "priority": "high", "status": "completed", "days_until_due": -7, "decision_key": "content-licensing"},
    {"key": "guide-writedown", "title": "Guide inventory writedown completed — $32M charge recognized in Q2", "owner": "CFO", "priority": "medium", "status": "completed", "days_until_due": -21, "decision_key": "guide-discontinuation"},
    {"key": "rental-launch-done", "title": "Rental program launched in all US markets", "owner": "CRO", "priority": "high", "status": "completed", "days_until_due": -10, "decision_key": "rental-program"},
    {"key": "marketing-realloc", "title": "Marketing budget reallocation complete — 60% now subscription-focused", "owner": "VP Marketing", "priority": "medium", "status": "completed", "days_until_due": -3, "decision_key": "marketing-pivot"},

    # deferred (3)
    {"key": "intl-expansion", "title": "International expansion scoping — market sizing for Germany and Australia", "owner": "VP Strategy", "priority": "low", "status": "deferred", "days_until_due": 30, "decision_key": None},
    {"key": "next-gen-hardware", "title": "Next-gen Bike hardware R&D kickoff — form factor and cost targets", "owner": "CTO", "priority": "low", "status": "deferred", "days_until_due": 60, "decision_key": None},
    {"key": "enterprise-sso", "title": "Enterprise SSO integration for corporate wellness customers", "owner": "CTO", "priority": "low", "status": "deferred", "days_until_due": 45, "decision_key": "corporate-wellness"},
]

KPI_ANOMALIES = [
    {
        "key": "churn-spike",
        "kpi_id": "monthly_churn_rate",
        "anomaly_type": "sudden_change",
        "description": "Monthly connected fitness churn spiked to 2.1% in Q3 (trailing avg: 1.4%)",
        "severity": "critical",
        "enrichment_question": "Connected fitness churn spiked following the Apple Fitness+ free bundle announcement. Is this primarily driven by Apple ecosystem users, or are you seeing elevated churn across all cohorts?",
    },
    {
        "key": "arpu-decline",
        "kpi_id": "subscription_arpu",
        "anomaly_type": "deviation_from_trend",
        "description": "Subscription ARPU declined 6% QoQ despite subscriber growth",
        "severity": "important",
        "enrichment_question": "ARPU declined 6% QoQ. Is this driven by the lower-priced App tier mix shift, or are you seeing downgrades from All-Access?",
    },
    {
        "key": "hardware-margin-negative",
        "kpi_id": "gross_margin_connected_fitness",
        "anomaly_type": "deviation_from_trend",
        "description": "Connected Fitness segment gross margin turned negative at -8.2%",
        "severity": "critical",
        "enrichment_question": "Connected Fitness segment gross margin turned negative. Is this a deliberate investment in subscriber acquisition via hardware discounting, or is there a supply chain cost issue?",
    },
    {
        "key": "fcf-improvement",
        "kpi_id": "free_cash_flow",
        "anomaly_type": "sudden_change",
        "description": "Free cash flow improved $45M QoQ while revenue declined",
        "severity": "notable",
        "enrichment_question": "FCF improved $45M QoQ while revenue declined. Is this sustainable cost discipline or one-time items (restructuring charges already behind)?",
    },
    {
        "key": "wellness-pipeline-surge",
        "kpi_id": "revenue_subscription",
        "anomaly_type": "sudden_change",
        "description": "Corporate wellness qualified pipeline grew from $2M to $6M ARR",
        "severity": "notable",
        "enrichment_question": "Corporate wellness qualified pipeline grew 3x QoQ. What's driving the acceleration — is it the Gympass partnership or direct enterprise sales?",
    },
]


async def seed_decision_records(session: AsyncSession, *, bundle: SeedBundle) -> None:
    """Seed 8 decision records for Peloton."""
    company = bundle.hero()
    fund = bundle.fund
    now = datetime.now(timezone.utc)

    for i, dr in enumerate(DECISION_RECORDS):
        artifact_id = _uuid(_DECISION_NS, f"peloton:{dr['key']}")
        version_id = _uuid(_DECISION_NS, f"peloton:{dr['key']}:v1")
        created = now - timedelta(days=(len(DECISION_RECORDS) - i) * 7)

        # Create as artifact in both legacy and new tables
        # (dual_write.py handles the new tables separately)
        stmt = pg_insert(ArtifactRow).values(
            id=str(artifact_id),
            organization_id=str(fund.id),
            artifact_type="decision_record",
            title=dr["title"],
            visibility="shared",
            active_version_id=str(version_id),
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[ArtifactRow.id],
            set_={"title": stmt.excluded.title, "active_version_id": stmt.excluded.active_version_id},
        )
        await session.execute(stmt)

        # Version with structured blocks
        blocks = [
            {"block_id": "context", "heading": "Context", "body": dr["context"]},
            {"block_id": "decision", "heading": "Decision", "body": dr["decision"]},
            {"block_id": "rationale", "heading": "Rationale", "body": dr["rationale"]},
        ]
        ver_stmt = pg_insert(ArtifactVersionRow).values(
            id=str(version_id),
            artifact_id=str(artifact_id),
            version_number=1,
            state="active",
            confidence_label="verified",
            blocks=blocks,
            summary=dr["decision"][:200],
        )
        ver_stmt = ver_stmt.on_conflict_do_update(
            index_elements=[ArtifactVersionRow.id],
            set_={"blocks": ver_stmt.excluded.blocks, "summary": ver_stmt.excluded.summary},
        )
        await session.execute(ver_stmt)

    logger.info("seeded %d decision records for peloton", len(DECISION_RECORDS))


async def seed_action_items(session: AsyncSession, *, bundle: SeedBundle) -> None:
    """Seed 20 action items in various states."""
    company = bundle.hero()
    fund = bundle.fund
    now = datetime.now(timezone.utc)

    for ai in ACTION_ITEMS:
        item_id = _uuid(_ACTION_NS, f"peloton:{ai['key']}")
        due = now + timedelta(days=ai["days_until_due"])

        stmt = pg_insert(ActionItemTable).values(
            id=item_id,
            owner_tenant_id=fund.id,
            subject_company_id=company.id,
            title=ai["title"],
            owner=ai["owner"],
            priority=ai["priority"],
            status=ai["status"],
            due_date=due.date(),
            source_artifact_id=_uuid(_DECISION_NS, f"peloton:{ai['decision_key']}") if ai["decision_key"] else None,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[ActionItemTable.id],
            set_={
                "title": stmt.excluded.title,
                "status": stmt.excluded.status,
                "due_date": stmt.excluded.due_date,
            },
        )
        await session.execute(stmt)

    logger.info("seeded %d action items for peloton", len(ACTION_ITEMS))


async def seed_kpi_anomalies(session: AsyncSession, *, bundle: SeedBundle) -> None:
    """Seed 5 KPI anomalies with enrichment questions."""
    company = bundle.hero()
    fund = bundle.fund

    for anomaly in KPI_ANOMALIES:
        anomaly_id = _uuid(_ANOMALY_NS, f"peloton:{anomaly['key']}")

        stmt = pg_insert(KpiAnomalyRow).values(
            id=str(anomaly_id),
            organization_id=str(fund.id),
            kpi_id=anomaly["kpi_id"],
            anomaly_type=anomaly["anomaly_type"],
            description=anomaly["description"],
            severity=anomaly["severity"],
            enrichment_question=anomaly["enrichment_question"],
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=["id"],
            set_={
                "description": stmt.excluded.description,
                "enrichment_question": stmt.excluded.enrichment_question,
            },
        )
        await session.execute(stmt)

    logger.info("seeded %d KPI anomalies for peloton", len(KPI_ANOMALIES))
```

Note: The exact table column names may need adjustment — the implementer should check the actual table definitions in `action_items/infrastructure/tables/` and `kpis/infrastructure/tables/` and adjust column names to match.

- [ ] **Step 2: Wire into loader.py**

```python
from seed.scenarios import seed_decision_records, seed_action_items, seed_kpi_anomalies

# In _run(), during the focus discipline phase:
await seed_decision_records(session, bundle=bundle)
await seed_action_items(session, bundle=bundle)
await seed_kpi_anomalies(session, bundle=bundle)
```

- [ ] **Step 3: Run and verify**

```bash
make docker-seed-full
```

Verify action items: `curl 'http://localhost:8000/actions/items?organization_id=...'`
Verify decision records appear as artifacts in the artifact store.

- [ ] **Step 4: Commit**

```bash
git add seed/scenarios.py seed/loader.py
git commit -m "feat(seed): add Peloton decision records, action items, and KPI anomalies"
```

---

## Task 8: Dual-Write Bridge

**Files:**
- Create: `seed/dual_write.py`
- Modify: `seed/loader.py`

- [ ] **Step 1: Create dual_write.py**

This module ensures every artifact, KPI value, and document written to legacy tables is also written to the corresponding new tables. It reads from one set and writes to the other.

```python
"""Dual-write bridge — sync legacy tables to new tables for demo parity.

The intelligence planner tools query legacy tables (knowledge.*, documents.*).
The Phase 1-3 UI queries new tables (artifacts.*, kpis.*).
This bridge ensures both are populated from the same seed data.

Run AFTER all other seeders have populated the legacy tables.
"""

from __future__ import annotations

import logging
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.infrastructure.tables import (
    ArtifactRow as NewArtifactRow,
    ArtifactVersionRow as NewArtifactVersionRow,
    ArtifactSubjectRow as NewArtifactSubjectRow,
)
from prescient.knowledge.infrastructure.tables import (
    ArtifactTable as LegacyArtifactTable,
    ArtifactVersionTable as LegacyVersionTable,
    ArtifactSubjectTable as LegacySubjectTable,
)

logger = logging.getLogger(__name__)


async def sync_legacy_to_new_artifacts(session: AsyncSession) -> int:
    """Copy all legacy knowledge.artifacts to new artifacts.* tables.

    Returns the number of artifacts synced.
    """
    legacy_rows = (await session.execute(select(LegacyArtifactTable))).scalars().all()
    count = 0

    for row in legacy_rows:
        stmt = pg_insert(NewArtifactRow).values(
            id=str(row.id),
            organization_id=str(row.owner_tenant_id),
            artifact_type=row.artifact_type,
            title=row.title,
            visibility=row.visibility,
            active_version_id=str(row.active_version_id) if row.active_version_id else None,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[NewArtifactRow.id],
            set_={"title": stmt.excluded.title, "active_version_id": stmt.excluded.active_version_id},
        )
        await session.execute(stmt)
        count += 1

    # Sync versions
    legacy_versions = (await session.execute(select(LegacyVersionTable))).scalars().all()
    for ver in legacy_versions:
        stmt = pg_insert(NewArtifactVersionRow).values(
            id=str(ver.id),
            artifact_id=str(ver.artifact_id),
            version_number=ver.version_number,
            state=ver.state,
            confidence_label=ver.confidence_label,
            blocks=ver.blocks,
            summary=ver.summary,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[NewArtifactVersionRow.id],
            set_={"blocks": stmt.excluded.blocks, "summary": stmt.excluded.summary},
        )
        await session.execute(stmt)

    # Sync subjects
    legacy_subjects = (await session.execute(select(LegacySubjectTable))).scalars().all()
    for subj in legacy_subjects:
        stmt = pg_insert(NewArtifactSubjectRow).values(
            artifact_version_id=str(subj.artifact_version_id),
            subject_type=subj.subject_type,
            subject_id=str(subj.subject_id),
            is_primary=subj.is_primary,
        )
        stmt = stmt.on_conflict_do_nothing()
        await session.execute(stmt)

    logger.info("synced %d legacy artifacts to new artifact store", count)
    return count
```

The implementer should also add a `sync_legacy_to_new_kpis()` function that copies `documents.kpi_values` → `kpis.kpi_values` with appropriate field mapping.

- [ ] **Step 2: Wire into loader.py as final step**

```python
from seed.dual_write import sync_legacy_to_new_artifacts

# At the end of _run(), after all other seeders:
logger.info("syncing legacy tables to new tables...")
await sync_legacy_to_new_artifacts(session)
await session.commit()
```

- [ ] **Step 3: Run and verify both table sets have data**

```bash
make docker-seed-full
```

Verify legacy: intelligence tools work (curl the copilot endpoint with a test question if API key available).
Verify new: `curl http://localhost:8000/artifact-store/` returns artifacts from the new tables.

- [ ] **Step 4: Commit**

```bash
git add seed/dual_write.py seed/loader.py
git commit -m "feat(seed): add dual-write bridge — sync legacy to new artifact/KPI tables"
```

---

## Task 9: Update Monitoring Findings

**Files:**
- Modify: `seed/focus.py`

- [ ] **Step 1: Replace monitoring findings with Peloton competitor signals**

Replace the existing `seed_monitor_findings()` findings list with the 15 findings from the spec (Section 2.4 of the design doc). Update watched slugs to Peloton competitors. Create targets for each competitor company.

The findings should be sourced from the competitor profile YAML files (Task 5) plus additional standalone signals. Follow the existing pattern exactly — `pg_insert().on_conflict_do_update()` with deterministic UUIDs.

- [ ] **Step 2: Run and verify findings appear in the headspace digest**

```bash
make docker-seed-full
curl http://localhost:8000/intelligence/headspace/digest
```

Should return 15 findings across Peloton's watched competitors.

- [ ] **Step 3: Commit**

```bash
git add seed/focus.py
git commit -m "feat(seed): add 15 Peloton competitor monitoring findings"
```

---

## Task 10: Update Verification and Cleanup

**Files:**
- Modify: `seed/loader.py` (verify command)
- Delete: old company directories

- [ ] **Step 1: Update verify command thresholds**

Update the `_verify()` function in `seed/loader.py` with new expected minimums:

```python
# Peloton (hero): deep dataset
MIN_HERO_DOCS = 40       # 30-40 filing sections + transcript segments
MIN_HERO_KPIS = 80       # 12 KPIs × 8 quarters minimum
MIN_HERO_CHUNKS = 200    # filing sections + transcript segments chunked

# Overall
MIN_OBSERVATIONS = 5
MIN_FINDINGS = 15
MIN_ARTIFACTS = 12        # strategic focus + 5 strategic + 4 competitor profiles + legal + product
MIN_DECISIONS = 8
MIN_ACTIONS = 20
```

- [ ] **Step 2: Run full seed pipeline end-to-end**

```bash
make docker-seed-full
make docker-seed-verify  # or equivalent verify command
```

All thresholds should pass.

- [ ] **Step 3: Verify the complete demo data is accessible**

Test key API endpoints:
```bash
# Companies
curl http://localhost:8000/companies | python3 -m json.tool | head -20

# KPI series
curl "http://localhost:8000/companies/peloton/kpis" | python3 -m json.tool | head -20

# Artifacts
curl "http://localhost:8000/companies/peloton/artifacts" | python3 -m json.tool | head -20

# Attention queue
curl http://localhost:8000/intelligence/attention-queue | python3 -m json.tool | head -20

# Briefing
curl "http://localhost:8000/briefing/today?user_id=...&organization_id=...&role=operator" | python3 -m json.tool | head -20

# Actions
curl "http://localhost:8000/actions/items?organization_id=..." | python3 -m json.tool | head -20
```

- [ ] **Step 4: Final commit**

```bash
git add -u
git commit -m "feat(seed): complete Peloton seed pipeline — verification passing"
```

---

## Summary

| Task | What it does | Key files |
|------|-------------|-----------|
| 1 | Company graph: Peloton + 4 competitors | `companies.yaml`, `config.py`, `companies.py` |
| 2 | Demo user: Sarah Chen, CEO | `seed_demo_user.py` |
| 3 | KPI definitions + Peloton values | `kpi_definitions.yaml`, `peloton/kpi_values.yaml` |
| 4 | Earnings transcript fetcher | `transcripts.py`, `loader.py` |
| 5 | Competitor profile artifacts | `competitors/*.yaml`, `focus.py` |
| 6 | Peloton strategic artifacts | `focus.py` |
| 7 | Decision records, action items, anomalies | `scenarios.py`, `loader.py` |
| 8 | Dual-write bridge | `dual_write.py`, `loader.py` |
| 9 | Monitoring findings (15 signals) | `focus.py` |
| 10 | Verification + cleanup | `loader.py` |
