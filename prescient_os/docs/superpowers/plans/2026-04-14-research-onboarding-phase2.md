# Research Engine & Smart Onboarding Phase 2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build async research infrastructure, remaining copilot tools, data model updates, onboarding interview UI, and standalone research page.

**Architecture:** Infrastructure-first approach. Layer 1 adds async Celery tasks, source tiers, research completion events, and notifications. Layer 2 adds `research_market` and `research_person` tools plus a quick→deep upgrade helper. Layer 3 updates the data model (watched companies, platform goals). Layers 4-5 build the frontend: onboarding interview with concurrent research, and a split-view `/research` page with depth/source controls.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Celery + Redis, Next.js 15 (app router), TypeScript, Zustand, React Query

**Spec:** `docs/superpowers/specs/2026-04-14-research-onboarding-phase2-design.md`

---

## File Structure

### Modified: Research domain

```
apps/api/src/prescient/research/
├── domain/
│   ├── entities/
│   │   └── research_task.py          # Add STANDARD mode, SourceTier enum
│   └── ports.py                      # Replace quick_pass with tier: SourceTier
├── application/
│   ├── source_registry.py            # Tier-based + enabled_sources filtering
│   ├── pipeline.py                   # Accept enabled_sources param
│   └── use_cases/
│       ├── run_research.py           # Add enabled_sources to command
│       └── upgrade_research.py       # NEW: quick→deep upgrade helper
├── infrastructure/
│   ├── sources/
│   │   ├── website_scraper.py        # Replace quick_pass with tier
│   │   ├── search_engine_source.py   # Replace quick_pass with tier
│   │   ├── dns_lookup_source.py      # Replace quick_pass with tier
│   │   ├── news_source.py            # Replace quick_pass with tier
│   │   └── edgar_source.py           # Replace quick_pass with tier
│   └── tasks/
│       ├── __init__.py               # NEW
│       └── deep_research.py          # NEW: Celery tasks
└── api/
    └── routes.py                     # Add GET /sources, enabled_sources param
```

### New: Research event subscriber

```
apps/api/src/prescient/events/subscribers/
└── research_to_notification.py       # NEW: research.task.completed → notification
```

### Modified: Notifications

```
apps/api/src/prescient/notifications/domain/enums.py  # Add RESEARCH category
```

### New: Copilot tools

```
apps/api/src/prescient/intelligence/infrastructure/tools/
├── research_market.py                # NEW
├── research_person.py                # NEW
└── __init__.py                       # Register new tools
```

### Modified: Onboarding

```
apps/api/src/prescient/onboarding/
├── domain/entities/user_context.py   # Rename field, add platform_goals
├── infrastructure/tables/user_context.py  # Add columns
└── application/use_cases/run_interview.py # Update prompts
```

### Modified: Celery & outbox

```
apps/api/src/prescient/celery_app.py  # Add research tasks autodiscover
apps/api/src/prescient/shared/infrastructure/tasks/outbox_drain.py  # Add subscriber
```

### New: Migration

```
apps/api/alembic/versions/20260414_research_phase2.py
```

### New: Tests

```
apps/api/tests/unit/research/test_source_tier.py
apps/api/tests/unit/research/test_upgrade_research.py
apps/api/tests/unit/research/test_deep_research_task.py
apps/api/tests/unit/intelligence/test_research_market_tool.py
apps/api/tests/unit/intelligence/test_research_person_tool.py
apps/api/tests/unit/onboarding/test_user_context_v2.py
apps/api/tests/unit/events/test_research_to_notification.py
```

---

## Task 1: Source Tier Enum and ResearchMode Expansion

**Files:**
- Modify: `apps/api/src/prescient/research/domain/entities/research_task.py`
- Modify: `apps/api/src/prescient/research/domain/ports.py`
- Test: `apps/api/tests/unit/research/test_source_tier.py`

- [ ] **Step 1: Write tests for SourceTier and expanded ResearchMode**

Create `apps/api/tests/unit/research/test_source_tier.py`:

```python
"""Unit tests for SourceTier enum and ResearchMode expansion."""

from __future__ import annotations

import pytest

from prescient.research.domain.entities.research_task import ResearchMode, SourceTier


class TestSourceTier:
    def test_tier_values(self) -> None:
        assert SourceTier.QUICK == "quick"
        assert SourceTier.STANDARD == "standard"
        assert SourceTier.DEEP == "deep"

    def test_tier_ordering(self) -> None:
        """Tiers have a natural ordering: quick < standard < deep."""
        assert SourceTier.QUICK.rank < SourceTier.STANDARD.rank
        assert SourceTier.STANDARD.rank < SourceTier.DEEP.rank

    def test_tier_included_in_mode(self) -> None:
        """A source is included if its tier rank <= the mode rank."""
        assert SourceTier.QUICK.included_in(ResearchMode.QUICK)
        assert SourceTier.QUICK.included_in(ResearchMode.STANDARD)
        assert SourceTier.QUICK.included_in(ResearchMode.DEEP)

        assert not SourceTier.STANDARD.included_in(ResearchMode.QUICK)
        assert SourceTier.STANDARD.included_in(ResearchMode.STANDARD)
        assert SourceTier.STANDARD.included_in(ResearchMode.DEEP)

        assert not SourceTier.DEEP.included_in(ResearchMode.QUICK)
        assert not SourceTier.DEEP.included_in(ResearchMode.STANDARD)
        assert SourceTier.DEEP.included_in(ResearchMode.DEEP)


class TestResearchModeExpanded:
    def test_standard_mode_exists(self) -> None:
        assert ResearchMode.STANDARD == "standard"

    def test_all_modes(self) -> None:
        assert set(ResearchMode) == {ResearchMode.QUICK, ResearchMode.STANDARD, ResearchMode.DEEP}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/research/test_source_tier.py -v`
Expected: ImportError for `SourceTier`

- [ ] **Step 3: Add SourceTier enum and STANDARD mode**

In `apps/api/src/prescient/research/domain/entities/research_task.py`, add `SourceTier` enum and `STANDARD` to `ResearchMode`:

```python
class ResearchMode(StrEnum):
    QUICK = "quick"
    STANDARD = "standard"
    DEEP = "deep"


_MODE_RANK = {
    ResearchMode.QUICK: 0,
    ResearchMode.STANDARD: 1,
    ResearchMode.DEEP: 2,
}

_TIER_RANK = {
    "quick": 0,
    "standard": 1,
    "deep": 2,
}


class SourceTier(StrEnum):
    QUICK = "quick"
    STANDARD = "standard"
    DEEP = "deep"

    @property
    def rank(self) -> int:
        return _TIER_RANK[self.value]

    def included_in(self, mode: ResearchMode) -> bool:
        """Return True if this tier should run for the given mode."""
        return self.rank <= _MODE_RANK[mode]
```

- [ ] **Step 4: Update ResearchSource protocol**

In `apps/api/src/prescient/research/domain/ports.py`, replace `quick_pass: bool` with `tier: SourceTier`:

