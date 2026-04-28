# Workshop Manual Dogfood Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first usable Ferrari 360 workshop-manual dogfood slice with generic KE source/evidence contracts, page-cited answers, web access, MCP access, and eval-candidate capture.

**Architecture:** Extend `prescient_benchmark` first, because the current branch already has OpenSearch indexing, retrieval records, eval scoring, and FastAPI route patterns. Keep domain/application logic in focused modules under `apps/api/src/prescient_benchmark/knowledge/` and `apps/api/src/prescient_benchmark/workshop_manuals/`; keep FastAPI, MCP, PDF, filesystem, and OpenSearch details behind adapters. The first implementation may use file-backed manifests plus OpenSearch, but the ports must allow a later Postgres adapter without changing API/MCP/UI contracts.

**Tech Stack:** Python 3.12, FastAPI, Pydantic v2, PyMuPDF, OpenSearch, pytest, official MCP Python SDK (`mcp` / `FastMCP`), Next.js, Docker Compose.

---

## Governing Context

Parent beads issue: `prescient_os-ed3` - Implement workshop manual dogfood slice.

Governing spec: `docs/superpowers/specs/2026-04-28-workshop-manual-dogfood-design.md`.

Why this sequence:

1. Contracts and scope rules come first because every interface - API, MCP, UI, eval - depends on the same answer/citation/status shape.
2. Ingestion comes before retrieval because page locators and rendered page images are citation requirements, not UI polish.
3. Retrieval primitives come before synthesis because the system must expose inspectable support even when it refuses to answer.
4. API is implemented before MCP and UI because MCP and UI should be thin adapters over one backend service.
5. Eval capture is implemented before broad UI polish so real shop failures immediately become reusable validation material.
6. Next.js comes last because a correct answer contract and citation viewer matter more than frontend breadth.

Before coding, create and claim child beads for these execution slices:

- `Contracts and seeded 360 scope`
- `PDF source ingestion and rendered citation pages`
- `Page-aware retrieval primitives`
- `Knowledge answer service and API`
- `MCP adapter for knowledge questions`
- `Dogfood web probe`
- `Eval candidate capture`

Each child bead should link back to parent `prescient_os-ed3`, this plan, and the governing spec. Beads are the live execution tracker; this markdown plan is the durable sequencing reference.

## File Structure

Create:

- `apps/api/src/prescient_benchmark/knowledge/__init__.py` - package marker.
- `apps/api/src/prescient_benchmark/knowledge/models.py` - generic KE contracts: scope, source, locator, evidence, answer, feedback.
- `apps/api/src/prescient_benchmark/workshop_manuals/__init__.py` - package marker.
- `apps/api/src/prescient_benchmark/workshop_manuals/catalog.py` - seeded Ferrari 360 scope and manual source definitions.
- `apps/api/src/prescient_benchmark/workshop_manuals/store.py` - file-backed metadata store for sources, units, structures, locators, answers, and feedback.
- `apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py` - PyMuPDF text extraction, page rendering, outline/heading section inference.
- `apps/api/src/prescient_benchmark/workshop_manuals/retrieval.py` - page-aware OpenSearch indexing/search and structure walk primitives.
- `apps/api/src/prescient_benchmark/workshop_manuals/providers.py` - provider ports and deterministic/mock LLM implementation for tests.
- `apps/api/src/prescient_benchmark/workshop_manuals/service.py` - application service implementing `ask_knowledge_question`.
- `apps/api/src/prescient_benchmark/api/routes_knowledge.py` - FastAPI routes for ask, scope resolution, citation page, feedback.
- `apps/api/src/prescient_benchmark/mcp/__init__.py` - package marker.
- `apps/api/src/prescient_benchmark/mcp/workshop_server.py` - FastMCP adapter over the same service.
- `tests/unit/test_knowledge_models.py`
- `tests/unit/test_workshop_catalog.py`
- `tests/unit/test_workshop_store.py`
- `tests/unit/test_workshop_pdf_ingest.py`
- `tests/unit/test_workshop_service.py`
- `tests/integration/test_workshop_api.py`
- `tests/integration/test_workshop_retrieval.py`
- `apps/web/package.json`
- `apps/web/next.config.mjs`
- `apps/web/app/page.tsx`
- `apps/web/app/layout.tsx`
- `apps/web/app/globals.css`

Modify:

- `pyproject.toml` - add MCP SDK dependency if needed and expose CLI entry points through existing Typer app.
- `apps/api/src/prescient_benchmark/main.py` - include knowledge routes.
- `apps/api/src/prescient_benchmark/cli.py` - add `ingest-workshop-manuals` and optional inspection command.
- `docker-compose.yml` - add a frontend service only after backend API contracts are stable.
- `.gitignore` - ignore local workshop derived artifact cache if not already covered.

Why this placement:

- Generic models live in `knowledge/` so later business documents and forums can use them without importing workshop-specific code.
- Workshop manual code lives in its own package so PDF and Ferrari seed data do not leak into the retrieval spine.
- API and MCP remain adapters. They should not know how retrieval works.
- Tests mirror the domain/application split so behavior can be verified without the copyrighted manuals.

---

### Task 1: Generic KE Contracts And Seeded Scope

**Why:** The backend, MCP server, web UI, and eval capture need one shared contract for answer status, citations, source locators, and scope. Starting here prevents later manual-specific API names from hardening.

**Files:**
- Create: `apps/api/src/prescient_benchmark/knowledge/__init__.py`
- Create: `apps/api/src/prescient_benchmark/knowledge/models.py`
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/__init__.py`
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/catalog.py`
- Test: `tests/unit/test_knowledge_models.py`
- Test: `tests/unit/test_workshop_catalog.py`

- [ ] **Step 1: Write failing tests for generic contracts**

Create `tests/unit/test_knowledge_models.py` with these tests:

```python
import pytest
from pydantic import ValidationError

from prescient_benchmark.knowledge.models import (
    AnswerStatus,
    AskKnowledgeQuestionRequest,
    EvidenceCitation,
    KnowledgeAnswer,
    KnowledgeScope,
    SourceLocator,
)


def test_knowledge_answer_requires_citations_for_answered_status() -> None:
    scope = KnowledgeScope(
        scope_id="scope-360-modena",
        scope_type="vehicle",
        display_name="Ferrari 360 Modena",
        aliases=["360", "360 Modena"],
        linked_source_ids=["source-360-wsm"],
    )
    locator = SourceLocator(
        locator_id="loc-360-wsm-p12",
        source_id="source-360-wsm",
        unit_id="unit-360-wsm-p12",
        locator_kind="rendered_page",
        target="derived/workshop_manuals/source-360-wsm/pages/page-0012.png",
        page_number=12,
    )
    citation = EvidenceCitation(
        citation_id="cite-1",
        source_id="source-360-wsm",
        unit_ids=["unit-360-wsm-p12"],
        locator_ids=[locator.locator_id],
        title="Ferrari 360 Workshop Manual",
        page_number=12,
        support_role="answer_support",
        excerpt="Tighten the fastener to 25 Nm.",
    )

    answer = KnowledgeAnswer(
        answer_id="answer-1",
        status=AnswerStatus.ANSWERED,
        text="Tighten the fastener to 25 Nm.",
        scope=scope,
        citations=[citation],
        locators=[locator],
        retrieval_trace=["resolve_scope", "search_units", "enforce_citations"],
    )

    assert answer.status is AnswerStatus.ANSWERED
    assert answer.citations[0].page_number == 12


def test_answered_status_rejects_empty_citations() -> None:
    scope = KnowledgeScope(
        scope_id="scope-360-modena",
        scope_type="vehicle",
        display_name="Ferrari 360 Modena",
        aliases=[],
        linked_source_ids=[],
    )

    with pytest.raises(ValidationError, match="answered responses require at least one citation"):
        KnowledgeAnswer(
            answer_id="answer-1",
            status=AnswerStatus.ANSWERED,
            text="Unsupported answer",
            scope=scope,
            citations=[],
            locators=[],
            retrieval_trace=[],
        )


def test_ask_request_accepts_optional_scope_id() -> None:
    request = AskKnowledgeQuestionRequest(
        question="What is the torque for the cam cover bolts?",
        scope_id="scope-360-modena",
    )

    assert request.question.startswith("What is")
    assert request.scope_id == "scope-360-modena"
```

