# Research Engine & Smart Onboarding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a multi-source research pipeline engine, smart conversational onboarding, and chat-based market research tools.

**Architecture:** Three phases sharing a core pipeline. Phase 1 builds the research pipeline engine as a new `research` bounded context with staged execution (discover → fetch → extract → synthesize → persist), source plugins, and a Postgres-backed task queue. Phase 2 replaces the current onboarding wizard with quick research + chat interview + deep async research. Phase 3 adds research tools to the copilot for on-demand market research with promote-to-watched/artifact capabilities.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Postgres 16 (JSONB), Anthropic Claude, httpx (web fetching), BeautifulSoup4 (HTML parsing)

**Spec:** `docs/superpowers/specs/2026-04-13-research-engine-and-smart-onboarding-design.md`

---

## File Structure

### New: `apps/api/src/prescient/research/` (Research bounded context)

```
research/
├── __init__.py
├── domain/
│   ├── __init__.py
│   ├── entities/
│   │   ├── __init__.py
│   │   ├── research_task.py          # ResearchTask entity + enums
│   │   ├── research_subject.py       # CompanySubject, PersonSubject, MarketSubject
│   │   └── finding.py                # Finding, RawFindings value objects
│   ├── ports.py                      # ResearchSource protocol, ResearchTaskRepo protocol
│   └── errors.py                     # ResearchError, SourceFetchError, etc.
├── application/
│   ├── __init__.py
│   ├── pipeline.py                   # ResearchPipeline orchestrator (staged execution)
│   ├── use_cases/
│   │   ├── __init__.py
│   │   ├── run_research.py           # RunResearch use case (entry point)
│   │   └── extract_and_synthesize.py # LLM extraction + synthesis logic
│   └── source_registry.py            # SourceRegistry (discovers + manages source plugins)
├── infrastructure/
│   ├── __init__.py
│   ├── tables/
│   │   ├── __init__.py
│   │   └── research_task.py          # ResearchTaskRow SQLAlchemy model
│   ├── repositories/
│   │   ├── __init__.py
│   │   └── research_task_repository.py
│   └── sources/
│       ├── __init__.py
│       ├── website_scraper.py        # WebsiteScraperSource
│       ├── edgar_source.py           # EdgarResearchSource (wraps existing)
│       ├── news_source.py            # NewsAggregatorSource
│       ├── search_engine_source.py   # SearchEngineSource
│       └── dns_lookup_source.py      # DnsLookupSource
└── api/
    ├── __init__.py
    └── routes.py                     # Research API routes (status, results)
```

### New: `apps/api/src/prescient/onboarding/domain/entities/user_context.py`

UserContext entity for interview results.

### New: `apps/api/src/prescient/onboarding/infrastructure/tables/user_context.py`

UserContext SQLAlchemy table.

### New: `apps/api/src/prescient/onboarding/application/use_cases/run_interview.py`

Interview orchestration use case.

### Modified: Existing files

- `apps/api/src/prescient/shared/metadata.py` — Register research tables
- `apps/api/src/prescient/shared/types.py` — Add ResearchTaskId
- `apps/api/src/prescient/main.py` — Register research router
- `apps/api/src/prescient/artifacts/domain/entities/artifact.py` — Add PERSON_PROFILE artifact type
- `apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py` — Register new research tools
- `apps/api/src/prescient/onboarding/api/routes.py` — New onboarding endpoints

### New: Research copilot tools

- `apps/api/src/prescient/intelligence/infrastructure/tools/research_company.py`
- `apps/api/src/prescient/intelligence/infrastructure/tools/research_market.py`
- `apps/api/src/prescient/intelligence/infrastructure/tools/research_person.py`
- `apps/api/src/prescient/intelligence/infrastructure/tools/watch_company.py`
- `apps/api/src/prescient/intelligence/infrastructure/tools/save_artifact.py`

### New: Migration

- `apps/api/alembic/versions/20260413_research_engine.py`

### New: Tests

- `apps/api/tests/unit/research/test_research_task_entity.py`
- `apps/api/tests/unit/research/test_finding.py`
- `apps/api/tests/unit/research/test_source_registry.py`
- `apps/api/tests/unit/research/test_pipeline.py`
- `apps/api/tests/unit/research/test_website_scraper.py`
- `apps/api/tests/unit/research/test_run_research.py`
- `apps/api/tests/unit/research/test_extract_and_synthesize.py`
- `apps/api/tests/unit/onboarding/test_user_context_entity.py`
- `apps/api/tests/unit/intelligence/test_research_company_tool.py`
- `apps/api/tests/unit/intelligence/test_watch_company_tool.py`

---

## Phase 1: Research Pipeline Engine

### Task 1: Research Domain Entities

**Files:**
- Create: `apps/api/src/prescient/research/__init__.py`
- Create: `apps/api/src/prescient/research/domain/__init__.py`
- Create: `apps/api/src/prescient/research/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/research/domain/entities/research_subject.py`
- Create: `apps/api/src/prescient/research/domain/entities/finding.py`
- Create: `apps/api/src/prescient/research/domain/entities/research_task.py`
- Create: `apps/api/src/prescient/research/domain/errors.py`
- Modify: `apps/api/src/prescient/shared/types.py`
- Test: `apps/api/tests/unit/research/test_research_task_entity.py`
- Test: `apps/api/tests/unit/research/test_finding.py`

- [ ] **Step 1: Write tests for research subjects and findings**

Create `apps/api/tests/unit/research/__init__.py` (empty) and `apps/api/tests/unit/research/test_finding.py`:

```python
"""Unit tests for research domain value objects."""

from __future__ import annotations

import pytest

from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    MarketSubject,
    PersonSubject,
    SubjectType,
)


class TestCompanySubject:
    def test_create_with_required_fields(self) -> None:
        subject = CompanySubject(name="Acme Corp")
        assert subject.name == "Acme Corp"
        assert subject.subject_type == SubjectType.COMPANY
        assert subject.website is None
        assert subject.cik is None

    def test_create_with_all_fields(self) -> None:
        subject = CompanySubject(
            name="Acme Corp",
            website="https://acme.com",
            cik="0001234567",
            industry="Manufacturing",
        )
        assert subject.website == "https://acme.com"
        assert subject.cik == "0001234567"
        assert subject.industry == "Manufacturing"


class TestPersonSubject:
    def test_create_with_required_fields(self) -> None:
        subject = PersonSubject(name="Jane Doe")
        assert subject.name == "Jane Doe"
        assert subject.subject_type == SubjectType.PERSON
        assert subject.email is None

    def test_create_with_email_and_company(self) -> None:
        subject = PersonSubject(
            name="Jane Doe",
            email="jane@acme.com",
            company_name="Acme Corp",
            role="CEO",
        )
        assert subject.email == "jane@acme.com"
        assert subject.company_name == "Acme Corp"
        assert subject.role == "CEO"


class TestMarketSubject:
    def test_create_from_company(self) -> None:
        subject = MarketSubject(
            seed_company_name="Acme Corp",
            industry="Manufacturing",
        )
        assert subject.subject_type == SubjectType.MARKET
        assert subject.seed_company_name == "Acme Corp"


class TestFinding:
    def test_create_finding(self) -> None:
        finding = Finding(
            content="Acme Corp reported $10M revenue.",
            source_url="https://acme.com/about",
            category="financials",
        )
        assert finding.content == "Acme Corp reported $10M revenue."
        assert finding.source_url == "https://acme.com/about"
        assert finding.category == "financials"

    def test_finding_without_optional_fields(self) -> None:
        finding = Finding(content="Some info.")
        assert finding.source_url is None
        assert finding.category is None


class TestRawFindings:
    def test_create_empty(self) -> None:
        findings = RawFindings(source_name="test_source")
        assert findings.source_name == "test_source"
        assert findings.findings == ()
        assert findings.has_data is False

    def test_has_data_when_populated(self) -> None:
        findings = RawFindings(
            source_name="test_source",
            findings=(Finding(content="data"),),
        )
        assert findings.has_data is True

    def test_all_text_concatenates_findings(self) -> None:
        findings = RawFindings(
            source_name="test_source",
            findings=(
                Finding(content="First finding."),
                Finding(content="Second finding."),
            ),
        )
        text = findings.all_text()
        assert "First finding." in text
        assert "Second finding." in text
```

- [ ] **Step 2: Write tests for ResearchTask entity**

Create `apps/api/tests/unit/research/test_research_task_entity.py`:

```python
"""Unit tests for ResearchTask domain entity."""

from __future__ import annotations

import pytest

from prescient.research.domain.entities.research_task import (
    ResearchMode,
    ResearchTask,
    ResearchTrigger,
    SourceStatus,
    TaskStatus,
)
from prescient.research.domain.entities.research_subject import CompanySubject
from prescient.research.domain.entities.finding import RawFindings, Finding


class TestResearchTask:
    def test_create_pending_task(self) -> None:
        subject = CompanySubject(name="Acme Corp")
        task = ResearchTask.create(
            subject=subject,
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.ONBOARDING,
            tenant_id="org-001",
            created_by="user-001",
        )
        assert task.status == TaskStatus.PENDING
        assert task.mode == ResearchMode.QUICK
        assert task.triggered_by == ResearchTrigger.ONBOARDING
        assert task.source_results == {}

    def test_advance_to_discovering(self) -> None:
        subject = CompanySubject(name="Acme Corp")
        task = ResearchTask.create(
            subject=subject,
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.COPILOT,
            tenant_id="org-001",
            created_by="user-001",
        )
        task = task.advance_to(TaskStatus.DISCOVERING)
        assert task.status == TaskStatus.DISCOVERING

    def test_record_source_success(self) -> None:
        subject = CompanySubject(name="Acme Corp")
        task = ResearchTask.create(
            subject=subject,
            mode=ResearchMode.DEEP,
            triggered_by=ResearchTrigger.COPILOT,
            tenant_id="org-001",
            created_by="user-001",
        )
        findings = RawFindings(
            source_name="website_scraper",
            findings=(Finding(content="Acme makes widgets."),),
        )
        task = task.record_source_result(
            source_name="website_scraper",
            status=SourceStatus.SUCCEEDED,
            findings=findings,
        )
        assert task.source_results["website_scraper"]["status"] == "succeeded"
        assert len(task.source_results["website_scraper"]["findings"]) == 1

    def test_record_source_failure(self) -> None:
        subject = CompanySubject(name="Acme Corp")
        task = ResearchTask.create(
            subject=subject,
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.ONBOARDING,
            tenant_id="org-001",
            created_by="user-001",
        )
        task = task.record_source_result(
            source_name="website_scraper",
            status=SourceStatus.FAILED,
            error="Connection timeout",
        )
        assert task.source_results["website_scraper"]["status"] == "failed"
        assert task.source_results["website_scraper"]["error"] == "Connection timeout"

    def test_all_findings_aggregates_across_sources(self) -> None:
        subject = CompanySubject(name="Acme Corp")
        task = ResearchTask.create(
            subject=subject,
            mode=ResearchMode.DEEP,
            triggered_by=ResearchTrigger.COPILOT,
            tenant_id="org-001",
            created_by="user-001",
        )
        task = task.record_source_result(
            source_name="website_scraper",
            status=SourceStatus.SUCCEEDED,
            findings=RawFindings(
                source_name="website_scraper",
                findings=(Finding(content="Web data"),),
            ),
        )
        task = task.record_source_result(
            source_name="edgar",
            status=SourceStatus.SUCCEEDED,
            findings=RawFindings(
                source_name="edgar",
                findings=(Finding(content="Filing data"),),
            ),
        )
        all_findings = task.all_findings()
        assert len(all_findings) == 2

    def test_mark_completed_with_artifact(self) -> None:
        subject = CompanySubject(name="Acme Corp")
        task = ResearchTask.create(
            subject=subject,
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.ONBOARDING,
            tenant_id="org-001",
            created_by="user-001",
        )
        task = task.mark_completed(
            result_artifact_id="artifact-123",
            synthesized_result={"summary": "Acme is a widget company."},
        )
        assert task.status == TaskStatus.COMPLETED
        assert task.result_artifact_id == "artifact-123"

    def test_mark_failed(self) -> None:
        subject = CompanySubject(name="Acme Corp")
        task = ResearchTask.create(
            subject=subject,
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.ONBOARDING,
            tenant_id="org-001",
            created_by="user-001",
        )
        task = task.mark_failed()
        assert task.status == TaskStatus.FAILED
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/research/ -v`
Expected: ModuleNotFoundError for `prescient.research`