```python
from prescient.research.domain.entities.research_task import SourceTier

@runtime_checkable
class ResearchSource(Protocol):
    """Contract for research source plugins."""

    name: str
    timeout_seconds: int
    tier: SourceTier

    def can_handle(self, subject_type: SubjectType) -> bool: ...
    async def fetch(self, subject: Any) -> RawFindings: ...
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/research/test_source_tier.py -v`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/research/domain/entities/research_task.py
git add apps/api/src/prescient/research/domain/ports.py
git add apps/api/tests/unit/research/test_source_tier.py
git commit -m "feat(research): add SourceTier enum and STANDARD research mode"
```

---

## Task 2: Update Source Plugins and SourceRegistry for Tiers

**Files:**
- Modify: `apps/api/src/prescient/research/infrastructure/sources/website_scraper.py`
- Modify: `apps/api/src/prescient/research/infrastructure/sources/search_engine_source.py`
- Modify: `apps/api/src/prescient/research/infrastructure/sources/dns_lookup_source.py`
- Modify: `apps/api/src/prescient/research/infrastructure/sources/news_source.py`
- Modify: `apps/api/src/prescient/research/infrastructure/sources/edgar_source.py`
- Modify: `apps/api/src/prescient/research/application/source_registry.py`
- Modify: `apps/api/tests/unit/research/test_source_registry.py`

- [ ] **Step 1: Update all source plugins to use tier instead of quick_pass**

In each source file, replace `quick_pass = True/False` with the appropriate `tier`:

`website_scraper.py`:
```python
from prescient.research.domain.entities.research_task import SourceTier
# Replace: quick_pass = True
tier = SourceTier.QUICK
```

`search_engine_source.py`:
```python
from prescient.research.domain.entities.research_task import SourceTier
# Replace: quick_pass = True
tier = SourceTier.QUICK
```

`dns_lookup_source.py`:
```python
from prescient.research.domain.entities.research_task import SourceTier
# Replace: quick_pass = True
tier = SourceTier.QUICK
```

`news_source.py`:
```python
from prescient.research.domain.entities.research_task import SourceTier
# Replace: quick_pass = False
tier = SourceTier.STANDARD
```

`edgar_source.py`:
```python
from prescient.research.domain.entities.research_task import SourceTier
# Replace: quick_pass = False
tier = SourceTier.DEEP
```

- [ ] **Step 2: Update SourceRegistry to use tier-based filtering and enabled_sources**

Replace the `sources_for` method in `apps/api/src/prescient/research/application/source_registry.py`:

```python
"""Source registry — discovers and manages research source plugins."""

from __future__ import annotations

from prescient.research.domain.entities.research_subject import SubjectType
from prescient.research.domain.entities.research_task import ResearchMode
from prescient.research.domain.ports import ResearchSource


class SourceRegistry:
    """Manages available research source plugins."""

    def __init__(self) -> None:
        self._sources: dict[str, ResearchSource] = {}

    def register(self, source: ResearchSource) -> None:
        if source.name in self._sources:
            raise ValueError(f"source {source.name!r} already registered")
        self._sources[source.name] = source

    def all_sources(self) -> list[ResearchSource]:
        return list(self._sources.values())

    def sources_for(
        self,
        subject_type: SubjectType,
        mode: ResearchMode | None = None,
        enabled_sources: list[str] | None = None,
    ) -> list[ResearchSource]:
        """Return sources that can handle the given subject type, mode, and user selection."""
        result = []
        for source in self._sources.values():
            if not source.can_handle(subject_type):
                continue
            if mode is not None and not source.tier.included_in(mode):
                continue
            if enabled_sources is not None and source.name not in enabled_sources:
                continue
            result.append(source)
        return result

    def source_descriptors(self) -> list[dict]:
        """Return metadata about all registered sources for the discovery API."""
        return [
            {
                "name": s.name,
                "label": s.name.replace("_", " ").title(),
                "tier": s.tier.value,
                "timeout_seconds": s.timeout_seconds,
            }
            for s in self._sources.values()
        ]
```

- [ ] **Step 3: Update existing source registry tests**

Update `apps/api/tests/unit/research/test_source_registry.py` — replace the fake sources' `quick_pass` with `tier`:

```python
"""Unit tests for SourceRegistry."""

from __future__ import annotations

import pytest

from prescient.research.application.source_registry import SourceRegistry
from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    PersonSubject,
    SubjectType,
)
from prescient.research.domain.entities.research_task import ResearchMode, SourceTier
from prescient.research.domain.ports import ResearchSource


class _FakeWebSource:
    """Fake source that handles companies only."""

    name = "website_scraper"
    timeout_seconds = 10
    tier = SourceTier.QUICK

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: CompanySubject) -> RawFindings:
        return RawFindings(
            source_name=self.name,
            findings=(Finding(content="Website data for " + subject.name),),
        )


class _FakeEdgarSource:
    """Fake source that handles companies, deep tier only."""

    name = "edgar"
    timeout_seconds = 30
    tier = SourceTier.DEEP

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: CompanySubject) -> RawFindings:
        return RawFindings(
            source_name=self.name,
            findings=(Finding(content="Filing data"),),
        )


class _FakeNewsSource:
    """Fake source at standard tier."""

    name = "news"
    timeout_seconds = 15
    tier = SourceTier.STANDARD

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: CompanySubject) -> RawFindings:
        return RawFindings(
            source_name=self.name,
            findings=(Finding(content="News data"),),
        )