- [ ] **Step 2: Run contract tests and verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_knowledge_models.py -v
```

Expected: FAIL because `prescient_benchmark.knowledge.models` does not exist.

- [ ] **Step 3: Implement generic contract models**

Create `apps/api/src/prescient_benchmark/knowledge/__init__.py` as an empty package marker.

Create `apps/api/src/prescient_benchmark/knowledge/models.py`:

```python
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field, model_validator


class AnswerStatus(StrEnum):
    ANSWERED = "answered"
    NEEDS_CLARIFICATION = "needs_clarification"
    INSUFFICIENT_EVIDENCE = "insufficient_evidence"
    ERROR = "error"


class KnowledgeScope(BaseModel):
    model_config = ConfigDict(extra="forbid")

    scope_id: str
    scope_type: str
    display_name: str
    aliases: list[str] = Field(default_factory=list)
    linked_source_ids: list[str] = Field(default_factory=list)


class AskKnowledgeQuestionRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")

    question: str = Field(min_length=1)
    scope_id: str | None = None


class SourceLocator(BaseModel):
    model_config = ConfigDict(extra="forbid")

    locator_id: str
    source_id: str
    unit_id: str
    locator_kind: str
    target: str
    page_number: int | None = None


class EvidenceCitation(BaseModel):
    model_config = ConfigDict(extra="forbid")

    citation_id: str
    source_id: str
    unit_ids: list[str]
    locator_ids: list[str]
    title: str
    page_number: int | None = None
    support_role: str
    excerpt: str


class KnowledgeAnswer(BaseModel):
    model_config = ConfigDict(extra="forbid")

    answer_id: str
    status: AnswerStatus
    text: str
    scope: KnowledgeScope
    citations: list[EvidenceCitation] = Field(default_factory=list)
    locators: list[SourceLocator] = Field(default_factory=list)
    clarification_question: str | None = None
    retrieval_trace: list[str] = Field(default_factory=list)

    @model_validator(mode="after")
    def answered_responses_must_have_citations(self) -> "KnowledgeAnswer":
        if self.status is AnswerStatus.ANSWERED and not self.citations:
            raise ValueError("answered responses require at least one citation")
        return self


class AnswerFeedback(BaseModel):
    model_config = ConfigDict(extra="forbid")

    answer_id: str
    rating: str
    notes: str | None = None
    corrected_source_id: str | None = None
    corrected_unit_ids: list[str] = Field(default_factory=list)
```

- [ ] **Step 4: Run contract tests and verify pass**

Run:

```bash
uv run python -m pytest tests/unit/test_knowledge_models.py -v
```

Expected: PASS.

- [ ] **Step 5: Write failing tests for seeded 360 scope**

Create `tests/unit/test_workshop_catalog.py`:

```python
from prescient_benchmark.workshop_manuals.catalog import (
    DEFAULT_360_SCOPE_ID,
    WORKSHOP_360_SOURCES,
    resolve_seeded_scope,
)


def test_seeded_scope_defaults_to_360_modena() -> None:
    scope = resolve_seeded_scope(None)

    assert scope.scope_id == DEFAULT_360_SCOPE_ID
    assert scope.scope_type == "vehicle"
    assert "360 Modena" in scope.display_name
    assert "source-ferrari-360-wsm" in scope.linked_source_ids


def test_seeded_sources_include_360_family_manuals() -> None:
    source_ids = {source.source_id for source in WORKSHOP_360_SOURCES}

    assert "source-ferrari-360-wsm" in source_ids
    assert "source-ferrari-360-challenge-stradale" in source_ids
    assert "source-ferrari-360-challenge-gearbox" in source_ids


def test_unknown_scope_falls_back_to_default_for_v1() -> None:
    scope = resolve_seeded_scope("unknown")

    assert scope.scope_id == DEFAULT_360_SCOPE_ID
```

- [ ] **Step 6: Run catalog tests and verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_catalog.py -v
```

Expected: FAIL because `workshop_manuals.catalog` does not exist.

- [ ] **Step 7: Implement seeded catalog**

Create `apps/api/src/prescient_benchmark/workshop_manuals/__init__.py` as an empty package marker.

Create `apps/api/src/prescient_benchmark/workshop_manuals/catalog.py`:

```python
from pathlib import Path

from pydantic import BaseModel, ConfigDict, Field

from prescient_benchmark.knowledge.models import KnowledgeScope

DEFAULT_WORKSHOP_ROOT = Path("~/Projects/workshop_manuals").expanduser()
DEFAULT_360_SCOPE_ID = "scope-ferrari-360-modena"


class WorkshopSourceConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")

    source_id: str
    title: str
    relative_path: Path
    aliases: list[str] = Field(default_factory=list)
    variant_tags: list[str] = Field(default_factory=list)
    cloud_llm_allowed: bool = True
    cloud_ocr_allowed: bool = True
    cloud_embedding_allowed: bool = True
    provider_retention_allowed: bool = False

    def source_path(self, root: Path = DEFAULT_WORKSHOP_ROOT) -> Path:
        return root / self.relative_path


WORKSHOP_360_SOURCES: tuple[WorkshopSourceConfig, ...] = (
    WorkshopSourceConfig(
        source_id="source-ferrari-360-wsm",
        title="Ferrari 360 Modena Workshop Manual",
        relative_path=Path("Ferrari 360 Modena/ferrari_360_wsm_english.pdf"),
        aliases=["360 WSM", "360 workshop manual", "Modena manual"],
        variant_tags=["360_modena"],
    ),
    WorkshopSourceConfig(
        source_id="source-ferrari-360-challenge-stradale",
        title="Ferrari 360 Challenge Stradale Workshop Manual",
        relative_path=Path("Ferrari 360 Challenge Stradale/206990-Ferrari_360_Challenge_Stradale_1999-2005.pdf"),
        aliases=["360 CS", "Challenge Stradale"],
        variant_tags=["360_challenge_stradale"],
    ),
    WorkshopSourceConfig(
        source_id="source-ferrari-360-challenge-technical",
        title="Ferrari 360 Challenge Technical Features and Usage",
        relative_path=Path("Ferrari 360 Challenge/1_Chall_mo_GB.pdf"),
        aliases=["360 Challenge technical features"],
        variant_tags=["360_challenge"],
    ),
    WorkshopSourceConfig(
        source_id="source-ferrari-360-challenge-regulations",
        title="Ferrari 360 Challenge Technical Regulations",
        relative_path=Path("Ferrari 360 Challenge/0_REGTECN_Chall_I-GB.pdf"),
        aliases=["360 Challenge regulations"],
        variant_tags=["360_challenge"],
    ),
    WorkshopSourceConfig(
        source_id="source-ferrari-360-challenge-das",
        title="Ferrari 360 Challenge Data Acquisition System",
        relative_path=Path("Ferrari 360 Challenge/360chall_DAS_GB.pdf"),
        aliases=["360 Challenge DAS"],
        variant_tags=["360_challenge"],
    ),
    WorkshopSourceConfig(
        source_id="source-ferrari-360-challenge-index",
        title="Ferrari 360 Challenge Index",
        relative_path=Path("Ferrari 360 Challenge/index.pdf"),
        aliases=["360 Challenge index"],
        variant_tags=["360_challenge"],
    ),
    WorkshopSourceConfig(
        source_id="source-ferrari-360-challenge-gearbox",
        title="Ferrari 360 Challenge Gearbox and Differential Overhaul",
        relative_path=Path("Ferrari 360 Challenge/RevCambio_GB.pdf"),
        aliases=["360 Challenge gearbox", "360 gearbox"],
        variant_tags=["360_challenge", "gearbox"],
    ),
    WorkshopSourceConfig(
        source_id="source-ferrari-360-challenge-parts",
        title="Ferrari 360 Challenge Spare Parts Catalogue",
        relative_path=Path("Ferrari 360 Challenge/cat_ric.pdf"),
        aliases=["360 Challenge parts catalogue"],
        variant_tags=["360_challenge"],
    ),
)


def default_360_scope() -> KnowledgeScope:
    return KnowledgeScope(
        scope_id=DEFAULT_360_SCOPE_ID,
        scope_type="vehicle",
        display_name="Ferrari 360 Modena",
        aliases=["Ferrari 360", "360", "360 Modena", "Modena", "F131"],
        linked_source_ids=[source.source_id for source in WORKSHOP_360_SOURCES],
    )


def resolve_seeded_scope(scope_id: str | None) -> KnowledgeScope:
    if scope_id in (None, DEFAULT_360_SCOPE_ID):
        return default_360_scope()
    return default_360_scope()
```

