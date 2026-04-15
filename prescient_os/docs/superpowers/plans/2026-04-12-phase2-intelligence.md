# Phase 2: Intelligence Engine, Onboarding Wizard & Company Research

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the company research pipeline, onboarding wizard, and monitoring configuration — so a user can name a company, the system researches it and creates a structured Company Profile artifact, and the user can optionally "keep watching" it.

**Architecture:** A `ResearchService` orchestrates data gathering from multiple sources (SEC EDGAR, web search) and uses the Anthropic LLM to synthesize findings into structured Company Profile artifacts. The Onboarding wizard uses this service to auto-research competitors during company setup. The Monitoring context gets infrastructure tables and API to persist watch configurations. User interest profiles are stored as a lightweight model in the Organizations context.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Anthropic SDK (Claude), OpenSearch (BM25), httpx for web fetching, existing EDGAR client from seed pipeline.

**Spec:** `docs/superpowers/specs/2026-04-12-prescient-os-redesign.md` — Modules 2 (Intelligence Engine) and 7 (Onboarding Wizard)

**Depends on:** Phase 1 foundation (Organizations, Artifacts, Auth contexts complete).

---

## File Structure

### New Files (API)

```
apps/api/src/prescient/
├── intelligence/
│   ├── application/
│   │   ├── research/
│   │   │   ├── __init__.py
│   │   │   ├── research_service.py          # Orchestrator: gather → synthesize → persist
│   │   │   └── company_profile_builder.py   # LLM prompt + response parsing for profiles
│   │   └── use_cases/
│   │       └── run_company_research.py      # Use case entry point
│   └── infrastructure/
│       └── sources/
│           ├── __init__.py
│           ├── edgar_source.py              # Wraps seed/edgar for async company research
│           ├── web_search_source.py         # Basic web search via httpx (news, general info)
│           └── source_aggregator.py         # Combines results from multiple sources
├── monitoring/
│   ├── application/
│   │   ├── __init__.py
│   │   └── use_cases/
│   │       ├── __init__.py
│   │       ├── add_watch_target.py          # Add a company/topic to monitoring
│   │       ├── list_watch_targets.py        # List active watch targets for an org
│   │       └── remove_watch_target.py       # Deactivate a watch target
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   ├── tables/
│   │   │   ├── __init__.py
│   │   │   ├── monitor_target.py
│   │   │   ├── monitor_source.py
│   │   │   ├── monitor_run.py
│   │   │   └── monitor_finding.py
│   │   └── repositories/
│   │       └── __init__.py
│   └── api/
│       ├── __init__.py
│       └── routes.py                        # CRUD for watch targets
├── onboarding/
│   ├── __init__.py
│   ├── domain/
│   │   ├── __init__.py
│   │   └── entities/
│   │       ├── __init__.py
│   │       ├── company_setup.py             # Company onboarding state
│   │       └── interest_profile.py          # User interest profile (Pinterest-style)
│   ├── application/
│   │   ├── __init__.py
│   │   └── use_cases/
│   │       ├── __init__.py
│   │       ├── setup_company.py             # Company setup wizard orchestration
│   │       ├── add_competitor.py            # Add competitor + trigger research
│   │       └── setup_user_interests.py      # Save user interest profile
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   └── tables/
│   │       ├── __init__.py
│   │       ├── company_setup.py
│   │       └── interest_profile.py
│   └── api/
│       ├── __init__.py
│       └── routes.py                        # Onboarding wizard endpoints
```

### New Files (Frontend)

```
apps/web/src/
├── app/(main)/
│   ├── onboarding/
│   │   ├── page.tsx                         # Onboarding wizard entry
│   │   ├── company/page.tsx                 # Company setup step
│   │   ├── competitors/page.tsx             # Competitor naming + live research
│   │   └── interests/page.tsx               # Interest profile setup
│   └── research/
│       └── [slug]/page.tsx                  # Ad-hoc company research results
├── components/
│   ├── research-status.tsx                  # Research progress indicator
│   └── company-profile-card.tsx             # Rendered company profile artifact
```

### New Files (Tests)

```
tests/
├── intelligence/
│   ├── test_research_service.py
│   ├── test_company_profile_builder.py
│   ├── test_edgar_source.py
│   └── test_run_company_research.py
├── monitoring/
│   ├── test_watch_target_domain.py
│   └── test_monitoring_api.py
├── onboarding/
│   ├── test_company_setup.py
│   ├── test_interest_profile.py
│   └── test_onboarding_api.py
```

### Modified Files

```
apps/api/src/prescient/shared/types.py           # Add InterestProfileId, CompanySetupId
apps/api/src/prescient/shared/metadata.py         # Register new tables
apps/api/src/prescient/main.py                    # Add monitoring, onboarding routers
apps/api/src/prescient/config/base.py             # Add web search config
apps/api/pyproject.toml                           # Any new deps if needed
```

---

## Task 1: Add Phase 2 Config and Types

**Files:**
- Modify: `apps/api/src/prescient/shared/types.py`
- Modify: `apps/api/src/prescient/config/base.py`

- [ ] **Step 1: Add new ID types**

In `apps/api/src/prescient/shared/types.py`, add:

```python
InterestProfileId = NewType("InterestProfileId", UUID)
CompanySetupId = NewType("CompanySetupId", UUID)
MonitorTargetId = NewType("MonitorTargetId", UUID)
MonitorSourceId = NewType("MonitorSourceId", UUID)
MonitorRunId = NewType("MonitorRunId", UUID)
MonitorFindingId = NewType("MonitorFindingId", UUID)
```

- [ ] **Step 2: Add research config**

In `apps/api/src/prescient/config/base.py`, add:

```python
# Research service config
research_max_sources: int = Field(default=5)
research_llm_model: str = Field(default="claude-sonnet-4-5-20250929")
sec_edgar_user_agent: str = Field(default="Prescient OS Demo dev@prescient.local")
```

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/shared/types.py apps/api/src/prescient/config/base.py
git commit -m "feat: add Phase 2 ID types and research config"
```

---

## Task 2: EDGAR Research Source Adapter

**Files:**
- Create: `apps/api/src/prescient/intelligence/infrastructure/sources/__init__.py`
- Create: `apps/api/src/prescient/intelligence/infrastructure/sources/edgar_source.py`
- Test: `apps/api/tests/intelligence/test_edgar_source.py`

This wraps the existing `seed/edgar.py` client for use by the research service. It fetches recent SEC filings for a company and extracts key sections.

- [ ] **Step 1: Create sources package**

Create empty `apps/api/src/prescient/intelligence/infrastructure/sources/__init__.py`.

- [ ] **Step 2: Write failing test**

Create `apps/api/tests/intelligence/__init__.py` (empty) and `apps/api/tests/intelligence/test_edgar_source.py`:

```python
from datetime import datetime, UTC
from unittest.mock import AsyncMock, MagicMock, patch
from uuid import uuid4

import pytest

from prescient.intelligence.infrastructure.sources.edgar_source import (
    EdgarResearchSource,
    EdgarResearchResult,
)


class TestEdgarResearchSource:
    @pytest.mark.asyncio
    async def test_research_returns_structured_result(self) -> None:
        mock_edgar = AsyncMock()
        mock_edgar.list_filings.return_value = [
            MagicMock(
                form="10-K",
                filing_date="2025-03-15",
                accession_number="0001868726-25-000001",
            ),
        ]
        mock_edgar.fetch_filing.return_value = "/tmp/fake-filing.html"

        mock_sections = {"Business": "Olaplex is a hair care company..."}

        with patch(
            "prescient.intelligence.infrastructure.sources.edgar_source.extract_sections",
            return_value=mock_sections,
        ):
            source = EdgarResearchSource(edgar_client=mock_edgar)
            result = await source.research(cik="0001868726", company_name="Olaplex Holdings")

        assert isinstance(result, EdgarResearchResult)
        assert result.company_name == "Olaplex Holdings"
        assert len(result.filings) >= 1
        assert result.filings[0]["form"] == "10-K"
        assert "Business" in result.filings[0]["sections"]

    @pytest.mark.asyncio
    async def test_handles_no_filings_gracefully(self) -> None:
        mock_edgar = AsyncMock()
        mock_edgar.list_filings.return_value = []

        source = EdgarResearchSource(edgar_client=mock_edgar)
        result = await source.research(cik="0000000000", company_name="Unknown Corp")

        assert result.company_name == "Unknown Corp"
        assert result.filings == []
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && uv run python -m pytest tests/intelligence/test_edgar_source.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement EDGAR source adapter**

Create `apps/api/src/prescient/intelligence/infrastructure/sources/edgar_source.py`:

```python
from __future__ import annotations

import logging
from dataclasses import dataclass, field
from pathlib import Path

from seed.edgar import EdgarClient, FilingRef
from seed.parser import extract_sections

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class EdgarResearchResult:
    company_name: str
    cik: str
    filings: list[dict] = field(default_factory=list)

    @property
    def has_data(self) -> bool:
        return len(self.filings) > 0

    def summary_text(self) -> str:
        """Flatten all filing sections into a single text block for LLM consumption."""
        parts: list[str] = []
        for filing in self.filings:
            parts.append(f"## {filing['form']} filed {filing['filing_date']}")
            for section_name, section_text in filing.get("sections", {}).items():
                parts.append(f"### {section_name}")
                parts.append(section_text[:3000])  # Cap per section for LLM context
        return "\n\n".join(parts)


class EdgarResearchSource:
    """Wraps the seed EDGAR client for ad-hoc company research."""

    def __init__(self, edgar_client: EdgarClient) -> None:
        self._edgar = edgar_client

    async def research(
        self,
        cik: str,
        company_name: str,
        form_counts: dict[str, int] | None = None,
    ) -> EdgarResearchResult:
        if form_counts is None:
            form_counts = {"10-K": 1, "10-Q": 2}

        try:
            filing_refs = await self._edgar.list_filings(cik, form_counts)
        except Exception:
            logger.exception("Failed to list filings for CIK %s", cik)
            return EdgarResearchResult(company_name=company_name, cik=cik, filings=[])

        filings: list[dict] = []
        for ref in filing_refs:
            try:
                html_path = await self._edgar.fetch_filing(ref)
                sections = extract_sections(Path(html_path))
                filings.append({
                    "form": ref.form,
                    "filing_date": str(ref.filing_date),
                    "accession_number": ref.accession_number,
                    "sections": sections,
                })
            except Exception:
                logger.exception("Failed to fetch/parse filing %s", ref.accession_number)
                continue

        return EdgarResearchResult(
            company_name=company_name,
            cik=cik,
            filings=filings,
        )
```