- [ ] **Step 4: Implement research subject value objects**

Create `apps/api/src/prescient/research/__init__.py` (empty).
Create `apps/api/src/prescient/research/domain/__init__.py` (empty).
Create `apps/api/src/prescient/research/domain/entities/__init__.py` (empty).

Create `apps/api/src/prescient/research/domain/entities/research_subject.py`:

```python
"""Research subject value objects — what we're researching."""

from __future__ import annotations

from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field


class SubjectType(StrEnum):
    COMPANY = "company"
    PERSON = "person"
    MARKET = "market"


class CompanySubject(BaseModel):
    model_config = ConfigDict(frozen=True)

    subject_type: SubjectType = SubjectType.COMPANY
    name: str = Field(min_length=1, max_length=256)
    website: str | None = None
    cik: str | None = None
    industry: str | None = None


class PersonSubject(BaseModel):
    model_config = ConfigDict(frozen=True)

    subject_type: SubjectType = SubjectType.PERSON
    name: str = Field(min_length=1, max_length=256)
    email: str | None = None
    company_name: str | None = None
    role: str | None = None


class MarketSubject(BaseModel):
    model_config = ConfigDict(frozen=True)

    subject_type: SubjectType = SubjectType.MARKET
    seed_company_name: str = Field(min_length=1, max_length=256)
    industry: str | None = None


ResearchSubject = CompanySubject | PersonSubject | MarketSubject
```

- [ ] **Step 5: Implement Finding and RawFindings value objects**

Create `apps/api/src/prescient/research/domain/entities/finding.py`:

```python
"""Finding value objects — raw data from research sources."""

from __future__ import annotations

from datetime import datetime, UTC

from pydantic import BaseModel, ConfigDict, Field


class Finding(BaseModel):
    """A single piece of research data from a source."""

    model_config = ConfigDict(frozen=True)

    content: str
    source_url: str | None = None
    category: str | None = None


class RawFindings(BaseModel):
    """Collection of findings from a single source."""

    model_config = ConfigDict(frozen=True)

    source_name: str
    findings: tuple[Finding, ...] = ()
    fetched_at: datetime = Field(default_factory=lambda: datetime.now(UTC))
    confidence: str = "medium"

    @property
    def has_data(self) -> bool:
        return len(self.findings) > 0

    def all_text(self) -> str:
        """Concatenate all finding content into a single text block."""
        return "\n\n".join(f.content for f in self.findings)
```

- [ ] **Step 6: Implement ResearchTask entity**

Create `apps/api/src/prescient/research/domain/entities/research_task.py`:

```python
"""ResearchTask domain entity — tracks a pipeline execution."""

from __future__ import annotations

import uuid
from datetime import UTC, datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field

from prescient.research.domain.entities.finding import RawFindings
from prescient.research.domain.entities.research_subject import ResearchSubject


class TaskStatus(StrEnum):
    PENDING = "pending"
    DISCOVERING = "discovering"
    FETCHING = "fetching"
    EXTRACTING = "extracting"
    SYNTHESIZING = "synthesizing"
    PERSISTING = "persisting"
    COMPLETED = "completed"
    FAILED = "failed"


class ResearchMode(StrEnum):
    QUICK = "quick"
    DEEP = "deep"


class ResearchTrigger(StrEnum):
    ONBOARDING = "onboarding"
    COPILOT = "copilot"
    SYSTEM = "system"


class SourceStatus(StrEnum):
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    TIMED_OUT = "timed_out"
    SKIPPED = "skipped"


class ResearchTask(BaseModel):
    """Immutable domain entity representing a research pipeline run."""

    model_config = ConfigDict(frozen=True)

    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    subject: ResearchSubject
    mode: ResearchMode
    status: TaskStatus
    triggered_by: ResearchTrigger
    tenant_id: str
    created_by: str
    source_results: dict[str, dict] = Field(default_factory=dict)
    extracted_data: dict | None = None
    synthesized_result: dict | None = None
    result_artifact_id: str | None = None
    created_at: datetime = Field(default_factory=lambda: datetime.now(UTC))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(UTC))

    @classmethod
    def create(
        cls,
        *,
        subject: ResearchSubject,
        mode: ResearchMode,
        triggered_by: ResearchTrigger,
        tenant_id: str,
        created_by: str,
    ) -> ResearchTask:
        return cls(
            subject=subject,
            mode=mode,
            status=TaskStatus.PENDING,
            triggered_by=triggered_by,
            tenant_id=tenant_id,
            created_by=created_by,
        )

    def advance_to(self, status: TaskStatus) -> ResearchTask:
        return self.model_copy(
            update={"status": status, "updated_at": datetime.now(UTC)},
        )

    def record_source_result(
        self,
        *,
        source_name: str,
        status: SourceStatus,
        findings: RawFindings | None = None,
        error: str | None = None,
    ) -> ResearchTask:
        entry: dict = {"status": status.value}
        if findings is not None:
            entry["findings"] = [f.model_dump() for f in findings.findings]
        if error is not None:
            entry["error"] = error
        new_results = {**self.source_results, source_name: entry}
        return self.model_copy(
            update={"source_results": new_results, "updated_at": datetime.now(UTC)},
        )

    def all_findings(self) -> list[RawFindings]:
        """Collect all successful source findings."""
        result = []
        for source_name, data in self.source_results.items():
            if data.get("status") == SourceStatus.SUCCEEDED.value and data.get("findings"):
                from prescient.research.domain.entities.finding import Finding

                findings_list = tuple(
                    Finding(**f) for f in data["findings"]
                )
                result.append(
                    RawFindings(source_name=source_name, findings=findings_list)
                )
        return result

    def mark_completed(
        self,
        *,
        result_artifact_id: str | None = None,
        synthesized_result: dict | None = None,
    ) -> ResearchTask:
        return self.model_copy(
            update={
                "status": TaskStatus.COMPLETED,
                "result_artifact_id": result_artifact_id,
                "synthesized_result": synthesized_result,
                "updated_at": datetime.now(UTC),
            },
        )

    def mark_failed(self) -> ResearchTask:
        return self.model_copy(
            update={"status": TaskStatus.FAILED, "updated_at": datetime.now(UTC)},
        )
```

- [ ] **Step 7: Implement domain errors**

Create `apps/api/src/prescient/research/domain/errors.py`:

```python
"""Research domain errors."""

from __future__ import annotations

from prescient.shared.errors import DomainError


class ResearchError(DomainError):
    """Base error for research domain."""

    pass


class SourceFetchError(ResearchError):
    """A research source failed to fetch data."""

    pass


class ResearchTaskNotFoundError(ResearchError):
    """Requested research task does not exist."""

    pass
```

- [ ] **Step 8: Add ResearchTaskId to shared types**

Add to `apps/api/src/prescient/shared/types.py`:

```python
ResearchTaskId = NewType("ResearchTaskId", UUID)
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/research/ -v`
Expected: All tests pass.

- [ ] **Step 10: Commit**

```bash
git add apps/api/src/prescient/research/domain/ apps/api/src/prescient/research/__init__.py apps/api/tests/unit/research/ apps/api/src/prescient/shared/types.py
git commit -m "feat(research): add research domain entities, subjects, and findings"
```

---

### Task 2: ResearchSource Protocol & Source Registry

**Files:**
- Create: `apps/api/src/prescient/research/domain/ports.py`
- Create: `apps/api/src/prescient/research/application/__init__.py`
- Create: `apps/api/src/prescient/research/application/source_registry.py`
- Test: `apps/api/tests/unit/research/test_source_registry.py`

- [ ] **Step 1: Write tests for source registry**

Create `apps/api/tests/unit/research/test_source_registry.py`:

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
from prescient.research.domain.entities.research_task import ResearchMode
from prescient.research.domain.ports import ResearchSource


class _FakeWebSource:
    """Fake source that handles companies only."""

    name = "website_scraper"
    timeout_seconds = 10
    quick_pass = True

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: CompanySubject) -> RawFindings:
        return RawFindings(
            source_name=self.name,
            findings=(Finding(content="Website data for " + subject.name),),
        )