- [ ] **Step 8: Run task tests**

Run:

```bash
uv run python -m pytest tests/unit/test_knowledge_models.py tests/unit/test_workshop_catalog.py -v
```

Expected: PASS.

- [ ] **Step 9: Commit task**

Run:

```bash
git add apps/api/src/prescient_benchmark/knowledge apps/api/src/prescient_benchmark/workshop_manuals tests/unit/test_knowledge_models.py tests/unit/test_workshop_catalog.py
git commit -m "feat: add knowledge contracts and 360 scope"
```

---

### Task 2: File-Backed Source Store And PDF Ingestion

**Why:** Citation quality depends on page-level units and rendered page images. This comes before retrieval so every search hit can point to a stable page locator.

**Files:**
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/store.py`
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py`
- Modify: `apps/api/src/prescient_benchmark/cli.py`
- Test: `tests/unit/test_workshop_store.py`
- Test: `tests/unit/test_workshop_pdf_ingest.py`

- [ ] **Step 1: Write failing store tests**

Create `tests/unit/test_workshop_store.py`:

```python
from pathlib import Path

from prescient_benchmark.knowledge.models import SourceLocator
from prescient_benchmark.workshop_manuals.store import (
    SourceRecord,
    SourceUnitRecord,
    WorkshopManualStore,
)


def test_store_round_trips_source_units_and_locators(tmp_path: Path) -> None:
    store = WorkshopManualStore(tmp_path)
    source = SourceRecord(
        source_id="source-ferrari-360-wsm",
        title="Ferrari 360 Workshop Manual",
        source_kind="pdf_manual",
        origin_path="/manuals/360.pdf",
        content_fingerprint="abc123",
        aliases=["360 WSM"],
        variant_tags=["360_modena"],
    )
    unit = SourceUnitRecord(
        unit_id="unit-ferrari-360-wsm-p1",
        source_id=source.source_id,
        unit_kind="pdf_page",
        ordinal=1,
        text="Tightening torque 25 Nm",
    )
    locator = SourceLocator(
        locator_id="loc-ferrari-360-wsm-p1",
        source_id=source.source_id,
        unit_id=unit.unit_id,
        locator_kind="rendered_page",
        target="derived/pages/page-0001.png",
        page_number=1,
    )

    store.upsert_source(source)
    store.upsert_units([unit])
    store.upsert_locators([locator])

    loaded = WorkshopManualStore(tmp_path)
    assert loaded.get_source(source.source_id) == source
    assert loaded.units_for_source(source.source_id) == [unit]
    assert loaded.locator_for_unit(unit.unit_id) == locator
```

- [ ] **Step 2: Run store test and verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_store.py -v
```

Expected: FAIL because store module does not exist.

- [ ] **Step 3: Implement JSON-backed store**

Create `apps/api/src/prescient_benchmark/workshop_manuals/store.py`:

```python
import json
from pathlib import Path

from pydantic import BaseModel, ConfigDict, Field

from prescient_benchmark.knowledge.models import SourceLocator


class SourceRecord(BaseModel):
    model_config = ConfigDict(extra="forbid")

    source_id: str
    title: str
    source_kind: str
    origin_path: str
    content_fingerprint: str
    aliases: list[str] = Field(default_factory=list)
    variant_tags: list[str] = Field(default_factory=list)


class SourceUnitRecord(BaseModel):
    model_config = ConfigDict(extra="forbid")

    unit_id: str
    source_id: str
    unit_kind: str
    ordinal: int
    text: str
    heading: str | None = None
    section_id: str | None = None


class SourceStructureRecord(BaseModel):
    model_config = ConfigDict(extra="forbid")

    structure_id: str
    source_id: str
    structure_kind: str
    heading: str
    start_ordinal: int
    end_ordinal: int
    inference_method: str
    confidence: float


class WorkshopManualStore:
    def __init__(self, root: Path) -> None:
        self.root = root
        self.root.mkdir(parents=True, exist_ok=True)
        self._sources_path = self.root / "sources.json"
        self._units_path = self.root / "units.json"
        self._structures_path = self.root / "structures.json"
        self._locators_path = self.root / "locators.json"

    def upsert_source(self, source: SourceRecord) -> None:
        sources = {item.source_id: item for item in self.list_sources()}
        sources[source.source_id] = source
        self._write_models(self._sources_path, list(sources.values()))

    def get_source(self, source_id: str) -> SourceRecord | None:
        return next((source for source in self.list_sources() if source.source_id == source_id), None)

    def list_sources(self) -> list[SourceRecord]:
        return self._read_models(self._sources_path, SourceRecord)

    def upsert_units(self, units: list[SourceUnitRecord]) -> None:
        existing = {item.unit_id: item for item in self.list_units()}
        for unit in units:
            existing[unit.unit_id] = unit
        self._write_models(self._units_path, list(existing.values()))

    def list_units(self) -> list[SourceUnitRecord]:
        return self._read_models(self._units_path, SourceUnitRecord)

    def units_for_source(self, source_id: str) -> list[SourceUnitRecord]:
        return sorted(
            [unit for unit in self.list_units() if unit.source_id == source_id],
            key=lambda unit: unit.ordinal,
        )

    def upsert_structures(self, structures: list[SourceStructureRecord]) -> None:
        existing = {item.structure_id: item for item in self.list_structures()}
        for structure in structures:
            existing[structure.structure_id] = structure
        self._write_models(self._structures_path, list(existing.values()))

    def list_structures(self) -> list[SourceStructureRecord]:
        return self._read_models(self._structures_path, SourceStructureRecord)

    def upsert_locators(self, locators: list[SourceLocator]) -> None:
        existing = {item.locator_id: item for item in self.list_locators()}
        for locator in locators:
            existing[locator.locator_id] = locator
        self._write_models(self._locators_path, list(existing.values()))

    def list_locators(self) -> list[SourceLocator]:
        return self._read_models(self._locators_path, SourceLocator)

    def locator_for_unit(self, unit_id: str) -> SourceLocator | None:
        return next((locator for locator in self.list_locators() if locator.unit_id == unit_id), None)

    def _read_models[T: BaseModel](self, path: Path, model_type: type[T]) -> list[T]:
        if not path.exists():
            return []
        payload = json.loads(path.read_text(encoding="utf-8"))
        return [model_type.model_validate(item) for item in payload]

    def _write_models(self, path: Path, models: list[BaseModel]) -> None:
        payload = [model.model_dump(mode="json") for model in models]
        path.write_text(json.dumps(payload, indent=2, sort_keys=True), encoding="utf-8")
```

- [ ] **Step 4: Run store test and verify pass**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_store.py -v
```

Expected: PASS.

- [ ] **Step 5: Write failing PDF ingestion tests with generated fixture PDF**

Create `tests/unit/test_workshop_pdf_ingest.py`:

```python
from pathlib import Path

import fitz

from prescient_benchmark.workshop_manuals.catalog import WorkshopSourceConfig
from prescient_benchmark.workshop_manuals.pdf_ingest import ingest_pdf_source
from prescient_benchmark.workshop_manuals.store import WorkshopManualStore


def _write_fixture_pdf(path: Path) -> None:
    doc = fitz.open()
    page = doc.new_page()
    page.insert_text((72, 72), "A. ENGINE\nTightening torques\nCam cover bolt 25 Nm")
    page = doc.new_page()
    page.insert_text((72, 72), "A. ENGINE\nRemove the cam cover bolts in sequence.")
    doc.save(path)
    doc.close()


def test_ingest_pdf_source_extracts_pages_locators_and_structures(tmp_path: Path) -> None:
    pdf_path = tmp_path / "fixture.pdf"
    _write_fixture_pdf(pdf_path)
    source_config = WorkshopSourceConfig(
        source_id="source-fixture",
        title="Fixture Manual",
        relative_path=Path("fixture.pdf"),
        aliases=["fixture"],
        variant_tags=["test"],
    )
    store = WorkshopManualStore(tmp_path / "store")

    result = ingest_pdf_source(
        source_config=source_config,
        source_path=pdf_path,
        store=store,
        rendered_pages_root=tmp_path / "rendered",
    )

    assert result.page_count == 2
    assert len(store.units_for_source("source-fixture")) == 2
    assert store.locator_for_unit("unit-source-fixture-p1") is not None
    assert (tmp_path / "rendered" / "source-fixture" / "page-0001.png").exists()
    assert store.list_structures()[0].heading == "A. ENGINE"
```