- [ ] **Step 5: Run tests**

Run: `cd apps/api && uv run python -m pytest tests/intelligence/test_edgar_source.py -v`
Expected: 2 PASSED

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/sources/ tests/intelligence/
git commit -m "feat: add EDGAR research source adapter for company research"
```

---

## Task 3: Company Profile Builder (LLM Synthesis)

**Files:**
- Create: `apps/api/src/prescient/intelligence/application/research/__init__.py`
- Create: `apps/api/src/prescient/intelligence/application/research/company_profile_builder.py`
- Test: `apps/api/tests/intelligence/test_company_profile_builder.py`

The profile builder takes raw research data and uses the LLM to produce structured blocks for a Company Profile artifact.

- [ ] **Step 1: Create research package**

Create empty `apps/api/src/prescient/intelligence/application/research/__init__.py`.

- [ ] **Step 2: Write failing test**

Create `apps/api/tests/intelligence/test_company_profile_builder.py`:

```python
from unittest.mock import AsyncMock

import pytest

from prescient.intelligence.application.research.company_profile_builder import (
    CompanyProfileBuilder,
    ProfileBuildRequest,
    ProfileBuildResult,
)


class TestCompanyProfileBuilder:
    @pytest.mark.asyncio
    async def test_builds_profile_blocks_from_research(self) -> None:
        mock_llm = AsyncMock()
        mock_llm.send.return_value = {
            "content": [
                {
                    "type": "text",
                    "text": (
                        '{"blocks": ['
                        '{"block_id": "overview", "heading": "Company Overview", '
                        '"body": "Olaplex is a premium hair care company.", "order": 0},'
                        '{"block_id": "financials", "heading": "Financial Summary", '
                        '"body": "Revenue was $422M in FY2024.", "order": 1},'
                        '{"block_id": "competitive", "heading": "Competitive Position", '
                        '"body": "Competes with K18 and Living Proof.", "order": 2}'
                        '], "summary": "Olaplex is a premium hair care company specializing in bond-building technology."}'
                    ),
                }
            ],
            "stop_reason": "end_turn",
        }

        builder = CompanyProfileBuilder(llm_adapter=mock_llm)
        result = await builder.build(
            ProfileBuildRequest(
                company_name="Olaplex Holdings",
                raw_research_text="## 10-K filed 2025-03-15\n### Business\nOlaplex is a hair care company...",
                industry="Beauty & Personal Care",
            )
        )

        assert isinstance(result, ProfileBuildResult)
        assert len(result.blocks) == 3
        assert result.blocks[0]["block_id"] == "overview"
        assert result.summary is not None
        assert mock_llm.send.called

    @pytest.mark.asyncio
    async def test_handles_malformed_llm_response(self) -> None:
        mock_llm = AsyncMock()
        mock_llm.send.return_value = {
            "content": [{"type": "text", "text": "This is not valid JSON"}],
            "stop_reason": "end_turn",
        }

        builder = CompanyProfileBuilder(llm_adapter=mock_llm)
        result = await builder.build(
            ProfileBuildRequest(
                company_name="Unknown Corp",
                raw_research_text="Some raw text",
                industry=None,
            )
        )

        # Should return a fallback single-block profile rather than crashing
        assert len(result.blocks) >= 1
        assert result.blocks[0]["block_id"] == "raw_research"
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && uv run python -m pytest tests/intelligence/test_company_profile_builder.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement profile builder**

Create `apps/api/src/prescient/intelligence/application/research/company_profile_builder.py`:

```python
from __future__ import annotations

import json
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)

PROFILE_SYSTEM_PROMPT = """You are a research analyst building a structured company profile.

Given raw research data about a company, produce a JSON object with this exact structure:
{
  "blocks": [
    {"block_id": "overview", "heading": "Company Overview", "body": "...", "order": 0},
    {"block_id": "financials", "heading": "Financial Summary", "body": "...", "order": 1},
    {"block_id": "products", "heading": "Products & Services", "body": "...", "order": 2},
    {"block_id": "competitive", "heading": "Competitive Position", "body": "...", "order": 3},
    {"block_id": "risks", "heading": "Key Risks", "body": "...", "order": 4},
    {"block_id": "recent", "heading": "Recent Developments", "body": "...", "order": 5}
  ],
  "summary": "One-paragraph executive summary of the company."
}

Rules:
- Only include blocks where you have substantive information. Omit blocks with no data.
- Each body should be 2-4 sentences, factual, grounded in the provided data.
- Do not invent information. If the data doesn't cover a topic, skip that block.
- Return ONLY the JSON object, no markdown fences or explanation."""


@dataclass(frozen=True)
class ProfileBuildRequest:
    company_name: str
    raw_research_text: str
    industry: str | None = None


@dataclass(frozen=True)
class ProfileBuildResult:
    blocks: list[dict]
    summary: str | None


class CompanyProfileBuilder:
    def __init__(self, llm_adapter: object) -> None:
        self._llm = llm_adapter

    async def build(self, request: ProfileBuildRequest) -> ProfileBuildResult:
        user_message = f"Company: {request.company_name}"
        if request.industry:
            user_message += f"\nIndustry: {request.industry}"
        user_message += f"\n\nRaw Research Data:\n{request.raw_research_text[:12000]}"

        try:
            response = await self._llm.send(
                system=PROFILE_SYSTEM_PROMPT,
                messages=[{"role": "user", "content": user_message}],
                tools=[],
            )
            text = self._extract_text(response)
            parsed = json.loads(text)
            blocks = parsed.get("blocks", [])
            summary = parsed.get("summary")
            if not blocks:
                return self._fallback(request)
            return ProfileBuildResult(blocks=blocks, summary=summary)
        except (json.JSONDecodeError, KeyError, TypeError):
            logger.warning("LLM returned malformed profile for %s, using fallback", request.company_name)
            return self._fallback(request)

    def _extract_text(self, response: dict) -> str:
        for block in response.get("content", []):
            if block.get("type") == "text":
                text = block["text"].strip()
                # Strip markdown fences if present
                if text.startswith("```"):
                    lines = text.split("\n")
                    text = "\n".join(lines[1:-1]) if len(lines) > 2 else text
                return text
        return "{}"

    def _fallback(self, request: ProfileBuildRequest) -> ProfileBuildResult:
        return ProfileBuildResult(
            blocks=[{
                "block_id": "raw_research",
                "heading": "Research Data",
                "body": request.raw_research_text[:2000],
                "order": 0,
            }],
            summary=f"Raw research data for {request.company_name}. LLM synthesis unavailable.",
        )
```

- [ ] **Step 5: Run tests**

Run: `cd apps/api && uv run python -m pytest tests/intelligence/test_company_profile_builder.py -v`
Expected: 2 PASSED

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/intelligence/application/research/ tests/intelligence/
git commit -m "feat: add CompanyProfileBuilder — LLM synthesis for company research"
```

---

## Task 4: Research Service (Orchestrator)

**Files:**
- Create: `apps/api/src/prescient/intelligence/application/research/research_service.py`
- Create: `apps/api/src/prescient/intelligence/application/use_cases/run_company_research.py`
- Test: `apps/api/tests/intelligence/test_research_service.py`

The research service orchestrates: gather data from sources → synthesize via LLM → persist as Company Profile artifact.

- [ ] **Step 1: Write failing test for research service**

Create `apps/api/tests/intelligence/test_research_service.py`:

```python
from datetime import datetime, UTC
from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.intelligence.application.research.research_service import (
    ResearchService,
    ResearchRequest,
    ResearchResult,
)
from prescient.intelligence.infrastructure.sources.edgar_source import EdgarResearchResult


class TestResearchService:
    @pytest.fixture
    def mock_edgar_source(self) -> AsyncMock:
        source = AsyncMock()
        source.research.return_value = EdgarResearchResult(
            company_name="Olaplex Holdings",
            cik="0001868726",
            filings=[{
                "form": "10-K",
                "filing_date": "2025-03-15",
                "accession_number": "0001868726-25-000001",
                "sections": {"Business": "Olaplex is a hair care company..."},
            }],
        )
        return source

    @pytest.fixture
    def mock_profile_builder(self) -> AsyncMock:
        builder = AsyncMock()
        builder.build.return_value = MagicMock(
            blocks=[
                {"block_id": "overview", "heading": "Company Overview", "body": "Olaplex overview.", "order": 0},
            ],
            summary="Olaplex is a premium hair care company.",
        )
        return builder

    @pytest.fixture
    def mock_session(self) -> AsyncMock:
        return AsyncMock()

    @pytest.mark.asyncio
    async def test_research_creates_artifact(
        self, mock_edgar_source: AsyncMock, mock_profile_builder: AsyncMock, mock_session: AsyncMock,
    ) -> None:
        service = ResearchService(
            edgar_source=mock_edgar_source,
            profile_builder=mock_profile_builder,
            session=mock_session,
        )
        result = await service.research(ResearchRequest(
            organization_id=str(uuid4()),
            company_name="Olaplex Holdings",
            cik="0001868726",
            industry="Beauty & Personal Care",
            requested_by="ryan",
        ))

        assert isinstance(result, ResearchResult)
        assert result.artifact_id is not None
        assert result.version_id is not None
        assert result.profile_summary == "Olaplex is a premium hair care company."
        mock_edgar_source.research.assert_called_once()
        mock_profile_builder.build.assert_called_once()
        assert mock_session.add.called

    @pytest.mark.asyncio
    async def test_research_without_cik_skips_edgar(
        self, mock_edgar_source: AsyncMock, mock_profile_builder: AsyncMock, mock_session: AsyncMock,
    ) -> None:
        service = ResearchService(
            edgar_source=mock_edgar_source,
            profile_builder=mock_profile_builder,
            session=mock_session,
        )
        result = await service.research(ResearchRequest(
            organization_id=str(uuid4()),
            company_name="Private Startup Inc",
            cik=None,
            industry="Technology",
            requested_by="ryan",
        ))

        assert result.artifact_id is not None
        mock_edgar_source.research.assert_not_called()
        mock_profile_builder.build.assert_called_once()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run python -m pytest tests/intelligence/test_research_service.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement research service**

Create `apps/api/src/prescient/intelligence/application/research/research_service.py`:

```python
from __future__ import annotations

import logging
from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.domain.entities.artifact import ArtifactType
from prescient.artifacts.infrastructure.tables.artifact import ArtifactRow
from prescient.artifacts.infrastructure.tables.artifact_version import ArtifactVersionRow
from prescient.intelligence.application.research.company_profile_builder import (
    CompanyProfileBuilder,
    ProfileBuildRequest,
)
from prescient.intelligence.infrastructure.sources.edgar_source import (
    EdgarResearchSource,
)

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class ResearchRequest:
    organization_id: str
    company_name: str
    cik: str | None = None
    industry: str | None = None
    requested_by: str = "system"


@dataclass(frozen=True)
class ResearchResult:
    artifact_id: str
    version_id: str
    profile_summary: str | None
    sources_used: list[str]


class ResearchService:
    def __init__(
        self,
        edgar_source: EdgarResearchSource,
        profile_builder: CompanyProfileBuilder,
        session: AsyncSession,
    ) -> None:
        self._edgar = edgar_source
        self._builder = profile_builder
        self._session = session

    async def research(self, request: ResearchRequest) -> ResearchResult:
        raw_parts: list[str] = []
        sources_used: list[str] = []

        # Gather from EDGAR if CIK available
        if request.cik:
            edgar_result = await self._edgar.research(
                cik=request.cik,
                company_name=request.company_name,
            )
            if edgar_result.has_data:
                raw_parts.append(edgar_result.summary_text())
                sources_used.append("sec_edgar")

        # Combine raw research
        raw_text = "\n\n---\n\n".join(raw_parts) if raw_parts else f"Company: {request.company_name}"
        if request.industry:
            raw_text = f"Industry: {request.industry}\n\n{raw_text}"

        # Synthesize via LLM
        profile_result = await self._builder.build(
            ProfileBuildRequest(
                company_name=request.company_name,
                raw_research_text=raw_text,
                industry=request.industry,
            )
        )

        # Persist as Company Profile artifact
        now = datetime.now(UTC)
        artifact_id = str(uuid4())
        version_id = str(uuid4())

        self._session.add(ArtifactRow(
            id=artifact_id,
            organization_id=request.organization_id,
            artifact_type=ArtifactType.COMPANY_PROFILE.value,
            title=f"{request.company_name} — Company Profile",
            visibility="shared",
            active_version_id=version_id,
            created_at=now,
            updated_at=now,
        ))

        self._session.add(ArtifactVersionRow(
            id=version_id,
            artifact_id=artifact_id,
            version_number=1,
            state="draft",
            confidence_label="medium",
            blocks=profile_result.blocks,
            citations=[],
            summary=profile_result.summary,
            created_by=request.requested_by,
            created_at=now,
        ))

        return ResearchResult(
            artifact_id=artifact_id,
            version_id=version_id,
            profile_summary=profile_result.summary,
            sources_used=sources_used,
        )
```

- [ ] **Step 4: Run tests**

Run: `cd apps/api && uv run python -m pytest tests/intelligence/test_research_service.py -v`
Expected: 2 PASSED

- [ ] **Step 5: Implement run_company_research use case**

Create `apps/api/src/prescient/intelligence/application/use_cases/__init__.py` (empty if not exists).

Create `apps/api/src/prescient/intelligence/application/use_cases/run_company_research.py`:

```python
from __future__ import annotations

from dataclasses import dataclass

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.intelligence.application.research.company_profile_builder import CompanyProfileBuilder
from prescient.intelligence.application.research.research_service import (
    ResearchRequest,
    ResearchResult,
    ResearchService,
)
from prescient.intelligence.infrastructure.sources.edgar_source import EdgarResearchSource


@dataclass(frozen=True)
class RunCompanyResearchCommand:
    organization_id: str
    company_name: str
    cik: str | None = None
    industry: str | None = None
    requested_by: str = "system"


class RunCompanyResearch:
    def __init__(
        self,
        session: AsyncSession,
        edgar_source: EdgarResearchSource,
        llm_adapter: object,
    ) -> None:
        self._session = session
        self._edgar_source = edgar_source
        self._llm_adapter = llm_adapter

    async def execute(self, command: RunCompanyResearchCommand) -> ResearchResult:
        builder = CompanyProfileBuilder(llm_adapter=self._llm_adapter)
        service = ResearchService(
            edgar_source=self._edgar_source,
            profile_builder=builder,
            session=self._session,
        )
        return await service.research(
            ResearchRequest(
                organization_id=command.organization_id,
                company_name=command.company_name,
                cik=command.cik,
                industry=command.industry,
                requested_by=command.requested_by,
            )
        )
```

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/intelligence/application/ tests/intelligence/
git commit -m "feat: add ResearchService — orchestrates EDGAR + LLM into Company Profile artifacts"
```

---

## Task 5: Research API Endpoint

**Files:**
- Create: `apps/api/src/prescient/intelligence/api/research_routes.py`
- Modify: `apps/api/src/prescient/main.py`
- Test: `apps/api/tests/intelligence/test_run_company_research.py`

- [ ] **Step 1: Write failing test for research use case end-to-end**

Create `apps/api/tests/intelligence/test_run_company_research.py`:

```python
from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.intelligence.application.use_cases.run_company_research import (
    RunCompanyResearch,
    RunCompanyResearchCommand,
)
from prescient.intelligence.infrastructure.sources.edgar_source import EdgarResearchResult


class TestRunCompanyResearch:
    @pytest.mark.asyncio
    async def test_executes_research_pipeline(self) -> None:
        mock_session = AsyncMock()
        mock_edgar = AsyncMock()
        mock_edgar.research.return_value = EdgarResearchResult(
            company_name="Olaplex", cik="0001868726",
            filings=[{"form": "10-K", "filing_date": "2025-03-15", "sections": {"Business": "Hair care..."}}],
        )
        mock_llm = AsyncMock()
        mock_llm.send.return_value = {
            "content": [{"type": "text", "text": '{"blocks": [{"block_id": "overview", "heading": "Overview", "body": "Olaplex overview.", "order": 0}], "summary": "Olaplex summary."}'}],
            "stop_reason": "end_turn",
        }

        use_case = RunCompanyResearch(
            session=mock_session, edgar_source=mock_edgar, llm_adapter=mock_llm,
        )
        result = await use_case.execute(RunCompanyResearchCommand(
            organization_id=str(uuid4()), company_name="Olaplex Holdings",
            cik="0001868726", industry="Beauty", requested_by="ryan",
        ))

        assert result.artifact_id is not None
        assert result.profile_summary == "Olaplex summary."
        assert "sec_edgar" in result.sources_used
```

- [ ] **Step 2: Run test**

Run: `cd apps/api && uv run python -m pytest tests/intelligence/test_run_company_research.py -v`
Expected: 1 PASSED

- [ ] **Step 3: Implement research API route**

Create `apps/api/src/prescient/intelligence/api/research_routes.py`:

```python
from __future__ import annotations

from pydantic import BaseModel, Field
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.config import get_settings
from prescient.db import get_session
from prescient.intelligence.application.use_cases.run_company_research import (
    RunCompanyResearch,
    RunCompanyResearchCommand,
)
from prescient.intelligence.infrastructure.anthropic_adapter import AnthropicAdapter
from prescient.intelligence.infrastructure.sources.edgar_source import (
    EdgarResearchSource,
)

router = APIRouter(prefix="/research", tags=["research"])


class ResearchCompanyBody(BaseModel):
    organization_id: str
    company_name: str = Field(min_length=1, max_length=256)
    cik: str | None = None
    industry: str | None = None
    requested_by: str = "system"


class ResearchCompanyResponse(BaseModel):
    artifact_id: str
    version_id: str
    profile_summary: str | None
    sources_used: list[str]


@router.post("/companies", response_model=ResearchCompanyResponse, status_code=201)
async def research_company(
    body: ResearchCompanyBody,
    session: AsyncSession = Depends(get_session),
) -> ResearchCompanyResponse:
    settings = get_settings()

    # Build dependencies
    from seed.edgar import EdgarClient

    edgar_client = EdgarClient(user_agent=settings.sec_edgar_user_agent)
    edgar_source = EdgarResearchSource(edgar_client=edgar_client)
    llm_adapter = AnthropicAdapter(
        api_key=settings.anthropic_api_key,
        model=settings.research_llm_model,
    )

    use_case = RunCompanyResearch(
        session=session,
        edgar_source=edgar_source,
        llm_adapter=llm_adapter,
    )
    result = await use_case.execute(RunCompanyResearchCommand(
        organization_id=body.organization_id,
        company_name=body.company_name,
        cik=body.cik,
        industry=body.industry,
        requested_by=body.requested_by,
    ))
    await session.commit()

    return ResearchCompanyResponse(
        artifact_id=result.artifact_id,
        version_id=result.version_id,
        profile_summary=result.profile_summary,
        sources_used=result.sources_used,
    )
```