class _FakeEdgarSource:
    """Fake source that handles companies, deep pass only."""

    name = "edgar"
    timeout_seconds = 30
    quick_pass = False

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: CompanySubject) -> RawFindings:
        return RawFindings(
            source_name=self.name,
            findings=(Finding(content="Filing data"),),
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

    def test_quick_pass_filters_deep_sources(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        registry.register(_FakeEdgarSource())

        sources = registry.sources_for(
            SubjectType.COMPANY, mode=ResearchMode.QUICK
        )
        assert len(sources) == 1
        assert sources[0].name == "website_scraper"

    def test_deep_pass_includes_all_sources(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        registry.register(_FakeEdgarSource())

        sources = registry.sources_for(
            SubjectType.COMPANY, mode=ResearchMode.DEEP
        )
        assert len(sources) == 2

    def test_duplicate_name_raises(self) -> None:
        registry = SourceRegistry()
        registry.register(_FakeWebSource())
        with pytest.raises(ValueError, match="already registered"):
            registry.register(_FakeWebSource())
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/research/test_source_registry.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement ResearchSource protocol**

Create `apps/api/src/prescient/research/domain/ports.py`:

```python
"""Research domain ports (protocols)."""

from __future__ import annotations

from typing import Any, Protocol, runtime_checkable

from prescient.research.domain.entities.finding import RawFindings
from prescient.research.domain.entities.research_subject import SubjectType


@runtime_checkable
class ResearchSource(Protocol):
    """Contract for research source plugins."""

    name: str
    timeout_seconds: int
    quick_pass: bool

    def can_handle(self, subject_type: SubjectType) -> bool: ...
    async def fetch(self, subject: Any) -> RawFindings: ...


class ResearchTaskRepositoryPort(Protocol):
    """Port for persisting research tasks."""

    async def save(self, task_data: dict) -> None: ...
    async def get(self, task_id: str) -> dict | None: ...
    async def update(self, task_id: str, updates: dict) -> None: ...
```

- [ ] **Step 4: Implement SourceRegistry**

Create `apps/api/src/prescient/research/application/__init__.py` (empty).

Create `apps/api/src/prescient/research/application/source_registry.py`:

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
    ) -> list[ResearchSource]:
        """Return sources that can handle the given subject type and mode."""
        result = []
        for source in self._sources.values():
            if not source.can_handle(subject_type):
                continue
            if mode == ResearchMode.QUICK and not source.quick_pass:
                continue
            result.append(source)
        return result
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/research/test_source_registry.py -v`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/research/domain/ports.py apps/api/src/prescient/research/application/
git add apps/api/tests/unit/research/test_source_registry.py
git commit -m "feat(research): add ResearchSource protocol and SourceRegistry"
```

---

### Task 3: Research Pipeline Orchestrator

**Files:**
- Create: `apps/api/src/prescient/research/application/pipeline.py`
- Create: `apps/api/src/prescient/research/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/research/application/use_cases/extract_and_synthesize.py`
- Test: `apps/api/tests/unit/research/test_pipeline.py`
- Test: `apps/api/tests/unit/research/test_extract_and_synthesize.py`

- [ ] **Step 1: Write tests for extract-and-synthesize**

Create `apps/api/tests/unit/research/test_extract_and_synthesize.py`:

```python
"""Unit tests for LLM extraction and synthesis."""

from __future__ import annotations

from unittest.mock import AsyncMock

import pytest

from prescient.research.application.use_cases.extract_and_synthesize import (
    ExtractAndSynthesize,
    ExtractionResult,
)
from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import CompanySubject


class TestExtractAndSynthesize:
    @pytest.mark.asyncio
    async def test_synthesizes_company_profile_from_findings(self) -> None:
        mock_llm = AsyncMock()
        mock_llm.send.return_value = {
            "role": "assistant",
            "content": [
                {
                    "type": "text",
                    "text": '{"blocks": [{"block_id": "overview", "heading": "Overview", "body": "Acme makes widgets.", "order": 0}], "summary": "Acme is a widget manufacturer."}',
                }
            ],
            "stop_reason": "end_turn",
            "usage": {"input_tokens": 100, "output_tokens": 50},
        }

        use_case = ExtractAndSynthesize(llm=mock_llm)
        subject = CompanySubject(name="Acme Corp", industry="Manufacturing")
        findings = [
            RawFindings(
                source_name="website_scraper",
                findings=(Finding(content="Acme Corp makes widgets for industry."),),
            ),
        ]

        result = await use_case.execute(subject=subject, findings=findings)

        assert isinstance(result, ExtractionResult)
        assert len(result.blocks) == 1
        assert result.blocks[0]["block_id"] == "overview"
        assert result.summary == "Acme is a widget manufacturer."
        mock_llm.send.assert_called_once()

    @pytest.mark.asyncio
    async def test_handles_empty_findings(self) -> None:
        mock_llm = AsyncMock()
        mock_llm.send.return_value = {
            "role": "assistant",
            "content": [
                {
                    "type": "text",
                    "text": '{"blocks": [{"block_id": "overview", "heading": "Overview", "body": "Limited information available.", "order": 0}], "summary": null}',
                }
            ],
            "stop_reason": "end_turn",
            "usage": {"input_tokens": 50, "output_tokens": 30},
        }

        use_case = ExtractAndSynthesize(llm=mock_llm)
        subject = CompanySubject(name="Unknown LLC")

        result = await use_case.execute(subject=subject, findings=[])

        assert result.summary is None
        assert len(result.blocks) >= 1

    @pytest.mark.asyncio
    async def test_falls_back_on_json_parse_error(self) -> None:
        mock_llm = AsyncMock()
        mock_llm.send.return_value = {
            "role": "assistant",
            "content": [{"type": "text", "text": "This is not valid JSON at all."}],
            "stop_reason": "end_turn",
            "usage": {"input_tokens": 50, "output_tokens": 30},
        }

        use_case = ExtractAndSynthesize(llm=mock_llm)
        subject = CompanySubject(name="Acme Corp")
        findings = [
            RawFindings(
                source_name="website_scraper",
                findings=(Finding(content="Some data"),),
            ),
        ]

        result = await use_case.execute(subject=subject, findings=findings)

        assert len(result.blocks) == 1
        assert result.blocks[0]["block_id"] == "raw_research"
```

- [ ] **Step 2: Write tests for pipeline orchestrator**

Create `apps/api/tests/unit/research/test_pipeline.py`:

```python
"""Unit tests for ResearchPipeline orchestrator."""

from __future__ import annotations

from unittest.mock import AsyncMock

import pytest

from prescient.research.application.pipeline import ResearchPipeline
from prescient.research.application.source_registry import SourceRegistry
from prescient.research.application.use_cases.extract_and_synthesize import (
    ExtractionResult,
)
from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    SubjectType,
)
from prescient.research.domain.entities.research_task import (
    ResearchMode,
    ResearchTask,
    ResearchTrigger,
    TaskStatus,
)


class _FakeSource:
    def __init__(self, name: str, *, quick_pass: bool = True) -> None:
        self.name = name
        self.timeout_seconds = 10
        self.quick_pass = quick_pass
        self.fetch_called = False

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: CompanySubject) -> RawFindings:
        self.fetch_called = True
        return RawFindings(
            source_name=self.name,
            findings=(Finding(content=f"Data from {self.name}"),),
        )


class _FailingSource:
    name = "failing_source"
    timeout_seconds = 10
    quick_pass = True

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: CompanySubject) -> RawFindings:
        raise ConnectionError("Source unavailable")


class TestResearchPipeline:
    @pytest.mark.asyncio
    async def test_quick_pass_runs_quick_sources_only(self) -> None:
        web_source = _FakeSource("website_scraper", quick_pass=True)
        edgar_source = _FakeSource("edgar", quick_pass=False)
        registry = SourceRegistry()
        registry.register(web_source)
        registry.register(edgar_source)

        mock_synthesizer = AsyncMock()
        mock_synthesizer.execute.return_value = ExtractionResult(
            blocks=[{"block_id": "overview", "heading": "Overview", "body": "Test", "order": 0}],
            summary="Test summary",
        )

        mock_repo = AsyncMock()

        pipeline = ResearchPipeline(
            source_registry=registry,
            synthesizer=mock_synthesizer,
            artifact_repo=mock_repo,
        )

        subject = CompanySubject(name="Acme Corp")
        task = await pipeline.run(
            subject=subject,
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.ONBOARDING,
            tenant_id="org-001",
            created_by="user-001",
        )

        assert web_source.fetch_called is True
        assert edgar_source.fetch_called is False
        assert task.status == TaskStatus.COMPLETED

    @pytest.mark.asyncio
    async def test_deep_pass_runs_all_sources(self) -> None:
        web_source = _FakeSource("website_scraper", quick_pass=True)
        edgar_source = _FakeSource("edgar", quick_pass=False)
        registry = SourceRegistry()
        registry.register(web_source)
        registry.register(edgar_source)

        mock_synthesizer = AsyncMock()
        mock_synthesizer.execute.return_value = ExtractionResult(
            blocks=[{"block_id": "overview", "heading": "Overview", "body": "Test", "order": 0}],
            summary="Test summary",
        )

        mock_repo = AsyncMock()

        pipeline = ResearchPipeline(
            source_registry=registry,
            synthesizer=mock_synthesizer,
            artifact_repo=mock_repo,
        )

        subject = CompanySubject(name="Acme Corp")
        task = await pipeline.run(
            subject=subject,
            mode=ResearchMode.DEEP,
            triggered_by=ResearchTrigger.COPILOT,
            tenant_id="org-001",
            created_by="user-001",
        )

        assert web_source.fetch_called is True
        assert edgar_source.fetch_called is True
        assert task.status == TaskStatus.COMPLETED

    @pytest.mark.asyncio
    async def test_source_failure_does_not_block_pipeline(self) -> None:
        working_source = _FakeSource("website_scraper")
        failing_source = _FailingSource()
        registry = SourceRegistry()
        registry.register(working_source)
        registry.register(failing_source)

        mock_synthesizer = AsyncMock()
        mock_synthesizer.execute.return_value = ExtractionResult(
            blocks=[{"block_id": "overview", "heading": "Overview", "body": "Partial", "order": 0}],
            summary="Partial data",
        )

        mock_repo = AsyncMock()

        pipeline = ResearchPipeline(
            source_registry=registry,
            synthesizer=mock_synthesizer,
            artifact_repo=mock_repo,
        )

        subject = CompanySubject(name="Acme Corp")
        task = await pipeline.run(
            subject=subject,
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.ONBOARDING,
            tenant_id="org-001",
            created_by="user-001",
        )

        assert task.status == TaskStatus.COMPLETED
        assert task.source_results["failing_source"]["status"] == "failed"
        assert task.source_results["website_scraper"]["status"] == "succeeded"

    @pytest.mark.asyncio
    async def test_all_sources_fail_still_completes_with_empty_profile(self) -> None:
        failing_source = _FailingSource()
        registry = SourceRegistry()
        registry.register(failing_source)

        mock_synthesizer = AsyncMock()
        mock_synthesizer.execute.return_value = ExtractionResult(
            blocks=[{"block_id": "overview", "heading": "Overview", "body": "No data available.", "order": 0}],
            summary=None,
        )

        mock_repo = AsyncMock()

        pipeline = ResearchPipeline(
            source_registry=registry,
            synthesizer=mock_synthesizer,
            artifact_repo=mock_repo,
        )

        subject = CompanySubject(name="Ghost Corp")
        task = await pipeline.run(
            subject=subject,
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.ONBOARDING,
            tenant_id="org-001",
            created_by="user-001",
        )

        assert task.status == TaskStatus.COMPLETED
        mock_synthesizer.execute.assert_called_once()
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/research/test_pipeline.py tests/unit/research/test_extract_and_synthesize.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 4: Implement ExtractAndSynthesize**

Create `apps/api/src/prescient/research/application/use_cases/__init__.py` (empty).

Create `apps/api/src/prescient/research/application/use_cases/extract_and_synthesize.py`:

```python
"""LLM-based extraction and synthesis of research findings."""

from __future__ import annotations

import json
import logging
from dataclasses import dataclass, field

from prescient.research.domain.entities.finding import RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    MarketSubject,
    PersonSubject,
    ResearchSubject,
)

logger = logging.getLogger(__name__)

_MAX_RESEARCH_TEXT = 16_000

_COMPANY_SYSTEM_PROMPT = """\
You are a research analyst. Given raw research data about a company, produce a \
structured JSON profile.

Output ONLY valid JSON with this structure:
{
  "blocks": [
    {"block_id": "overview", "heading": "Company Overview", "body": "...", "order": 0},
    {"block_id": "products", "heading": "Products & Services", "body": "...", "order": 1},
    {"block_id": "financials", "heading": "Financial Summary", "body": "...", "order": 2},
    {"block_id": "competitive", "heading": "Competitive Position", "body": "...", "order": 3},
    {"block_id": "leadership", "heading": "Leadership", "body": "...", "order": 4},
    {"block_id": "recent", "heading": "Recent Developments", "body": "...", "order": 5}
  ],
  "summary": "One paragraph executive summary."
}

Omit any block for which you have no information. If data is very limited, \
include only the blocks you can populate. Set summary to null if insufficient data.
"""

_PERSON_SYSTEM_PROMPT = """\
You are a research analyst. Given raw research data about a person, produce a \
structured JSON profile.

Output ONLY valid JSON with this structure:
{
  "blocks": [
    {"block_id": "overview", "heading": "Overview", "body": "...", "order": 0},
    {"block_id": "career", "heading": "Career History", "body": "...", "order": 1},
    {"block_id": "expertise", "heading": "Areas of Expertise", "body": "...", "order": 2},
    {"block_id": "public", "heading": "Public Presence", "body": "...", "order": 3}
  ],
  "summary": "One paragraph summary."
}

Omit any block for which you have no information.
"""

_MARKET_SYSTEM_PROMPT = """\
You are a research analyst. Given raw research data about a market or competitive \
landscape, produce a structured JSON analysis.

Output ONLY valid JSON with this structure:
{
  "blocks": [
    {"block_id": "overview", "heading": "Market Overview", "body": "...", "order": 0},
    {"block_id": "competitors", "heading": "Key Competitors", "body": "...", "order": 1},
    {"block_id": "trends", "heading": "Market Trends", "body": "...", "order": 2},
    {"block_id": "dynamics", "heading": "Competitive Dynamics", "body": "...", "order": 3}
  ],
  "summary": "One paragraph summary."
}

Omit any block for which you have no information.
"""


@dataclass(frozen=True)
class ExtractionResult:
    blocks: list[dict]
    summary: str | None = None


class ExtractAndSynthesize:
    """Uses an LLM to extract structured data from raw findings and synthesize a profile."""

    def __init__(self, llm: object) -> None:
        self._llm = llm

    async def execute(
        self,
        *,
        subject: ResearchSubject,
        findings: list[RawFindings],
    ) -> ExtractionResult:
        raw_text = self._build_raw_text(subject, findings)
        system_prompt = self._system_prompt_for(subject)

        user_message = f"Research subject: {self._describe_subject(subject)}\n\nRaw research data:\n{raw_text}"
        if not raw_text.strip():
            user_message = f"Research subject: {self._describe_subject(subject)}\n\nNo research data was available. Produce a minimal profile noting the lack of data."

        response = await self._llm.send(
            system=system_prompt,
            messages=[{"role": "user", "content": user_message}],
            tools=[],
            model="default",
        )

        text = self._extract_text(response)
        return self._parse_result(text, raw_text)

    def _build_raw_text(
        self, subject: ResearchSubject, findings: list[RawFindings]
    ) -> str:
        parts = []
        for rf in findings:
            if rf.has_data:
                parts.append(f"## Source: {rf.source_name}\n{rf.all_text()}")
        combined = "\n\n".join(parts)
        return combined[:_MAX_RESEARCH_TEXT]

    def _system_prompt_for(self, subject: ResearchSubject) -> str:
        if isinstance(subject, CompanySubject):
            return _COMPANY_SYSTEM_PROMPT
        if isinstance(subject, PersonSubject):
            return _PERSON_SYSTEM_PROMPT
        if isinstance(subject, MarketSubject):
            return _MARKET_SYSTEM_PROMPT
        return _COMPANY_SYSTEM_PROMPT

    def _describe_subject(self, subject: ResearchSubject) -> str:
        if isinstance(subject, CompanySubject):
            parts = [subject.name]
            if subject.industry:
                parts.append(f"Industry: {subject.industry}")
            if subject.website:
                parts.append(f"Website: {subject.website}")
            return ", ".join(parts)
        if isinstance(subject, PersonSubject):
            parts = [subject.name]
            if subject.role:
                parts.append(f"Role: {subject.role}")
            if subject.company_name:
                parts.append(f"Company: {subject.company_name}")
            return ", ".join(parts)
        if isinstance(subject, MarketSubject):
            return f"Market around {subject.seed_company_name} ({subject.industry or 'unknown industry'})"
        return str(subject)

    def _extract_text(self, response: dict) -> str:
        for block in response.get("content", []):
            if block.get("type") == "text":
                return block["text"]
        return ""

    def _parse_result(self, text: str, raw_text: str) -> ExtractionResult:
        try:
            data = json.loads(text)
            return ExtractionResult(
                blocks=data.get("blocks", []),
                summary=data.get("summary"),
            )
        except (json.JSONDecodeError, KeyError, TypeError):
            logger.warning("Failed to parse LLM synthesis output, falling back to raw block")
            return ExtractionResult(
                blocks=[
                    {
                        "block_id": "raw_research",
                        "heading": "Research Data",
                        "body": raw_text[:4000] if raw_text else "No data available.",
                        "order": 0,
                    }
                ],
                summary=None,
            )
```

- [ ] **Step 5: Implement ResearchPipeline orchestrator**

Create `apps/api/src/prescient/research/application/pipeline.py`:

```python
"""Research pipeline orchestrator — staged execution with checkpointing."""

from __future__ import annotations

import asyncio
import logging

from prescient.research.application.source_registry import SourceRegistry
from prescient.research.application.use_cases.extract_and_synthesize import (
    ExtractAndSynthesize,
)
from prescient.research.domain.entities.research_subject import ResearchSubject
from prescient.research.domain.entities.research_task import (
    ResearchMode,
    ResearchTask,
    ResearchTrigger,
    SourceStatus,
    TaskStatus,
)

logger = logging.getLogger(__name__)


class ResearchPipeline:
    """Orchestrates the full research pipeline: discover → fetch → extract → synthesize → persist."""

    def __init__(
        self,
        *,
        source_registry: SourceRegistry,
        synthesizer: ExtractAndSynthesize,
        artifact_repo: object,
    ) -> None:
        self._sources = source_registry
        self._synthesizer = synthesizer
        self._artifact_repo = artifact_repo

    async def run(
        self,
        *,
        subject: ResearchSubject,
        mode: ResearchMode,
        triggered_by: ResearchTrigger,
        tenant_id: str,
        created_by: str,
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
        sources = self._sources.sources_for(subject.subject_type, mode=mode)

        # Stage 2: Fetch from all sources in parallel
        task = task.advance_to(TaskStatus.FETCHING)
        task = await self._fetch_all(task, sources, subject)

        # Stage 3+4: Extract and synthesize via LLM
        task = task.advance_to(TaskStatus.EXTRACTING)
        all_findings = task.all_findings()

        extraction_result = await self._synthesizer.execute(
            subject=subject,
            findings=all_findings,
        )

        task = task.advance_to(TaskStatus.SYNTHESIZING)

        # Stage 5: Persist (artifact creation handled by caller or repo)
        task = task.advance_to(TaskStatus.PERSISTING)
        task = task.mark_completed(
            synthesized_result={
                "blocks": extraction_result.blocks,
                "summary": extraction_result.summary,
            },
        )

        return task

    async def _fetch_all(
        self,
        task: ResearchTask,
        sources: list,
        subject: ResearchSubject,
    ) -> ResearchTask:
        """Fetch from all sources concurrently, recording results on the task."""

        async def _fetch_one(source: object) -> tuple[str, SourceStatus, object]:
            source_name = source.name  # type: ignore[attr-defined]
            timeout = source.timeout_seconds  # type: ignore[attr-defined]
            try:
                findings = await asyncio.wait_for(
                    source.fetch(subject),  # type: ignore[attr-defined]
                    timeout=timeout,
                )
                return (source_name, SourceStatus.SUCCEEDED, findings)
            except asyncio.TimeoutError:
                logger.warning("Source %s timed out after %ds", source_name, timeout)
                return (source_name, SourceStatus.TIMED_OUT, None)
            except Exception as exc:
                logger.warning("Source %s failed: %s", source_name, exc)
                return (source_name, SourceStatus.FAILED, str(exc))

        results = await asyncio.gather(*[_fetch_one(s) for s in sources])

        for source_name, status, data in results:
            if status == SourceStatus.SUCCEEDED:
                task = task.record_source_result(
                    source_name=source_name,
                    status=status,
                    findings=data,
                )
            elif status == SourceStatus.FAILED:
                task = task.record_source_result(
                    source_name=source_name,
                    status=status,
                    error=data if isinstance(data, str) else None,
                )
            else:
                task = task.record_source_result(
                    source_name=source_name,
                    status=status,
                )

        return task
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/research/ -v`
Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/research/application/ apps/api/tests/unit/research/test_pipeline.py apps/api/tests/unit/research/test_extract_and_synthesize.py
git commit -m "feat(research): add pipeline orchestrator and extract-and-synthesize use case"
```

---

### Task 4: Website Scraper Source Plugin

**Files:**
- Create: `apps/api/src/prescient/research/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/research/infrastructure/sources/__init__.py`
- Create: `apps/api/src/prescient/research/infrastructure/sources/website_scraper.py`
- Test: `apps/api/tests/unit/research/test_website_scraper.py`

- [ ] **Step 1: Write tests for website scraper**

Create `apps/api/tests/unit/research/test_website_scraper.py`:

```python
"""Unit tests for WebsiteScraperSource."""

from __future__ import annotations

from unittest.mock import AsyncMock, patch

import pytest

from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    PersonSubject,
    SubjectType,
)
from prescient.research.infrastructure.sources.website_scraper import (
    WebsiteScraperSource,
)


class TestWebsiteScraperSource:
    def test_can_handle_company(self) -> None:
        source = WebsiteScraperSource()
        assert source.can_handle(SubjectType.COMPANY) is True

    def test_can_handle_person(self) -> None:
        source = WebsiteScraperSource()
        assert source.can_handle(SubjectType.PERSON) is True

    def test_cannot_handle_market(self) -> None:
        source = WebsiteScraperSource()
        assert source.can_handle(SubjectType.MARKET) is False

    def test_is_quick_pass(self) -> None:
        source = WebsiteScraperSource()
        assert source.quick_pass is True

    @pytest.mark.asyncio
    async def test_fetch_company_with_website(self) -> None:
        source = WebsiteScraperSource()
        subject = CompanySubject(name="Acme Corp", website="https://acme.com")

        mock_response = AsyncMock()
        mock_response.status_code = 200
        mock_response.text = """
        <html><body>
        <h1>About Acme Corp</h1>
        <p>We are a leading manufacturer of widgets since 1990.</p>
        <p>Our products serve the industrial sector worldwide.</p>
        </body></html>
        """

        with patch(
            "prescient.research.infrastructure.sources.website_scraper.httpx.AsyncClient"
        ) as mock_client_cls:
            mock_client = AsyncMock()
            mock_client.__aenter__ = AsyncMock(return_value=mock_client)
            mock_client.__aexit__ = AsyncMock(return_value=False)
            mock_client.get.return_value = mock_response
            mock_client_cls.return_value = mock_client

            result = await source.fetch(subject)

        assert result.source_name == "website_scraper"
        assert result.has_data is True
        assert any("widget" in f.content.lower() for f in result.findings)

    @pytest.mark.asyncio
    async def test_fetch_company_without_website_returns_empty(self) -> None:
        source = WebsiteScraperSource()
        subject = CompanySubject(name="Mystery Corp")

        result = await source.fetch(subject)

        assert result.has_data is False

    @pytest.mark.asyncio
    async def test_fetch_handles_http_error(self) -> None:
        source = WebsiteScraperSource()
        subject = CompanySubject(name="Acme Corp", website="https://acme.com")

        mock_response = AsyncMock()
        mock_response.status_code = 404
        mock_response.text = "Not Found"

        with patch(
            "prescient.research.infrastructure.sources.website_scraper.httpx.AsyncClient"
        ) as mock_client_cls:
            mock_client = AsyncMock()
            mock_client.__aenter__ = AsyncMock(return_value=mock_client)
            mock_client.__aexit__ = AsyncMock(return_value=False)
            mock_client.get.return_value = mock_response
            mock_client_cls.return_value = mock_client

            result = await source.fetch(subject)

        assert result.has_data is False
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/research/test_website_scraper.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement WebsiteScraperSource**

Create `apps/api/src/prescient/research/infrastructure/__init__.py` (empty).
Create `apps/api/src/prescient/research/infrastructure/sources/__init__.py` (empty).

Create `apps/api/src/prescient/research/infrastructure/sources/website_scraper.py`:

```python
"""Website scraper research source — fetches and extracts text from company websites."""

from __future__ import annotations

import logging
from html.parser import HTMLParser

import httpx

from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    PersonSubject,
    ResearchSubject,
    SubjectType,
)

logger = logging.getLogger(__name__)

_MAX_BODY_CHARS = 8_000
_SKIP_TAGS = frozenset({"script", "style", "nav", "footer", "header", "noscript"})


class _TextExtractor(HTMLParser):
    """Simple HTML → text extractor that skips scripts, styles, and nav."""

    def __init__(self) -> None:
        super().__init__()
        self._parts: list[str] = []
        self._skip_depth = 0

    def handle_starttag(self, tag: str, attrs: list[tuple[str, str | None]]) -> None:
        if tag in _SKIP_TAGS:
            self._skip_depth += 1

    def handle_endtag(self, tag: str) -> None:
        if tag in _SKIP_TAGS and self._skip_depth > 0:
            self._skip_depth -= 1

    def handle_data(self, data: str) -> None:
        if self._skip_depth == 0:
            text = data.strip()
            if text:
                self._parts.append(text)

    def get_text(self) -> str:
        return "\n".join(self._parts)


def _extract_text(html: str) -> str:
    parser = _TextExtractor()
    parser.feed(html)
    return parser.get_text()[:_MAX_BODY_CHARS]


class WebsiteScraperSource:
    """Fetches and extracts text from a company or person's website."""

    name = "website_scraper"
    timeout_seconds = 10
    quick_pass = True

    _PATHS = ("/", "/about", "/about-us", "/company")

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type in (SubjectType.COMPANY, SubjectType.PERSON)

    async def fetch(self, subject: ResearchSubject) -> RawFindings:
        website = self._get_website(subject)
        if not website:
            return RawFindings(source_name=self.name)

        base_url = website.rstrip("/")
        findings: list[Finding] = []

        async with httpx.AsyncClient(
            timeout=self.timeout_seconds,
            follow_redirects=True,
            headers={"User-Agent": "PrescientOS-Research/1.0"},
        ) as client:
            for path in self._PATHS:
                url = base_url + path
                try:
                    response = await client.get(url)
                    if response.status_code != 200:
                        continue
                    text = _extract_text(response.text)
                    if text and len(text) > 50:
                        findings.append(
                            Finding(
                                content=text,
                                source_url=url,
                                category="website",
                            )
                        )
                except httpx.HTTPError as exc:
                    logger.debug("Failed to fetch %s: %s", url, exc)

        return RawFindings(source_name=self.name, findings=tuple(findings))

    def _get_website(self, subject: ResearchSubject) -> str | None:
        if isinstance(subject, CompanySubject):
            return subject.website
        if isinstance(subject, PersonSubject):
            return None
        return None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/research/test_website_scraper.py -v`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/research/infrastructure/ apps/api/tests/unit/research/test_website_scraper.py
git commit -m "feat(research): add website scraper source plugin"
```

---

### Task 5: Remaining Source Plugins (EDGAR wrapper, Search Engine, News, DNS)

**Files:**
- Create: `apps/api/src/prescient/research/infrastructure/sources/edgar_source.py`
- Create: `apps/api/src/prescient/research/infrastructure/sources/search_engine_source.py`
- Create: `apps/api/src/prescient/research/infrastructure/sources/news_source.py`
- Create: `apps/api/src/prescient/research/infrastructure/sources/dns_lookup_source.py`

- [ ] **Step 1: Implement EdgarResearchSource wrapper**

This wraps the existing `prescient.intelligence.infrastructure.sources.edgar_source.EdgarResearchSource` to conform to the new `ResearchSource` protocol.

Create `apps/api/src/prescient/research/infrastructure/sources/edgar_source.py`:

```python
"""EDGAR research source — wraps existing EdgarResearchSource to fit the plugin protocol."""

from __future__ import annotations

import logging

from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    ResearchSubject,
    SubjectType,
)

logger = logging.getLogger(__name__)


class EdgarSource:
    """Wraps the existing EDGAR client to conform to the ResearchSource protocol."""

    name = "edgar"
    timeout_seconds = 30
    quick_pass = False

    def __init__(self, edgar_research_source: object) -> None:
        self._inner = edgar_research_source

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: ResearchSubject) -> RawFindings:
        if not isinstance(subject, CompanySubject) or not subject.cik:
            return RawFindings(source_name=self.name)

        result = await self._inner.research(  # type: ignore[attr-defined]
            cik=subject.cik,
            company_name=subject.name,
        )

        if not result.has_data:
            return RawFindings(source_name=self.name)

        findings = []
        for filing in result.filings:
            sections_text = "\n".join(
                f"### {name}\n{text}"
                for name, text in filing.get("sections", {}).items()
            )
            findings.append(
                Finding(
                    content=f"## {filing['form']} filed {filing['filing_date']}\n{sections_text}",
                    source_url=None,
                    category="sec_filing",
                )
            )

        return RawFindings(source_name=self.name, findings=tuple(findings))
```

- [ ] **Step 2: Implement SearchEngineSource**

Create `apps/api/src/prescient/research/infrastructure/sources/search_engine_source.py`:

```python
"""Search engine research source — uses web search for general discovery."""

from __future__ import annotations

import logging

import httpx

from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    MarketSubject,
    PersonSubject,
    ResearchSubject,
    SubjectType,
)

logger = logging.getLogger(__name__)


class SearchEngineSource:
    """Uses web search API for general research discovery.

    Starts with a simple implementation using DuckDuckGo HTML search.
    Can be upgraded to a paid search API later.
    """

    name = "search_engine"
    timeout_seconds = 10
    quick_pass = True

    def can_handle(self, subject_type: SubjectType) -> bool:
        return True

    async def fetch(self, subject: ResearchSubject) -> RawFindings:
        query = self._build_query(subject)

        try:
            async with httpx.AsyncClient(
                timeout=self.timeout_seconds,
                follow_redirects=True,
                headers={"User-Agent": "PrescientOS-Research/1.0"},
            ) as client:
                response = await client.get(
                    "https://html.duckduckgo.com/html/",
                    params={"q": query},
                )
                if response.status_code != 200:
                    return RawFindings(source_name=self.name)

                # Extract snippets from DuckDuckGo HTML results
                findings = self._parse_ddg_html(response.text, query)
                return RawFindings(source_name=self.name, findings=tuple(findings))

        except httpx.HTTPError as exc:
            logger.warning("Search engine fetch failed: %s", exc)
            return RawFindings(source_name=self.name)

    def _build_query(self, subject: ResearchSubject) -> str:
        if isinstance(subject, CompanySubject):
            parts = [subject.name]
            if subject.industry:
                parts.append(subject.industry)
            parts.append("company overview")
            return " ".join(parts)
        if isinstance(subject, PersonSubject):
            parts = [subject.name]
            if subject.company_name:
                parts.append(subject.company_name)
            if subject.role:
                parts.append(subject.role)
            return " ".join(parts)
        if isinstance(subject, MarketSubject):
            parts = [subject.seed_company_name, "competitors", "market"]
            if subject.industry:
                parts.append(subject.industry)
            return " ".join(parts)
        return ""

    def _parse_ddg_html(self, html: str, query: str) -> list[Finding]:
        """Extract result snippets from DuckDuckGo HTML response.

        This is a lightweight parser — extracts text between result markers.
        A more robust approach would use BeautifulSoup, but we keep
        dependencies minimal for now.
        """
        from html.parser import HTMLParser

        class _SnippetExtractor(HTMLParser):
            def __init__(self) -> None:
                super().__init__()
                self.in_result = False
                self.snippets: list[str] = []
                self._current: list[str] = []

            def handle_starttag(self, tag: str, attrs: list) -> None:
                attr_dict = dict(attrs)
                if tag == "a" and "result__snippet" in attr_dict.get("class", ""):
                    self.in_result = True
                    self._current = []

            def handle_data(self, data: str) -> None:
                if self.in_result:
                    self._current.append(data.strip())

            def handle_endtag(self, tag: str) -> None:
                if self.in_result and tag == "a":
                    self.in_result = False
                    text = " ".join(self._current)
                    if text:
                        self.snippets.append(text)

        parser = _SnippetExtractor()
        parser.feed(html)

        return [
            Finding(content=snippet, category="web_search")
            for snippet in parser.snippets[:10]
        ]
```

- [ ] **Step 3: Implement NewsAggregatorSource**

Create `apps/api/src/prescient/research/infrastructure/sources/news_source.py`:

```python
"""News aggregator research source — fetches recent news via RSS/web search."""

from __future__ import annotations

import logging
import xml.etree.ElementTree as ET

import httpx

from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    MarketSubject,
    PersonSubject,
    ResearchSubject,
    SubjectType,
)

logger = logging.getLogger(__name__)


class NewsAggregatorSource:
    """Fetches recent news coverage using Google News RSS.

    Deep pass only — news fetching is slower and less critical
    for the initial onboarding quick pass.
    """

    name = "news_aggregator"
    timeout_seconds = 15
    quick_pass = False

    def can_handle(self, subject_type: SubjectType) -> bool:
        return True

    async def fetch(self, subject: ResearchSubject) -> RawFindings:
        query = self._build_query(subject)

        try:
            async with httpx.AsyncClient(
                timeout=self.timeout_seconds,
                follow_redirects=True,
                headers={"User-Agent": "PrescientOS-Research/1.0"},
            ) as client:
                response = await client.get(
                    "https://news.google.com/rss/search",
                    params={"q": query, "hl": "en-US", "gl": "US", "ceid": "US:en"},
                )
                if response.status_code != 200:
                    return RawFindings(source_name=self.name)

                findings = self._parse_rss(response.text)
                return RawFindings(source_name=self.name, findings=tuple(findings))

        except httpx.HTTPError as exc:
            logger.warning("News aggregator fetch failed: %s", exc)
            return RawFindings(source_name=self.name)

    def _build_query(self, subject: ResearchSubject) -> str:
        if isinstance(subject, CompanySubject):
            return subject.name
        if isinstance(subject, PersonSubject):
            parts = [subject.name]
            if subject.company_name:
                parts.append(subject.company_name)
            return " ".join(parts)
        if isinstance(subject, MarketSubject):
            return f"{subject.seed_company_name} {subject.industry or 'market'}"
        return ""

    def _parse_rss(self, xml_text: str) -> list[Finding]:
        """Parse Google News RSS feed into findings."""
        findings = []
        try:
            root = ET.fromstring(xml_text)
            for item in root.findall(".//item")[:10]:
                title = item.findtext("title", "")
                description = item.findtext("description", "")
                link = item.findtext("link", "")
                pub_date = item.findtext("pubDate", "")
                content = f"{title}\n{description}"
                if pub_date:
                    content = f"[{pub_date}] {content}"
                findings.append(
                    Finding(
                        content=content,
                        source_url=link or None,
                        category="news",
                    )
                )
        except ET.ParseError:
            logger.warning("Failed to parse news RSS feed")

        return findings
```

- [ ] **Step 4: Implement DnsLookupSource**

Create `apps/api/src/prescient/research/infrastructure/sources/dns_lookup_source.py`:

```python
"""DNS lookup research source — lightweight company size/tech signals from domain."""

from __future__ import annotations

import logging
import socket
from urllib.parse import urlparse

from prescient.research.domain.entities.finding import Finding, RawFindings
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    ResearchSubject,
    SubjectType,
)

logger = logging.getLogger(__name__)


class DnsLookupSource:
    """Extracts lightweight signals from a company's domain.

    Quick pass — very fast, provides basic infrastructure signals.
    """

    name = "dns_lookup"
    timeout_seconds = 5
    quick_pass = True

    def can_handle(self, subject_type: SubjectType) -> bool:
        return subject_type == SubjectType.COMPANY

    async def fetch(self, subject: ResearchSubject) -> RawFindings:
        if not isinstance(subject, CompanySubject) or not subject.website:
            return RawFindings(source_name=self.name)

        domain = self._extract_domain(subject.website)
        if not domain:
            return RawFindings(source_name=self.name)

        findings: list[Finding] = []

        try:
            # Basic DNS resolution to confirm domain is active
            addresses = socket.getaddrinfo(domain, None)
            if addresses:
                findings.append(
                    Finding(
                        content=f"Domain {domain} is active and resolves to {len(addresses)} address(es).",
                        category="dns",
                    )
                )
        except socket.gaierror:
            findings.append(
                Finding(
                    content=f"Domain {domain} does not resolve — may be inactive or misconfigured.",
                    category="dns",
                )
            )

        return RawFindings(source_name=self.name, findings=tuple(findings))

    def _extract_domain(self, website: str) -> str | None:
        try:
            parsed = urlparse(website if "://" in website else f"https://{website}")
            return parsed.hostname
        except Exception:
            return None
```

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/research/infrastructure/sources/
git commit -m "feat(research): add EDGAR, search engine, news, and DNS source plugins"
```

---

### Task 6: Database Table, Migration, and RunResearch Use Case

**Files:**
- Create: `apps/api/src/prescient/research/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/research/infrastructure/tables/research_task.py`
- Create: `apps/api/src/prescient/research/infrastructure/repositories/__init__.py`
- Create: `apps/api/src/prescient/research/infrastructure/repositories/research_task_repository.py`
- Create: `apps/api/src/prescient/research/application/use_cases/run_research.py`
- Create: `apps/api/alembic/versions/20260413_research_engine.py`
- Modify: `apps/api/src/prescient/shared/metadata.py`
- Test: `apps/api/tests/unit/research/test_run_research.py`

- [ ] **Step 1: Write test for RunResearch use case**

Create `apps/api/tests/unit/research/test_run_research.py`:

```python
"""Unit tests for RunResearch use case."""

from __future__ import annotations

from unittest.mock import AsyncMock

import pytest

from prescient.research.application.use_cases.run_research import (
    RunResearch,
    RunResearchCommand,
    RunResearchResult,
)
from prescient.research.domain.entities.research_task import (
    ResearchMode,
    ResearchTask,
    ResearchTrigger,
    TaskStatus,
)
from prescient.research.domain.entities.research_subject import CompanySubject


class TestRunResearch:
    @pytest.mark.asyncio
    async def test_execute_returns_result_with_task_id(self) -> None:
        mock_pipeline = AsyncMock()
        mock_pipeline.run.return_value = ResearchTask.create(
            subject=CompanySubject(name="Acme Corp"),
            mode=ResearchMode.QUICK,
            triggered_by=ResearchTrigger.ONBOARDING,
            tenant_id="org-001",
            created_by="user-001",
        ).mark_completed(
            result_artifact_id="artifact-123",
            synthesized_result={"summary": "Acme is a widget company."},
        )

        mock_session = AsyncMock()

        use_case = RunResearch(pipeline=mock_pipeline, session=mock_session)

        command = RunResearchCommand(
            subject_type="company",
            subject_data={"name": "Acme Corp", "website": "https://acme.com"},
            mode="quick",
            triggered_by="onboarding",
            tenant_id="org-001",
            created_by="user-001",
        )

        result = await use_case.execute(command)

        assert isinstance(result, RunResearchResult)
        assert result.task_id != ""
        assert result.status == TaskStatus.COMPLETED
        assert result.artifact_id == "artifact-123"
        mock_pipeline.run.assert_called_once()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/research/test_run_research.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement ResearchTaskRow table**

Create `apps/api/src/prescient/research/infrastructure/tables/__init__.py`:

```python
"""Register research tables on Base.metadata."""

from prescient.research.infrastructure.tables.research_task import ResearchTaskRow  # noqa: F401
```

Create `apps/api/src/prescient/research/infrastructure/tables/research_task.py`:

```python
"""SQLAlchemy model for research_tasks table."""

from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, Integer, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class ResearchTaskRow(Base):
    __tablename__ = "research_tasks"
    __table_args__ = {"schema": "research"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    subject_type: Mapped[str] = mapped_column(String(16), nullable=False, index=True)
    subject: Mapped[dict] = mapped_column(JSONB, nullable=False)
    mode: Mapped[str] = mapped_column(String(8), nullable=False)
    status: Mapped[str] = mapped_column(String(16), nullable=False, server_default="pending")
    triggered_by: Mapped[str] = mapped_column(String(16), nullable=False)
    source_results: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="{}")
    extracted_data: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    synthesized_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    result_artifact_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    tenant_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    created_by: Mapped[str] = mapped_column(String(64), nullable=False)
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

- [ ] **Step 4: Register research tables in metadata**

Add to `apps/api/src/prescient/shared/metadata.py`:

```python
import prescient.research.infrastructure.tables
```

- [ ] **Step 5: Implement ResearchTaskRepository**

Create `apps/api/src/prescient/research/infrastructure/repositories/__init__.py` (empty).

Create `apps/api/src/prescient/research/infrastructure/repositories/research_task_repository.py`:

```python
"""Concrete repository for persisting research task rows."""

from __future__ import annotations

from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.research.infrastructure.tables.research_task import ResearchTaskRow


class ResearchTaskRepository:
    """Persists and retrieves research tasks."""

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def save(self, task_data: dict) -> None:
        row = ResearchTaskRow(**task_data)
        self._session.add(row)

    async def get(self, task_id: str) -> dict | None:
        stmt = select(ResearchTaskRow).where(ResearchTaskRow.id == task_id)
        result = await self._session.execute(stmt)
        row = result.scalar_one_or_none()
        if row is None:
            return None
        return {
            "id": row.id,
            "subject_type": row.subject_type,
            "subject": row.subject,
            "mode": row.mode,
            "status": row.status,
            "triggered_by": row.triggered_by,
            "source_results": row.source_results,
            "extracted_data": row.extracted_data,
            "synthesized_result": row.synthesized_result,
            "result_artifact_id": row.result_artifact_id,
            "tenant_id": row.tenant_id,
            "created_by": row.created_by,
        }

    async def update(self, task_id: str, updates: dict) -> None:
        stmt = (
            update(ResearchTaskRow)
            .where(ResearchTaskRow.id == task_id)
            .values(**updates)
        )
        await self._session.execute(stmt)
```

- [ ] **Step 6: Implement RunResearch use case**

Create `apps/api/src/prescient/research/application/use_cases/run_research.py`:

```python
"""RunResearch use case — entry point for triggering research."""

from __future__ import annotations

from dataclasses import dataclass

from prescient.research.application.pipeline import ResearchPipeline
from prescient.research.domain.entities.research_subject import (
    CompanySubject,
    MarketSubject,
    PersonSubject,
    ResearchSubject,
)
from prescient.research.domain.entities.research_task import (
    ResearchMode,
    ResearchTask,
    ResearchTrigger,
    TaskStatus,
)


@dataclass(frozen=True)
class RunResearchCommand:
    subject_type: str  # "company" | "person" | "market"
    subject_data: dict  # Fields depend on subject_type
    mode: str  # "quick" | "deep"
    triggered_by: str  # "onboarding" | "copilot" | "system"
    tenant_id: str
    created_by: str


@dataclass(frozen=True)
class RunResearchResult:
    task_id: str
    status: TaskStatus
    artifact_id: str | None
    summary: str | None


class RunResearch:
    """Orchestrates a research pipeline run and persists the task record."""

    def __init__(self, *, pipeline: ResearchPipeline, session: object) -> None:
        self._pipeline = pipeline
        self._session = session

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
        )

        return RunResearchResult(
            task_id=task.id,
            status=task.status,
            artifact_id=task.result_artifact_id,
            summary=task.synthesized_result.get("summary") if task.synthesized_result else None,
        )

    def _build_subject(self, command: RunResearchCommand) -> ResearchSubject:
        data = command.subject_data
        if command.subject_type == "company":
            return CompanySubject(
                name=data["name"],
                website=data.get("website"),
                cik=data.get("cik"),
                industry=data.get("industry"),
            )
        if command.subject_type == "person":
            return PersonSubject(
                name=data["name"],
                email=data.get("email"),
                company_name=data.get("company_name"),
                role=data.get("role"),
            )
        if command.subject_type == "market":
            return MarketSubject(
                seed_company_name=data["seed_company_name"],
                industry=data.get("industry"),
            )
        raise ValueError(f"Unknown subject type: {command.subject_type}")