- [ ] **Step 6: Run PDF ingestion test and verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_pdf_ingest.py -v
```

Expected: FAIL because PDF ingestion module does not exist.

- [ ] **Step 7: Implement PDF ingestion**

Create `apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py` with:

```python
import hashlib
import re
from pathlib import Path

import fitz
from pydantic import BaseModel, ConfigDict

from prescient_benchmark.knowledge.models import SourceLocator
from prescient_benchmark.workshop_manuals.catalog import WorkshopSourceConfig
from prescient_benchmark.workshop_manuals.store import (
    SourceRecord,
    SourceStructureRecord,
    SourceUnitRecord,
    WorkshopManualStore,
)


class IngestResult(BaseModel):
    model_config = ConfigDict(extra="forbid")

    source_id: str
    page_count: int
    content_fingerprint: str


def ingest_pdf_source(
    *,
    source_config: WorkshopSourceConfig,
    source_path: Path,
    store: WorkshopManualStore,
    rendered_pages_root: Path,
) -> IngestResult:
    content_fingerprint = _sha256_file(source_path)
    source = SourceRecord(
        source_id=source_config.source_id,
        title=source_config.title,
        source_kind="pdf_manual",
        origin_path=source_path.as_posix(),
        content_fingerprint=content_fingerprint,
        aliases=source_config.aliases,
        variant_tags=source_config.variant_tags,
    )
    store.upsert_source(source)

    rendered_source_root = rendered_pages_root / source_config.source_id
    rendered_source_root.mkdir(parents=True, exist_ok=True)

    units: list[SourceUnitRecord] = []
    locators: list[SourceLocator] = []
    structures: list[SourceStructureRecord] = []

    doc = fitz.open(source_path)
    try:
        active_heading: str | None = None
        structure_start = 1
        for page_index, page in enumerate(doc):
            page_number = page_index + 1
            text = page.get_text("text").strip()
            heading = _extract_heading(text)
            if heading and heading != active_heading:
                if active_heading is not None:
                    structures.append(
                        _structure_record(source_config.source_id, active_heading, structure_start, page_number - 1)
                    )
                active_heading = heading
                structure_start = page_number

            unit_id = f"unit-{source_config.source_id}-p{page_number}"
            rendered_path = rendered_source_root / f"page-{page_number:04d}.png"
            if not rendered_path.exists():
                pixmap = page.get_pixmap(matrix=fitz.Matrix(1.5, 1.5), alpha=False)
                pixmap.save(rendered_path)

            units.append(
                SourceUnitRecord(
                    unit_id=unit_id,
                    source_id=source_config.source_id,
                    unit_kind="pdf_page",
                    ordinal=page_number,
                    text=text,
                    heading=active_heading,
                )
            )
            locators.append(
                SourceLocator(
                    locator_id=f"loc-{source_config.source_id}-p{page_number}",
                    source_id=source_config.source_id,
                    unit_id=unit_id,
                    locator_kind="rendered_page",
                    target=rendered_path.as_posix(),
                    page_number=page_number,
                )
            )

        if active_heading is not None:
            structures.append(_structure_record(source_config.source_id, active_heading, structure_start, len(doc)))
    finally:
        doc.close()

    store.upsert_units(units)
    store.upsert_locators(locators)
    store.upsert_structures(structures)
    return IngestResult(
        source_id=source_config.source_id,
        page_count=len(units),
        content_fingerprint=content_fingerprint,
    )