- [ ] **Step 4: Wire research router into main.py**

In `apps/api/src/prescient/main.py`, add:

```python
from prescient.intelligence.api.research_routes import router as research_router
```

And include: `app.include_router(research_router)`

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/intelligence/api/research_routes.py apps/api/src/prescient/main.py tests/intelligence/
git commit -m "feat: add POST /research/companies endpoint for ad-hoc company research"
```

---

## Task 6: Monitoring Infrastructure Tables

**Files:**
- Create: `apps/api/src/prescient/monitoring/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/monitoring/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_target.py`
- Create: `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_source.py`
- Create: `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_run.py`
- Create: `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_finding.py`

All tables use schema "monitoring".

- [ ] **Step 1: Create infrastructure package**

Create empty `__init__.py` files for monitoring infrastructure.

- [ ] **Step 2: Implement monitor_target table**

Create `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_target.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import Boolean, DateTime, String, Text
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class MonitorTargetRow(Base):
    __tablename__ = "monitor_targets"
    __table_args__ = {"schema": "monitoring"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    target_type: Mapped[str] = mapped_column(String(16), nullable=False)
    target_name: Mapped[str] = mapped_column(String(256), nullable=False)
    target_metadata: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    purpose: Mapped[str] = mapped_column(String(32), nullable=False)
    monitoring_config: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    created_by_user_id: Mapped[str | None] = mapped_column(String(64), nullable=True)
    active: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default="true")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 3: Implement monitor_source, monitor_run, monitor_finding tables**

Create `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_source.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import Boolean, DateTime, String
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class MonitorSourceRow(Base):
    __tablename__ = "monitor_sources"
    __table_args__ = {"schema": "monitoring"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    source_type: Mapped[str] = mapped_column(String(32), nullable=False)
    config: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="{}")
    credentials_ref: Mapped[str | None] = mapped_column(String(256), nullable=True)
    active: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default="true")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

Create `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_run.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, ForeignKey, Integer, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class MonitorRunRow(Base):
    __tablename__ = "monitor_runs"
    __table_args__ = {"schema": "monitoring"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    source_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("monitoring.monitor_sources.id"), nullable=False, index=True
    )
    started_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    finished_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    status: Mapped[str] = mapped_column(String(16), nullable=False, server_default="running")
    items_found: Mapped[int] = mapped_column(Integer, nullable=False, server_default="0")
    error: Mapped[str | None] = mapped_column(Text, nullable=True)
```

Create `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_finding.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import Boolean, DateTime, ForeignKey, String, Text
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class MonitorFindingRow(Base):
    __tablename__ = "monitor_findings"
    __table_args__ = {"schema": "monitoring"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    run_id: Mapped[str | None] = mapped_column(
        String(36), ForeignKey("monitoring.monitor_runs.id"), nullable=True, index=True
    )
    target_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("monitoring.monitor_targets.id"), nullable=False, index=True
    )
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    finding_type: Mapped[str] = mapped_column(String(32), nullable=False)
    title: Mapped[str] = mapped_column(String(256), nullable=False)
    summary: Mapped[str | None] = mapped_column(Text, nullable=True)
    severity: Mapped[str] = mapped_column(String(16), nullable=False, server_default="info")
    occurred_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    discovered_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    source_url: Mapped[str | None] = mapped_column(String(1024), nullable=True)
    raw_payload: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    processed: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default="false")
```

- [ ] **Step 4: Create tables __init__.py**

Update `apps/api/src/prescient/monitoring/infrastructure/tables/__init__.py`:

```python
from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow
from prescient.monitoring.infrastructure.tables.monitor_source import MonitorSourceRow
from prescient.monitoring.infrastructure.tables.monitor_run import MonitorRunRow
from prescient.monitoring.infrastructure.tables.monitor_finding import MonitorFindingRow

__all__ = [
    "MonitorTargetRow",
    "MonitorSourceRow",
    "MonitorRunRow",
    "MonitorFindingRow",
]
```

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/monitoring/infrastructure/
git commit -m "feat: add Monitoring context SQLAlchemy table definitions"
```

---

## Task 7: Onboarding Domain Entities

**Files:**
- Create: `apps/api/src/prescient/onboarding/__init__.py`
- Create: `apps/api/src/prescient/onboarding/domain/__init__.py`
- Create: `apps/api/src/prescient/onboarding/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/onboarding/domain/entities/company_setup.py`
- Create: `apps/api/src/prescient/onboarding/domain/entities/interest_profile.py`
- Test: `apps/api/tests/onboarding/__init__.py`
- Test: `apps/api/tests/onboarding/test_company_setup.py`
- Test: `apps/api/tests/onboarding/test_interest_profile.py`

- [ ] **Step 1: Create onboarding package**

Create empty `__init__.py` files for onboarding context.

- [ ] **Step 2: Write failing test for CompanySetup**

Create `apps/api/tests/onboarding/test_company_setup.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.onboarding.domain.entities.company_setup import (
    CompanySetup,
    CompanySetupStatus,
    CompetitorEntry,
    MonitoringScope,
)
from prescient.shared.types import CompanySetupId, OrganizationId


class TestCompanySetup:
    def test_create_company_setup(self) -> None:
        now = datetime.now(UTC)
        setup = CompanySetup(
            id=CompanySetupId(uuid4()),
            organization_id=OrganizationId(uuid4()),
            company_name="Olaplex Holdings",
            industry="Beauty & Personal Care",
            status=CompanySetupStatus.IN_PROGRESS,
            competitors=(),
            created_at=now,
            updated_at=now,
        )
        assert setup.status == CompanySetupStatus.IN_PROGRESS
        assert len(setup.competitors) == 0

    def test_add_competitor(self) -> None:
        now = datetime.now(UTC)
        setup = CompanySetup(
            id=CompanySetupId(uuid4()),
            organization_id=OrganizationId(uuid4()),
            company_name="Olaplex Holdings",
            industry="Beauty & Personal Care",
            status=CompanySetupStatus.IN_PROGRESS,
            competitors=(),
            created_at=now,
            updated_at=now,
        )
        competitor = CompetitorEntry(
            name="K18 Biomimetic Hair",
            how_they_compete="Product overlap in bond-building hair treatments",
            cik=None,
            monitoring_scope=MonitoringScope.FULL,
        )
        updated = setup.model_copy(update={"competitors": (*setup.competitors, competitor)})
        assert len(updated.competitors) == 1
        assert updated.competitors[0].name == "K18 Biomimetic Hair"

    def test_all_statuses(self) -> None:
        expected = {"not_started", "in_progress", "completed"}
        assert {s.value for s in CompanySetupStatus} == expected

    def test_all_monitoring_scopes(self) -> None:
        expected = {"full", "news_only", "filings_only", "minimal"}
        assert {s.value for s in MonitoringScope} == expected
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && uv run python -m pytest tests/onboarding/test_company_setup.py -v`
Expected: FAIL

- [ ] **Step 4: Implement CompanySetup**

Create `apps/api/src/prescient/onboarding/domain/entities/company_setup.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field

from prescient.shared.types import CompanySetupId, OrganizationId


class CompanySetupStatus(StrEnum):
    NOT_STARTED = "not_started"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"


class MonitoringScope(StrEnum):
    FULL = "full"
    NEWS_ONLY = "news_only"
    FILINGS_ONLY = "filings_only"
    MINIMAL = "minimal"


class CompetitorEntry(BaseModel):
    model_config = ConfigDict(frozen=True)

    name: str = Field(min_length=1, max_length=256)
    how_they_compete: str | None = None
    cik: str | None = None
    monitoring_scope: MonitoringScope = MonitoringScope.FULL


class CompanySetup(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: CompanySetupId
    organization_id: OrganizationId
    company_name: str = Field(min_length=1, max_length=256)
    industry: str | None = None
    status: CompanySetupStatus
    competitors: tuple[CompetitorEntry, ...] = ()
    created_at: datetime
    updated_at: datetime
```

- [ ] **Step 5: Run tests**

Run: `cd apps/api && uv run python -m pytest tests/onboarding/test_company_setup.py -v`
Expected: 4 PASSED

- [ ] **Step 6: Write failing test for InterestProfile**