```

- [ ] **Step 7: Create migration**

Create `apps/api/alembic/versions/20260413_research_engine.py`:

```python
"""research engine schema

Revision ID: 20260413_research
Revises: 20260413_km_schema
Create Date: 2026-04-13
"""

from __future__ import annotations

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB

revision = "20260413_research"
down_revision = "20260413_km_schema"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("CREATE SCHEMA IF NOT EXISTS research")

    op.create_table(
        "research_tasks",
        sa.Column("id", sa.String(36), primary_key=True),
        sa.Column("subject_type", sa.String(16), nullable=False, index=True),
        sa.Column("subject", JSONB, nullable=False),
        sa.Column("mode", sa.String(8), nullable=False),
        sa.Column("status", sa.String(16), nullable=False, server_default="pending"),
        sa.Column("triggered_by", sa.String(16), nullable=False),
        sa.Column("source_results", JSONB, nullable=False, server_default="{}"),
        sa.Column("extracted_data", JSONB, nullable=True),
        sa.Column("synthesized_result", JSONB, nullable=True),
        sa.Column("result_artifact_id", sa.String(36), nullable=True),
        sa.Column("tenant_id", sa.String(36), nullable=False, index=True),
        sa.Column("created_by", sa.String(64), nullable=False),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        schema="research",
    )

    op.create_table(
        "user_contexts",
        sa.Column("id", sa.String(36), primary_key=True),
        sa.Column("user_id", sa.String(64), nullable=False, index=True),
        sa.Column("company_id", sa.String(36), nullable=True),
        sa.Column("role_description", sa.Text(), nullable=True),
        sa.Column("primary_focus", sa.Text(), nullable=True),
        sa.Column("pain_points", JSONB, nullable=False, server_default="[]"),
        sa.Column("desired_outcomes", JSONB, nullable=False, server_default="[]"),
        sa.Column("competitors_mentioned", JSONB, nullable=False, server_default="[]"),
        sa.Column("raw_interview_summary", sa.Text(), nullable=True),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        schema="onboarding",
    )