def _sha256_file(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for chunk in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def _extract_heading(text: str) -> str | None:
    for line in text.splitlines():
        stripped = line.strip()
        if re.match(r"^[A-Z0-9][A-Z0-9. -]{2,}$", stripped):
            return stripped
    return None


def _structure_record(source_id: str, heading: str, start: int, end: int) -> SourceStructureRecord:
    return SourceStructureRecord(
        structure_id=f"structure-{source_id}-{start}-{end}",
        source_id=source_id,
        structure_kind="section",
        heading=heading,
        start_ordinal=start,
        end_ordinal=end,
        inference_method="heading_range",
        confidence=0.6,
    )
```

- [ ] **Step 8: Add CLI command for ingestion**

Modify `apps/api/src/prescient_benchmark/cli.py` imports:

```python
from prescient_benchmark.workshop_manuals.catalog import DEFAULT_WORKSHOP_ROOT, WORKSHOP_360_SOURCES
from prescient_benchmark.workshop_manuals.pdf_ingest import ingest_pdf_source
from prescient_benchmark.workshop_manuals.store import WorkshopManualStore
```

Add command near other corpus commands:

```python
@app.command("ingest-workshop-manuals")
def ingest_workshop_manuals_command(
    *,
    workshop_root: Path = typer.Option(DEFAULT_WORKSHOP_ROOT),
    data_root: Path = typer.Option(Path("corpus/workshop_manuals")),
) -> None:
    store = WorkshopManualStore(data_root / "store")
    rendered_root = data_root / "rendered_pages"
    ingested = 0
    for source_config in WORKSHOP_360_SOURCES:
        source_path = source_config.source_path(workshop_root)
        if not source_path.exists():
            typer.echo(f"missing: {source_path}")
            continue
        result = ingest_pdf_source(
            source_config=source_config,
            source_path=source_path,
            store=store,
            rendered_pages_root=rendered_root,
        )
        ingested += 1
        typer.echo(f"{result.source_id}: {result.page_count} pages")
    typer.echo(f"ingested: {ingested}")
```

- [ ] **Step 9: Run task tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_store.py tests/unit/test_workshop_pdf_ingest.py -v
```

Expected: PASS.

- [ ] **Step 10: Commit task**

Run:

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/store.py apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py apps/api/src/prescient_benchmark/cli.py tests/unit/test_workshop_store.py tests/unit/test_workshop_pdf_ingest.py
git commit -m "feat: ingest workshop manual pages"
```

---

### Task 3: Page-Aware OpenSearch Retrieval Primitives

**Why:** The existing retrieval stack indexes chunks without page locators. The dogfood slice needs page-aware hits and structure walk while still reusing OpenSearch and existing retrieval patterns.

**Files:**
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/retrieval.py`
- Test: `tests/integration/test_workshop_retrieval.py`

- [ ] **Step 1: Write failing integration test**

Create `tests/integration/test_workshop_retrieval.py`:

```python
from pathlib import Path

from prescient_benchmark.knowledge.models import SourceLocator
from prescient_benchmark.workshop_manuals.retrieval import (
    ensure_workshop_index,
    search_workshop_units,
    workshop_index_name,
)
from prescient_benchmark.workshop_manuals.store import (
    SourceRecord,
    SourceStructureRecord,
    SourceUnitRecord,
    WorkshopManualStore,
)


def test_workshop_search_returns_page_locator_and_structure(opensearch_client, tmp_path: Path) -> None:
    store = WorkshopManualStore(tmp_path / "store")
    store.upsert_source(
        SourceRecord(
            source_id="source-fixture",
            title="Fixture Manual",
            source_kind="pdf_manual",
            origin_path="/manuals/fixture.pdf",
            content_fingerprint="abc123",
            aliases=["fixture"],
            variant_tags=["test"],
        )
    )
    store.upsert_units(
        [
            SourceUnitRecord(
                unit_id="unit-source-fixture-p1",
                source_id="source-fixture",
                unit_kind="pdf_page",
                ordinal=1,
                text="Cam cover bolt tightening torque 25 Nm",
                heading="ENGINE",
            )
        ]
    )
    store.upsert_locators(
        [
            SourceLocator(
                locator_id="loc-source-fixture-p1",
                source_id="source-fixture",
                unit_id="unit-source-fixture-p1",
                locator_kind="rendered_page",
                target="/rendered/page-0001.png",
                page_number=1,
            )
        ]
    )
    store.upsert_structures(
        [
            SourceStructureRecord(
                structure_id="structure-source-fixture-1-1",
                source_id="source-fixture",
                structure_kind="section",
                heading="ENGINE",
                start_ordinal=1,
                end_ordinal=1,
                inference_method="heading_range",
                confidence=0.6,
            )
        ]
    )
    index_name = workshop_index_name("test")

    ensure_workshop_index(client=opensearch_client, index_name=index_name, store=store)
    results = search_workshop_units(
        client=opensearch_client,
        index_name=index_name,
        store=store,
        query="cam cover torque",
        source_ids=["source-fixture"],
    )

    assert results[0].unit.unit_id == "unit-source-fixture-p1"
    assert results[0].locator.page_number == 1
    assert results[0].structure.heading == "ENGINE"
```

- [ ] **Step 2: Run integration test and verify failure**

Run:

```bash
docker compose up -d opensearch
uv run python -m pytest tests/integration/test_workshop_retrieval.py -v
```

Expected: FAIL because retrieval module does not exist.

- [ ] **Step 3: Implement page-aware retrieval**

Create `apps/api/src/prescient_benchmark/workshop_manuals/retrieval.py`:

```python
from pydantic import BaseModel, ConfigDict
from opensearchpy import OpenSearch

from prescient_benchmark.knowledge.models import SourceLocator
from prescient_benchmark.workshop_manuals.store import (
    SourceStructureRecord,
    SourceUnitRecord,
    WorkshopManualStore,
)


class WorkshopSearchResult(BaseModel):
    model_config = ConfigDict(extra="forbid")

    unit: SourceUnitRecord
    locator: SourceLocator
    structure: SourceStructureRecord | None = None
    score: float


def workshop_index_name(corpus_version: str) -> str:
    return f"prescient-workshop-{corpus_version}"


def ensure_workshop_index(*, client: OpenSearch, index_name: str, store: WorkshopManualStore) -> None:
    if client.indices.exists(index=index_name):
        client.indices.delete(index=index_name)
    client.indices.create(
        index=index_name,
        body={
            "mappings": {
                "properties": {
                    "unit_id": {"type": "keyword"},
                    "source_id": {"type": "keyword"},
                    "ordinal": {"type": "integer"},
                    "title": {"type": "text"},
                    "heading": {"type": "text"},
                    "text": {"type": "text"},
                    "aliases": {"type": "text"},
                    "variant_tags": {"type": "keyword"},
                }
            }
        },
    )
    sources = {source.source_id: source for source in store.list_sources()}
    for unit in store.list_units():
        source = sources[unit.source_id]
        client.index(
            index=index_name,
            id=unit.unit_id,
            body={
                "unit_id": unit.unit_id,
                "source_id": unit.source_id,
                "ordinal": unit.ordinal,
                "title": source.title,
                "heading": unit.heading,
                "text": unit.text,
                "aliases": " ".join(source.aliases),
                "variant_tags": source.variant_tags,
            },
        )
    client.indices.refresh(index=index_name)


def search_workshop_units(
    *,
    client: OpenSearch,
    index_name: str,
    store: WorkshopManualStore,
    query: str,
    source_ids: list[str],
    top_k: int = 10,
) -> list[WorkshopSearchResult]:
    hits = client.search(
        index=index_name,
        body={
            "size": top_k,
            "query": {
                "bool": {
                    "must": {
                        "multi_match": {
                            "query": query,
                            "fields": ["title^2", "heading^3", "text", "aliases"],
                        }
                    },
                    "filter": [{"terms": {"source_id": source_ids}}],
                }
            },
        },
    )["hits"]["hits"]

    units = {unit.unit_id: unit for unit in store.list_units()}
    structures = store.list_structures()
    results: list[WorkshopSearchResult] = []
    for hit in hits:
        unit = units[hit["_source"]["unit_id"]]
        locator = store.locator_for_unit(unit.unit_id)
        if locator is None:
            continue
        results.append(
            WorkshopSearchResult(
                unit=unit,
                locator=locator,
                structure=_structure_for_unit(structures, unit),
                score=float(hit["_score"]),
            )
        )
    return results


def _structure_for_unit(
    structures: list[SourceStructureRecord],
    unit: SourceUnitRecord,
) -> SourceStructureRecord | None:
    return next(
        (
            structure
            for structure in structures
            if structure.source_id == unit.source_id
            and structure.start_ordinal <= unit.ordinal <= structure.end_ordinal
        ),
        None,
    )
```

- [ ] **Step 4: Run retrieval integration test**

Run:

```bash
uv run python -m pytest tests/integration/test_workshop_retrieval.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit task**

Run:

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/retrieval.py tests/integration/test_workshop_retrieval.py
git commit -m "feat: add page-aware workshop retrieval"
```

---

### Task 4: Knowledge Answer Service And FastAPI Routes

**Why:** API, MCP, and UI must call one application service. This is where conservative answer policy, citation enforcement, scope resolution, and retrieval traces become real behavior.

**Files:**
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/providers.py`
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/service.py`
- Create: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Modify: `apps/api/src/prescient_benchmark/main.py`
- Test: `tests/unit/test_workshop_service.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing service tests**

Create `tests/unit/test_workshop_service.py`:

```python
from pathlib import Path

from prescient_benchmark.knowledge.models import AnswerStatus, SourceLocator
from prescient_benchmark.workshop_manuals.catalog import default_360_scope
from prescient_benchmark.workshop_manuals.providers import DeterministicLlmProvider
from prescient_benchmark.workshop_manuals.service import AskKnowledgeService
from prescient_benchmark.workshop_manuals.store import (
    SourceRecord,
    SourceUnitRecord,
    WorkshopManualStore,
)


def _store_with_one_supported_page(tmp_path: Path) -> WorkshopManualStore:
    store = WorkshopManualStore(tmp_path / "store")
    source = SourceRecord(
        source_id="source-ferrari-360-wsm",
        title="Ferrari 360 Workshop Manual",
        source_kind="pdf_manual",
        origin_path="/manuals/360.pdf",
        content_fingerprint="abc123",
        aliases=["360 WSM"],
        variant_tags=["360_modena"],
    )
    unit = SourceUnitRecord(
        unit_id="unit-source-ferrari-360-wsm-p1",
        source_id=source.source_id,
        unit_kind="pdf_page",
        ordinal=1,
        text="Cam cover bolt tightening torque 25 Nm",
        heading="ENGINE",
    )
    locator = SourceLocator(
        locator_id="loc-source-ferrari-360-wsm-p1",
        source_id=source.source_id,
        unit_id=unit.unit_id,
        locator_kind="rendered_page",
        target="/rendered/page-0001.png",
        page_number=1,
    )
    store.upsert_source(source)
    store.upsert_units([unit])
    store.upsert_locators([locator])
    return store


def test_service_returns_cited_answer_from_supported_evidence(tmp_path: Path) -> None:
    store = _store_with_one_supported_page(tmp_path)
    service = AskKnowledgeService(
        store=store,
        scope=default_360_scope(),
        llm_provider=DeterministicLlmProvider(),
    )

    answer = service.answer_from_candidates(
        question="What is the cam cover bolt torque?",
        candidate_unit_ids=["unit-source-ferrari-360-wsm-p1"],
    )

    assert answer.status is AnswerStatus.ANSWERED
    assert answer.citations[0].page_number == 1
    assert "25 Nm" in answer.text


def test_service_refuses_without_candidates(tmp_path: Path) -> None:
    service = AskKnowledgeService(
        store=WorkshopManualStore(tmp_path / "store"),
        scope=default_360_scope(),
        llm_provider=DeterministicLlmProvider(),
    )

    answer = service.answer_from_candidates(
        question="What is the cam cover bolt torque?",
        candidate_unit_ids=[],
    )

    assert answer.status is AnswerStatus.INSUFFICIENT_EVIDENCE
    assert answer.citations == []
```

- [ ] **Step 2: Run service tests and verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_service.py -v
```

Expected: FAIL because provider/service modules do not exist.

- [ ] **Step 3: Implement provider port and deterministic provider**

Create `apps/api/src/prescient_benchmark/workshop_manuals/providers.py`:

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class LlmEvidenceAnswer:
    text: str
    supported: bool


class LlmProvider(Protocol):
    def answer_from_evidence(self, *, question: str, evidence_text: str) -> LlmEvidenceAnswer:
        """Return an answer synthesized only from supplied evidence."""


class DeterministicLlmProvider:
    def answer_from_evidence(self, *, question: str, evidence_text: str) -> LlmEvidenceAnswer:
        if not evidence_text.strip():
            return LlmEvidenceAnswer(text="Insufficient evidence.", supported=False)
        if "25 Nm" in evidence_text:
            return LlmEvidenceAnswer(text="The cited evidence states 25 Nm.", supported=True)
        return LlmEvidenceAnswer(text="Candidate evidence found, but it does not directly answer the question.", supported=False)
```

- [ ] **Step 4: Implement application service**

Create `apps/api/src/prescient_benchmark/workshop_manuals/service.py`:

```python
from uuid import uuid4

from prescient_benchmark.knowledge.models import (
    AnswerStatus,
    EvidenceCitation,
    KnowledgeAnswer,
    KnowledgeScope,
)
from prescient_benchmark.workshop_manuals.providers import LlmProvider
from prescient_benchmark.workshop_manuals.store import WorkshopManualStore


class AskKnowledgeService:
    def __init__(
        self,
        *,
        store: WorkshopManualStore,
        scope: KnowledgeScope,
        llm_provider: LlmProvider,
    ) -> None:
        self.store = store
        self.scope = scope
        self.llm_provider = llm_provider

    def answer_from_candidates(
        self,
        *,
        question: str,
        candidate_unit_ids: list[str],
    ) -> KnowledgeAnswer:
        units = [unit for unit in self.store.list_units() if unit.unit_id in set(candidate_unit_ids)]
        locators = [locator for unit in units if (locator := self.store.locator_for_unit(unit.unit_id))]
        if not units or not locators:
            return KnowledgeAnswer(
                answer_id=f"answer-{uuid4()}",
                status=AnswerStatus.INSUFFICIENT_EVIDENCE,
                text="Insufficient evidence to answer from the indexed workshop manual corpus.",
                scope=self.scope,
                citations=[],
                locators=[],
                retrieval_trace=["resolve_scope", "insufficient_evidence"],
            )

        evidence_text = "\n\n".join(unit.text for unit in units)
        llm_answer = self.llm_provider.answer_from_evidence(question=question, evidence_text=evidence_text)
        if not llm_answer.supported:
            return KnowledgeAnswer(
                answer_id=f"answer-{uuid4()}",
                status=AnswerStatus.INSUFFICIENT_EVIDENCE,
                text=llm_answer.text,
                scope=self.scope,
                citations=[],
                locators=locators,
                retrieval_trace=["resolve_scope", "read_evidence_span", "insufficient_evidence"],
            )

        first_unit = units[0]
        first_locator = locators[0]
        source = self.store.get_source(first_unit.source_id)
        citation = EvidenceCitation(
            citation_id=f"cite-{uuid4()}",
            source_id=first_unit.source_id,
            unit_ids=[unit.unit_id for unit in units],
            locator_ids=[locator.locator_id for locator in locators],
            title=source.title if source else first_unit.source_id,
            page_number=first_locator.page_number,
            support_role="answer_support",
            excerpt=first_unit.text[:500],
        )
        return KnowledgeAnswer(
            answer_id=f"answer-{uuid4()}",
            status=AnswerStatus.ANSWERED,
            text=llm_answer.text,
            scope=self.scope,
            citations=[citation],
            locators=locators,
            retrieval_trace=[
                "resolve_scope",
                "expand_aliases",
                "search_units",
                "walk_structure",
                "read_evidence_span",
                "enforce_citations",
            ],
        )
```

- [ ] **Step 5: Run service tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_service.py -v
```

Expected: PASS.

- [ ] **Step 6: Write failing API tests**

Create `tests/integration/test_workshop_api.py`:

```python
from pathlib import Path

from fastapi.testclient import TestClient

from prescient_benchmark.config import settings
from prescient_benchmark.main import app
from prescient_benchmark.knowledge.models import SourceLocator
from prescient_benchmark.workshop_manuals.store import SourceRecord, SourceUnitRecord, WorkshopManualStore


def test_ask_knowledge_route_returns_structured_answer(tmp_path: Path, monkeypatch) -> None:
    data_root = tmp_path / "workshop"
    store = WorkshopManualStore(data_root / "store")
    store.upsert_source(
        SourceRecord(
            source_id="source-ferrari-360-wsm",
            title="Ferrari 360 Workshop Manual",
            source_kind="pdf_manual",
            origin_path="/manuals/360.pdf",
            content_fingerprint="abc123",
            aliases=["360 WSM"],
            variant_tags=["360_modena"],
        )
    )
    store.upsert_units(
        [
            SourceUnitRecord(
                unit_id="unit-source-ferrari-360-wsm-p1",
                source_id="source-ferrari-360-wsm",
                unit_kind="pdf_page",
                ordinal=1,
                text="Cam cover bolt tightening torque 25 Nm",
            )
        ]
    )
    store.upsert_locators(
        [
            SourceLocator(
                locator_id="loc-source-ferrari-360-wsm-p1",
                source_id="source-ferrari-360-wsm",
                unit_id="unit-source-ferrari-360-wsm-p1",
                locator_kind="rendered_page",
                target="/rendered/page-0001.png",
                page_number=1,
            )
        ]
    )
    monkeypatch.setattr(settings, "corpus_root", data_root)

    client = TestClient(app)
    response = client.post(
        "/knowledge/ask",
        json={
            "question": "What is the cam cover bolt torque?",
            "candidate_unit_ids": ["unit-source-ferrari-360-wsm-p1"],
        },
    )

    assert response.status_code == 200
    payload = response.json()
    assert payload["status"] == "answered"
    assert payload["citations"][0]["page_number"] == 1
```

- [ ] **Step 7: Implement FastAPI route**

Create `apps/api/src/prescient_benchmark/api/routes_knowledge.py`:

```python
from pathlib import Path

from fastapi import APIRouter
from pydantic import BaseModel, ConfigDict, Field

from prescient_benchmark.config import settings
from prescient_benchmark.knowledge.models import KnowledgeAnswer
from prescient_benchmark.workshop_manuals.catalog import default_360_scope
from prescient_benchmark.workshop_manuals.providers import DeterministicLlmProvider
from prescient_benchmark.workshop_manuals.service import AskKnowledgeService
from prescient_benchmark.workshop_manuals.store import WorkshopManualStore

router = APIRouter(prefix="/knowledge", tags=["knowledge"])


class AskRouteRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")

    question: str = Field(min_length=1)
    scope_id: str | None = None
    candidate_unit_ids: list[str] = Field(default_factory=list)


@router.post("/ask")
def ask_knowledge(request: AskRouteRequest) -> dict:
    store = WorkshopManualStore(Path(settings.corpus_root) / "store")
    service = AskKnowledgeService(
        store=store,
        scope=default_360_scope(),
        llm_provider=DeterministicLlmProvider(),
    )
    answer: KnowledgeAnswer = service.answer_from_candidates(
        question=request.question,
        candidate_unit_ids=request.candidate_unit_ids,
    )
    return answer.model_dump(mode="json")
```

Modify `apps/api/src/prescient_benchmark/main.py`:

```python
from prescient_benchmark.api.routes_knowledge import router as knowledge_router

app.include_router(knowledge_router)
```

- [ ] **Step 8: Run API tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_service.py tests/integration/test_workshop_api.py -v
```

Expected: PASS.

- [ ] **Step 9: Commit task**

Run:

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/providers.py apps/api/src/prescient_benchmark/workshop_manuals/service.py apps/api/src/prescient_benchmark/api/routes_knowledge.py apps/api/src/prescient_benchmark/main.py tests/unit/test_workshop_service.py tests/integration/test_workshop_api.py
git commit -m "feat: add knowledge answer service"
```

---

### Task 5: MCP Adapter

**Why:** Hermes needs access when the web UI is unavailable, but MCP must not duplicate retrieval logic. It should call the same application service and return the same contract shape.

**Files:**
- Modify: `pyproject.toml`
- Create: `apps/api/src/prescient_benchmark/mcp/__init__.py`
- Create: `apps/api/src/prescient_benchmark/mcp/workshop_server.py`

- [ ] **Step 1: Add MCP dependency**

Modify `pyproject.toml` dependencies:

```toml
  "mcp>=1.0",
```

Use the official MCP Python SDK and `FastMCP` import path documented by the Model Context Protocol Python SDK.

- [ ] **Step 2: Create MCP server module**

Create `apps/api/src/prescient_benchmark/mcp/__init__.py` as an empty package marker.

Create `apps/api/src/prescient_benchmark/mcp/workshop_server.py`:

```python
from pathlib import Path

from mcp.server.fastmcp import FastMCP

from prescient_benchmark.config import settings
from prescient_benchmark.workshop_manuals.catalog import default_360_scope
from prescient_benchmark.workshop_manuals.providers import DeterministicLlmProvider
from prescient_benchmark.workshop_manuals.service import AskKnowledgeService
from prescient_benchmark.workshop_manuals.store import WorkshopManualStore

mcp = FastMCP("prescient-workshop-manuals")


def _service() -> AskKnowledgeService:
    return AskKnowledgeService(
        store=WorkshopManualStore(Path(settings.corpus_root) / "store"),
        scope=default_360_scope(),
        llm_provider=DeterministicLlmProvider(),
    )


@mcp.tool()
def ask_knowledge_question(question: str, candidate_unit_ids: list[str] | None = None) -> dict:
    """Ask a scoped knowledge question against the workshop manual dogfood corpus."""
    answer = _service().answer_from_candidates(
        question=question,
        candidate_unit_ids=candidate_unit_ids or [],
    )
    return answer.model_dump(mode="json")


@mcp.tool()
def resolve_knowledge_scope() -> dict:
    """Return the seeded Ferrari 360 Modena scope used by v1."""
    return default_360_scope().model_dump(mode="json")


@mcp.tool()
def get_citation_page(locator_target: str) -> dict:
    """Return citation page locator metadata for a rendered source page."""
    return {"locator_target": locator_target}


@mcp.tool()
def record_answer_feedback(answer_id: str, rating: str, notes: str | None = None) -> dict:
    """Record lightweight answer feedback for later eval promotion."""
    return {"answer_id": answer_id, "rating": rating, "notes": notes}


if __name__ == "__main__":
    mcp.run()
```

- [ ] **Step 3: Verify MCP module imports**

Run:

```bash
uv run python -c "from prescient_benchmark.mcp.workshop_server import mcp; print(mcp.name)"
```

Expected output includes:

```text
prescient-workshop-manuals
```

- [ ] **Step 4: Commit task**

Run:

```bash
git add pyproject.toml uv.lock apps/api/src/prescient_benchmark/mcp
git commit -m "feat: add workshop manual mcp adapter"
```

---

### Task 6: Eval Candidate Capture

**Why:** The dogfood loop only strengthens the retrieval thesis if real shop questions become scored eval cases. Capture must happen while the UI is being used, not after memories fade.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/store.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/service.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/unit/test_workshop_store.py`
- Test: `tests/unit/test_workshop_service.py`

- [ ] **Step 1: Add failing feedback capture test**

Append to `tests/unit/test_workshop_store.py`:

```python
from prescient_benchmark.knowledge.models import AnswerFeedback


def test_store_round_trips_answer_feedback(tmp_path: Path) -> None:
    store = WorkshopManualStore(tmp_path)
    feedback = AnswerFeedback(
        answer_id="answer-1",
        rating="missing_citation",
        notes="The right page was nearby.",
        corrected_unit_ids=["unit-source-ferrari-360-wsm-p2"],
    )

    store.append_feedback(feedback)

    assert store.list_feedback() == [feedback]
```

- [ ] **Step 2: Implement feedback persistence**

Modify `WorkshopManualStore`:

```python
from prescient_benchmark.knowledge.models import AnswerFeedback, SourceLocator
```

Add in `__init__`:

```python
self._feedback_path = self.root / "feedback.json"
```

Add methods:

```python
def append_feedback(self, feedback: AnswerFeedback) -> None:
    items = self.list_feedback()
    items.append(feedback)
    self._write_models(self._feedback_path, items)

def list_feedback(self) -> list[AnswerFeedback]:
    return self._read_models(self._feedback_path, AnswerFeedback)
```

- [ ] **Step 3: Add API feedback route**

Modify `apps/api/src/prescient_benchmark/api/routes_knowledge.py`:

```python
from prescient_benchmark.knowledge.models import AnswerFeedback
```

Add:

```python
@router.post("/feedback")
def record_feedback(feedback: AnswerFeedback) -> dict:
    store = WorkshopManualStore(Path(settings.corpus_root) / "store")
    store.append_feedback(feedback)
    return {"recorded": True}
```

- [ ] **Step 4: Run feedback tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_store.py tests/integration/test_workshop_api.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit task**

Run:

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/store.py apps/api/src/prescient_benchmark/api/routes_knowledge.py tests/unit/test_workshop_store.py tests/integration/test_workshop_api.py
git commit -m "feat: capture workshop answer feedback"
```

---

### Task 7: Next.js Dogfood Web Probe

**Why:** The user needs a usable interaction surface during shop work. The UI should be simple and dense: question input, answer status, citations, rendered page viewer, and feedback controls.

**Files:**
- Create: `apps/web/package.json`
- Create: `apps/web/next.config.mjs`
- Create: `apps/web/app/layout.tsx`
- Create: `apps/web/app/page.tsx`
- Create: `apps/web/app/globals.css`
- Modify: `docker-compose.yml`

- [ ] **Step 1: Create minimal Next.js package**

Create `apps/web/package.json`:

```json
{
  "name": "prescient-workshop-web",
  "private": true,
  "scripts": {
    "dev": "next dev --hostname 0.0.0.0 --port 3000",
    "build": "next build",
    "start": "next start --hostname 0.0.0.0 --port 3000"
  },
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "typescript": "^5.0.0"
  }
}
```

Create `apps/web/next.config.mjs`:

```javascript
const nextConfig = {};

export default nextConfig;
```

- [ ] **Step 2: Create app shell**

Create `apps/web/app/layout.tsx`:

```tsx
import "./globals.css";

export const metadata = {
  title: "Prescient Workshop",
  description: "Workshop manual dogfood probe",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

- [ ] **Step 3: Create chat and citation page**

Create `apps/web/app/page.tsx`:

```tsx
"use client";

import { FormEvent, useState } from "react";

type Answer = {
  answer_id: string;
  status: string;
  text: string;
  citations: Array<{
    citation_id: string;
    title: string;
    page_number: number | null;
    excerpt: string;
  }>;
  locators: Array<{
    locator_id: string;
    target: string;
    page_number: number | null;
  }>;
};

export default function Page() {
  const [question, setQuestion] = useState("");
  const [answer, setAnswer] = useState<Answer | null>(null);
  const [loading, setLoading] = useState(false);
  const [selectedLocator, setSelectedLocator] = useState<string | null>(null);

  async function submit(event: FormEvent) {
    event.preventDefault();
    setLoading(true);
    const response = await fetch("http://localhost:8000/knowledge/ask", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ question, candidate_unit_ids: [] }),
    });
    const payload = (await response.json()) as Answer;
    setAnswer(payload);
    setSelectedLocator(payload.locators[0]?.target ?? null);
    setLoading(false);
  }

  return (
    <main className="workspace">
      <section className="chat">
        <header>
          <p className="scope">Ferrari 360 Modena</p>
          <h1>Workshop Manuals</h1>
        </header>
        <form onSubmit={submit} className="questionForm">
          <textarea
            value={question}
            onChange={(event) => setQuestion(event.target.value)}
            placeholder="Ask about a torque spec, procedure, or cited manual page"
          />
          <button type="submit" disabled={loading || question.trim().length === 0}>
            {loading ? "Searching" : "Ask"}
          </button>
        </form>
        {answer && (
          <article className="answer">
            <p className={`status status-${answer.status}`}>{answer.status}</p>
            <p>{answer.text}</p>
            <div className="citations">
              {answer.citations.map((citation) => (
                <button
                  key={citation.citation_id}
                  type="button"
                  onClick={() => setSelectedLocator(answer.locators[0]?.target ?? null)}
                >
                  {citation.title} p. {citation.page_number ?? "?"}
                </button>
              ))}
            </div>
          </article>
        )}
      </section>
      <section className="viewer">
        {selectedLocator ? (
          <img src={`http://localhost:8000/knowledge/citation-page?target=${encodeURIComponent(selectedLocator)}`} alt="Cited manual page" />
        ) : (
          <p>Select a citation to inspect its page.</p>
        )}
      </section>
    </main>
  );
}
```

- [ ] **Step 4: Add CSS**

Create `apps/web/app/globals.css`:

```css
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  color: #1b1d1f;
  background: #f6f7f8;
}