Create `apps/api/tests/onboarding/test_interest_profile.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.onboarding.domain.entities.interest_profile import (
    InterestProfile,
    InterestCategory,
)
from prescient.shared.types import InterestProfileId, UserId, OrganizationId


class TestInterestProfile:
    def test_create_interest_profile(self) -> None:
        now = datetime.now(UTC)
        profile = InterestProfile(
            id=InterestProfileId(uuid4()),
            user_id=UserId("ryan"),
            organization_id=OrganizationId(uuid4()),
            topics=("AI infrastructure", "cloud cost optimization", "engineering talent market"),
            industries=("Beauty & Personal Care", "Consumer Goods"),
            technologies=("machine learning", "data pipelines"),
            competitors_of_interest=(),
            created_at=now,
            updated_at=now,
        )
        assert len(profile.topics) == 3
        assert "AI infrastructure" in profile.topics

    def test_empty_profile_is_valid(self) -> None:
        now = datetime.now(UTC)
        profile = InterestProfile(
            id=InterestProfileId(uuid4()),
            user_id=UserId("analyst1"),
            organization_id=OrganizationId(uuid4()),
            topics=(),
            industries=(),
            technologies=(),
            competitors_of_interest=(),
            created_at=now,
            updated_at=now,
        )
        assert profile.has_interests is False

    def test_profile_with_any_interest_has_interests(self) -> None:
        now = datetime.now(UTC)
        profile = InterestProfile(
            id=InterestProfileId(uuid4()),
            user_id=UserId("ryan"),
            organization_id=OrganizationId(uuid4()),
            topics=("supply chain",),
            industries=(),
            technologies=(),
            competitors_of_interest=(),
            created_at=now,
            updated_at=now,
        )
        assert profile.has_interests is True
```

- [ ] **Step 7: Implement InterestProfile**

Create `apps/api/src/prescient/onboarding/domain/entities/interest_profile.py`:

```python
from __future__ import annotations

from datetime import datetime

from pydantic import BaseModel, ConfigDict

from prescient.shared.types import InterestProfileId, OrganizationId, UserId


class InterestCategory:
    """Constants for interest categorization (no enum needed — just tags)."""

    TOPIC = "topic"
    INDUSTRY = "industry"
    TECHNOLOGY = "technology"
    COMPETITOR = "competitor"


class InterestProfile(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: InterestProfileId
    user_id: UserId
    organization_id: OrganizationId
    topics: tuple[str, ...] = ()
    industries: tuple[str, ...] = ()
    technologies: tuple[str, ...] = ()
    competitors_of_interest: tuple[str, ...] = ()
    created_at: datetime
    updated_at: datetime

    @property
    def has_interests(self) -> bool:
        return bool(self.topics or self.industries or self.technologies or self.competitors_of_interest)
```

- [ ] **Step 8: Run all onboarding tests**

Run: `cd apps/api && uv run python -m pytest tests/onboarding/ -v`
Expected: 7 PASSED

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/onboarding/ tests/onboarding/
git commit -m "feat: add onboarding domain — CompanySetup and InterestProfile entities"
```

---

## Task 8: Onboarding Infrastructure Tables

**Files:**
- Create: `apps/api/src/prescient/onboarding/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/onboarding/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/onboarding/infrastructure/tables/company_setup.py`
- Create: `apps/api/src/prescient/onboarding/infrastructure/tables/interest_profile.py`

- [ ] **Step 1: Create infrastructure package**

Create empty `__init__.py` files.

- [ ] **Step 2: Implement company_setup table**

Create `apps/api/src/prescient/onboarding/infrastructure/tables/company_setup.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, String
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class CompanySetupRow(Base):
    __tablename__ = "company_setups"
    __table_args__ = {"schema": "onboarding"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    company_name: Mapped[str] = mapped_column(String(256), nullable=False)
    industry: Mapped[str | None] = mapped_column(String(128), nullable=True)
    status: Mapped[str] = mapped_column(String(16), nullable=False, server_default="not_started")
    competitors: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="[]")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 3: Implement interest_profile table**

Create `apps/api/src/prescient/onboarding/infrastructure/tables/interest_profile.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, String
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class InterestProfileRow(Base):
    __tablename__ = "interest_profiles"
    __table_args__ = {"schema": "onboarding"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    user_id: Mapped[str] = mapped_column(String(64), nullable=False, unique=True, index=True)
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    topics: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="[]")
    industries: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="[]")
    technologies: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="[]")
    competitors_of_interest: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="[]")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 4: Create tables __init__.py**

Update `apps/api/src/prescient/onboarding/infrastructure/tables/__init__.py`:

```python
from prescient.onboarding.infrastructure.tables.company_setup import CompanySetupRow
from prescient.onboarding.infrastructure.tables.interest_profile import InterestProfileRow

__all__ = ["CompanySetupRow", "InterestProfileRow"]
```

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/onboarding/infrastructure/
git commit -m "feat: add onboarding infrastructure — company_setup and interest_profile tables"
```

---

## Task 9: Alembic Migration for Phase 2 Schemas

**Files:**
- Modify: `apps/api/src/prescient/shared/metadata.py`
- Create: new migration in `apps/api/alembic/versions/`

- [ ] **Step 1: Update metadata.py**

Add imports for new table modules:

```python
import prescient.monitoring.infrastructure.tables  # noqa: F401
import prescient.onboarding.infrastructure.tables  # noqa: F401
```

- [ ] **Step 2: Create schemas if needed**

```bash
PGPASSWORD=prescient psql -h localhost -U prescient -d prescient -c "CREATE SCHEMA IF NOT EXISTS monitoring; CREATE SCHEMA IF NOT EXISTS onboarding;"
```

- [ ] **Step 3: Auto-generate migration**

Run: `cd apps/api && uv run python -m alembic revision --autogenerate -m "phase2_monitoring_onboarding"`

- [ ] **Step 4: Review and fix the migration**

Ensure it includes `CREATE SCHEMA IF NOT EXISTS` for `monitoring` and `onboarding` at the top of `upgrade()`. Should create:
- `monitoring` schema: monitor_targets, monitor_sources, monitor_runs, monitor_findings
- `onboarding` schema: company_setups, interest_profiles

- [ ] **Step 5: Run the migration**

Run: `cd apps/api && uv run python -m alembic upgrade head`

- [ ] **Step 6: Verify tables**

Run: `PGPASSWORD=prescient psql -h localhost -U prescient -d prescient -c "\dt monitoring.*" -c "\dt onboarding.*"`

- [ ] **Step 7: Commit**

```bash
git add apps/api/alembic/ apps/api/src/prescient/shared/metadata.py
git commit -m "feat: add Alembic migration for Phase 2 schemas — monitoring, onboarding"
```

---

## Task 10: Monitoring API — Watch Target CRUD

**Files:**
- Create: `apps/api/src/prescient/monitoring/application/__init__.py`
- Create: `apps/api/src/prescient/monitoring/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/monitoring/application/use_cases/add_watch_target.py`
- Create: `apps/api/src/prescient/monitoring/application/use_cases/list_watch_targets.py`
- Create: `apps/api/src/prescient/monitoring/application/use_cases/remove_watch_target.py`
- Create: `apps/api/src/prescient/monitoring/api/__init__.py`
- Create: `apps/api/src/prescient/monitoring/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`
- Test: `apps/api/tests/monitoring/__init__.py`
- Test: `apps/api/tests/monitoring/test_monitoring_api.py`

- [ ] **Step 1: Create application and API packages**

Create empty `__init__.py` files for monitoring application, use_cases, and api.

- [ ] **Step 2: Write failing test**

Create `apps/api/tests/monitoring/test_monitoring_api.py`:

```python
from datetime import datetime, UTC
from unittest.mock import AsyncMock
from uuid import uuid4

import pytest

from prescient.monitoring.application.use_cases.add_watch_target import (
    AddWatchTarget,
    AddWatchTargetRequest,
)


class TestAddWatchTarget:
    @pytest.fixture
    def mock_session(self) -> AsyncMock:
        return AsyncMock()

    @pytest.mark.asyncio
    async def test_add_watch_target(self, mock_session: AsyncMock) -> None:
        use_case = AddWatchTarget(session=mock_session)
        result = await use_case.execute(AddWatchTargetRequest(
            organization_id=str(uuid4()),
            target_type="company",
            target_name="K18 Biomimetic Hair",
            purpose="competitive_intel",
            monitoring_config={"scope": "full"},
            created_by_user_id="ryan",
        ))
        assert result.target_id is not None
        assert mock_session.add.called
```

- [ ] **Step 3: Implement add_watch_target use case**

Create `apps/api/src/prescient/monitoring/application/use_cases/add_watch_target.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow


@dataclass(frozen=True)
class AddWatchTargetRequest:
    organization_id: str
    target_type: str
    target_name: str
    purpose: str
    monitoring_config: dict | None = None
    created_by_user_id: str | None = None


@dataclass(frozen=True)
class AddWatchTargetResult:
    target_id: str


class AddWatchTarget:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, request: AddWatchTargetRequest) -> AddWatchTargetResult:
        target_id = str(uuid4())
        self._session.add(MonitorTargetRow(
            id=target_id,
            organization_id=request.organization_id,
            target_type=request.target_type,
            target_name=request.target_name,
            purpose=request.purpose,
            monitoring_config=request.monitoring_config,
            created_by_user_id=request.created_by_user_id,
            active=True,
            created_at=datetime.now(UTC),
        ))
        return AddWatchTargetResult(target_id=target_id)
```

- [ ] **Step 4: Implement list and remove use cases**

Create `apps/api/src/prescient/monitoring/application/use_cases/list_watch_targets.py`:

```python
from __future__ import annotations

from dataclasses import dataclass

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow


@dataclass(frozen=True)
class WatchTargetSummary:
    target_id: str
    target_type: str
    target_name: str
    purpose: str
    active: bool


class ListWatchTargets:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, organization_id: str) -> list[WatchTargetSummary]:
        result = await self._session.execute(
            select(MonitorTargetRow).where(
                MonitorTargetRow.organization_id == organization_id,
                MonitorTargetRow.active.is_(True),
            )
        )
        return [
            WatchTargetSummary(
                target_id=r.id,
                target_type=r.target_type,
                target_name=r.target_name,
                purpose=r.purpose,
                active=r.active,
            )
            for r in result.scalars().all()
        ]
```