class TestSourceRegistry:
    def test_register_and_list(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        registry.register(_FakeEdgarSource())
        assert len(registry.all_sources()) == 2

    def test_sources_for_subject_filters_by_type(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        registry.register(_FakeEdgarSource())

        company = CompanySubject(name="Acme")
        sources = registry.sources_for(company.subject_type)
        assert len(sources) == 2

        person = PersonSubject(name="Jane")
        sources = registry.sources_for(person.subject_type)
        assert len(sources) == 0

    def test_quick_mode_filters_to_quick_tier(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        registry.register(_FakeEdgarSource())
        registry.register(_FakeNewsSource())

        sources = registry.sources_for(SubjectType.COMPANY, mode=ResearchMode.QUICK)
        assert len(sources) == 1
        assert sources[0].name == "website_scraper"

    def test_standard_mode_includes_quick_and_standard(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        registry.register(_FakeEdgarSource())
        registry.register(_FakeNewsSource())

        sources = registry.sources_for(SubjectType.COMPANY, mode=ResearchMode.STANDARD)
        names = {s.name for s in sources}
        assert names == {"website_scraper", "news"}

    def test_deep_mode_includes_all(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        registry.register(_FakeEdgarSource())
        registry.register(_FakeNewsSource())

        sources = registry.sources_for(SubjectType.COMPANY, mode=ResearchMode.DEEP)
        assert len(sources) == 3

    def test_enabled_sources_filter(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        registry.register(_FakeEdgarSource())
        registry.register(_FakeNewsSource())

        sources = registry.sources_for(
            SubjectType.COMPANY,
            mode=ResearchMode.DEEP,
            enabled_sources=["website_scraper", "edgar"],
        )
        names = {s.name for s in sources}
        assert names == {"website_scraper", "edgar"}

    def test_duplicate_name_raises(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        with pytest.raises(ValueError, match="already registered"):
            registry.register(_FakeWebSource())

    def test_source_descriptors(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        descriptors = registry.source_descriptors()
        assert len(descriptors) == 1
        assert descriptors[0]["name"] == "website_scraper"
        assert descriptors[0]["tier"] == "quick"
```

- [ ] **Step 4: Run all research tests to verify nothing broke**

Run: `cd apps/api && uv run pytest tests/unit/research/ -v`
Expected: All tests pass (existing and new).

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/research/infrastructure/sources/
git add apps/api/src/prescient/research/application/source_registry.py
git add apps/api/tests/unit/research/test_source_registry.py
git commit -m "feat(research): migrate sources from quick_pass to tier system with enabled_sources filtering"
```

---

## Task 3: Update Pipeline and RunResearch for enabled_sources

**Files:**
- Modify: `apps/api/src/prescient/research/application/pipeline.py`
- Modify: `apps/api/src/prescient/research/application/use_cases/run_research.py`
- Modify: `apps/api/src/prescient/research/api/routes.py`

- [ ] **Step 1: Add enabled_sources to pipeline**

In `apps/api/src/prescient/research/application/pipeline.py`, update `run()` to accept `enabled_sources`:

```python
    async def run(
        self,
        *,
        subject: ResearchSubject,
        mode: ResearchMode,
        triggered_by: ResearchTrigger,
        tenant_id: str,
        created_by: str,
        enabled_sources: list[str] | None = None,
    ) -> ResearchTask:
        task = ResearchTask.create(
            subject=subject,
            mode=mode,
            triggered_by=triggered_by,
            tenant_id=tenant_id,
            created_by=created_by,
        )

        # Stage 1: Discover available sources
        task = task.advance_to(TaskStatus.DISCOVERING)
        sources = self._sources.sources_for(
            subject.subject_type, mode=mode, enabled_sources=enabled_sources,
        )

        # ... rest unchanged
```

- [ ] **Step 2: Add enabled_sources to RunResearchCommand**

In `apps/api/src/prescient/research/application/use_cases/run_research.py`:

```python
@dataclass(frozen=True)
class RunResearchCommand:
    subject_type: str  # "company" | "person" | "market"
    subject_data: dict  # Fields depend on subject_type
    mode: str  # "quick" | "standard" | "deep"
    triggered_by: str  # "onboarding" | "copilot" | "system"
    tenant_id: str
    created_by: str
    enabled_sources: list[str] | None = None
```

And pass it through in `execute()`:

```python
    async def execute(self, command: RunResearchCommand) -> RunResearchResult:
        subject = self._build_subject(command)
        mode = ResearchMode(command.mode)
        triggered_by = ResearchTrigger(command.triggered_by)

        task = await self._pipeline.run(
            subject=subject,
            mode=mode,
            triggered_by=triggered_by,
            tenant_id=command.tenant_id,
            created_by=command.created_by,
            enabled_sources=command.enabled_sources,
        )

        return RunResearchResult(
            task_id=task.id,
            status=task.status,
            artifact_id=task.result_artifact_id,
            summary=task.synthesized_result.get("summary") if task.synthesized_result else None,
        )
```

- [ ] **Step 3: Update research API routes**

In `apps/api/src/prescient/research/api/routes.py`:

Update `RunResearchBody` to accept `enabled_sources` and allow `"standard"` mode:

```python
class RunResearchBody(BaseModel):
    model_config = ConfigDict(frozen=True)

    subject_type: str = Field(pattern="^(company|person|market)$")
    subject_data: dict
    mode: str = Field(default="deep", pattern="^(quick|standard|deep)$")
    enabled_sources: list[str] | None = None
```

Add the sources discovery endpoint and pass `enabled_sources` through:

```python
@router.get("/sources")
async def list_sources() -> list[dict]:
    """Return available research sources with their tier and metadata."""
    from prescient.research.application.source_registry import SourceRegistry
    from prescient.research.infrastructure.sources.dns_lookup_source import DnsLookupSource
    from prescient.research.infrastructure.sources.news_source import NewsAggregatorSource
    from prescient.research.infrastructure.sources.search_engine_source import SearchEngineSource
    from prescient.research.infrastructure.sources.website_scraper import WebsiteScraperSource

    registry = SourceRegistry()
    registry.register(WebsiteScraperSource())
    registry.register(SearchEngineSource())
    registry.register(NewsAggregatorSource())
    registry.register(DnsLookupSource())
    return registry.source_descriptors()
```

And in the existing `run_research` endpoint, pass `enabled_sources`:

```python
    result = await use_case.execute(
        RunResearchCommand(
            subject_type=body.subject_type,
            subject_data=body.subject_data,
            mode=body.mode,
            triggered_by="copilot",
            tenant_id=str(ctx.user.fund_id),
            created_by=str(ctx.user.user_id),
            enabled_sources=body.enabled_sources,
        )
    )
```

- [ ] **Step 4: Run all research tests**

Run: `cd apps/api && uv run pytest tests/unit/research/ -v`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/research/application/pipeline.py
git add apps/api/src/prescient/research/application/use_cases/run_research.py
git add apps/api/src/prescient/research/api/routes.py
git commit -m "feat(research): thread enabled_sources through pipeline and add GET /sources endpoint"
```

---

## Task 4: Deep Research Celery Task

**Files:**
- Create: `apps/api/src/prescient/research/infrastructure/tasks/__init__.py`
- Create: `apps/api/src/prescient/research/infrastructure/tasks/deep_research.py`
- Modify: `apps/api/src/prescient/celery_app.py`
- Test: `apps/api/tests/unit/research/test_deep_research_task.py`

- [ ] **Step 1: Write tests for deep research task**

Create `apps/api/tests/unit/research/test_deep_research_task.py`:

```python
"""Unit tests for deep research Celery task logic."""

from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from prescient.research.infrastructure.tasks.deep_research import (
    _run_deep_research_async,
)
from prescient.research.domain.entities.research_task import TaskStatus


class TestRunDeepResearchAsync:
    @pytest.mark.asyncio
    async def test_runs_pipeline_and_emits_event(self) -> None:
        mock_result = MagicMock()
        mock_result.task_id = "task-001"
        mock_result.status = TaskStatus.COMPLETED
        mock_result.artifact_id = "artifact-001"
        mock_result.summary = "Acme makes widgets."

        mock_use_case = AsyncMock()
        mock_use_case.execute.return_value = mock_result

        mock_session = AsyncMock()
        mock_session.__aenter__ = AsyncMock(return_value=mock_session)
        mock_session.__aexit__ = AsyncMock(return_value=False)

        with patch(
            "prescient.research.infrastructure.tasks.deep_research._build_run_research",
            return_value=mock_use_case,
        ), patch(
            "prescient.research.infrastructure.tasks.deep_research.SessionLocal",
            return_value=mock_session,
        ), patch(
            "prescient.research.infrastructure.tasks.deep_research.append_event",
        ) as mock_append:
            result = await _run_deep_research_async(
                subject_type="company",
                subject_data={"name": "Acme Corp"},
                mode="deep",
                triggered_by="copilot",
                tenant_id="org-001",
                created_by="user-001",
                enabled_sources=None,
            )

        assert result["task_id"] == "task-001"
        assert result["status"] == "completed"
        mock_append.assert_called_once()
        mock_session.commit.assert_called_once()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/research/test_deep_research_task.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement deep research task**

Create `apps/api/src/prescient/research/infrastructure/tasks/__init__.py` (empty).

Create `apps/api/src/prescient/research/infrastructure/tasks/deep_research.py`:

```python
"""Deep research Celery tasks — async pipeline execution with event emission."""

from __future__ import annotations

import asyncio
import logging

from celery import shared_task

from prescient.research.application.use_cases.run_research import (
    RunResearch,
    RunResearchCommand,
)
from prescient.shared.events import DomainEvent
from prescient.shared.outbox import append_event

logger = logging.getLogger(__name__)


def _build_run_research() -> object:
    """Build a fully-wired RunResearch use case. Import here to avoid circular deps."""
    from prescient.config import get_settings
    from prescient.intelligence.infrastructure.llm import AnthropicAdapter
    from prescient.research.application.pipeline import ResearchPipeline
    from prescient.research.application.source_registry import SourceRegistry
    from prescient.research.application.use_cases.extract_and_synthesize import (
        ExtractAndSynthesize,
    )
    from prescient.research.infrastructure.sources.dns_lookup_source import DnsLookupSource
    from prescient.research.infrastructure.sources.edgar_source import EdgarSource
    from prescient.research.infrastructure.sources.news_source import NewsAggregatorSource
    from prescient.research.infrastructure.sources.search_engine_source import SearchEngineSource
    from prescient.research.infrastructure.sources.website_scraper import WebsiteScraperSource

    settings = get_settings()
    llm = AnthropicAdapter(
        api_key=settings.anthropic_api_key,
        default_model=settings.anthropic_model,
    )
    registry = SourceRegistry()
    registry.register(WebsiteScraperSource())
    registry.register(SearchEngineSource())
    registry.register(NewsAggregatorSource())
    registry.register(DnsLookupSource())
    # EdgarSource wraps existing EDGAR client — skip if not configured
    try:
        from seed.edgar import EdgarClient

        registry.register(EdgarSource(edgar_research_source=EdgarClient()))
    except Exception:
        logger.info("EDGAR source not available, skipping")
    synthesizer = ExtractAndSynthesize(llm=llm)
    pipeline = ResearchPipeline(
        source_registry=registry,
        synthesizer=synthesizer,
        artifact_repo=None,
    )
    return RunResearch(pipeline=pipeline, session=None)


async def _run_deep_research_async(
    *,
    subject_type: str,
    subject_data: dict,
    mode: str,
    triggered_by: str,
    tenant_id: str,
    created_by: str,
    enabled_sources: list[str] | None,
) -> dict:
    from prescient.db import SessionLocal

    use_case = _build_run_research()

    result = await use_case.execute(
        RunResearchCommand(
            subject_type=subject_type,
            subject_data=subject_data,
            mode=mode,
            triggered_by=triggered_by,
            tenant_id=tenant_id,
            created_by=created_by,
            enabled_sources=enabled_sources,
        )
    )

    # Emit domain event for completion
    async with SessionLocal() as session:
        subject_name = subject_data.get("name") or subject_data.get("seed_company_name", "Unknown")
        await append_event(
            session,
            DomainEvent(
                aggregate_id=result.task_id,
                aggregate_type="research.task",
                event_type="research.task.completed",
                payload={
                    "task_id": result.task_id,
                    "subject_type": subject_type,
                    "subject_name": subject_name,
                    "status": result.status.value,
                    "artifact_id": result.artifact_id,
                    "tenant_id": tenant_id,
                    "created_by": created_by,
                },
            ),
        )
        await session.commit()

    return {
        "task_id": result.task_id,
        "status": result.status.value,
        "artifact_id": result.artifact_id,
    }


@shared_task(
    name="prescient.research.infrastructure.tasks.deep_research.run_deep_research",
    soft_time_limit=360,
    time_limit=420,
)
def run_deep_research(
    subject_type: str,
    subject_data: dict,
    mode: str,
    triggered_by: str,
    tenant_id: str,
    created_by: str,
    enabled_sources: list[str] | None = None,
) -> dict:
    """Run research pipeline asynchronously as a Celery task."""
    return asyncio.run(
        _run_deep_research_async(
            subject_type=subject_type,
            subject_data=subject_data,
            mode=mode,
            triggered_by=triggered_by,
            tenant_id=tenant_id,
            created_by=created_by,
            enabled_sources=enabled_sources,
        )
    )
```

- [ ] **Step 4: Add research tasks to Celery autodiscover**

In `apps/api/src/prescient/celery_app.py`, add `"prescient.research.infrastructure.tasks"` to the autodiscover list:

```python
    app.autodiscover_tasks(
        [
            "prescient.knowledge_mgmt.infrastructure.tasks",
            "prescient.monitoring.infrastructure.tasks",
            "prescient.kpis.infrastructure.tasks",
            "prescient.shared.infrastructure.tasks",
            "prescient.events.subscribers",
            "prescient.research.infrastructure.tasks",
        ]
    )
```

- [ ] **Step 5: Run tests**

Run: `cd apps/api && uv run pytest tests/unit/research/test_deep_research_task.py -v`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/research/infrastructure/tasks/
git add apps/api/src/prescient/celery_app.py
git add apps/api/tests/unit/research/test_deep_research_task.py
git commit -m "feat(research): add deep research Celery task with event emission"
```

---

## Task 5: Research Completion Notification Subscriber

**Files:**
- Create: `apps/api/src/prescient/events/subscribers/research_to_notification.py`
- Modify: `apps/api/src/prescient/shared/infrastructure/tasks/outbox_drain.py`
- Modify: `apps/api/src/prescient/notifications/domain/enums.py`
- Test: `apps/api/tests/unit/events/test_research_to_notification.py`

- [ ] **Step 1: Write test for subscriber**

Create `apps/api/tests/unit/events/__init__.py` (empty) and `apps/api/tests/unit/events/test_research_to_notification.py`:

```python
"""Unit tests for research_to_notification event subscriber."""

from __future__ import annotations

from unittest.mock import AsyncMock, patch

import pytest

from prescient.events.subscribers.research_to_notification import (
    _handle_research_completed,
)


class TestResearchToNotification:
    @pytest.mark.asyncio
    async def test_publishes_notification_on_research_complete(self) -> None:
        payload = {
            "task_id": "task-001",
            "subject_type": "company",
            "subject_name": "Acme Corp",
            "status": "completed",
            "artifact_id": "artifact-001",
            "tenant_id": "org-001",
            "created_by": "user-001",
        }

        mock_session = AsyncMock()
        mock_session.__aenter__ = AsyncMock(return_value=mock_session)
        mock_session.__aexit__ = AsyncMock(return_value=False)

        with patch(
            "prescient.events.subscribers.research_to_notification.SessionLocal",
            return_value=mock_session,
        ):
            await _handle_research_completed(payload)

        mock_session.add.assert_called_once()
        mock_session.commit.assert_called_once()

        # Verify the notification row was created with correct fields
        row = mock_session.add.call_args[0][0]
        assert row.title == "Research complete: Acme Corp"
        assert row.category == "research"
        assert row.recipient_id == "user-001"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/events/test_research_to_notification.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Add RESEARCH to NotificationCategory**

In `apps/api/src/prescient/notifications/domain/enums.py`:

```python
class NotificationCategory(StrEnum):
    STRATEGY = "strategy"
    ACTION_ITEM = "action_item"
    TRIAGE = "triage"
    MONITORING = "monitoring"
    RESEARCH = "research"
    SYSTEM = "system"
```

- [ ] **Step 4: Implement the subscriber**

Create `apps/api/src/prescient/events/subscribers/research_to_notification.py`:

```python
"""Event subscriber: research.task.completed -> publish notification."""

from __future__ import annotations

import asyncio
import logging

from celery import shared_task

from prescient.notifications.domain.enums import NotificationCategory, NotificationPriority

logger = logging.getLogger(__name__)


async def _handle_research_completed(payload: dict) -> None:
    from uuid import uuid4

    from prescient.db import SessionLocal
    from prescient.notifications.infrastructure.tables.notification import NotificationTable

    subject_name = payload.get("subject_name", "Unknown")
    created_by = payload.get("created_by", "")
    organization_id = payload.get("tenant_id", "")
    task_id = payload.get("task_id", "")
    artifact_id = payload.get("artifact_id")

    action_url = f"/research?task={task_id}"
    if artifact_id:
        action_url = f"/research?task={task_id}&artifact={artifact_id}"

    async with SessionLocal() as session:
        session.add(
            NotificationTable(
                id=uuid4(),
                recipient_id=created_by,
                organization_id=organization_id,
                category=NotificationCategory.RESEARCH.value,
                priority=NotificationPriority.NORMAL.value,
                title=f"Research complete: {subject_name}",
                body=f"Deep research on {subject_name} has finished. View the full profile.",
                source_type="research_task",
                source_id=uuid4(),
                action_url=action_url,
                read=False,
                dismissed=False,
                read_at=None,
            )
        )
        await session.commit()
    logger.info("Published notification for research task %s", task_id)


@shared_task(
    name="prescient.events.subscribers.research_to_notification.notify_research_complete",
    max_retries=3,
    default_retry_delay=10,
)
def notify_research_complete(payload: dict) -> dict:
    asyncio.run(_handle_research_completed(payload))
    return {"task_id": payload.get("task_id"), "status": "notified"}
```

- [ ] **Step 5: Register subscriber in outbox drain**

In `apps/api/src/prescient/shared/infrastructure/tasks/outbox_drain.py`, update `_get_subscriber_registry()`:

```python
def _get_subscriber_registry() -> dict[str, list]:
    from prescient.events.subscribers.anomaly_to_triage import create_triage_from_anomaly
    from prescient.events.subscribers.research_to_notification import notify_research_complete
    from prescient.events.subscribers.target_immediate_check import run_immediate_target_check

    return {
        "kpi.anomaly_detected": [create_triage_from_anomaly],
        "monitoring.target.created": [run_immediate_target_check],
        "research.task.completed": [notify_research_complete],
    }
```

- [ ] **Step 6: Run tests**

Run: `cd apps/api && uv run pytest tests/unit/events/test_research_to_notification.py -v`
Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/events/subscribers/research_to_notification.py
git add apps/api/src/prescient/shared/infrastructure/tasks/outbox_drain.py
git add apps/api/src/prescient/notifications/domain/enums.py
git add apps/api/tests/unit/events/
git commit -m "feat(research): add research completion notification subscriber"
```

---

## Task 6: UpgradeResearch Use Case

**Files:**
- Create: `apps/api/src/prescient/research/application/use_cases/upgrade_research.py`
- Test: `apps/api/tests/unit/research/test_upgrade_research.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/research/test_upgrade_research.py`:

```python
"""Unit tests for UpgradeResearch use case."""

from __future__ import annotations

from unittest.mock import MagicMock, patch

import pytest

from prescient.research.application.use_cases.upgrade_research import (
    UpgradeResearch,
    UpgradeResearchCommand,
    UpgradeResearchResult,
)
from prescient.research.domain.entities.research_subject import CompanySubject
from prescient.research.domain.entities.research_task import (
    ResearchMode,
    ResearchTask,
    ResearchTrigger,
    TaskStatus,
)


class TestUpgradeResearch:
    def test_dispatches_deep_research_task(self) -> None:
        quick_task = ResearchTask.create(
            subject=CompanySubject(name="Acme Corp", website="https://acme.com"),
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.COPILOT,
            tenant_id="org-001",
            created_by="user-001",
        ).mark_completed(synthesized_result={"summary": "Quick results"})

        with patch(
            "prescient.research.infrastructure.tasks.deep_research.run_deep_research"
        ) as mock_task:
            mock_task.delay = MagicMock(return_value=MagicMock(id="celery-task-id"))

            use_case = UpgradeResearch()
            result = use_case.execute(
                UpgradeResearchCommand(quick_task=quick_task)
            )

        assert isinstance(result, UpgradeResearchResult)
        assert "deeper" in result.message.lower() or "deep" in result.message.lower()
        mock_task.delay.assert_called_once()
        call_kwargs = mock_task.delay.call_args
        assert call_kwargs[1]["mode"] == "deep"
        assert call_kwargs[1]["subject_type"] == "company"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/research/test_upgrade_research.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement UpgradeResearch**

Create `apps/api/src/prescient/research/application/use_cases/upgrade_research.py`:

```python
"""UpgradeResearch use case — dispatches a deep research task from a completed quick task."""

from __future__ import annotations

from dataclasses import dataclass

from prescient.research.domain.entities.research_task import ResearchTask


@dataclass(frozen=True)
class UpgradeResearchCommand:
    quick_task: ResearchTask


@dataclass(frozen=True)
class UpgradeResearchResult:
    message: str


class UpgradeResearch:
    """Dispatches a deep async research task based on a completed quick task."""

    def execute(self, command: UpgradeResearchCommand) -> UpgradeResearchResult:
        from prescient.research.infrastructure.tasks.deep_research import run_deep_research

        task = command.quick_task
        subject = task.subject

        run_deep_research.delay(
            subject_type=subject.subject_type.value,
            subject_data=subject.model_dump(),
            mode="deep",
            triggered_by=task.triggered_by.value,
            tenant_id=task.tenant_id,
            created_by=task.created_by,
            enabled_sources=None,
        )

        subject_name = getattr(subject, "name", None) or getattr(subject, "seed_company_name", "")
        return UpgradeResearchResult(
            message=f"I've kicked off a deeper research pass on {subject_name}. "
            "You'll get a notification when it's ready."
        )
```

- [ ] **Step 4: Run tests**

Run: `cd apps/api && uv run pytest tests/unit/research/test_upgrade_research.py -v`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/research/application/use_cases/upgrade_research.py
git add apps/api/tests/unit/research/test_upgrade_research.py
git commit -m "feat(research): add UpgradeResearch use case for quick-to-deep upgrade"
```

---

## Task 7: research_market Copilot Tool

**Files:**
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/research_market.py`
- Test: `apps/api/tests/unit/intelligence/test_research_market_tool.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/intelligence/test_research_market_tool.py`:

```python
"""Unit tests for research_market copilot tool."""

from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.intelligence.infrastructure.tools.research_market import (
    ResearchMarketArguments,
    ResearchMarketTool,
)
from prescient.research.domain.entities.research_task import TaskStatus


class TestResearchMarketTool:
    def test_name_and_description(self) -> None:
        tool = ResearchMarketTool(pipeline_factory=MagicMock())
        assert tool.name == "research_market"
        assert "competitive" in tool.description.lower() or "market" in tool.description.lower()

    @pytest.mark.asyncio
    async def test_execute_returns_market_analysis(self) -> None:
        mock_pipeline_factory = MagicMock()
        mock_run_research = AsyncMock()
        mock_run_research.execute.return_value = MagicMock(
            task_id="task-001",
            status=TaskStatus.COMPLETED,
            artifact_id="artifact-001",
            summary="Manufacturing market dominated by 3 players.",
        )
        mock_pipeline_factory.return_value = mock_run_research

        tool = ResearchMarketTool(pipeline_factory=mock_pipeline_factory)

        context = MagicMock()
        context.tenant_id = uuid4()
        context.user_id = "user-001"

        args = ResearchMarketArguments(
            seed_company_name="Acme Corp",
            industry="Manufacturing",
        )

        result = await tool.execute(context, args)

        assert result.status.value == "ok"
        assert len(result.records) > 0
        assert result.records[0]["seed_company_name"] == "Acme Corp"

    @pytest.mark.asyncio
    async def test_execute_with_minimal_args(self) -> None:
        mock_pipeline_factory = MagicMock()
        mock_run_research = AsyncMock()
        mock_run_research.execute.return_value = MagicMock(
            task_id="task-002",
            status=TaskStatus.COMPLETED,
            artifact_id=None,
            summary=None,
        )
        mock_pipeline_factory.return_value = mock_run_research

        tool = ResearchMarketTool(pipeline_factory=mock_pipeline_factory)

        context = MagicMock()
        context.tenant_id = uuid4()
        context.user_id = "user-001"

        args = ResearchMarketArguments(seed_company_name="Unknown Corp")

        result = await tool.execute(context, args)

        assert result.status.value == "ok"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_research_market_tool.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement research_market tool**

Create `apps/api/src/prescient/intelligence/infrastructure/tools/research_market.py`:

```python
"""research_market copilot tool — analyzes the competitive landscape around a company."""

from __future__ import annotations

from pydantic import BaseModel, ConfigDict, Field

from prescient.intelligence.domain.entities.tool_execution import ToolExecution
from prescient.intelligence.infrastructure.tools.base import ToolContext


class ResearchMarketArguments(BaseModel):
    model_config = ConfigDict(extra="forbid")

    seed_company_name: str = Field(
        min_length=1, max_length=256, description="Anchor company for market discovery"
    )
    industry: str | None = Field(default=None, description="Industry sector")
    sources: list[str] | None = Field(default=None, description="Specific sources to use")


class ResearchMarketTool:
    name = "research_market"
    description = (
        "Analyze the competitive landscape around a company. Discovers competitors, "
        "market trends, and competitive dynamics. Use when the user asks about a market, "
        "competitive landscape, or wants to understand who the key players are in an industry."
    )
    arguments_model = ResearchMarketArguments

    def __init__(self, pipeline_factory: object) -> None:
        self._pipeline_factory = pipeline_factory

    async def execute(
        self, context: ToolContext, arguments: ResearchMarketArguments
    ) -> ToolExecution:
        from prescient.research.application.use_cases.run_research import (
            RunResearchCommand,
        )

        run_research = self._pipeline_factory()  # type: ignore[operator]

        result = await run_research.execute(
            RunResearchCommand(
                subject_type="market",
                subject_data={
                    "seed_company_name": arguments.seed_company_name,
                    "industry": arguments.industry,
                },
                mode="quick",
                triggered_by="copilot",
                tenant_id=str(context.tenant_id),
                created_by=str(context.user_id),
                enabled_sources=arguments.sources,
            )
        )

        records = [
            {
                "seed_company_name": arguments.seed_company_name,
                "task_id": result.task_id,
                "status": result.status.value,
                "artifact_id": result.artifact_id,
                "summary": result.summary or "Market analysis completed.",
            }
        ]

        return ToolExecution.ok(
            self.name,
            summary=f"Analyzed market around {arguments.seed_company_name}: {result.summary or 'analysis generated'}",
            records=records,
        )
```

- [ ] **Step 4: Run tests**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_research_market_tool.py -v`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/tools/research_market.py
git add apps/api/tests/unit/intelligence/test_research_market_tool.py
git commit -m "feat(intelligence): add research_market copilot tool"
```

---

## Task 8: research_person Copilot Tool

**Files:**
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/research_person.py`
- Test: `apps/api/tests/unit/intelligence/test_research_person_tool.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/intelligence/test_research_person_tool.py`:

```python
"""Unit tests for research_person copilot tool."""

from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.intelligence.infrastructure.tools.research_person import (
    ResearchPersonArguments,
    ResearchPersonTool,
)
from prescient.research.domain.entities.research_task import TaskStatus


class TestResearchPersonTool:
    def test_name_and_description(self) -> None:
        tool = ResearchPersonTool(pipeline_factory=MagicMock())
        assert tool.name == "research_person"
        assert "person" in tool.description.lower()

    @pytest.mark.asyncio
    async def test_execute_returns_profile(self) -> None:
        mock_pipeline_factory = MagicMock()
        mock_run_research = AsyncMock()
        mock_run_research.execute.return_value = MagicMock(
            task_id="task-001",
            status=TaskStatus.COMPLETED,
            artifact_id="artifact-001",
            summary="Jane Doe is CEO of Acme Corp.",
        )
        mock_pipeline_factory.return_value = mock_run_research

        tool = ResearchPersonTool(pipeline_factory=mock_pipeline_factory)

        context = MagicMock()
        context.tenant_id = uuid4()
        context.user_id = "user-001"

        args = ResearchPersonArguments(
            name="Jane Doe",
            company_name="Acme Corp",
            role="CEO",
        )

        result = await tool.execute(context, args)

        assert result.status.value == "ok"
        assert result.records[0]["name"] == "Jane Doe"

    @pytest.mark.asyncio
    async def test_execute_with_minimal_args(self) -> None:
        mock_pipeline_factory = MagicMock()
        mock_run_research = AsyncMock()
        mock_run_research.execute.return_value = MagicMock(
            task_id="task-002",
            status=TaskStatus.COMPLETED,
            artifact_id=None,
            summary=None,
        )
        mock_pipeline_factory.return_value = mock_run_research

        tool = ResearchPersonTool(pipeline_factory=mock_pipeline_factory)

        context = MagicMock()
        context.tenant_id = uuid4()
        context.user_id = "user-001"

        args = ResearchPersonArguments(name="John Smith")

        result = await tool.execute(context, args)

        assert result.status.value == "ok"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_research_person_tool.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement research_person tool**

Create `apps/api/src/prescient/intelligence/infrastructure/tools/research_person.py`:

```python
"""research_person copilot tool — researches a person's professional background."""

from __future__ import annotations

from pydantic import BaseModel, ConfigDict, Field

from prescient.intelligence.domain.entities.tool_execution import ToolExecution
from prescient.intelligence.infrastructure.tools.base import ToolContext


class ResearchPersonArguments(BaseModel):
    model_config = ConfigDict(extra="forbid")

    name: str = Field(min_length=1, max_length=256, description="Person's full name")
    company_name: str | None = Field(default=None, description="Company they work at")
    role: str | None = Field(default=None, description="Their role or title")
    email: str | None = Field(default=None, description="Email address if known")
    sources: list[str] | None = Field(default=None, description="Specific sources to use")


class ResearchPersonTool:
    name = "research_person"
    description = (
        "Research a person's professional background. Finds career history, expertise, "
        "and public presence. Use when the user asks about a specific person, wants "
        "background on an executive, or needs to prepare for a meeting with someone."
    )
    arguments_model = ResearchPersonArguments

    def __init__(self, pipeline_factory: object) -> None:
        self._pipeline_factory = pipeline_factory

    async def execute(
        self, context: ToolContext, arguments: ResearchPersonArguments
    ) -> ToolExecution:
        from prescient.research.application.use_cases.run_research import (
            RunResearchCommand,
        )

        run_research = self._pipeline_factory()  # type: ignore[operator]

        result = await run_research.execute(
            RunResearchCommand(
                subject_type="person",
                subject_data={
                    "name": arguments.name,
                    "company_name": arguments.company_name,
                    "role": arguments.role,
                    "email": arguments.email,
                },
                mode="quick",
                triggered_by="copilot",
                tenant_id=str(context.tenant_id),
                created_by=str(context.user_id),
                enabled_sources=arguments.sources,
            )
        )

        records = [
            {
                "name": arguments.name,
                "task_id": result.task_id,
                "status": result.status.value,
                "artifact_id": result.artifact_id,
                "summary": result.summary or "Person profile generated.",
            }
        ]

        return ToolExecution.ok(
            self.name,
            summary=f"Researched {arguments.name}: {result.summary or 'profile generated'}",
            records=records,
        )
```

- [ ] **Step 4: Run tests**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_research_person_tool.py -v`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/tools/research_person.py
git add apps/api/tests/unit/intelligence/test_research_person_tool.py
git commit -m "feat(intelligence): add research_person copilot tool"
```

---

## Task 9: Register New Tools

**Files:**
- Modify: `apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py`

- [ ] **Step 1: Add tool imports and registration**

In `apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py`, add imports:

```python
from prescient.intelligence.infrastructure.tools.research_market import (
    ResearchMarketTool,
)
from prescient.intelligence.infrastructure.tools.research_person import (
    ResearchPersonTool,
)
```

Add factory function (same pattern as `_build_research_company_tool`):

```python
def _build_research_market_tool() -> ResearchMarketTool:
    """Build ResearchMarketTool with its pipeline factory wired up."""

    def _factory() -> object:
        from prescient.config import get_settings
        from prescient.intelligence.infrastructure.llm import AnthropicAdapter
        from prescient.research.application.pipeline import ResearchPipeline
        from prescient.research.application.source_registry import SourceRegistry
        from prescient.research.application.use_cases.extract_and_synthesize import (
            ExtractAndSynthesize,
        )
        from prescient.research.application.use_cases.run_research import RunResearch
        from prescient.research.infrastructure.sources.dns_lookup_source import DnsLookupSource
        from prescient.research.infrastructure.sources.news_source import NewsAggregatorSource
        from prescient.research.infrastructure.sources.search_engine_source import SearchEngineSource
        from prescient.research.infrastructure.sources.website_scraper import WebsiteScraperSource

        settings = get_settings()
        llm = AnthropicAdapter(api_key=settings.anthropic_api_key, default_model=settings.anthropic_model)
        registry = SourceRegistry()
        registry.register(WebsiteScraperSource())
        registry.register(SearchEngineSource())
        registry.register(NewsAggregatorSource())
        registry.register(DnsLookupSource())
        synthesizer = ExtractAndSynthesize(llm=llm)
        pipeline = ResearchPipeline(source_registry=registry, synthesizer=synthesizer, artifact_repo=None)
        return RunResearch(pipeline=pipeline, session=None)

    return ResearchMarketTool(pipeline_factory=_factory)


def _build_research_person_tool() -> ResearchPersonTool:
    """Build ResearchPersonTool with its pipeline factory wired up."""

    def _factory() -> object:
        from prescient.config import get_settings
        from prescient.intelligence.infrastructure.llm import AnthropicAdapter
        from prescient.research.application.pipeline import ResearchPipeline
        from prescient.research.application.source_registry import SourceRegistry
        from prescient.research.application.use_cases.extract_and_synthesize import (
            ExtractAndSynthesize,
        )
        from prescient.research.application.use_cases.run_research import RunResearch
        from prescient.research.infrastructure.sources.dns_lookup_source import DnsLookupSource
        from prescient.research.infrastructure.sources.news_source import NewsAggregatorSource
        from prescient.research.infrastructure.sources.search_engine_source import SearchEngineSource
        from prescient.research.infrastructure.sources.website_scraper import WebsiteScraperSource

        settings = get_settings()
        llm = AnthropicAdapter(api_key=settings.anthropic_api_key, default_model=settings.anthropic_model)
        registry = SourceRegistry()
        registry.register(WebsiteScraperSource())
        registry.register(SearchEngineSource())
        registry.register(NewsAggregatorSource())
        registry.register(DnsLookupSource())
        synthesizer = ExtractAndSynthesize(llm=llm)
        pipeline = ResearchPipeline(source_registry=registry, synthesizer=synthesizer, artifact_repo=None)
        return RunResearch(pipeline=pipeline, session=None)

    return ResearchPersonTool(pipeline_factory=_factory)
```

Add to `build_tool_registry()`:

```python
    registry.register(_build_research_market_tool())
    registry.register(_build_research_person_tool())
```

Add to `__all__`:

```python
    "ResearchMarketTool",
    "ResearchPersonTool",
```

- [ ] **Step 2: Run all intelligence tests**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/ -v`
Expected: All tests pass.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py
git commit -m "feat(intelligence): register research_market and research_person tools"
```

---

## Task 10: UserContext Data Model Updates

**Files:**
- Modify: `apps/api/src/prescient/onboarding/domain/entities/user_context.py`
- Modify: `apps/api/src/prescient/onboarding/infrastructure/tables/user_context.py`
- Modify: `apps/api/src/prescient/onboarding/application/use_cases/run_interview.py`
- Test: `apps/api/tests/unit/onboarding/test_user_context_v2.py`

- [ ] **Step 1: Write tests for updated UserContext**

Create `apps/api/tests/unit/onboarding/test_user_context_v2.py`:

```python
"""Unit tests for UserContext v2 — watched_companies + platform_goals."""

from __future__ import annotations

import pytest

from prescient.onboarding.domain.entities.user_context import UserContext


class TestUserContextWatchedCompanies:
    def test_watched_companies_default_empty(self) -> None:
        ctx = UserContext.create(user_id="user-001")
        assert ctx.watched_companies == []

    def test_watched_companies_with_relationships(self) -> None:
        ctx = UserContext.create(
            user_id="user-001",
            watched_companies=[
                {"name": "Rival Corp", "relationship": "competitor", "context": "Main competitor"},
                {"name": "Partner Inc", "relationship": "partner", "context": "Key supplier"},
            ],
        )
        assert len(ctx.watched_companies) == 2
        assert ctx.watched_companies[0]["relationship"] == "competitor"
        assert ctx.watched_companies[1]["relationship"] == "partner"


class TestUserContextPlatformGoals:
    def test_platform_goals_default_empty(self) -> None:
        ctx = UserContext.create(user_id="user-001")
        assert ctx.platform_goals == ()

    def test_platform_goals_with_values(self) -> None:
        ctx = UserContext.create(
            user_id="user-001",
            platform_goals=("track_competitor_moves", "prepare_board_meetings"),
        )
        assert len(ctx.platform_goals) == 2
        assert "track_competitor_moves" in ctx.platform_goals

    def test_update_platform_goals(self) -> None:
        ctx = UserContext.create(user_id="user-001")
        ctx = ctx.update(platform_goals=("monitor_market_trends",))
        assert ctx.platform_goals == ("monitor_market_trends",)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/onboarding/test_user_context_v2.py -v`
Expected: AttributeError for `watched_companies` and `platform_goals`

- [ ] **Step 3: Update UserContext entity**

In `apps/api/src/prescient/onboarding/domain/entities/user_context.py`, rename `competitors_mentioned` to `watched_companies` and add `platform_goals`:

```python
"""UserContext domain entity — captures interview results."""

from __future__ import annotations

import uuid
from datetime import UTC, datetime

from pydantic import BaseModel, ConfigDict, Field


class UserContext(BaseModel):
    """Captures what the onboarding interview learns about a user."""

    model_config = ConfigDict(frozen=True)

    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    user_id: str
    company_id: str | None = None
    role_description: str | None = None
    primary_focus: str | None = None
    pain_points: tuple[str, ...] = ()
    desired_outcomes: tuple[str, ...] = ()
    watched_companies: list[dict] = Field(default_factory=list)
    platform_goals: tuple[str, ...] = ()
    raw_interview_summary: str | None = None
    created_at: datetime = Field(default_factory=lambda: datetime.now(UTC))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(UTC))

    @classmethod
    def create(
        cls,
        *,
        user_id: str,
        company_id: str | None = None,
        role_description: str | None = None,
        primary_focus: str | None = None,
        pain_points: tuple[str, ...] = (),
        desired_outcomes: tuple[str, ...] = (),
        watched_companies: list[dict] | None = None,
        platform_goals: tuple[str, ...] = (),
        raw_interview_summary: str | None = None,
    ) -> UserContext:
        return cls(
            user_id=user_id,
            company_id=company_id,
            role_description=role_description,
            primary_focus=primary_focus,
            pain_points=pain_points,
            desired_outcomes=desired_outcomes,
            watched_companies=watched_companies or [],
            platform_goals=platform_goals,
            raw_interview_summary=raw_interview_summary,
        )

    def update(self, **kwargs: object) -> UserContext:
        return self.model_copy(
            update={**kwargs, "updated_at": datetime.now(UTC)},
        )
```

- [ ] **Step 4: Update UserContextRow table**

In `apps/api/src/prescient/onboarding/infrastructure/tables/user_context.py`, rename the column and add `platform_goals`:

```python
"""SQLAlchemy model for user_contexts table."""

from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class UserContextRow(Base):
    __tablename__ = "user_contexts"
    __table_args__ = {"schema": "onboarding"}  # noqa: RUF012

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    user_id: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    company_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    role_description: Mapped[str | None] = mapped_column(Text(), nullable=True)
    primary_focus: Mapped[str | None] = mapped_column(Text(), nullable=True)
    pain_points: Mapped[list] = mapped_column(JSONB, nullable=False, server_default="[]")
    desired_outcomes: Mapped[list] = mapped_column(JSONB, nullable=False, server_default="[]")
    watched_companies: Mapped[list] = mapped_column(JSONB, nullable=False, server_default="[]")
    platform_goals: Mapped[list] = mapped_column(JSONB, nullable=False, server_default="[]")
    raw_interview_summary: Mapped[str | None] = mapped_column(Text(), nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        server_default=func.now(),
        onupdate=func.now(),
    )
```

- [ ] **Step 5: Update interview prompts**

In `apps/api/src/prescient/onboarding/application/use_cases/run_interview.py`, update the system prompt and extraction prompt:

Replace `"5. Find out which competitors they watch most closely"` with:

```python
"5. Find out which companies they watch most closely (competitors, partners, customers)"
```

Update `_EXTRACTION_PROMPT`:

```python
_EXTRACTION_PROMPT = """\
Analyze the following onboarding interview conversation and extract structured data.

Output ONLY valid JSON with this structure:
{
  "role_description": "What the user actually does day-to-day",
  "primary_focus": "Their current top priority",
  "pain_points": ["pain point 1", "pain point 2"],
  "desired_outcomes": ["what they want solved"],
  "watched_companies": [{"name": "Company Name", "relationship": "competitor|partner|customer|prospect|other", "context": "why they mentioned them"}],
  "platform_goals": ["track_competitor_moves", "prepare_board_meetings", "monitor_market_trends", "source_deals", "manage_portfolio_risk", "stay_informed_industry"],
  "raw_interview_summary": "2-3 sentence summary of the conversation"
}

For platform_goals, pick from the standard list above based on what the user described wanting.
Set any field to null if the user didn't mention it.
"""
```

Update `ExtractedUserContext` to match:

```python
@dataclass(frozen=True)
class ExtractedUserContext:
    role_description: str | None = None
    primary_focus: str | None = None
    pain_points: list[str] = field(default_factory=list)
    desired_outcomes: list[str] = field(default_factory=list)
    watched_companies: list[dict] = field(default_factory=list)
    platform_goals: list[str] = field(default_factory=list)
    raw_interview_summary: str | None = None
```

- [ ] **Step 6: Update existing UserContext tests**

In `apps/api/tests/unit/onboarding/test_user_context_entity.py`, update references from `competitors_mentioned` to `watched_companies`:

Replace `competitors_mentioned=[{"name": "Rival Corp", "context": "Main competitor"}]` with `watched_companies=[{"name": "Rival Corp", "relationship": "competitor", "context": "Main competitor"}]`.

Replace `ctx.competitors_mentioned[0]["name"]` with `ctx.watched_companies[0]["name"]`.

- [ ] **Step 7: Run all onboarding tests**

Run: `cd apps/api && uv run pytest tests/unit/onboarding/ -v`
Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/onboarding/domain/entities/user_context.py
git add apps/api/src/prescient/onboarding/infrastructure/tables/user_context.py
git add apps/api/src/prescient/onboarding/application/use_cases/run_interview.py
git add apps/api/tests/unit/onboarding/
git commit -m "feat(onboarding): rename competitors to watched_companies, add platform_goals"
```

---

## Task 11: Database Migration

**Files:**
- Create: `apps/api/alembic/versions/20260414_research_phase2.py`

- [ ] **Step 1: Create migration**

Create `apps/api/alembic/versions/20260414_research_phase2.py`:

```python
"""research phase 2 — watched_companies rename + platform_goals

Revision ID: 20260414_research_p2
Revises: 20260414_ke_foundation
Create Date: 2026-04-14
"""

from __future__ import annotations

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB

revision = "20260414_research_p2"
down_revision = "20260414_ke_foundation"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Rename competitors_mentioned -> watched_companies
    op.alter_column(
        "user_contexts",
        "competitors_mentioned",
        new_column_name="watched_companies",
        schema="onboarding",
    )

    # Add platform_goals column
    op.add_column(
        "user_contexts",
        sa.Column("platform_goals", JSONB, nullable=False, server_default="[]"),
        schema="onboarding",
    )


def downgrade() -> None:
    op.drop_column("user_contexts", "platform_goals", schema="onboarding")
    op.alter_column(
        "user_contexts",
        "watched_companies",
        new_column_name="competitors_mentioned",
        schema="onboarding",
    )
```

- [ ] **Step 2: Commit**

```bash
git add apps/api/alembic/versions/20260414_research_phase2.py
git commit -m "feat(onboarding): add migration for watched_companies rename and platform_goals"
```

---

## Task 12: Run All Tests and Final Verification

- [ ] **Step 1: Run all unit tests**

Run: `cd apps/api && uv run pytest tests/unit/ -v`
Expected: All tests pass (existing + new).

- [ ] **Step 2: Run linting**

Run: `cd apps/api && uv run ruff check src/ tests/`
Expected: No errors.

- [ ] **Step 3: Fix any issues found**

Address any test failures or lint errors.

- [ ] **Step 4: Final commit if fixes were needed**

```bash
git add -A
git commit -m "fix: resolve lint/test issues from research phase 2 implementation"
```

---

## Summary

| Layer | Tasks | What It Delivers |
|-------|-------|-----------------|
| **Infrastructure** | Tasks 1-5 | Source tiers, enabled_sources filtering, deep research Celery task, research completion notifications |
| **Tools** | Tasks 6-9 | UpgradeResearch use case, research_market tool, research_person tool, tool registration |
| **Data Model** | Tasks 10-11 | watched_companies rename, platform_goals field, updated interview prompts, migration |
| **Verification** | Task 12 | All tests pass, lint clean |

**Total tasks:** 12
**Estimated commits:** 12-14

**Not included in this plan (frontend — separate plan):**
- Onboarding interview page (`/onboarding/interview`)
- Standalone `/research` page with split-view
- Research controls UI (depth preset + source toggles)
- Research result cards with live updates
- SSE `research_complete` event handling in frontend
- Copilot nudge for completed research