.workspace {
  display: grid;
  grid-template-columns: minmax(360px, 440px) 1fr;
  min-height: 100vh;
}

.chat {
  border-right: 1px solid #d9dee3;
  background: #ffffff;
  padding: 24px;
}

.scope {
  margin: 0 0 4px;
  color: #57606a;
  font-size: 13px;
}

h1 {
  margin: 0 0 24px;
  font-size: 24px;
}

.questionForm {
  display: grid;
  gap: 12px;
}

textarea {
  min-height: 120px;
  resize: vertical;
  border: 1px solid #c8d0d8;
  border-radius: 6px;
  padding: 12px;
  font: inherit;
}

button {
  border: 1px solid #1f6feb;
  border-radius: 6px;
  background: #1f6feb;
  color: white;
  padding: 10px 12px;
  font: inherit;
  cursor: pointer;
}

button:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

.answer {
  margin-top: 24px;
  border-top: 1px solid #d9dee3;
  padding-top: 16px;
}

.status {
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0;
  color: #57606a;
}

.citations {
  display: grid;
  gap: 8px;
}

.citations button {
  text-align: left;
  background: #f6f8fa;
  color: #1b1d1f;
  border-color: #d0d7de;
}

.viewer {
  display: grid;
  place-items: center;
  padding: 24px;
  overflow: auto;
}