Create `apps/api/src/prescient/monitoring/application/use_cases/remove_watch_target.py`:

```python
from __future__ import annotations

from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow
from prescient.shared.errors import NotFoundError


class RemoveWatchTarget:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, target_id: str) -> None:
        result = await self._session.execute(
            select(MonitorTargetRow).where(MonitorTargetRow.id == target_id)
        )
        row = result.scalar_one_or_none()
        if row is None:
            raise NotFoundError(f"Watch target not found: {target_id}")
        await self._session.execute(
            update(MonitorTargetRow)
            .where(MonitorTargetRow.id == target_id)
            .values(active=False)
        )
```

- [ ] **Step 5: Implement monitoring API routes**

Create `apps/api/src/prescient/monitoring/api/routes.py`:

```python
from __future__ import annotations

from pydantic import BaseModel, Field
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.db import get_session
from prescient.monitoring.application.use_cases.add_watch_target import (
    AddWatchTarget,
    AddWatchTargetRequest,
)
from prescient.monitoring.application.use_cases.list_watch_targets import ListWatchTargets
from prescient.monitoring.application.use_cases.remove_watch_target import RemoveWatchTarget

router = APIRouter(prefix="/monitoring", tags=["monitoring"])


class AddWatchTargetBody(BaseModel):
    organization_id: str
    target_type: str = Field(min_length=1, max_length=16)
    target_name: str = Field(min_length=1, max_length=256)
    purpose: str = Field(min_length=1, max_length=32)
    monitoring_config: dict | None = None
    created_by_user_id: str | None = None


class WatchTargetResponse(BaseModel):
    target_id: str
    target_type: str
    target_name: str
    purpose: str
    active: bool


@router.post("/targets", response_model=WatchTargetResponse, status_code=201)
async def add_watch_target(
    body: AddWatchTargetBody,
    session: AsyncSession = Depends(get_session),
) -> WatchTargetResponse:
    use_case = AddWatchTarget(session=session)
    result = await use_case.execute(AddWatchTargetRequest(
        organization_id=body.organization_id,
        target_type=body.target_type,
        target_name=body.target_name,
        purpose=body.purpose,
        monitoring_config=body.monitoring_config,
        created_by_user_id=body.created_by_user_id,
    ))
    return WatchTargetResponse(
        target_id=result.target_id,
        target_type=body.target_type,
        target_name=body.target_name,
        purpose=body.purpose,
        active=True,
    )


@router.get("/targets", response_model=list[WatchTargetResponse])
async def list_watch_targets(
    organization_id: str,
    session: AsyncSession = Depends(get_session),
) -> list[WatchTargetResponse]:
    use_case = ListWatchTargets(session=session)
    targets = await use_case.execute(organization_id)
    return [
        WatchTargetResponse(
            target_id=t.target_id,
            target_type=t.target_type,
            target_name=t.target_name,
            purpose=t.purpose,
            active=t.active,
        )
        for t in targets
    ]


@router.delete("/targets/{target_id}", status_code=204)
async def remove_watch_target(
    target_id: str,
    session: AsyncSession = Depends(get_session),
) -> None:
    use_case = RemoveWatchTarget(session=session)
    await use_case.execute(target_id)
    await session.commit()
```

- [ ] **Step 6: Wire monitoring router into main.py**

In `apps/api/src/prescient/main.py`, add:

```python
from prescient.monitoring.api.routes import router as monitoring_router
```

And include: `app.include_router(monitoring_router)`

- [ ] **Step 7: Run tests**

Run: `cd apps/api && uv run python -m pytest tests/monitoring/ -v`
Expected: PASSED

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/monitoring/ apps/api/src/prescient/main.py tests/monitoring/
git commit -m "feat: add Monitoring API — watch target add, list, remove"
```

---

## Task 11: Onboarding API — Company Setup + Competitor Research

**Files:**
- Create: `apps/api/src/prescient/onboarding/application/__init__.py`
- Create: `apps/api/src/prescient/onboarding/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/onboarding/application/use_cases/setup_company.py`
- Create: `apps/api/src/prescient/onboarding/application/use_cases/add_competitor.py`
- Create: `apps/api/src/prescient/onboarding/application/use_cases/setup_user_interests.py`
- Create: `apps/api/src/prescient/onboarding/api/__init__.py`
- Create: `apps/api/src/prescient/onboarding/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`
- Test: `apps/api/tests/onboarding/test_onboarding_api.py`

- [ ] **Step 1: Create application and API packages**

Create empty `__init__.py` files.

- [ ] **Step 2: Write failing test for setup_company**

Create `apps/api/tests/onboarding/test_onboarding_api.py`:

```python
from datetime import datetime, UTC
from unittest.mock import AsyncMock
from uuid import uuid4

import pytest

from prescient.onboarding.application.use_cases.setup_company import (
    SetupCompany,
    SetupCompanyRequest,
)


class TestSetupCompany:
    @pytest.fixture
    def mock_session(self) -> AsyncMock:
        return AsyncMock()

    @pytest.mark.asyncio
    async def test_creates_company_setup(self, mock_session: AsyncMock) -> None:
        use_case = SetupCompany(session=mock_session)
        result = await use_case.execute(SetupCompanyRequest(
            organization_id=str(uuid4()),
            company_name="Olaplex Holdings",
            industry="Beauty & Personal Care",
        ))
        assert result.setup_id is not None
        assert mock_session.add.called
```

- [ ] **Step 3: Implement setup_company use case**

Create `apps/api/src/prescient/onboarding/application/use_cases/setup_company.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.onboarding.infrastructure.tables.company_setup import CompanySetupRow


@dataclass(frozen=True)
class SetupCompanyRequest:
    organization_id: str
    company_name: str
    industry: str | None = None


@dataclass(frozen=True)
class SetupCompanyResult:
    setup_id: str


class SetupCompany:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, request: SetupCompanyRequest) -> SetupCompanyResult:
        now = datetime.now(UTC)
        setup_id = str(uuid4())
        self._session.add(CompanySetupRow(
            id=setup_id,
            organization_id=request.organization_id,
            company_name=request.company_name,
            industry=request.industry,
            status="in_progress",
            competitors=[],
            created_at=now,
            updated_at=now,
        ))
        return SetupCompanyResult(setup_id=setup_id)
```

- [ ] **Step 4: Implement add_competitor use case**

This is the key use case — adding a competitor triggers research automatically.

Create `apps/api/src/prescient/onboarding/application/use_cases/add_competitor.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC

from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.intelligence.application.research.research_service import ResearchRequest, ResearchResult, ResearchService
from prescient.intelligence.application.research.company_profile_builder import CompanyProfileBuilder
from prescient.intelligence.infrastructure.sources.edgar_source import EdgarResearchSource
from prescient.monitoring.application.use_cases.add_watch_target import AddWatchTarget, AddWatchTargetRequest
from prescient.onboarding.infrastructure.tables.company_setup import CompanySetupRow
from prescient.shared.errors import NotFoundError


@dataclass(frozen=True)
class AddCompetitorRequest:
    setup_id: str
    organization_id: str
    competitor_name: str
    how_they_compete: str | None = None
    cik: str | None = None
    monitoring_scope: str = "full"
    requested_by: str = "system"


@dataclass(frozen=True)
class AddCompetitorResult:
    artifact_id: str | None
    profile_summary: str | None
    watch_target_id: str | None


class AddCompetitor:
    def __init__(
        self,
        session: AsyncSession,
        edgar_source: EdgarResearchSource,
        llm_adapter: object,
    ) -> None:
        self._session = session
        self._edgar_source = edgar_source
        self._llm_adapter = llm_adapter

    async def execute(self, request: AddCompetitorRequest) -> AddCompetitorResult:
        # Verify setup exists
        result = await self._session.execute(
            select(CompanySetupRow).where(CompanySetupRow.id == request.setup_id)
        )
        setup_row = result.scalar_one_or_none()
        if setup_row is None:
            raise NotFoundError(f"Company setup not found: {request.setup_id}")

        # Run research on the competitor
        builder = CompanyProfileBuilder(llm_adapter=self._llm_adapter)
        research_svc = ResearchService(
            edgar_source=self._edgar_source,
            profile_builder=builder,
            session=self._session,
        )
        research_result = await research_svc.research(ResearchRequest(
            organization_id=request.organization_id,
            company_name=request.competitor_name,
            cik=request.cik,
            requested_by=request.requested_by,
        ))

        # Add to watch targets
        watch_use_case = AddWatchTarget(session=self._session)
        watch_result = await watch_use_case.execute(AddWatchTargetRequest(
            organization_id=request.organization_id,
            target_type="company",
            target_name=request.competitor_name,
            purpose="competitive_intel",
            monitoring_config={"scope": request.monitoring_scope, "cik": request.cik},
            created_by_user_id=request.requested_by,
        ))

        # Update competitors list on setup
        competitors = list(setup_row.competitors or [])
        competitors.append({
            "name": request.competitor_name,
            "how_they_compete": request.how_they_compete,
            "cik": request.cik,
            "monitoring_scope": request.monitoring_scope,
            "artifact_id": research_result.artifact_id,
        })
        await self._session.execute(
            update(CompanySetupRow)
            .where(CompanySetupRow.id == request.setup_id)
            .values(competitors=competitors, updated_at=datetime.now(UTC))
        )

        return AddCompetitorResult(
            artifact_id=research_result.artifact_id,
            profile_summary=research_result.profile_summary,
            watch_target_id=watch_result.target_id,
        )