def downgrade() -> None:
    op.drop_table("user_contexts", schema="onboarding")
    op.drop_table("research_tasks", schema="research")
    op.execute("DROP SCHEMA IF EXISTS research")
```

- [ ] **Step 8: Run test to verify it passes**

Run: `cd apps/api && uv run pytest tests/unit/research/test_run_research.py -v`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/research/infrastructure/tables/ apps/api/src/prescient/research/infrastructure/repositories/
git add apps/api/src/prescient/research/application/use_cases/run_research.py
git add apps/api/alembic/versions/20260413_research_engine.py
git add apps/api/src/prescient/shared/metadata.py
git commit -m "feat(research): add database table, migration, repository, and RunResearch use case"
```

---

### Task 7: Research API Routes

**Files:**
- Create: `apps/api/src/prescient/research/api/__init__.py`
- Create: `apps/api/src/prescient/research/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`

- [ ] **Step 1: Implement research API routes**

Create `apps/api/src/prescient/research/api/__init__.py` (empty).

Create `apps/api/src/prescient/research/api/routes.py`:

```python
"""Research API routes — trigger research and check status."""

from __future__ import annotations

from pydantic import BaseModel, ConfigDict, Field
from fastapi import APIRouter

from prescient.db import SessionDep
from prescient.auth.dependencies import CtxDep

router = APIRouter(prefix="/research", tags=["research"])


class RunResearchBody(BaseModel):
    model_config = ConfigDict(frozen=True)

    subject_type: str = Field(pattern="^(company|person|market)$")
    subject_data: dict
    mode: str = Field(default="deep", pattern="^(quick|deep)$")


class RunResearchResponse(BaseModel):
    model_config = ConfigDict(frozen=True)

    task_id: str
    status: str
    artifact_id: str | None = None
    summary: str | None = None


class ResearchTaskStatusResponse(BaseModel):
    model_config = ConfigDict(frozen=True)

    task_id: str
    status: str
    artifact_id: str | None = None
    synthesized_result: dict | None = None


@router.post("", response_model=RunResearchResponse, status_code=201)
async def run_research(
    body: RunResearchBody,
    ctx: CtxDep,
    session: SessionDep,
) -> RunResearchResponse:
    """Trigger a research pipeline run."""
    from prescient.research.application.pipeline import ResearchPipeline
    from prescient.research.application.source_registry import SourceRegistry
    from prescient.research.application.use_cases.extract_and_synthesize import (
        ExtractAndSynthesize,
    )
    from prescient.research.application.use_cases.run_research import (
        RunResearch,
        RunResearchCommand,
    )
    from prescient.research.infrastructure.sources.dns_lookup_source import DnsLookupSource
    from prescient.research.infrastructure.sources.news_source import NewsAggregatorSource
    from prescient.research.infrastructure.sources.search_engine_source import SearchEngineSource
    from prescient.research.infrastructure.sources.website_scraper import WebsiteScraperSource
    from prescient.intelligence.infrastructure.llm import AnthropicAdapter
    from prescient.config import get_settings

    settings = get_settings()
    llm = AnthropicAdapter(api_key=settings.anthropic_api_key, default_model=settings.anthropic_model)

    registry = SourceRegistry()
    registry.register(WebsiteScraperSource())
    registry.register(SearchEngineSource())
    registry.register(NewsAggregatorSource())
    registry.register(DnsLookupSource())

    synthesizer = ExtractAndSynthesize(llm=llm)
    pipeline = ResearchPipeline(
        source_registry=registry,
        synthesizer=synthesizer,
        artifact_repo=None,
    )

    use_case = RunResearch(pipeline=pipeline, session=session)
    result = await use_case.execute(
        RunResearchCommand(
            subject_type=body.subject_type,
            subject_data=body.subject_data,
            mode=body.mode,
            triggered_by="copilot",
            tenant_id=str(ctx.user.fund_id),
            created_by=str(ctx.user.user_id),
        )
    )

    await session.commit()

    return RunResearchResponse(
        task_id=result.task_id,
        status=result.status.value,
        artifact_id=result.artifact_id,
        summary=result.summary,
    )
```