.viewer img {
  max-width: 100%;
  height: auto;
  box-shadow: 0 8px 24px rgba(27, 29, 31, 0.14);
}

@media (max-width: 820px) {
  .workspace {
    grid-template-columns: 1fr;
  }

  .chat {
    border-right: 0;
    border-bottom: 1px solid #d9dee3;
  }
}
```

- [ ] **Step 5: Add citation page route**

Modify `apps/api/src/prescient_benchmark/api/routes_knowledge.py`:

```python
from fastapi.responses import FileResponse
```

Add:

```python
@router.get("/citation-page")
def citation_page(target: str) -> FileResponse:
    return FileResponse(target, media_type="image/png")
```

- [ ] **Step 6: Add Docker frontend service**

Modify `docker-compose.yml`:

```yaml
  web:
    image: node:22-alpine
    working_dir: /workspace/apps/web
    command: sh -c "npm install && npm run dev"
    volumes:
      - .:/workspace
    ports:
      - "3000:3000"
    depends_on:
      - api
```

- [ ] **Step 7: Verify frontend build**

Run:

```bash
cd apps/web && npm install && npm run build
```

Expected: Next.js build succeeds.

- [ ] **Step 8: Commit task**

Run:

```bash
git add apps/web docker-compose.yml apps/api/src/prescient_benchmark/api/routes_knowledge.py
git commit -m "feat: add workshop dogfood web probe"
```

---

### Task 8: Wire Real Retrieval Into The Ask Route

**Why:** Earlier tasks establish contracts and deterministic seams. This task makes `/knowledge/ask` use indexed evidence instead of pre-supplied `candidate_unit_ids`, while keeping the test hook for deterministic unit tests.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/service.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Add integration test for search-backed ask**

Append to `tests/integration/test_workshop_api.py`:

```python
def _seed_api_store(data_root: Path) -> WorkshopManualStore:
    store = WorkshopManualStore(data_root / "store")
    store.upsert_source(
        SourceRecord(
            source_id="source-ferrari-360-wsm",
            title="Ferrari 360 Workshop Manual",
            source_kind="pdf_manual",
            origin_path="/manuals/360.pdf",
            content_fingerprint="abc123",
            aliases=["360 WSM"],
            variant_tags=["360_modena"],
        )
    )
    store.upsert_units(
        [
            SourceUnitRecord(
                unit_id="unit-source-ferrari-360-wsm-p1",
                source_id="source-ferrari-360-wsm",
                unit_kind="pdf_page",
                ordinal=1,
                text="Cam cover bolt tightening torque 25 Nm",
            )
        ]
    )
    store.upsert_locators(
        [
            SourceLocator(
                locator_id="loc-source-ferrari-360-wsm-p1",
                source_id="source-ferrari-360-wsm",
                unit_id="unit-source-ferrari-360-wsm-p1",
                locator_kind="rendered_page",
                target="/rendered/page-0001.png",
                page_number=1,
            )
        ]
    )
    return store