```

- [ ] **Step 5: Implement setup_user_interests use case**

Create `apps/api/src/prescient/onboarding/application/use_cases/setup_user_interests.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import uuid4

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.onboarding.infrastructure.tables.interest_profile import InterestProfileRow


@dataclass(frozen=True)
class SetupUserInterestsRequest:
    user_id: str
    organization_id: str
    topics: list[str]
    industries: list[str]
    technologies: list[str]
    competitors_of_interest: list[str]


@dataclass(frozen=True)
class SetupUserInterestsResult:
    profile_id: str


class SetupUserInterests:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, request: SetupUserInterestsRequest) -> SetupUserInterestsResult:
        now = datetime.now(UTC)

        # Check for existing profile
        existing = await self._session.execute(
            select(InterestProfileRow).where(InterestProfileRow.user_id == request.user_id)
        )
        row = existing.scalar_one_or_none()

        if row is not None:
            # Update existing
            row.topics = request.topics
            row.industries = request.industries
            row.technologies = request.technologies
            row.competitors_of_interest = request.competitors_of_interest
            row.updated_at = now
            return SetupUserInterestsResult(profile_id=row.id)

        # Create new
        profile_id = str(uuid4())
        self._session.add(InterestProfileRow(
            id=profile_id,
            user_id=request.user_id,
            organization_id=request.organization_id,
            topics=request.topics,
            industries=request.industries,
            technologies=request.technologies,
            competitors_of_interest=request.competitors_of_interest,
            created_at=now,
            updated_at=now,
        ))
        return SetupUserInterestsResult(profile_id=profile_id)
```

- [ ] **Step 6: Implement onboarding API routes**

Create `apps/api/src/prescient/onboarding/api/routes.py`:

```python
from __future__ import annotations

from pydantic import BaseModel, Field
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.config import get_settings
from prescient.db import get_session
from prescient.onboarding.application.use_cases.setup_company import SetupCompany, SetupCompanyRequest
from prescient.onboarding.application.use_cases.add_competitor import AddCompetitor, AddCompetitorRequest
from prescient.onboarding.application.use_cases.setup_user_interests import SetupUserInterests, SetupUserInterestsRequest

router = APIRouter(prefix="/onboarding", tags=["onboarding"])


class SetupCompanyBody(BaseModel):
    organization_id: str
    company_name: str = Field(min_length=1, max_length=256)
    industry: str | None = None


class SetupCompanyResponse(BaseModel):
    setup_id: str


class AddCompetitorBody(BaseModel):
    setup_id: str
    organization_id: str
    competitor_name: str = Field(min_length=1, max_length=256)
    how_they_compete: str | None = None
    cik: str | None = None
    monitoring_scope: str = "full"
    requested_by: str = "system"


class AddCompetitorResponse(BaseModel):
    artifact_id: str | None
    profile_summary: str | None
    watch_target_id: str | None


class SetupUserInterestsBody(BaseModel):
    user_id: str
    organization_id: str
    topics: list[str] = []
    industries: list[str] = []
    technologies: list[str] = []
    competitors_of_interest: list[str] = []


class SetupUserInterestsResponse(BaseModel):
    profile_id: str


@router.post("/company", response_model=SetupCompanyResponse, status_code=201)
async def setup_company(
    body: SetupCompanyBody,
    session: AsyncSession = Depends(get_session),
) -> SetupCompanyResponse:
    use_case = SetupCompany(session=session)
    result = await use_case.execute(SetupCompanyRequest(
        organization_id=body.organization_id,
        company_name=body.company_name,
        industry=body.industry,
    ))
    await session.commit()
    return SetupCompanyResponse(setup_id=result.setup_id)


@router.post("/company/competitors", response_model=AddCompetitorResponse, status_code=201)
async def add_competitor(
    body: AddCompetitorBody,
    session: AsyncSession = Depends(get_session),
) -> AddCompetitorResponse:
    settings = get_settings()
    from seed.edgar import EdgarClient
    from prescient.intelligence.infrastructure.anthropic_adapter import AnthropicAdapter
    from prescient.intelligence.infrastructure.sources.edgar_source import EdgarResearchSource

    edgar_client = EdgarClient(user_agent=settings.sec_edgar_user_agent)
    edgar_source = EdgarResearchSource(edgar_client=edgar_client)
    llm_adapter = AnthropicAdapter(api_key=settings.anthropic_api_key, model=settings.research_llm_model)

    use_case = AddCompetitor(session=session, edgar_source=edgar_source, llm_adapter=llm_adapter)
    result = await use_case.execute(AddCompetitorRequest(
        setup_id=body.setup_id,
        organization_id=body.organization_id,
        competitor_name=body.competitor_name,
        how_they_compete=body.how_they_compete,
        cik=body.cik,
        monitoring_scope=body.monitoring_scope,
        requested_by=body.requested_by,
    ))
    await session.commit()
    return AddCompetitorResponse(
        artifact_id=result.artifact_id,
        profile_summary=result.profile_summary,
        watch_target_id=result.watch_target_id,
    )


@router.post("/interests", response_model=SetupUserInterestsResponse, status_code=201)
async def setup_user_interests(
    body: SetupUserInterestsBody,
    session: AsyncSession = Depends(get_session),
) -> SetupUserInterestsResponse:
    use_case = SetupUserInterests(session=session)
    result = await use_case.execute(SetupUserInterestsRequest(
        user_id=body.user_id,
        organization_id=body.organization_id,
        topics=body.topics,
        industries=body.industries,
        technologies=body.technologies,
        competitors_of_interest=body.competitors_of_interest,
    ))
    await session.commit()
    return SetupUserInterestsResponse(profile_id=result.profile_id)
```

- [ ] **Step 7: Wire onboarding router into main.py**

In `apps/api/src/prescient/main.py`, add:

```python
from prescient.onboarding.api.routes import router as onboarding_router
```

And include: `app.include_router(onboarding_router)`

- [ ] **Step 8: Run all tests**

Run: `cd apps/api && uv run python -m pytest tests/onboarding/ tests/monitoring/ tests/intelligence/ -v`
Expected: All pass.

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/onboarding/ apps/api/src/prescient/main.py tests/onboarding/
git commit -m "feat: add onboarding API — company setup, competitor research, interest profiles"
```

---

## Task 12: Frontend — Onboarding Wizard Pages

**Files:**
- Create: `apps/web/src/app/(main)/onboarding/page.tsx`
- Create: `apps/web/src/app/(main)/onboarding/company/page.tsx`
- Create: `apps/web/src/app/(main)/onboarding/competitors/page.tsx`
- Create: `apps/web/src/app/(main)/onboarding/interests/page.tsx`
- Create: `apps/web/src/app/(main)/research/[slug]/page.tsx`
- Create: `apps/web/src/components/research-status.tsx`
- Create: `apps/web/src/components/company-profile-card.tsx`

- [ ] **Step 1: Read existing frontend patterns**

Read `apps/web/src/app/(main)/brief/page.tsx`, `apps/web/src/components/nav.tsx`, and `apps/web/src/lib/auth.ts` to understand current patterns.

- [ ] **Step 2: Create onboarding entry page**

Create `apps/web/src/app/(main)/onboarding/page.tsx`:

```tsx
"use client";

import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function OnboardingPage() {
  const router = useRouter();

  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-semibold">Get Started</h1>
      <p className="text-neutral-500">
        Set up your company profile, name your competitors, and tell us what you care about.
      </p>
      <div className="grid gap-4 md:grid-cols-3">
        <Card className="cursor-pointer hover:border-indigo-300" onClick={() => router.push("/onboarding/company")}>
          <CardHeader>
            <CardTitle className="text-lg">1. Company Setup</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-sm text-neutral-500">Name your company and industry</p>
          </CardContent>
        </Card>
        <Card className="cursor-pointer hover:border-indigo-300" onClick={() => router.push("/onboarding/competitors")}>
          <CardHeader>
            <CardTitle className="text-lg">2. Competitors</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-sm text-neutral-500">Name competitors and watch the system research them</p>
          </CardContent>
        </Card>
        <Card className="cursor-pointer hover:border-indigo-300" onClick={() => router.push("/onboarding/interests")}>
          <CardHeader>
            <CardTitle className="text-lg">3. Your Interests</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-sm text-neutral-500">Tell us what topics matter to you</p>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Create company setup page**

Create `apps/web/src/app/(main)/onboarding/company/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { getToken } from "@/lib/auth";

const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