- [ ] **Step 2: Register router in main.py**

Add import and include_router in `apps/api/src/prescient/main.py`:

```python
from prescient.research.api.routes import router as research_pipeline_router
```

And in `create_app()`:

```python
app.include_router(research_pipeline_router)
```

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/research/api/ apps/api/src/prescient/main.py
git commit -m "feat(research): add research API routes and register in app"
```

---

## Phase 2: Smart Onboarding

### Task 8: UserContext Entity and Table

**Files:**
- Create: `apps/api/src/prescient/onboarding/domain/entities/user_context.py`
- Create: `apps/api/src/prescient/onboarding/infrastructure/tables/user_context.py`
- Test: `apps/api/tests/unit/onboarding/test_user_context_entity.py`

- [ ] **Step 1: Write tests for UserContext entity**

Create `apps/api/tests/unit/onboarding/__init__.py` (empty) and `apps/api/tests/unit/onboarding/test_user_context_entity.py`:

```python
"""Unit tests for UserContext domain entity."""

from __future__ import annotations

import pytest

from prescient.onboarding.domain.entities.user_context import UserContext


class TestUserContext:
    def test_create_minimal(self) -> None:
        ctx = UserContext.create(user_id="user-001", company_id="company-001")
        assert ctx.user_id == "user-001"
        assert ctx.company_id == "company-001"
        assert ctx.pain_points == ()
        assert ctx.desired_outcomes == ()

    def test_create_with_all_fields(self) -> None:
        ctx = UserContext.create(
            user_id="user-001",
            company_id="company-001",
            role_description="I run the finance team",
            primary_focus="Cost reduction",
            pain_points=("Manual reporting", "No visibility"),
            desired_outcomes=("Automated reports",),
            competitors_mentioned=[{"name": "Rival Corp", "context": "Main competitor"}],
            raw_interview_summary="User is focused on cost reduction and automation.",
        )
        assert ctx.role_description == "I run the finance team"
        assert len(ctx.pain_points) == 2
        assert len(ctx.desired_outcomes) == 1
        assert ctx.competitors_mentioned[0]["name"] == "Rival Corp"

    def test_update_from_interview(self) -> None:
        ctx = UserContext.create(user_id="user-001", company_id="company-001")
        ctx = ctx.update(
            role_description="CEO",
            primary_focus="Growth",
            pain_points=("Hiring",),
        )
        assert ctx.role_description == "CEO"
        assert ctx.primary_focus == "Growth"
        assert ctx.pain_points == ("Hiring",)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/onboarding/test_user_context_entity.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement UserContext entity**