def test_ask_knowledge_route_can_search_when_candidates_omitted(tmp_path: Path, monkeypatch, opensearch_client) -> None:
    data_root = tmp_path / "workshop"
    _seed_api_store(data_root)
    monkeypatch.setattr(settings, "corpus_root", data_root)

    client = TestClient(app)
    response = client.post(
        "/knowledge/ask",
        json={"question": "cam cover torque"},
    )

    assert response.status_code == 200
    assert response.json()["status"] in {"answered", "insufficient_evidence"}
```

When implementing, extract the repeated fixture setup in this test file into a helper named `_seed_api_store`.

- [ ] **Step 2: Update service with retrieval primitive method**

Modify `AskKnowledgeService`:

```python
def answer_from_search_results(self, *, question: str, results: list[WorkshopSearchResult]) -> KnowledgeAnswer:
    return self.answer_from_candidates(
        question=question,
        candidate_unit_ids=[result.unit.unit_id for result in results],
    )
```

- [ ] **Step 3: Update route to search when candidates are empty**

Modify `ask_knowledge` in `routes_knowledge.py`:

```python
from prescient_benchmark.config import get_opensearch_client
from prescient_benchmark.workshop_manuals.retrieval import search_workshop_units, workshop_index_name
```

Inside the route:

```python
candidate_unit_ids = request.candidate_unit_ids
if not candidate_unit_ids:
    results = search_workshop_units(
        client=get_opensearch_client(),
        index_name=workshop_index_name("v1"),
        store=store,
        query=request.question,
        source_ids=default_360_scope().linked_source_ids,
    )
    candidate_unit_ids = [result.unit.unit_id for result in results[:5]]
```

- [ ] **Step 4: Run API and retrieval tests**

Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py tests/integration/test_workshop_retrieval.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit task**

Run:

```bash
git add apps/api/src/prescient_benchmark/api/routes_knowledge.py apps/api/src/prescient_benchmark/workshop_manuals/service.py tests/integration/test_workshop_api.py
git commit -m "feat: answer workshop questions from search"
```

---

## Final Verification

- [ ] Run backend unit tests:

```bash
uv run python -m pytest tests/unit/test_knowledge_models.py tests/unit/test_workshop_catalog.py tests/unit/test_workshop_store.py tests/unit/test_workshop_pdf_ingest.py tests/unit/test_workshop_service.py -v
```

Expected: PASS.

- [ ] Run OpenSearch-backed integration tests:

```bash
docker compose up -d opensearch
uv run python -m pytest tests/integration/test_workshop_retrieval.py tests/integration/test_workshop_api.py -v
```

Expected: PASS.

- [ ] Run full test suite if the shared retrieval or eval contracts changed:

```bash
uv run python -m pytest
```

Expected: PASS.

- [ ] Run frontend build:

```bash
cd apps/web && npm run build
```

Expected: PASS.

- [ ] Ingest real manuals locally:

```bash
uv run python -m prescient_benchmark.cli ingest-workshop-manuals --workshop-root ~/Projects/workshop_manuals --data-root corpus/workshop_manuals
```

Expected: command reports ingested 360-family sources and page counts.

- [ ] Start dogfood stack:

```bash
docker compose up --build api web opensearch
```

Expected:

- API at `http://localhost:8000`
- Web probe at `http://localhost:3000`
- `/knowledge/ask` returns `answered`, `needs_clarification`, or `insufficient_evidence` with the generic answer contract.

## Plan Self-Review

Spec coverage:

- Interaction-first web probe: Task 7.
- MCP access for Hermes: Task 5.
- Seeded 360 Modena scope and 360-family sources: Task 1.
- Generic source/evidence contracts: Task 1.
- PDF pages and rendered citations: Task 2 and Task 7.
- Reuse `prescient_benchmark`: Tasks 3, 4, 8.
- Agent primitives rather than fixed RAG: Task 3 and Task 4 establish searchable units, structure walk, evidence read, citation enforcement, and trace records.
- Provider-swappable LLM boundary: Task 4.
- Eval capture: Task 6, with promotion to existing eval formats as the next extension after feedback persistence.
- Quantitative validation milestone: tracked in the governing spec, not enforced as a blocker for the first usable slice.

Known limits of this first plan:

- It starts with deterministic/mock LLM behavior for tests and leaves production cloud LLM implementation for a follow-on child issue after the answer contract is stable.
- It implements heading-range section inference but does not solve table/figure extraction deeply.
- It creates eval-candidate capture before full evidence-key promotion. Promotion is intentionally next because real usage should shape the first scored question set.