export default function CompanySetupPage() {
  const [companyName, setCompanyName] = useState("");
  const [industry, setIndustry] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const router = useRouter();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);
    setLoading(true);
    try {
      const res = await fetch(`${API_BASE}/onboarding/company`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${getToken()}`,
        },
        body: JSON.stringify({
          organization_id: "placeholder",
          company_name: companyName,
          industry: industry || null,
        }),
      });
      if (!res.ok) throw new Error("Setup failed");
      const data = await res.json();
      localStorage.setItem("prescient_setup_id", data.setup_id);
      router.push("/onboarding/competitors");
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="mx-auto max-w-lg space-y-6">
      <h1 className="text-2xl font-semibold">Company Setup</h1>
      <Card>
        <CardHeader>
          <CardTitle>Tell us about your company</CardTitle>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit} className="flex flex-col gap-4">
            <Input placeholder="Company Name" value={companyName} onChange={(e) => setCompanyName(e.target.value)} required />
            <Input placeholder="Industry (optional)" value={industry} onChange={(e) => setIndustry(e.target.value)} />
            {error && <p className="text-sm text-red-600">{error}</p>}
            <Button type="submit" disabled={loading}>{loading ? "Setting up..." : "Continue"}</Button>
          </form>
        </CardContent>
      </Card>
    </div>
  );
}
```

- [ ] **Step 4: Create competitors page with live research**

Create `apps/web/src/app/(main)/onboarding/competitors/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { ResearchStatus } from "@/components/research-status";
import { getToken } from "@/lib/auth";

const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

type ResearchedCompetitor = {
  name: string;
  profile_summary: string | null;
  artifact_id: string | null;
  status: "idle" | "researching" | "done" | "error";
};

export default function CompetitorsPage() {
  const [competitorName, setCompetitorName] = useState("");
  const [howTheyCompete, setHowTheyCompete] = useState("");
  const [cik, setCik] = useState("");
  const [competitors, setCompetitors] = useState<ResearchedCompetitor[]>([]);
  const router = useRouter();

  const addCompetitor = async () => {
    if (!competitorName.trim()) return;
    const name = competitorName.trim();
    setCompetitors((prev) => [...prev, { name, profile_summary: null, artifact_id: null, status: "researching" }]);
    setCompetitorName("");
    setHowTheyCompete("");
    setCik("");

    try {
      const setupId = localStorage.getItem("prescient_setup_id") ?? "";
      const res = await fetch(`${API_BASE}/onboarding/company/competitors`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${getToken()}`,
        },
        body: JSON.stringify({
          setup_id: setupId,
          organization_id: "placeholder",
          competitor_name: name,
          how_they_compete: howTheyCompete || null,
          cik: cik || null,
        }),
      });
      if (!res.ok) throw new Error("Research failed");
      const data = await res.json();
      setCompetitors((prev) =>
        prev.map((c) =>
          c.name === name ? { ...c, profile_summary: data.profile_summary, artifact_id: data.artifact_id, status: "done" } : c
        )
      );
    } catch {
      setCompetitors((prev) => prev.map((c) => (c.name === name ? { ...c, status: "error" } : c)));
    }
  };

  return (
    <div className="mx-auto max-w-2xl space-y-6">
      <h1 className="text-2xl font-semibold">Name Your Competitors</h1>
      <p className="text-neutral-500">Add competitors and watch the system research them in real-time.</p>
      <Card>
        <CardContent className="pt-6">
          <div className="flex flex-col gap-3">
            <Input placeholder="Competitor name" value={competitorName} onChange={(e) => setCompetitorName(e.target.value)} />
            <Input placeholder="How do they compete with you? (optional)" value={howTheyCompete} onChange={(e) => setHowTheyCompete(e.target.value)} />
            <Input placeholder="SEC CIK number (optional, for public companies)" value={cik} onChange={(e) => setCik(e.target.value)} />
            <Button onClick={addCompetitor} disabled={!competitorName.trim()}>Add &amp; Research</Button>
          </div>
        </CardContent>
      </Card>

      {competitors.length > 0 && (
        <div className="space-y-3">
          <h2 className="text-lg font-medium">Research Results</h2>
          {competitors.map((c) => (
            <ResearchStatus key={c.name} name={c.name} status={c.status} summary={c.profile_summary} />
          ))}
        </div>
      )}

      <Button variant="outline" onClick={() => router.push("/onboarding/interests")} className="mt-4">
        Continue to Interests
      </Button>
    </div>
  );
}
```

- [ ] **Step 5: Create ResearchStatus component**

Create `apps/web/src/components/research-status.tsx`:

```tsx
import { Card, CardContent } from "@/components/ui/card";

type Props = {
  name: string;
  status: "idle" | "researching" | "done" | "error";
  summary: string | null;
};

export function ResearchStatus({ name, status, summary }: Props) {
  return (
    <Card>
      <CardContent className="flex items-start gap-3 pt-4">
        <div className="mt-0.5">
          {status === "researching" && <span className="inline-block h-3 w-3 animate-pulse rounded-full bg-indigo-500" />}
          {status === "done" && <span className="inline-block h-3 w-3 rounded-full bg-green-500" />}
          {status === "error" && <span className="inline-block h-3 w-3 rounded-full bg-red-500" />}
          {status === "idle" && <span className="inline-block h-3 w-3 rounded-full bg-neutral-300" />}
        </div>
        <div className="flex-1">
          <p className="font-medium">{name}</p>
          {status === "researching" && <p className="text-sm text-neutral-500">Researching...</p>}
          {status === "done" && summary && <p className="text-sm text-neutral-600">{summary}</p>}
          {status === "error" && <p className="text-sm text-red-600">Research failed — try again later</p>}
        </div>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 6: Create interests page**

Create `apps/web/src/app/(main)/onboarding/interests/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { getToken } from "@/lib/auth";

const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

const SUGGESTED_TOPICS = [
  "AI & Machine Learning", "Supply Chain", "Talent Market", "Regulatory Changes",
  "M&A Activity", "ESG / Sustainability", "Digital Transformation", "Pricing Strategy",
];

export default function InterestsPage() {
  const [selectedTopics, setSelectedTopics] = useState<string[]>([]);
  const [customTopic, setCustomTopic] = useState("");
  const [loading, setLoading] = useState(false);
  const router = useRouter();

  const toggleTopic = (topic: string) => {
    setSelectedTopics((prev) => (prev.includes(topic) ? prev.filter((t) => t !== topic) : [...prev, topic]));
  };

  const addCustom = () => {
    if (customTopic.trim() && !selectedTopics.includes(customTopic.trim())) {
      setSelectedTopics((prev) => [...prev, customTopic.trim()]);
      setCustomTopic("");
    }
  };

  const handleSave = async () => {
    setLoading(true);
    try {
      await fetch(`${API_BASE}/onboarding/interests`, {
        method: "POST",
        headers: { "Content-Type": "application/json", Authorization: `Bearer ${getToken()}` },
        body: JSON.stringify({
          user_id: localStorage.getItem("prescient_user_id") ?? "",
          organization_id: "placeholder",
          topics: selectedTopics,
          industries: [],
          technologies: [],
          competitors_of_interest: [],
        }),
      });
      router.push("/brief");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="mx-auto max-w-lg space-y-6">
      <h1 className="text-2xl font-semibold">What do you care about?</h1>
      <p className="text-neutral-500">Select topics that matter to your role. We will filter intelligence to what is worth your time.</p>
      <Card>
        <CardHeader>
          <CardTitle className="text-base">Select topics</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="flex flex-wrap gap-2">
            {SUGGESTED_TOPICS.map((topic) => (
              <button
                key={topic}
                onClick={() => toggleTopic(topic)}
                className={`rounded-full border px-3 py-1 text-sm transition ${
                  selectedTopics.includes(topic)
                    ? "border-indigo-500 bg-indigo-50 text-indigo-700"
                    : "border-neutral-300 text-neutral-600 hover:border-neutral-400"
                }`}
              >
                {topic}
              </button>
            ))}
          </div>
          <div className="mt-4 flex gap-2">
            <Input placeholder="Add a custom topic" value={customTopic} onChange={(e) => setCustomTopic(e.target.value)} onKeyDown={(e) => e.key === "Enter" && addCustom()} />
            <Button variant="outline" onClick={addCustom}>Add</Button>
          </div>
        </CardContent>
      </Card>
      <Button onClick={handleSave} disabled={loading}>{loading ? "Saving..." : "Finish Setup"}</Button>
    </div>
  );
}
```

- [ ] **Step 7: Verify frontend compiles**

Run: `cd apps/web && npx tsc --noEmit`
Expected: No errors.

- [ ] **Step 8: Commit**

```bash
git add apps/web/src/
git commit -m "feat: add onboarding wizard — company setup, competitor research, interest profiles"
```

---

## Task 13: Lint, Typecheck, and Final Verification

**Files:** None new — verification only.

- [ ] **Step 1: Run backend linter**

Run: `cd apps/api && uv run python -m ruff check src/prescient/intelligence/ src/prescient/monitoring/ src/prescient/onboarding/`

- [ ] **Step 2: Run backend type checker**

Run: `cd apps/api && uv run python -m mypy src/prescient/intelligence/application/research/ src/prescient/onboarding/ src/prescient/monitoring/application/ src/prescient/monitoring/api/`

- [ ] **Step 3: Run frontend type check**

Run: `cd apps/web && npx tsc --noEmit`

- [ ] **Step 4: Run full test suite**

Run: `cd apps/api && uv run python -m pytest tests/intelligence/ tests/monitoring/ tests/onboarding/ -v`

- [ ] **Step 5: Fix any issues**

Address lint, type, or test failures.

- [ ] **Step 6: Final commit if fixes needed**

```bash
git add -A && git commit -m "fix: address lint and typecheck issues from Phase 2"
```

---

## Summary

Phase 2 delivers:
- **Company Research Service** — orchestrates EDGAR + LLM to produce structured Company Profile artifacts. Endpoint: `POST /research/companies`
- **Company Profile Builder** — LLM prompt that synthesizes raw research into structured blocks (overview, financials, products, competitive position, risks, recent developments)
- **EDGAR Source Adapter** — wraps existing seed pipeline for async company research
- **Monitoring context** — SQLAlchemy tables for targets/sources/runs/findings + API for watch target CRUD (`POST/GET/DELETE /monitoring/targets`)
- **Onboarding Wizard** — Company setup (`POST /onboarding/company`), add competitor with auto-research (`POST /onboarding/company/competitors`), user interest profiles (`POST /onboarding/interests`)
- **Frontend** — 4-step onboarding wizard (entry → company → competitors with live research → interests), research status indicator component
- **Database** — Alembic migration for monitoring + onboarding schemas

Next: **Phase 3 — Operating Core: KPI Management, Action & Decision Log, Daily Briefing**