Create `apps/api/src/prescient/onboarding/domain/entities/user_context.py`:

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
    competitors_mentioned: list[dict] = Field(default_factory=list)
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
        competitors_mentioned: list[dict] | None = None,
        raw_interview_summary: str | None = None,
    ) -> UserContext:
        return cls(
            user_id=user_id,
            company_id=company_id,
            role_description=role_description,
            primary_focus=primary_focus,
            pain_points=pain_points,
            desired_outcomes=desired_outcomes,
            competitors_mentioned=competitors_mentioned or [],
            raw_interview_summary=raw_interview_summary,
        )

    def update(self, **kwargs: object) -> UserContext:
        return self.model_copy(
            update={**kwargs, "updated_at": datetime.now(UTC)},
        )
```

- [ ] **Step 4: Implement UserContextRow table**

Create `apps/api/src/prescient/onboarding/infrastructure/tables/user_context.py`:

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
    __table_args__ = {"schema": "onboarding"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    user_id: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    company_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    role_description: Mapped[str | None] = mapped_column(Text(), nullable=True)
    primary_focus: Mapped[str | None] = mapped_column(Text(), nullable=True)
    pain_points: Mapped[list] = mapped_column(JSONB, nullable=False, server_default="[]")
    desired_outcomes: Mapped[list] = mapped_column(JSONB, nullable=False, server_default="[]")
    competitors_mentioned: Mapped[list] = mapped_column(JSONB, nullable=False, server_default="[]")
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

Register in `apps/api/src/prescient/onboarding/infrastructure/tables/__init__.py` by adding:

```python
from prescient.onboarding.infrastructure.tables.user_context import UserContextRow  # noqa: F401
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/onboarding/test_user_context_entity.py -v`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/onboarding/domain/entities/user_context.py
git add apps/api/src/prescient/onboarding/infrastructure/tables/user_context.py
git add apps/api/src/prescient/onboarding/infrastructure/tables/__init__.py
git add apps/api/tests/unit/onboarding/
git commit -m "feat(onboarding): add UserContext entity and database table"
```

---

### Task 9: Onboarding Interview Use Case

**Files:**
- Create: `apps/api/src/prescient/onboarding/application/use_cases/run_interview.py`
- Modify: `apps/api/src/prescient/onboarding/api/routes.py`

This is the backend for the chat-based onboarding interview. It uses the copilot planner with a specialized system prompt that guides the conversation toward gathering UserContext fields.

- [ ] **Step 1: Implement RunInterview use case**

Create `apps/api/src/prescient/onboarding/application/use_cases/run_interview.py`:

```python
"""RunInterview use case — orchestrates the onboarding interview conversation."""

from __future__ import annotations

from dataclasses import dataclass, field

_INTERVIEW_SYSTEM_PROMPT = """\
You are conducting an onboarding interview for a new user of Prescient OS, \
a portfolio intelligence platform. Your goal is to understand this person \
and their priorities so the platform can be personalized for them.

{research_context}

Your interview goals (gather these, but conversationally — not as a checklist):
1. Confirm or correct what we found about their company
2. Understand their role and day-to-day focus
3. Learn what keeps them up at night (pain points)
4. Discover what they'd most want this platform to solve
5. Find out which competitors they watch most closely

Guidelines:
- Ask ONE question at a time
- Be conversational and natural — acknowledge their answers before moving on
- If the research already covers something well, skip it or confirm briefly
- Keep the interview to 5-8 questions max — respect their time
- When you have enough information, wrap up with a summary of what you've learned \
and what you're setting up for them

The user's name is {user_name} and their role is {user_role}.
"""


@dataclass(frozen=True)
class InterviewContext:
    user_name: str
    user_role: str
    company_name: str
    research_summary: str | None = None


def build_interview_system_prompt(context: InterviewContext) -> str:
    """Build the system prompt for the onboarding interview."""
    if context.research_summary:
        research_context = (
            f"Here's what our research found about {context.company_name}:\n"
            f"{context.research_summary}\n\n"
            "Use this to ask informed questions and skip what's already known."
        )
    else:
        research_context = (
            f"We weren't able to find much about {context.company_name} yet. "
            "You'll need to ask a few more questions to understand the company."
        )

    return _INTERVIEW_SYSTEM_PROMPT.format(
        research_context=research_context,
        user_name=context.user_name,
        user_role=context.user_role,
    )


@dataclass(frozen=True)
class ExtractedUserContext:
    role_description: str | None = None
    primary_focus: str | None = None
    pain_points: list[str] = field(default_factory=list)
    desired_outcomes: list[str] = field(default_factory=list)
    competitors_mentioned: list[dict] = field(default_factory=list)
    raw_interview_summary: str | None = None


_EXTRACTION_PROMPT = """\
Analyze the following onboarding interview conversation and extract structured data.

Output ONLY valid JSON with this structure:
{
  "role_description": "What the user actually does day-to-day",
  "primary_focus": "Their current top priority",
  "pain_points": ["pain point 1", "pain point 2"],
  "desired_outcomes": ["what they want solved"],
  "competitors_mentioned": [{"name": "Competitor Name", "context": "why they mentioned them"}],
  "raw_interview_summary": "2-3 sentence summary of the conversation"
}

Set any field to null if the user didn't mention it.
"""
```

- [ ] **Step 2: Add onboarding interview route**

Add to `apps/api/src/prescient/onboarding/api/routes.py` — a new endpoint that starts the interview and streams the conversation. This reuses the existing copilot streaming infrastructure with the interview system prompt.

```python
@router.post("/interview/start", status_code=200)
async def start_interview(
    session: SessionDep,
    ctx: CtxDep,
) -> dict:
    """Kick off the onboarding interview.

    Returns the interview context (research summary + first message)
    so the frontend can render the chat.
    """
    # This endpoint triggers quick research and returns the interview context.
    # The actual conversation uses the copilot stream endpoint with the interview system prompt.
    return {
        "status": "ready",
        "message": "Interview endpoint ready. Use copilot stream with interview mode.",
    }
```

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/onboarding/application/use_cases/run_interview.py
git add apps/api/src/prescient/onboarding/api/routes.py
git commit -m "feat(onboarding): add interview use case with system prompt and extraction"
```

---

## Phase 3: Copilot Research Tools

### Task 10: research_company Copilot Tool

**Files:**
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/research_company.py`
- Modify: `apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py`
- Test: `apps/api/tests/unit/intelligence/test_research_company_tool.py`

- [ ] **Step 1: Write test for research_company tool**

Create `apps/api/tests/unit/intelligence/test_research_company_tool.py`:

```python
"""Unit tests for research_company copilot tool."""

from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.intelligence.infrastructure.tools.research_company import (
    ResearchCompanyArguments,
    ResearchCompanyTool,
)
from prescient.research.domain.entities.research_task import TaskStatus


class TestResearchCompanyTool:
    def test_name_and_description(self) -> None:
        tool = ResearchCompanyTool(pipeline_factory=MagicMock())
        assert tool.name == "research_company"
        assert "research" in tool.description.lower()

    @pytest.mark.asyncio
    async def test_execute_returns_profile(self) -> None:
        mock_pipeline_factory = MagicMock()
        mock_run_research = AsyncMock()
        mock_run_research.execute.return_value = MagicMock(
            task_id="task-001",
            status=TaskStatus.COMPLETED,
            artifact_id="artifact-001",
            summary="Acme Corp is a widget manufacturer.",
        )
        mock_pipeline_factory.return_value = mock_run_research

        tool = ResearchCompanyTool(pipeline_factory=mock_pipeline_factory)

        context = MagicMock()
        context.tenant_id = uuid4()
        context.user_id = "user-001"

        args = ResearchCompanyArguments(
            company_name="Acme Corp",
            website="https://acme.com",
        )

        result = await tool.execute(context, args)

        assert result.status.value == "ok"
        assert len(result.records) > 0
        assert result.records[0]["company_name"] == "Acme Corp"

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

        tool = ResearchCompanyTool(pipeline_factory=mock_pipeline_factory)

        context = MagicMock()
        context.tenant_id = uuid4()
        context.user_id = "user-001"

        args = ResearchCompanyArguments(company_name="Unknown Corp")

        result = await tool.execute(context, args)

        assert result.status.value == "ok"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_research_company_tool.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement research_company tool**

Create `apps/api/src/prescient/intelligence/infrastructure/tools/research_company.py`:

```python
"""research_company copilot tool — triggers the research pipeline for a company."""

from __future__ import annotations

from pydantic import BaseModel, ConfigDict, Field

from prescient.intelligence.infrastructure.tools.base import ToolContext, ToolExecution


class ResearchCompanyArguments(BaseModel):
    model_config = ConfigDict(extra="forbid")

    company_name: str = Field(min_length=1, max_length=256, description="Name of the company to research")
    website: str | None = Field(default=None, description="Company website URL for deeper research")
    cik: str | None = Field(default=None, description="SEC CIK number if known")
    industry: str | None = Field(default=None, description="Industry sector")


class ResearchCompanyTool:
    name = "research_company"
    description = (
        "Research a company using multiple sources (website, SEC filings, news, web search). "
        "Returns a synthesized company profile. Use this when the user asks about an external "
        "company, wants competitive intelligence, or asks you to research a specific company."
    )
    arguments_model = ResearchCompanyArguments

    def __init__(self, pipeline_factory: object) -> None:
        self._pipeline_factory = pipeline_factory

    async def execute(
        self, context: ToolContext, arguments: ResearchCompanyArguments
    ) -> ToolExecution:
        from prescient.research.application.use_cases.run_research import (
            RunResearchCommand,
        )

        run_research = self._pipeline_factory()  # type: ignore[operator]

        result = await run_research.execute(
            RunResearchCommand(
                subject_type="company",
                subject_data={
                    "name": arguments.company_name,
                    "website": arguments.website,
                    "cik": arguments.cik,
                    "industry": arguments.industry,
                },
                mode="deep",
                triggered_by="copilot",
                tenant_id=str(context.tenant_id),
                created_by=str(context.user_id),
            )
        )

        records = [
            {
                "company_name": arguments.company_name,
                "task_id": result.task_id,
                "status": result.status.value,
                "artifact_id": result.artifact_id,
                "summary": result.summary or "Research completed but no summary available.",
            }
        ]

        return ToolExecution.ok(
            self.name,
            summary=f"Researched {arguments.company_name}: {result.summary or 'profile generated'}",
            records=records,
        )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_research_company_tool.py -v`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/tools/research_company.py
git add apps/api/tests/unit/intelligence/test_research_company_tool.py
git commit -m "feat(intelligence): add research_company copilot tool"
```

---

### Task 11: watch_company and save_artifact Copilot Tools

**Files:**
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/watch_company.py`
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/save_artifact.py`
- Test: `apps/api/tests/unit/intelligence/test_watch_company_tool.py`

- [ ] **Step 1: Write test for watch_company tool**

Create `apps/api/tests/unit/intelligence/test_watch_company_tool.py`:

```python
"""Unit tests for watch_company copilot tool."""

from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.intelligence.infrastructure.tools.watch_company import (
    WatchCompanyArguments,
    WatchCompanyTool,
)


class TestWatchCompanyTool:
    def test_name(self) -> None:
        tool = WatchCompanyTool()
        assert tool.name == "watch_company"

    @pytest.mark.asyncio
    async def test_execute_creates_watch_target(self) -> None:
        tool = WatchCompanyTool()

        context = MagicMock()
        context.tenant_id = uuid4()
        context.user_id = "user-001"
        context.session = AsyncMock()

        args = WatchCompanyArguments(
            company_name="Acme Corp",
            purpose="competitor",
        )

        result = await tool.execute(context, args)

        assert result.status.value == "ok"
        assert "Acme Corp" in result.summary
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_watch_company_tool.py -v`
Expected: ModuleNotFoundError

- [ ] **Step 3: Implement watch_company tool**

Create `apps/api/src/prescient/intelligence/infrastructure/tools/watch_company.py`:

```python
"""watch_company copilot tool — promotes a company to a watched entity."""

from __future__ import annotations

from pydantic import BaseModel, ConfigDict, Field

from prescient.intelligence.infrastructure.tools.base import ToolContext, ToolExecution
from prescient.monitoring.application.use_cases.add_watch_target import (
    AddWatchTarget,
    AddWatchTargetCommand,
)
from prescient.monitoring.infrastructure.tables.watch_target import WatchTargetRow


class WatchCompanyArguments(BaseModel):
    model_config = ConfigDict(extra="forbid")

    company_name: str = Field(min_length=1, max_length=256, description="Name of the company to watch")
    purpose: str = Field(
        default="competitor",
        description="Why watching this company: 'competitor', 'partner', 'customer', 'market'",
    )


class WatchCompanyTool:
    name = "watch_company"
    description = (
        "Add a company as a watched entity for ongoing monitoring. "
        "Use this when the user wants to track a company they've been researching, "
        "or when you suggest watching a company after research."
    )
    arguments_model = WatchCompanyArguments

    async def execute(
        self, context: ToolContext, arguments: WatchCompanyArguments
    ) -> ToolExecution:
        use_case = AddWatchTarget(
            session=context.session,
            row_factory=WatchTargetRow,
        )

        result = await use_case.execute(
            AddWatchTargetCommand(
                owner_tenant_id=str(context.tenant_id),
                target_type="company",
                target_id=arguments.company_name,
                purpose=arguments.purpose,
                created_by_user_id=str(context.user_id),
            )
        )

        return ToolExecution.ok(
            self.name,
            summary=f"Now watching {arguments.company_name} as {arguments.purpose}",
            records=[
                {
                    "watch_target_id": result.target_id,
                    "company_name": arguments.company_name,
                    "purpose": arguments.purpose,
                }
            ],
        )
```

- [ ] **Step 4: Implement save_artifact tool**

Create `apps/api/src/prescient/intelligence/infrastructure/tools/save_artifact.py`:

```python
"""save_artifact copilot tool — saves research output as a persistent artifact."""

from __future__ import annotations

from pydantic import BaseModel, ConfigDict, Field

from prescient.artifacts.application.use_cases.create_artifact import (
    CreateArtifact,
    CreateArtifactRequest,
)
from prescient.artifacts.domain.entities.artifact import ArtifactType
from prescient.intelligence.infrastructure.tools.base import ToolContext, ToolExecution


class SaveArtifactArguments(BaseModel):
    model_config = ConfigDict(extra="forbid")

    title: str = Field(min_length=1, max_length=256, description="Title for the artifact")
    summary: str = Field(description="Summary text to save")
    artifact_type: str = Field(
        default="company_profile",
        description="Type: 'company_profile', 'intelligence_signal', 'decision_record'",
    )


class SaveArtifactTool:
    name = "save_artifact"
    description = (
        "Save research output or analysis as a persistent artifact. "
        "Use this when the user wants to keep research results, "
        "or when you suggest saving a useful analysis."
    )
    arguments_model = SaveArtifactArguments

    async def execute(
        self, context: ToolContext, arguments: SaveArtifactArguments
    ) -> ToolExecution:
        use_case = CreateArtifact(session=context.session)

        result = await use_case.execute(
            CreateArtifactRequest(
                organization_id=str(context.tenant_id),
                artifact_type=ArtifactType(arguments.artifact_type),
                title=arguments.title,
                created_by=str(context.user_id),
                summary=arguments.summary,
                blocks=[
                    {
                        "block_id": "content",
                        "heading": arguments.title,
                        "body": arguments.summary,
                        "order": 0,
                    }
                ],
            )
        )

        return ToolExecution.ok(
            self.name,
            summary=f"Saved artifact: {arguments.title}",
            records=[
                {
                    "artifact_id": result.artifact_id,
                    "version_id": result.version_id,
                    "title": arguments.title,
                }
            ],
        )
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_watch_company_tool.py -v`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/tools/watch_company.py
git add apps/api/src/prescient/intelligence/infrastructure/tools/save_artifact.py
git add apps/api/tests/unit/intelligence/test_watch_company_tool.py
git commit -m "feat(intelligence): add watch_company and save_artifact copilot tools"
```

---

### Task 12: Register New Tools and Add PERSON_PROFILE Artifact Type

**Files:**
- Modify: `apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py`
- Modify: `apps/api/src/prescient/artifacts/domain/entities/artifact.py`

- [ ] **Step 1: Add PERSON_PROFILE to ArtifactType enum**

In `apps/api/src/prescient/artifacts/domain/entities/artifact.py`, add to the `ArtifactType` enum:

```python
PERSON_PROFILE = "person_profile"
```

- [ ] **Step 2: Register new tools in the tool registry**

Read the existing `__init__.py` to understand current registration pattern, then add the new research tools. The tools need to be registered in the function that builds the default registry.

Add imports and registrations for:
- `ResearchCompanyTool`
- `WatchCompanyTool`
- `SaveArtifactTool`

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py
git add apps/api/src/prescient/artifacts/domain/entities/artifact.py
git commit -m "feat: register research tools in copilot and add PERSON_PROFILE artifact type"
```

---

### Task 13: Run All Tests and Final Verification

- [ ] **Step 1: Run all unit tests**

Run: `cd apps/api && uv run pytest tests/unit/ -v`
Expected: All tests pass (both existing and new).

- [ ] **Step 2: Run linting**

Run: `cd apps/api && uv run ruff check src/ tests/`
Expected: No errors.

- [ ] **Step 3: Run type checking**

Run: `cd apps/api && uv run mypy src/prescient/research/`
Expected: No errors.

- [ ] **Step 4: Fix any issues found**

Address any test failures, lint errors, or type errors.

- [ ] **Step 5: Final commit if fixes were needed**

```bash
git add -A
git commit -m "fix: resolve lint/type/test issues from research engine implementation"
```

---

## Summary

| Phase | Tasks | What It Delivers |
|-------|-------|-----------------|
| **Phase 1** | Tasks 1-7 | Research pipeline engine: entities, source plugins, pipeline orchestrator, database, API |
| **Phase 2** | Tasks 8-9 | Smart onboarding: UserContext entity, interview use case with system prompt |
| **Phase 3** | Tasks 10-13 | Copilot tools: research_company, watch_company, save_artifact, tool registration, verification |

**Total tasks:** 13
**Estimated commits:** 13-15

**Not included in this plan (future work):**
- Frontend onboarding interview UI (chat-based onboarding page)
- Frontend standalone research copilot page (`/research`)
- research_market and research_person copilot tools (same pattern as research_company)
- Deep research async worker (background job scheduling)
- Notification system for "research complete" alerts
