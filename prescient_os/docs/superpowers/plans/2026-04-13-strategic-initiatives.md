# Strategic Initiatives & Unified Notifications — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a strategic initiative management system with proactive agent follow-ups, dependency/resource tracking, timeline visualization, a unified notifications system, and copilot + briefing integration.

**Architecture:** Two new bounded contexts (`strategy/` and `notifications/`) following the existing hexagonal pattern: frozen Pydantic domain models, Protocol-based ports, SQLAlchemy ORM tables, FastAPI routes as composition roots. Six vertical slices — each delivers a usable, demoable feature. The frontend is a Next.js page at `/strategy` with a Gantt-style timeline chart.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy (async), Alembic, Pydantic v2, PostgreSQL, Next.js 14, React, Tailwind CSS, gantt-task-react (or similar Gantt library)

**Spec:** `docs/superpowers/specs/2026-04-13-strategic-initiatives-design.md`

---

## File Structure

### Backend — Strategy Context

```
apps/api/src/prescient/strategy/
├── __init__.py
├── domain/
│   ├── __init__.py
│   ├── entities/
│   │   ├── __init__.py
│   │   ├── initiative.py          — Initiative frozen Pydantic model
│   │   ├── milestone.py           — Milestone frozen Pydantic model
│   │   ├── dependency.py          — Dependency frozen Pydantic model
│   │   ├── resource_assignment.py — ResourceAssignment frozen Pydantic model
│   │   ├── evidence.py            — Evidence frozen Pydantic model
│   │   ├── initiative_subject.py  — Polymorphic subject link
│   │   └── approval_request.py    — ApprovalRequest frozen Pydantic model
│   ├── enums.py                   — All strategy StrEnums
│   └── health.py                  — Health score computation (pure function)
├── application/
│   ├── __init__.py
│   ├── use_cases/
│   │   ├── __init__.py
│   │   ├── create_initiative.py
│   │   ├── update_initiative.py
│   │   ├── list_initiatives.py
│   │   ├── get_initiative.py
│   │   ├── create_milestone.py
│   │   ├── update_milestone.py
│   │   ├── create_dependency.py
│   │   ├── create_resource.py
│   │   ├── create_evidence.py
│   │   ├── extract_initiatives.py
│   │   └── resolve_approval.py
│   └── cadence_engine.py          — Scheduled process for staleness/risk detection
├── infrastructure/
│   ├── __init__.py
│   ├── tables/
│   │   ├── __init__.py
│   │   ├── initiative.py
│   │   ├── milestone.py
│   │   ├── dependency.py
│   │   ├── resource_assignment.py
│   │   ├── evidence.py
│   │   ├── initiative_subject.py
│   │   └── approval_request.py
│   └── extraction_adapter.py      — LLM-based initiative extraction
├── api/
│   ├── __init__.py
│   └── routes.py
└── queries.py                     — Published query functions for cross-context reads
```

### Backend — Notifications Context

```
apps/api/src/prescient/notifications/
├── __init__.py
├── domain/
│   ├── __init__.py
│   ├── entities/
│   │   ├── __init__.py
│   │   └── notification.py
│   └── enums.py
├── application/
│   ├── __init__.py
│   └── use_cases/
│       ├── __init__.py
│       ├── publish_notification.py
│       └── mark_read.py
├── infrastructure/
│   ├── __init__.py
│   └── tables/
│       ├── __init__.py
│       └── notification.py
├── api/
│   ├── __init__.py
│   └── routes.py
└── queries.py
```

### Frontend

```
apps/web/src/app/(main)/strategy/
├── page.tsx                       — Main strategy page with timeline
└── components/
    ├── timeline-chart.tsx         — Gantt chart wrapper
    ├── initiative-detail.tsx      — Right-panel drill-down
    ├── summary-bar.tsx            — Status counts + upcoming milestones
    ├── initiative-form.tsx        — Create/edit initiative modal
    ├── milestone-list.tsx         — Inline milestone editor
    ├── extraction-review.tsx      — Document extraction review flow
    └── dependency-overlay.tsx     — SVG dependency arrows

apps/web/src/components/
├── notifications-bell.tsx         — Header bell icon + dropdown
└── notifications-panel.tsx        — Notification list panel
```

### Migrations

```
apps/api/alembic/versions/
├── 20260413_strategy_schema.py
└── 20260413_notifications_schema.py
```

### Tests

```
apps/api/tests/unit/strategy/
├── __init__.py
├── test_enums.py
├── test_health_score.py
├── test_create_initiative.py
├── test_update_initiative.py
├── test_list_initiatives.py
├── test_create_milestone.py
├── test_update_milestone.py
├── test_create_dependency.py
├── test_create_resource.py
├── test_create_evidence.py
├── test_extract_initiatives.py
├── test_resolve_approval.py
└── test_cadence_engine.py

apps/api/tests/unit/notifications/
├── __init__.py
├── test_publish_notification.py
└── test_mark_read.py

apps/api/tests/unit/briefing/
└── test_strategy_source.py        — Tests for get_strategy_items on source adapter
```

---

## Slice 1: Initiative + Milestone Foundation

### Task 1: Strategy Domain Enums

**Files:**
- Create: `apps/api/src/prescient/strategy/__init__.py`
- Create: `apps/api/src/prescient/strategy/domain/__init__.py`
- Create: `apps/api/src/prescient/strategy/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/strategy/domain/enums.py`
- Test: `apps/api/tests/unit/strategy/__init__.py`
- Test: `apps/api/tests/unit/strategy/test_enums.py`

- [ ] **Step 1: Create package structure**

Create empty `__init__.py` files:

```python
# apps/api/src/prescient/strategy/__init__.py
# apps/api/src/prescient/strategy/domain/__init__.py
# apps/api/src/prescient/strategy/domain/entities/__init__.py
# apps/api/tests/unit/strategy/__init__.py
```

- [ ] **Step 2: Write failing test for enums**

```python
# apps/api/tests/unit/strategy/test_enums.py
"""Unit tests for strategy domain enums."""

from __future__ import annotations

from prescient.strategy.domain.enums import (
    InitiativePriority,
    InitiativeStatus,
    MilestoneStatus,
    TimeHorizon,
)


class TestInitiativeStatus:
    def test_draft_value(self) -> None:
        assert InitiativeStatus.DRAFT == "draft"

    def test_all_statuses_exist(self) -> None:
        expected = {"draft", "active", "at_risk", "on_hold", "completed", "cancelled"}
        actual = {s.value for s in InitiativeStatus}
        assert actual == expected


class TestInitiativePriority:
    def test_all_priorities_exist(self) -> None:
        expected = {"critical", "high", "medium", "low"}
        actual = {p.value for p in InitiativePriority}
        assert actual == expected


class TestMilestoneStatus:
    def test_all_statuses_exist(self) -> None:
        expected = {"pending", "in_progress", "completed", "missed", "deferred"}
        actual = {s.value for s in MilestoneStatus}
        assert actual == expected


class TestTimeHorizon:
    def test_all_horizons_exist(self) -> None:
        expected = {"q1", "q2", "q3", "q4", "annual", "multi_year"}
        actual = {h.value for h in TimeHorizon}
        assert actual == expected
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_enums.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'prescient.strategy.domain.enums'`

- [ ] **Step 4: Implement enums**

```python
# apps/api/src/prescient/strategy/domain/enums.py
"""Strategy context enums."""

from __future__ import annotations

from enum import StrEnum


class InitiativeStatus(StrEnum):
    DRAFT = "draft"
    ACTIVE = "active"
    AT_RISK = "at_risk"
    ON_HOLD = "on_hold"
    COMPLETED = "completed"
    CANCELLED = "cancelled"


class InitiativePriority(StrEnum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"


class TimeHorizon(StrEnum):
    Q1 = "q1"
    Q2 = "q2"
    Q3 = "q3"
    Q4 = "q4"
    ANNUAL = "annual"
    MULTI_YEAR = "multi_year"


class MilestoneStatus(StrEnum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    MISSED = "missed"
    DEFERRED = "deferred"


class DependencyType(StrEnum):
    BLOCKS = "blocks"
    INFORMS = "informs"
    SHARES_RESOURCES = "shares_resources"


class ResourceType(StrEnum):
    PERSON = "person"
    TEAM = "team"
    BUDGET = "budget"


class EvidenceType(StrEnum):
    KPI = "kpi"
    ACTION_ITEM = "action_item"
    ARTIFACT = "artifact"
    OBSERVATION = "observation"


class ApprovalRequestType(StrEnum):
    DATE_CHANGE = "date_change"
    STATUS_CHANGE = "status_change"
    SCOPE_CHANGE = "scope_change"


class ApprovalStatus(StrEnum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_enums.py -v`
Expected: PASS (all 4 tests)

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/strategy/ apps/api/tests/unit/strategy/
git commit -m "feat(strategy): add domain enums for initiatives, milestones, dependencies, resources"
```

---

### Task 2: Initiative and Milestone Domain Entities

**Files:**
- Create: `apps/api/src/prescient/strategy/domain/entities/initiative.py`
- Create: `apps/api/src/prescient/strategy/domain/entities/milestone.py`
- Create: `apps/api/src/prescient/strategy/domain/entities/initiative_subject.py`
- Modify: `apps/api/src/prescient/shared/types.py` — add `InitiativeId`, `MilestoneId`

- [ ] **Step 1: Add shared types**

Add to `apps/api/src/prescient/shared/types.py` after the `ResearchTaskId` line:

```python
InitiativeId = NewType("InitiativeId", UUID)
MilestoneId = NewType("MilestoneId", UUID)
NotificationId = NewType("NotificationId", UUID)
ApprovalRequestId = NewType("ApprovalRequestId", UUID)
```

- [ ] **Step 2: Create Initiative entity**

```python
# apps/api/src/prescient/strategy/domain/entities/initiative.py
"""Initiative domain entity."""

from __future__ import annotations

from datetime import date, datetime
from decimal import Decimal
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field

from prescient.strategy.domain.entities.initiative_subject import InitiativeSubject
from prescient.strategy.domain.entities.milestone import Milestone
from prescient.strategy.domain.enums import InitiativePriority, InitiativeStatus, TimeHorizon
from prescient.shared.types import FundId


class Initiative(BaseModel):
    model_config = ConfigDict(frozen=True, strict=True)

    id: UUID
    owner_tenant_id: FundId
    title: str = Field(min_length=1, max_length=256)
    description: str = Field(min_length=1)
    owner_id: UUID
    status: InitiativeStatus
    priority: InitiativePriority
    time_horizon: TimeHorizon
    start_date: date
    target_end_date: date
    health_score: Decimal | None = Field(default=None, ge=0, le=1)
    completion_pct: Decimal | None = Field(default=None, ge=0, le=1)
    source_artifact_id: UUID | None = None
    update_cadence_days: int = Field(default=7, ge=1)
    milestones: tuple[Milestone, ...] = Field(default_factory=tuple)
    subjects: tuple[InitiativeSubject, ...] = Field(default_factory=tuple)
    last_updated_at: datetime
    created_at: datetime
    updated_at: datetime
```

- [ ] **Step 3: Create Milestone entity**

```python
# apps/api/src/prescient/strategy/domain/entities/milestone.py
"""Milestone domain entity."""

from __future__ import annotations

from datetime import date, datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field

from prescient.strategy.domain.enums import MilestoneStatus


class Milestone(BaseModel):
    model_config = ConfigDict(frozen=True, strict=True)

    id: UUID
    initiative_id: UUID
    title: str = Field(min_length=1, max_length=256)
    description: str | None = None
    assignee_id: UUID | None = None
    status: MilestoneStatus
    target_date: date
    completed_date: date | None = None
    sort_order: int = 0
    created_at: datetime
```

- [ ] **Step 4: Create InitiativeSubject entity**

```python
# apps/api/src/prescient/strategy/domain/entities/initiative_subject.py
"""Polymorphic subject link for initiatives."""

from __future__ import annotations

from enum import StrEnum
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field


class SubjectType(StrEnum):
    COMPANY = "company"
    PRODUCT = "product"
    MARKET = "market"
    PERSON = "person"
    TOPIC = "topic"


class InitiativeSubject(BaseModel):
    model_config = ConfigDict(frozen=True)

    initiative_id: UUID
    subject_type: SubjectType
    subject_id: UUID
    role: str = Field(default="subject", min_length=1, max_length=32)
    is_primary: bool = False
```

- [ ] **Step 5: Verify imports work**

Run: `cd apps/api && python -c "from prescient.strategy.domain.entities.initiative import Initiative; print('OK')"`
Expected: `OK`

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/strategy/domain/entities/ apps/api/src/prescient/shared/types.py
git commit -m "feat(strategy): add Initiative, Milestone, and InitiativeSubject domain entities"
```

---

### Task 3: Health Score Computation

**Files:**
- Create: `apps/api/src/prescient/strategy/domain/health.py`
- Test: `apps/api/tests/unit/strategy/test_health_score.py`

- [ ] **Step 1: Write failing test**

```python
# apps/api/tests/unit/strategy/test_health_score.py
"""Unit tests for initiative health score computation."""

from __future__ import annotations

from datetime import UTC, date, datetime, timedelta
from decimal import Decimal
from uuid import uuid4

import pytest

from prescient.strategy.domain.enums import InitiativeStatus, MilestoneStatus
from prescient.strategy.domain.health import compute_health_score, HealthInput, MilestoneInput, UpstreamInput


NOW = datetime(2026, 4, 13, 12, 0, 0, tzinfo=UTC)


def _make_input(**overrides: object) -> HealthInput:
    defaults: dict[str, object] = {
        "milestones": [],
        "last_updated_at": NOW,
        "update_cadence_days": 7,
        "now": NOW,
        "upstream_initiatives": [],
    }
    defaults.update(overrides)
    return HealthInput(**defaults)  # type: ignore[arg-type]


class TestComputeHealthScore:
    def test_no_milestones_returns_base_score(self) -> None:
        inp = _make_input(milestones=[])
        score = compute_health_score(inp)
        assert score >= Decimal("0.5")

    def test_all_milestones_completed(self) -> None:
        milestones = [
            MilestoneInput(status=MilestoneStatus.COMPLETED, target_date=date(2026, 3, 1)),
            MilestoneInput(status=MilestoneStatus.COMPLETED, target_date=date(2026, 4, 1)),
        ]
        inp = _make_input(milestones=milestones)
        score = compute_health_score(inp)
        assert score >= Decimal("0.9")

    def test_stale_initiative_degrades_score(self) -> None:
        stale_date = NOW - timedelta(days=20)
        inp = _make_input(last_updated_at=stale_date, update_cadence_days=7)
        score = compute_health_score(inp)
        fresh_inp = _make_input(last_updated_at=NOW, update_cadence_days=7)
        fresh_score = compute_health_score(fresh_inp)
        assert score < fresh_score

    def test_overdue_milestone_penalizes(self) -> None:
        overdue = [
            MilestoneInput(status=MilestoneStatus.PENDING, target_date=date(2026, 3, 1)),
        ]
        inp = _make_input(milestones=overdue, now=NOW)
        on_time = [
            MilestoneInput(status=MilestoneStatus.PENDING, target_date=date(2026, 6, 1)),
        ]
        inp_on_time = _make_input(milestones=on_time, now=NOW)
        assert compute_health_score(inp) < compute_health_score(inp_on_time)

    def test_upstream_at_risk_penalizes(self) -> None:
        upstream = [UpstreamInput(status=InitiativeStatus.AT_RISK, dependency_type="blocks")]
        inp = _make_input(upstream_initiatives=upstream)
        clean_inp = _make_input(upstream_initiatives=[])
        assert compute_health_score(inp) < compute_health_score(clean_inp)

    def test_score_clamped_between_zero_and_one(self) -> None:
        worst = _make_input(
            milestones=[MilestoneInput(status=MilestoneStatus.MISSED, target_date=date(2026, 1, 1))],
            last_updated_at=NOW - timedelta(days=100),
            upstream_initiatives=[UpstreamInput(status=InitiativeStatus.AT_RISK, dependency_type="blocks")],
        )
        score = compute_health_score(worst)
        assert Decimal("0") <= score <= Decimal("1")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_health_score.py -v`
Expected: FAIL — `ModuleNotFoundError`

- [ ] **Step 3: Implement health score computation**

```python
# apps/api/src/prescient/strategy/domain/health.py
"""Health score computation — pure function, no side effects."""

from __future__ import annotations

from dataclasses import dataclass
from datetime import date, datetime
from decimal import Decimal

from prescient.strategy.domain.enums import InitiativeStatus, MilestoneStatus

# Weights
_MILESTONE_WEIGHT = Decimal("0.4")
_STALENESS_WEIGHT = Decimal("0.3")
_DEPENDENCY_WEIGHT = Decimal("0.2")
_OVERDUE_WEIGHT = Decimal("0.1")

# Dependency penalties by type
_DEP_PENALTY = {"blocks": Decimal("1.0"), "informs": Decimal("0.3"), "shares_resources": Decimal("0.5")}

_RISKY_STATUSES = {InitiativeStatus.AT_RISK, InitiativeStatus.ON_HOLD}


@dataclass(frozen=True)
class MilestoneInput:
    status: MilestoneStatus
    target_date: date


@dataclass(frozen=True)
class UpstreamInput:
    status: InitiativeStatus
    dependency_type: str


@dataclass(frozen=True)
class HealthInput:
    milestones: list[MilestoneInput]
    last_updated_at: datetime
    update_cadence_days: int
    now: datetime
    upstream_initiatives: list[UpstreamInput]


def compute_health_score(inp: HealthInput) -> Decimal:
    """Compute a 0-1 health score for an initiative."""
    milestone_score = _milestone_component(inp.milestones)
    staleness_score = _staleness_component(inp.last_updated_at, inp.update_cadence_days, inp.now)
    dependency_score = _dependency_component(inp.upstream_initiatives)
    overdue_score = _overdue_component(inp.milestones, inp.now.date())

    raw = (
        _MILESTONE_WEIGHT * milestone_score
        + _STALENESS_WEIGHT * staleness_score
        + _DEPENDENCY_WEIGHT * dependency_score
        + _OVERDUE_WEIGHT * overdue_score
    )
    return max(Decimal("0"), min(Decimal("1"), raw.quantize(Decimal("0.01"))))


def _milestone_component(milestones: list[MilestoneInput]) -> Decimal:
    if not milestones:
        return Decimal("0.7")  # no milestones → neutral
    completed = sum(1 for m in milestones if m.status == MilestoneStatus.COMPLETED)
    return Decimal(completed) / Decimal(len(milestones))


def _staleness_component(last_updated: datetime, cadence_days: int, now: datetime) -> Decimal:
    days_since = (now - last_updated).days
    threshold = cadence_days * 2
    if days_since <= cadence_days:
        return Decimal("1.0")
    if days_since >= threshold:
        return Decimal("0.0")
    # Linear degradation between cadence and 2x cadence
    remaining = Decimal(threshold - days_since) / Decimal(threshold - cadence_days)
    return remaining


def _dependency_component(upstreams: list[UpstreamInput]) -> Decimal:
    if not upstreams:
        return Decimal("1.0")
    total_penalty = Decimal("0")
    for u in upstreams:
        if u.status in _RISKY_STATUSES:
            total_penalty += _DEP_PENALTY.get(u.dependency_type, Decimal("0.3"))
    return max(Decimal("0"), Decimal("1") - total_penalty)


def _overdue_component(milestones: list[MilestoneInput], today: date) -> Decimal:
    if not milestones:
        return Decimal("1.0")
    overdue = sum(
        1 for m in milestones
        if m.target_date < today and m.status not in (MilestoneStatus.COMPLETED, MilestoneStatus.DEFERRED)
    )
    if overdue == 0:
        return Decimal("1.0")
    return max(Decimal("0"), Decimal("1") - Decimal(overdue) / Decimal(len(milestones)))
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_health_score.py -v`
Expected: PASS (all 6 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/strategy/domain/health.py apps/api/tests/unit/strategy/test_health_score.py
git commit -m "feat(strategy): add health score computation with milestone, staleness, dependency, and overdue factors"
```

---

### Task 4: Strategy ORM Tables + Migration

**Files:**
- Create: `apps/api/src/prescient/strategy/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/strategy/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/strategy/infrastructure/tables/initiative.py`
- Create: `apps/api/src/prescient/strategy/infrastructure/tables/milestone.py`
- Create: `apps/api/src/prescient/strategy/infrastructure/tables/initiative_subject.py`
- Create: `apps/api/alembic/versions/20260413_strategy_schema.py`
- Modify: `apps/api/src/prescient/shared/metadata.py` — register strategy tables

- [ ] **Step 1: Create infrastructure package structure**

Create empty `__init__.py` files:
```python
# apps/api/src/prescient/strategy/infrastructure/__init__.py
# apps/api/src/prescient/strategy/infrastructure/tables/__init__.py
```

- [ ] **Step 2: Create InitiativeTable**

```python
# apps/api/src/prescient/strategy/infrastructure/tables/initiative.py
"""`strategy.initiatives` — strategic initiative persistence."""

from __future__ import annotations

from datetime import date, datetime
from decimal import Decimal
from uuid import UUID

from sqlalchemy import Date, DateTime, Integer, Numeric, String, Text, func
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class InitiativeTable(Base):
    __tablename__ = "initiatives"
    __table_args__ = {"schema": "strategy"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    owner_tenant_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False, index=True)
    title: Mapped[str] = mapped_column(String(256), nullable=False)
    description: Mapped[str] = mapped_column(Text, nullable=False)
    owner_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False)
    status: Mapped[str] = mapped_column(String(32), nullable=False, index=True, server_default="draft")
    priority: Mapped[str] = mapped_column(String(16), nullable=False, server_default="medium")
    time_horizon: Mapped[str] = mapped_column(String(16), nullable=False)
    start_date: Mapped[date] = mapped_column(Date, nullable=False)
    target_end_date: Mapped[date] = mapped_column(Date, nullable=False)
    health_score: Mapped[Decimal | None] = mapped_column(Numeric(precision=3, scale=2), nullable=True)
    completion_pct: Mapped[Decimal | None] = mapped_column(Numeric(precision=3, scale=2), nullable=True)
    source_artifact_id: Mapped[UUID | None] = mapped_column(PgUUID(as_uuid=True), nullable=True)
    update_cadence_days: Mapped[int] = mapped_column(Integer, nullable=False, server_default="7")
    last_updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=func.now())
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=func.now())
```

- [ ] **Step 3: Create MilestoneTable**

```python
# apps/api/src/prescient/strategy/infrastructure/tables/milestone.py
"""`strategy.milestones` — initiative milestone persistence."""

from __future__ import annotations

from datetime import date, datetime
from uuid import UUID

from sqlalchemy import Date, DateTime, ForeignKey, Integer, String, Text, func
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class MilestoneTable(Base):
    __tablename__ = "milestones"
    __table_args__ = {"schema": "strategy"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    initiative_id: Mapped[UUID] = mapped_column(
        PgUUID(as_uuid=True),
        ForeignKey("strategy.initiatives.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    title: Mapped[str] = mapped_column(String(256), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    assignee_id: Mapped[UUID | None] = mapped_column(PgUUID(as_uuid=True), nullable=True)
    status: Mapped[str] = mapped_column(String(32), nullable=False, server_default="pending")
    target_date: Mapped[date] = mapped_column(Date, nullable=False)
    completed_date: Mapped[date | None] = mapped_column(Date, nullable=True)
    sort_order: Mapped[int] = mapped_column(Integer, nullable=False, server_default="0")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=func.now())
```

- [ ] **Step 4: Create InitiativeSubjectTable**

```python
# apps/api/src/prescient/strategy/infrastructure/tables/initiative_subject.py
"""`strategy.initiative_subjects` — polymorphic subject links."""

from __future__ import annotations

from uuid import UUID

from sqlalchemy import Boolean, String
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class InitiativeSubjectTable(Base):
    __tablename__ = "initiative_subjects"
    __table_args__ = {"schema": "strategy"}  # noqa: RUF012

    initiative_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    subject_type: Mapped[str] = mapped_column(String(16), primary_key=True)
    subject_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    role: Mapped[str] = mapped_column(String(32), server_default="subject")
    is_primary: Mapped[bool] = mapped_column(Boolean(), server_default="false")
```

- [ ] **Step 5: Register tables in shared metadata**

Add to `apps/api/src/prescient/shared/metadata.py` after the triage import:

```python
import prescient.strategy.infrastructure.tables  # noqa: F401
```

- [ ] **Step 6: Create Alembic migration**

Note: The current migration tree has two heads (`0007_board_prep_schema` and `20260413_research`), both descending from `0006_km_schema`. This migration uses a merge to unify them.

```python
# apps/api/alembic/versions/20260413_strategy_schema.py
"""strategy schema — initiatives, milestones, initiative_subjects

Revision ID: 20260413_strategy
Revises: 0007_board_prep_schema, 20260413_research
Create Date: 2026-04-13
"""

from __future__ import annotations

from collections.abc import Sequence

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects.postgresql import UUID

revision: str = "20260413_strategy"
down_revision: tuple[str, ...] = ("0007_board_prep_schema", "20260413_research")
branch_labels: str | Sequence[str] | None = None
depends_on: str | Sequence[str] | None = None


def upgrade() -> None:
    op.execute("CREATE SCHEMA IF NOT EXISTS strategy")

    op.create_table(
        "initiatives",
        sa.Column("id", UUID(as_uuid=True), nullable=False),
        sa.Column("owner_tenant_id", UUID(as_uuid=True), nullable=False, index=True),
        sa.Column("title", sa.String(256), nullable=False),
        sa.Column("description", sa.Text, nullable=False),
        sa.Column("owner_id", UUID(as_uuid=True), nullable=False),
        sa.Column("status", sa.String(32), nullable=False, server_default="draft", index=True),
        sa.Column("priority", sa.String(16), nullable=False, server_default="medium"),
        sa.Column("time_horizon", sa.String(16), nullable=False),
        sa.Column("start_date", sa.Date, nullable=False),
        sa.Column("target_end_date", sa.Date, nullable=False),
        sa.Column("health_score", sa.Numeric(precision=3, scale=2), nullable=True),
        sa.Column("completion_pct", sa.Numeric(precision=3, scale=2), nullable=True),
        sa.Column("source_artifact_id", UUID(as_uuid=True), nullable=True),
        sa.Column("update_cadence_days", sa.Integer, nullable=False, server_default="7"),
        sa.Column("last_updated_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.PrimaryKeyConstraint("id"),
        schema="strategy",
    )

    op.create_table(
        "milestones",
        sa.Column("id", UUID(as_uuid=True), nullable=False),
        sa.Column("initiative_id", UUID(as_uuid=True), sa.ForeignKey("strategy.initiatives.id", ondelete="CASCADE"), nullable=False, index=True),
        sa.Column("title", sa.String(256), nullable=False),
        sa.Column("description", sa.Text, nullable=True),
        sa.Column("assignee_id", UUID(as_uuid=True), nullable=True),
        sa.Column("status", sa.String(32), nullable=False, server_default="pending"),
        sa.Column("target_date", sa.Date, nullable=False),
        sa.Column("completed_date", sa.Date, nullable=True),
        sa.Column("sort_order", sa.Integer, nullable=False, server_default="0"),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.PrimaryKeyConstraint("id"),
        schema="strategy",
    )

    op.create_table(
        "initiative_subjects",
        sa.Column("initiative_id", UUID(as_uuid=True), nullable=False),
        sa.Column("subject_type", sa.String(16), nullable=False),
        sa.Column("subject_id", UUID(as_uuid=True), nullable=False),
        sa.Column("role", sa.String(32), server_default="subject"),
        sa.Column("is_primary", sa.Boolean(), server_default="false"),
        sa.PrimaryKeyConstraint("initiative_id", "subject_type", "subject_id"),
        schema="strategy",
    )


def downgrade() -> None:
    op.drop_table("initiative_subjects", schema="strategy")
    op.drop_table("milestones", schema="strategy")
    op.drop_table("initiatives", schema="strategy")
    op.execute("DROP SCHEMA IF EXISTS strategy CASCADE")
```

- [ ] **Step 7: Verify migration applies**

Run: `cd apps/api && docker compose exec api alembic upgrade head`
Expected: Migration applies without errors.

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/strategy/infrastructure/ apps/api/alembic/versions/20260413_strategy_schema.py apps/api/src/prescient/shared/metadata.py
git commit -m "feat(strategy): add ORM tables and Alembic migration for initiatives, milestones, and subjects"
```

---

### Task 5: CreateInitiative Use Case

**Files:**
- Create: `apps/api/src/prescient/strategy/application/__init__.py`
- Create: `apps/api/src/prescient/strategy/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/strategy/application/use_cases/create_initiative.py`
- Test: `apps/api/tests/unit/strategy/test_create_initiative.py`

- [ ] **Step 1: Create application package structure**

```python
# apps/api/src/prescient/strategy/application/__init__.py
# apps/api/src/prescient/strategy/application/use_cases/__init__.py
```

- [ ] **Step 2: Write failing test**

```python
# apps/api/tests/unit/strategy/test_create_initiative.py
"""Unit tests for CreateInitiative use case."""

from __future__ import annotations

from datetime import UTC, date, datetime
from uuid import uuid4

import pytest

from prescient.strategy.application.use_cases.create_initiative import (
    CreateInitiative,
    CreateInitiativeCommand,
    CreateInitiativeResult,
)
from prescient.strategy.domain.enums import InitiativePriority, InitiativeStatus, TimeHorizon

NOW = datetime(2026, 4, 13, 10, 0, 0, tzinfo=UTC)
TENANT_ID = uuid4()
OWNER_ID = uuid4()


def _make_command(**overrides: object) -> CreateInitiativeCommand:
    defaults: dict[str, object] = {
        "owner_tenant_id": TENANT_ID,
        "title": "Margin Expansion Program",
        "description": "Drive EBITDA margin from 18% to 25% over 12 months.",
        "owner_id": OWNER_ID,
        "priority": InitiativePriority.HIGH,
        "time_horizon": TimeHorizon.ANNUAL,
        "start_date": date(2026, 1, 1),
        "target_end_date": date(2026, 12, 31),
        "created_at": NOW,
    }
    defaults.update(overrides)
    return CreateInitiativeCommand(**defaults)  # type: ignore[arg-type]


class _FakeSession:
    def __init__(self) -> None:
        self.added: list[object] = []

    def add(self, row: object) -> None:
        self.added.append(row)


class _FakeRow:
    def __init__(self, **kwargs: object) -> None:
        for k, v in kwargs.items():
            setattr(self, k, v)


class TestCreateInitiative:
    @pytest.mark.asyncio
    async def test_returns_result_with_initiative_id(self) -> None:
        session = _FakeSession()
        use_case = CreateInitiative(session=session, row_factory=_FakeRow)
        command = _make_command()

        result = await use_case.execute(command)

        assert isinstance(result, CreateInitiativeResult)
        assert result.initiative_id == command.initiative_id

    @pytest.mark.asyncio
    async def test_adds_row_to_session(self) -> None:
        session = _FakeSession()
        use_case = CreateInitiative(session=session, row_factory=_FakeRow)

        await use_case.execute(_make_command())

        assert len(session.added) == 1

    @pytest.mark.asyncio
    async def test_row_has_draft_status(self) -> None:
        session = _FakeSession()
        use_case = CreateInitiative(session=session, row_factory=_FakeRow)

        await use_case.execute(_make_command())

        row = session.added[0]
        assert row.status == InitiativeStatus.DRAFT.value  # type: ignore[attr-defined]

    @pytest.mark.asyncio
    async def test_row_fields_match_command(self) -> None:
        session = _FakeSession()
        use_case = CreateInitiative(session=session, row_factory=_FakeRow)
        command = _make_command(title="Revenue Growth", priority=InitiativePriority.CRITICAL)

        await use_case.execute(command)

        row = session.added[0]
        assert row.title == "Revenue Growth"  # type: ignore[attr-defined]
        assert row.priority == InitiativePriority.CRITICAL.value  # type: ignore[attr-defined]
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_create_initiative.py -v`
Expected: FAIL — `ModuleNotFoundError`

- [ ] **Step 4: Implement use case**

```python
# apps/api/src/prescient/strategy/application/use_cases/create_initiative.py
"""CreateInitiative use case."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import date, datetime
from typing import Any, Protocol
from uuid import UUID, uuid4

from prescient.strategy.domain.enums import InitiativePriority, InitiativeStatus, TimeHorizon
from prescient.shared.types import utcnow


class CreateInitiativeSessionPort(Protocol):
    def add(self, row: Any) -> None: ...


@dataclass(frozen=True)
class CreateInitiativeCommand:
    owner_tenant_id: UUID
    title: str
    description: str
    owner_id: UUID
    priority: InitiativePriority
    time_horizon: TimeHorizon
    start_date: date
    target_end_date: date
    source_artifact_id: UUID | None = None
    update_cadence_days: int = 7
    initiative_id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=utcnow)


@dataclass(frozen=True)
class CreateInitiativeResult:
    initiative_id: UUID


class CreateInitiative:
    def __init__(self, session: CreateInitiativeSessionPort, row_factory: Any) -> None:
        self._session = session
        self._row_factory = row_factory

    async def execute(self, command: CreateInitiativeCommand) -> CreateInitiativeResult:
        row = self._row_factory(
            id=command.initiative_id,
            owner_tenant_id=command.owner_tenant_id,
            title=command.title,
            description=command.description,
            owner_id=command.owner_id,
            status=InitiativeStatus.DRAFT.value,
            priority=command.priority.value,
            time_horizon=command.time_horizon.value,
            start_date=command.start_date,
            target_end_date=command.target_end_date,
            health_score=None,
            completion_pct=None,
            source_artifact_id=command.source_artifact_id,
            update_cadence_days=command.update_cadence_days,
            last_updated_at=command.created_at,
            created_at=command.created_at,
            updated_at=command.created_at,
        )
        self._session.add(row)
        return CreateInitiativeResult(initiative_id=command.initiative_id)
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_create_initiative.py -v`
Expected: PASS (all 4 tests)

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/strategy/application/ apps/api/tests/unit/strategy/test_create_initiative.py
git commit -m "feat(strategy): add CreateInitiative use case with TDD"
```

---

### Task 6: UpdateInitiative, ListInitiatives, GetInitiative Use Cases

**Files:**
- Create: `apps/api/src/prescient/strategy/application/use_cases/update_initiative.py`
- Create: `apps/api/src/prescient/strategy/application/use_cases/list_initiatives.py`
- Create: `apps/api/src/prescient/strategy/application/use_cases/get_initiative.py`
- Create: `apps/api/src/prescient/strategy/queries.py`
- Test: `apps/api/tests/unit/strategy/test_update_initiative.py`
- Test: `apps/api/tests/unit/strategy/test_list_initiatives.py`

- [ ] **Step 1: Write failing test for UpdateInitiative**

```python
# apps/api/tests/unit/strategy/test_update_initiative.py
"""Unit tests for UpdateInitiative use case."""

from __future__ import annotations

from uuid import uuid4

import pytest

from prescient.strategy.application.use_cases.update_initiative import (
    UpdateInitiative,
    UpdateInitiativeCommand,
)
from prescient.strategy.domain.enums import InitiativeStatus


class _FakeRow:
    def __init__(self, **kwargs: object) -> None:
        for k, v in kwargs.items():
            setattr(self, k, v)


class _FakeQueryPort:
    def __init__(self, row: _FakeRow | None) -> None:
        self._row = row

    async def get_by_id(self, initiative_id: object) -> _FakeRow | None:
        return self._row


INITIATIVE_ID = uuid4()


class TestUpdateInitiative:
    @pytest.mark.asyncio
    async def test_updates_status(self) -> None:
        row = _FakeRow(id=INITIATIVE_ID, status="draft", priority="high", title="Test")
        port = _FakeQueryPort(row)
        use_case = UpdateInitiative(query_port=port)
        command = UpdateInitiativeCommand(
            initiative_id=INITIATIVE_ID,
            status=InitiativeStatus.ACTIVE,
        )

        await use_case.execute(command)

        assert row.status == InitiativeStatus.ACTIVE.value

    @pytest.mark.asyncio
    async def test_raises_on_not_found(self) -> None:
        port = _FakeQueryPort(None)
        use_case = UpdateInitiative(query_port=port)
        command = UpdateInitiativeCommand(initiative_id=uuid4(), status=InitiativeStatus.ACTIVE)

        with pytest.raises(ValueError, match="not found"):
            await use_case.execute(command)

    @pytest.mark.asyncio
    async def test_partial_update_leaves_other_fields(self) -> None:
        row = _FakeRow(id=INITIATIVE_ID, status="draft", priority="high", title="Keep This")
        port = _FakeQueryPort(row)
        use_case = UpdateInitiative(query_port=port)
        command = UpdateInitiativeCommand(initiative_id=INITIATIVE_ID, status=InitiativeStatus.ACTIVE)

        await use_case.execute(command)

        assert row.title == "Keep This"
        assert row.priority == "high"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_update_initiative.py -v`
Expected: FAIL

- [ ] **Step 3: Implement UpdateInitiative**

```python
# apps/api/src/prescient/strategy/application/use_cases/update_initiative.py
"""UpdateInitiative use case — partial update of initiative fields."""

from __future__ import annotations

from dataclasses import dataclass
from datetime import date
from typing import Any, Protocol
from uuid import UUID

from prescient.strategy.domain.enums import InitiativePriority, InitiativeStatus, TimeHorizon
from prescient.shared.types import utcnow


class UpdateInitiativeQueryPort(Protocol):
    async def get_by_id(self, initiative_id: UUID) -> Any | None: ...


@dataclass(frozen=True)
class UpdateInitiativeCommand:
    initiative_id: UUID
    title: str | None = None
    description: str | None = None
    status: InitiativeStatus | None = None
    priority: InitiativePriority | None = None
    time_horizon: TimeHorizon | None = None
    start_date: date | None = None
    target_end_date: date | None = None
    update_cadence_days: int | None = None


class UpdateInitiative:
    def __init__(self, query_port: UpdateInitiativeQueryPort) -> None:
        self._query_port = query_port

    async def execute(self, command: UpdateInitiativeCommand) -> None:
        row = await self._query_port.get_by_id(command.initiative_id)
        if row is None:
            raise ValueError(f"Initiative {command.initiative_id} not found")

        now = utcnow()
        if command.title is not None:
            row.title = command.title
        if command.description is not None:
            row.description = command.description
        if command.status is not None:
            row.status = command.status.value
        if command.priority is not None:
            row.priority = command.priority.value
        if command.time_horizon is not None:
            row.time_horizon = command.time_horizon.value
        if command.start_date is not None:
            row.start_date = command.start_date
        if command.target_end_date is not None:
            row.target_end_date = command.target_end_date
        if command.update_cadence_days is not None:
            row.update_cadence_days = command.update_cadence_days
        row.updated_at = now
        row.last_updated_at = now
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_update_initiative.py -v`
Expected: PASS (all 3 tests)

- [ ] **Step 5: Create queries module**

```python
# apps/api/src/prescient/strategy/queries.py
"""Published query functions for the strategy context.

Other contexts (e.g. briefing) read strategy data through these functions
rather than importing ORM tables directly.
"""

from __future__ import annotations

from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.strategy.infrastructure.tables.initiative import InitiativeTable
from prescient.strategy.infrastructure.tables.milestone import MilestoneTable


async def get_initiatives_for_tenant(
    session: AsyncSession,
    owner_tenant_id: str,
    *,
    status_filter: list[str] | None = None,
) -> list[InitiativeTable]:
    stmt = (
        select(InitiativeTable)
        .where(InitiativeTable.owner_tenant_id == UUID(owner_tenant_id))
        .order_by(InitiativeTable.created_at.desc())
    )
    if status_filter:
        stmt = stmt.where(InitiativeTable.status.in_(status_filter))
    result = await session.execute(stmt)
    return list(result.scalars().all())


async def get_initiative_by_id(
    session: AsyncSession,
    initiative_id: UUID,
) -> InitiativeTable | None:
    result = await session.execute(
        select(InitiativeTable).where(InitiativeTable.id == initiative_id)
    )
    return result.scalar_one_or_none()


async def get_milestones_for_initiative(
    session: AsyncSession,
    initiative_id: UUID,
) -> list[MilestoneTable]:
    result = await session.execute(
        select(MilestoneTable)
        .where(MilestoneTable.initiative_id == initiative_id)
        .order_by(MilestoneTable.sort_order)
    )
    return list(result.scalars().all())
```

- [ ] **Step 6: Implement ListInitiatives and GetInitiative use cases**

```python
# apps/api/src/prescient/strategy/application/use_cases/list_initiatives.py
"""ListInitiatives use case."""

from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Protocol
from uuid import UUID


class ListInitiativesQueryPort(Protocol):
    async def list_for_tenant(
        self, owner_tenant_id: str, *, status_filter: list[str] | None = None
    ) -> list[Any]: ...


@dataclass(frozen=True)
class ListInitiativesCommand:
    owner_tenant_id: str
    status_filter: list[str] | None = None


class ListInitiatives:
    def __init__(self, query_port: ListInitiativesQueryPort) -> None:
        self._query_port = query_port

    async def execute(self, command: ListInitiativesCommand) -> list[Any]:
        return await self._query_port.list_for_tenant(
            command.owner_tenant_id, status_filter=command.status_filter
        )
```

```python
# apps/api/src/prescient/strategy/application/use_cases/get_initiative.py
"""GetInitiative use case — returns full initiative detail with milestones."""

from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Protocol
from uuid import UUID


class GetInitiativeQueryPort(Protocol):
    async def get_by_id(self, initiative_id: UUID) -> Any | None: ...
    async def get_milestones(self, initiative_id: UUID) -> list[Any]: ...


@dataclass(frozen=True)
class GetInitiativeCommand:
    initiative_id: UUID


@dataclass(frozen=True)
class GetInitiativeResult:
    initiative: Any
    milestones: list[Any]


class GetInitiative:
    def __init__(self, query_port: GetInitiativeQueryPort) -> None:
        self._query_port = query_port

    async def execute(self, command: GetInitiativeCommand) -> GetInitiativeResult:
        initiative = await self._query_port.get_by_id(command.initiative_id)
        if initiative is None:
            raise ValueError(f"Initiative {command.initiative_id} not found")
        milestones = await self._query_port.get_milestones(command.initiative_id)
        return GetInitiativeResult(initiative=initiative, milestones=milestones)
```

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/strategy/application/use_cases/ apps/api/src/prescient/strategy/queries.py apps/api/tests/unit/strategy/
git commit -m "feat(strategy): add UpdateInitiative, ListInitiatives, GetInitiative use cases and queries"
```

---

### Task 7: CreateMilestone and UpdateMilestone Use Cases

**Files:**
- Create: `apps/api/src/prescient/strategy/application/use_cases/create_milestone.py`
- Create: `apps/api/src/prescient/strategy/application/use_cases/update_milestone.py`
- Test: `apps/api/tests/unit/strategy/test_create_milestone.py`
- Test: `apps/api/tests/unit/strategy/test_update_milestone.py`

- [ ] **Step 1: Write failing test for CreateMilestone**

```python
# apps/api/tests/unit/strategy/test_create_milestone.py
"""Unit tests for CreateMilestone use case."""

from __future__ import annotations

from datetime import UTC, date, datetime
from uuid import uuid4

import pytest

from prescient.strategy.application.use_cases.create_milestone import (
    CreateMilestone,
    CreateMilestoneCommand,
    CreateMilestoneResult,
)
from prescient.strategy.domain.enums import MilestoneStatus

NOW = datetime(2026, 4, 13, 10, 0, 0, tzinfo=UTC)
INITIATIVE_ID = uuid4()


def _make_command(**overrides: object) -> CreateMilestoneCommand:
    defaults: dict[str, object] = {
        "initiative_id": INITIATIVE_ID,
        "title": "Complete vendor selection",
        "target_date": date(2026, 6, 30),
        "created_at": NOW,
    }
    defaults.update(overrides)
    return CreateMilestoneCommand(**defaults)  # type: ignore[arg-type]


class _FakeSession:
    def __init__(self) -> None:
        self.added: list[object] = []

    def add(self, row: object) -> None:
        self.added.append(row)


class _FakeRow:
    def __init__(self, **kwargs: object) -> None:
        for k, v in kwargs.items():
            setattr(self, k, v)


class TestCreateMilestone:
    @pytest.mark.asyncio
    async def test_returns_result_with_milestone_id(self) -> None:
        session = _FakeSession()
        use_case = CreateMilestone(session=session, row_factory=_FakeRow)
        command = _make_command()

        result = await use_case.execute(command)

        assert isinstance(result, CreateMilestoneResult)
        assert result.milestone_id == command.milestone_id

    @pytest.mark.asyncio
    async def test_row_has_pending_status(self) -> None:
        session = _FakeSession()
        use_case = CreateMilestone(session=session, row_factory=_FakeRow)

        await use_case.execute(_make_command())

        row = session.added[0]
        assert row.status == MilestoneStatus.PENDING.value  # type: ignore[attr-defined]

    @pytest.mark.asyncio
    async def test_row_fields_match_command(self) -> None:
        session = _FakeSession()
        use_case = CreateMilestone(session=session, row_factory=_FakeRow)
        command = _make_command(title="Hire CFO", target_date=date(2026, 9, 1))

        await use_case.execute(command)

        row = session.added[0]
        assert row.title == "Hire CFO"  # type: ignore[attr-defined]
        assert row.target_date == date(2026, 9, 1)  # type: ignore[attr-defined]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_create_milestone.py -v`
Expected: FAIL

- [ ] **Step 3: Implement CreateMilestone**

```python
# apps/api/src/prescient/strategy/application/use_cases/create_milestone.py
"""CreateMilestone use case."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import date, datetime
from typing import Any, Protocol
from uuid import UUID, uuid4

from prescient.strategy.domain.enums import MilestoneStatus
from prescient.shared.types import utcnow


class CreateMilestoneSessionPort(Protocol):
    def add(self, row: Any) -> None: ...


@dataclass(frozen=True)
class CreateMilestoneCommand:
    initiative_id: UUID
    title: str
    target_date: date
    description: str | None = None
    assignee_id: UUID | None = None
    sort_order: int = 0
    milestone_id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=utcnow)


@dataclass(frozen=True)
class CreateMilestoneResult:
    milestone_id: UUID


class CreateMilestone:
    def __init__(self, session: CreateMilestoneSessionPort, row_factory: Any) -> None:
        self._session = session
        self._row_factory = row_factory

    async def execute(self, command: CreateMilestoneCommand) -> CreateMilestoneResult:
        row = self._row_factory(
            id=command.milestone_id,
            initiative_id=command.initiative_id,
            title=command.title,
            description=command.description,
            assignee_id=command.assignee_id,
            status=MilestoneStatus.PENDING.value,
            target_date=command.target_date,
            completed_date=None,
            sort_order=command.sort_order,
            created_at=command.created_at,
        )
        self._session.add(row)
        return CreateMilestoneResult(milestone_id=command.milestone_id)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_create_milestone.py -v`
Expected: PASS (all 3 tests)

- [ ] **Step 5: Implement UpdateMilestone (same TDD pattern)**

```python
# apps/api/src/prescient/strategy/application/use_cases/update_milestone.py
"""UpdateMilestone use case."""

from __future__ import annotations

from dataclasses import dataclass
from datetime import date
from typing import Any, Protocol
from uuid import UUID

from prescient.strategy.domain.enums import MilestoneStatus
from prescient.shared.types import utcnow


class UpdateMilestoneQueryPort(Protocol):
    async def get_by_id(self, milestone_id: UUID) -> Any | None: ...


@dataclass(frozen=True)
class UpdateMilestoneCommand:
    milestone_id: UUID
    title: str | None = None
    description: str | None = None
    assignee_id: UUID | None = None
    status: MilestoneStatus | None = None
    target_date: date | None = None
    completed_date: date | None = None
    sort_order: int | None = None


class UpdateMilestone:
    def __init__(self, query_port: UpdateMilestoneQueryPort) -> None:
        self._query_port = query_port

    async def execute(self, command: UpdateMilestoneCommand) -> None:
        row = await self._query_port.get_by_id(command.milestone_id)
        if row is None:
            raise ValueError(f"Milestone {command.milestone_id} not found")

        if command.title is not None:
            row.title = command.title
        if command.description is not None:
            row.description = command.description
        if command.assignee_id is not None:
            row.assignee_id = command.assignee_id
        if command.status is not None:
            row.status = command.status.value
            if command.status == MilestoneStatus.COMPLETED and command.completed_date is None:
                row.completed_date = utcnow().date()
        if command.target_date is not None:
            row.target_date = command.target_date
        if command.completed_date is not None:
            row.completed_date = command.completed_date
        if command.sort_order is not None:
            row.sort_order = command.sort_order
```

- [ ] **Step 6: Write and run UpdateMilestone test**

```python
# apps/api/tests/unit/strategy/test_update_milestone.py
"""Unit tests for UpdateMilestone use case."""

from __future__ import annotations

from datetime import date
from uuid import uuid4

import pytest

from prescient.strategy.application.use_cases.update_milestone import (
    UpdateMilestone,
    UpdateMilestoneCommand,
)
from prescient.strategy.domain.enums import MilestoneStatus

MILESTONE_ID = uuid4()


class _FakeRow:
    def __init__(self, **kwargs: object) -> None:
        for k, v in kwargs.items():
            setattr(self, k, v)


class _FakeQueryPort:
    def __init__(self, row: _FakeRow | None) -> None:
        self._row = row

    async def get_by_id(self, milestone_id: object) -> _FakeRow | None:
        return self._row


class TestUpdateMilestone:
    @pytest.mark.asyncio
    async def test_updates_status(self) -> None:
        row = _FakeRow(id=MILESTONE_ID, status="pending", title="Test", completed_date=None)
        use_case = UpdateMilestone(query_port=_FakeQueryPort(row))
        command = UpdateMilestoneCommand(milestone_id=MILESTONE_ID, status=MilestoneStatus.IN_PROGRESS)

        await use_case.execute(command)

        assert row.status == MilestoneStatus.IN_PROGRESS.value

    @pytest.mark.asyncio
    async def test_completed_sets_date_automatically(self) -> None:
        row = _FakeRow(id=MILESTONE_ID, status="pending", title="Test", completed_date=None)
        use_case = UpdateMilestone(query_port=_FakeQueryPort(row))
        command = UpdateMilestoneCommand(milestone_id=MILESTONE_ID, status=MilestoneStatus.COMPLETED)

        await use_case.execute(command)

        assert row.completed_date is not None

    @pytest.mark.asyncio
    async def test_raises_on_not_found(self) -> None:
        use_case = UpdateMilestone(query_port=_FakeQueryPort(None))

        with pytest.raises(ValueError, match="not found"):
            await use_case.execute(UpdateMilestoneCommand(milestone_id=uuid4()))
```

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_update_milestone.py -v`
Expected: PASS (all 3 tests)

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/strategy/application/use_cases/create_milestone.py apps/api/src/prescient/strategy/application/use_cases/update_milestone.py apps/api/tests/unit/strategy/test_create_milestone.py apps/api/tests/unit/strategy/test_update_milestone.py
git commit -m "feat(strategy): add CreateMilestone and UpdateMilestone use cases with TDD"
```

---

### Task 8: Strategy API Routes (CRUD)

**Files:**
- Create: `apps/api/src/prescient/strategy/api/__init__.py`
- Create: `apps/api/src/prescient/strategy/api/routes.py`
- Modify: `apps/api/src/prescient/main.py` — register strategy router

- [ ] **Step 1: Create API package**

```python
# apps/api/src/prescient/strategy/api/__init__.py
```

- [ ] **Step 2: Implement routes**

```python
# apps/api/src/prescient/strategy/api/routes.py
"""Strategy API routes — composition root for initiative CRUD."""

from __future__ import annotations

from datetime import date
from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.db import get_session
from prescient.strategy.application.use_cases.create_initiative import (
    CreateInitiative,
    CreateInitiativeCommand,
)
from prescient.strategy.application.use_cases.create_milestone import (
    CreateMilestone,
    CreateMilestoneCommand,
)
from prescient.strategy.application.use_cases.update_initiative import (
    UpdateInitiative,
    UpdateInitiativeCommand,
)
from prescient.strategy.application.use_cases.update_milestone import (
    UpdateMilestone,
    UpdateMilestoneCommand,
)
from prescient.strategy.domain.enums import (
    InitiativePriority,
    InitiativeStatus,
    MilestoneStatus,
    TimeHorizon,
)
from prescient.strategy.infrastructure.tables.initiative import InitiativeTable
from prescient.strategy.infrastructure.tables.milestone import MilestoneTable
from prescient.strategy.queries import (
    get_initiative_by_id,
    get_initiatives_for_tenant,
    get_milestones_for_initiative,
)

router = APIRouter(prefix="/strategy", tags=["strategy"])

SessionDep = Annotated[AsyncSession, Depends(get_session)]


# ---------------------------------------------------------------------------
# Request / Response models
# ---------------------------------------------------------------------------


class CreateInitiativeRequest(BaseModel):
    title: str = Field(min_length=1, max_length=256)
    description: str = Field(min_length=1)
    owner_id: str
    priority: InitiativePriority = InitiativePriority.MEDIUM
    time_horizon: TimeHorizon
    start_date: date
    target_end_date: date
    source_artifact_id: str | None = None
    update_cadence_days: int = 7


class UpdateInitiativeRequest(BaseModel):
    title: str | None = None
    description: str | None = None
    status: InitiativeStatus | None = None
    priority: InitiativePriority | None = None
    time_horizon: TimeHorizon | None = None
    start_date: date | None = None
    target_end_date: date | None = None
    update_cadence_days: int | None = None


class CreateMilestoneRequest(BaseModel):
    title: str = Field(min_length=1, max_length=256)
    target_date: date
    description: str | None = None
    assignee_id: str | None = None
    sort_order: int = 0


class UpdateMilestoneRequest(BaseModel):
    title: str | None = None
    description: str | None = None
    assignee_id: str | None = None
    status: MilestoneStatus | None = None
    target_date: date | None = None
    completed_date: date | None = None
    sort_order: int | None = None


class InitiativeResponse(BaseModel):
    id: str
    title: str
    description: str
    owner_id: str
    status: str
    priority: str
    time_horizon: str
    start_date: date
    target_end_date: date
    health_score: float | None
    completion_pct: float | None
    source_artifact_id: str | None
    update_cadence_days: int
    created_at: str
    updated_at: str


class MilestoneResponse(BaseModel):
    id: str
    initiative_id: str
    title: str
    description: str | None
    assignee_id: str | None
    status: str
    target_date: date
    completed_date: date | None
    sort_order: int
    created_at: str


class InitiativeDetailResponse(BaseModel):
    initiative: InitiativeResponse
    milestones: list[MilestoneResponse]


# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------


def _initiative_to_response(row: InitiativeTable) -> InitiativeResponse:
    return InitiativeResponse(
        id=str(row.id),
        title=row.title,
        description=row.description,
        owner_id=str(row.owner_id),
        status=row.status,
        priority=row.priority,
        time_horizon=row.time_horizon,
        start_date=row.start_date,
        target_end_date=row.target_end_date,
        health_score=float(row.health_score) if row.health_score is not None else None,
        completion_pct=float(row.completion_pct) if row.completion_pct is not None else None,
        source_artifact_id=str(row.source_artifact_id) if row.source_artifact_id else None,
        update_cadence_days=row.update_cadence_days,
        created_at=row.created_at.isoformat(),
        updated_at=row.updated_at.isoformat(),
    )


def _milestone_to_response(row: MilestoneTable) -> MilestoneResponse:
    return MilestoneResponse(
        id=str(row.id),
        initiative_id=str(row.initiative_id),
        title=row.title,
        description=row.description,
        assignee_id=str(row.assignee_id) if row.assignee_id else None,
        status=row.status,
        target_date=row.target_date,
        completed_date=row.completed_date,
        sort_order=row.sort_order,
        created_at=row.created_at.isoformat(),
    )


# ---------------------------------------------------------------------------
# Adapters (composition root — infra allowed here)
# ---------------------------------------------------------------------------


class _InitiativeQueryAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get_by_id(self, initiative_id: UUID) -> InitiativeTable | None:
        return await get_initiative_by_id(self._session, initiative_id)

    async def get_milestones(self, initiative_id: UUID) -> list[MilestoneTable]:
        return await get_milestones_for_initiative(self._session, initiative_id)

    async def list_for_tenant(
        self, owner_tenant_id: str, *, status_filter: list[str] | None = None
    ) -> list[InitiativeTable]:
        return await get_initiatives_for_tenant(
            self._session, owner_tenant_id, status_filter=status_filter
        )


class _MilestoneQueryAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get_by_id(self, milestone_id: UUID) -> MilestoneTable | None:
        result = await self._session.execute(
            select(MilestoneTable).where(MilestoneTable.id == milestone_id)
        )
        return result.scalar_one_or_none()


# ---------------------------------------------------------------------------
# Endpoints
# ---------------------------------------------------------------------------


@router.get("/initiatives")
async def list_initiatives(
    organization_id: str,
    session: SessionDep,
    status: str | None = None,
) -> list[InitiativeResponse]:
    adapter = _InitiativeQueryAdapter(session)
    status_filter = [s.strip() for s in status.split(",")] if status else None
    rows = await adapter.list_for_tenant(organization_id, status_filter=status_filter)
    return [_initiative_to_response(r) for r in rows]


@router.post("/initiatives", status_code=201)
async def create_initiative(
    payload: CreateInitiativeRequest,
    organization_id: str,
    session: SessionDep,
) -> InitiativeResponse:
    use_case = CreateInitiative(session=session, row_factory=InitiativeTable)
    result = await use_case.execute(
        CreateInitiativeCommand(
            owner_tenant_id=UUID(organization_id),
            title=payload.title,
            description=payload.description,
            owner_id=UUID(payload.owner_id),
            priority=payload.priority,
            time_horizon=payload.time_horizon,
            start_date=payload.start_date,
            target_end_date=payload.target_end_date,
            source_artifact_id=UUID(payload.source_artifact_id) if payload.source_artifact_id else None,
            update_cadence_days=payload.update_cadence_days,
        )
    )
    await session.flush()
    row = await get_initiative_by_id(session, result.initiative_id)
    return _initiative_to_response(row)  # type: ignore[arg-type]


@router.get("/initiatives/{initiative_id}")
async def get_initiative(
    initiative_id: str,
    session: SessionDep,
) -> InitiativeDetailResponse:
    adapter = _InitiativeQueryAdapter(session)
    row = await adapter.get_by_id(UUID(initiative_id))
    if row is None:
        raise HTTPException(status_code=404, detail="Initiative not found")
    milestones = await adapter.get_milestones(UUID(initiative_id))
    return InitiativeDetailResponse(
        initiative=_initiative_to_response(row),
        milestones=[_milestone_to_response(m) for m in milestones],
    )


@router.patch("/initiatives/{initiative_id}")
async def update_initiative(
    initiative_id: str,
    payload: UpdateInitiativeRequest,
    session: SessionDep,
) -> InitiativeResponse:
    adapter = _InitiativeQueryAdapter(session)
    use_case = UpdateInitiative(query_port=adapter)
    await use_case.execute(
        UpdateInitiativeCommand(
            initiative_id=UUID(initiative_id),
            title=payload.title,
            description=payload.description,
            status=payload.status,
            priority=payload.priority,
            time_horizon=payload.time_horizon,
            start_date=payload.start_date,
            target_end_date=payload.target_end_date,
            update_cadence_days=payload.update_cadence_days,
        )
    )
    await session.flush()
    row = await adapter.get_by_id(UUID(initiative_id))
    return _initiative_to_response(row)  # type: ignore[arg-type]


@router.delete("/initiatives/{initiative_id}", status_code=204)
async def delete_initiative(
    initiative_id: str,
    session: SessionDep,
) -> None:
    adapter = _InitiativeQueryAdapter(session)
    use_case = UpdateInitiative(query_port=adapter)
    await use_case.execute(
        UpdateInitiativeCommand(
            initiative_id=UUID(initiative_id),
            status=InitiativeStatus.CANCELLED,
        )
    )


@router.post("/initiatives/{initiative_id}/milestones", status_code=201)
async def create_milestone(
    initiative_id: str,
    payload: CreateMilestoneRequest,
    session: SessionDep,
) -> MilestoneResponse:
    use_case = CreateMilestone(session=session, row_factory=MilestoneTable)
    result = await use_case.execute(
        CreateMilestoneCommand(
            initiative_id=UUID(initiative_id),
            title=payload.title,
            target_date=payload.target_date,
            description=payload.description,
            assignee_id=UUID(payload.assignee_id) if payload.assignee_id else None,
            sort_order=payload.sort_order,
        )
    )
    await session.flush()
    ms_adapter = _MilestoneQueryAdapter(session)
    row = await ms_adapter.get_by_id(result.milestone_id)
    return _milestone_to_response(row)  # type: ignore[arg-type]


@router.patch("/milestones/{milestone_id}")
async def update_milestone(
    milestone_id: str,
    payload: UpdateMilestoneRequest,
    session: SessionDep,
) -> MilestoneResponse:
    adapter = _MilestoneQueryAdapter(session)
    use_case = UpdateMilestone(query_port=adapter)
    await use_case.execute(
        UpdateMilestoneCommand(
            milestone_id=UUID(milestone_id),
            title=payload.title,
            description=payload.description,
            assignee_id=UUID(payload.assignee_id) if payload.assignee_id else None,
            status=payload.status,
            target_date=payload.target_date,
            completed_date=payload.completed_date,
            sort_order=payload.sort_order,
        )
    )
    await session.flush()
    row = await adapter.get_by_id(UUID(milestone_id))
    return _milestone_to_response(row)  # type: ignore[arg-type]


@router.delete("/milestones/{milestone_id}", status_code=204)
async def delete_milestone(
    milestone_id: str,
    session: SessionDep,
) -> None:
    adapter = _MilestoneQueryAdapter(session)
    row = await adapter.get_by_id(UUID(milestone_id))
    if row is None:
        raise HTTPException(status_code=404, detail="Milestone not found")
    await session.delete(row)
```

- [ ] **Step 3: Register router in main.py**

Add import at `apps/api/src/prescient/main.py:38` (after the sponsor import):

```python
from prescient.strategy.api.routes import router as strategy_router
```

Add `app.include_router(strategy_router)` after line 135 (after `research_pipeline_router`).

- [ ] **Step 4: Verify API starts**

Run: `cd apps/api && docker compose exec api python -c "from prescient.main import app; print('Routes:', [r.path for r in app.routes if hasattr(r, 'path') and 'strategy' in r.path])"`
Expected: List of `/strategy/*` paths

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/strategy/api/ apps/api/src/prescient/main.py
git commit -m "feat(strategy): add API routes for initiative and milestone CRUD"
```

---

### Task 9: Strategy Timeline Page (Frontend)

**Files:**
- Create: `apps/web/src/app/(main)/strategy/page.tsx`
- Modify: `apps/web/src/components/nav.tsx` — add Strategy nav link

- [ ] **Step 1: Add nav link**

In `apps/web/src/components/nav.tsx`, add to `NAV_LINKS` array after the Knowledge entry:

```typescript
{ href: "/strategy", label: "Strategy" },
```

- [ ] **Step 2: Install Gantt chart library**

Run: `cd apps/web && npm install gantt-task-react`

If `gantt-task-react` doesn't work well with Next.js/React 18+, fallback:
Run: `cd apps/web && npm install frappe-gantt`

Note: The implementing engineer should evaluate the library and switch if needed. The key requirement is horizontal bars with milestone markers, colored by health status.

- [ ] **Step 3: Create the strategy page**

```tsx
// apps/web/src/app/(main)/strategy/page.tsx
"use client";

import { useEffect, useState } from "react";
import { getOrgId } from "@/lib/auth";
import { apiFetch } from "@/lib/api-client";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";

interface Milestone {
  id: string;
  initiative_id: string;
  title: string;
  status: string;
  target_date: string;
  completed_date: string | null;
  assignee_id: string | null;
  sort_order: number;
}

interface Initiative {
  id: string;
  title: string;
  description: string;
  owner_id: string;
  status: string;
  priority: string;
  time_horizon: string;
  start_date: string;
  target_end_date: string;
  health_score: number | null;
  completion_pct: number | null;
  update_cadence_days: number;
  created_at: string;
}

interface InitiativeDetail extends Initiative {
  milestones: Milestone[];
}

const HEALTH_COLORS: Record<string, string> = {
  green: "bg-green-500",
  yellow: "bg-amber-400",
  red: "bg-red-500",
  gray: "bg-neutral-400",
};

const STATUS_LABELS: Record<string, string> = {
  draft: "Draft",
  active: "Active",
  at_risk: "At Risk",
  on_hold: "On Hold",
  completed: "Completed",
  cancelled: "Cancelled",
};

const PRIORITY_STYLES: Record<string, string> = {
  critical: "bg-red-50 text-red-700 border border-red-200",
  high: "bg-orange-50 text-orange-700 border border-orange-200",
  medium: "bg-amber-50 text-amber-700 border border-amber-200",
  low: "bg-green-50 text-green-700 border border-green-200",
};

function healthColor(score: number | null, status: string): string {
  if (status === "on_hold" || status === "cancelled") return "gray";
  if (score === null) return "gray";
  if (score >= 0.7) return "green";
  if (score >= 0.4) return "yellow";
  return "red";
}

function Pill({ label, className }: { label: string; className: string }) {
  return (
    <span className={`inline-flex items-center rounded-md px-2 py-0.5 text-xs font-semibold capitalize ${className}`}>
      {label}
    </span>
  );
}

function SummaryBar({ initiatives }: { initiatives: Initiative[] }) {
  const counts = initiatives.reduce(
    (acc, i) => {
      acc[i.status] = (acc[i.status] || 0) + 1;
      return acc;
    },
    {} as Record<string, number>,
  );

  return (
    <div className="flex items-center gap-6 rounded-lg border border-neutral-200 bg-white px-6 py-3 text-sm">
      {Object.entries(counts).map(([status, count]) => (
        <div key={status} className="flex items-center gap-2">
          <span className="font-semibold text-neutral-900">{count}</span>
          <span className="text-neutral-500">{STATUS_LABELS[status] ?? status}</span>
        </div>
      ))}
    </div>
  );
}

function InitiativeRow({
  initiative,
  onClick,
}: {
  initiative: Initiative;
  onClick: () => void;
}) {
  const color = healthColor(initiative.health_score, initiative.status);
  return (
    <button
      onClick={onClick}
      className="flex w-full items-center gap-4 rounded-lg border border-neutral-200 bg-white px-4 py-3 text-left transition-colors hover:bg-neutral-50"
    >
      <div className={`h-3 w-3 rounded-full ${HEALTH_COLORS[color]}`} />
      <div className="min-w-0 flex-1">
        <p className="truncate font-medium text-neutral-900">{initiative.title}</p>
        <p className="truncate text-sm text-neutral-500">{initiative.description}</p>
      </div>
      <Pill label={initiative.priority} className={PRIORITY_STYLES[initiative.priority] ?? ""} />
      <span className="text-xs text-neutral-400">
        {initiative.start_date} — {initiative.target_end_date}
      </span>
    </button>
  );
}

export default function StrategyPage() {
  const [initiatives, setInitiatives] = useState<Initiative[]>([]);
  const [selected, setSelected] = useState<InitiativeDetail | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const orgId = getOrgId();
    if (!orgId) return;

    apiFetch<Initiative[]>(`/strategy/initiatives?organization_id=${orgId}`)
      .then(setInitiatives)
      .catch(console.error)
      .finally(() => setLoading(false));
  }, []);

  async function selectInitiative(id: string) {
    const detail = await apiFetch<{ initiative: Initiative; milestones: Milestone[] }>(
      `/strategy/initiatives/${id}`,
    );
    setSelected({ ...detail.initiative, milestones: detail.milestones });
  }

  if (loading) {
    return (
      <div className="mx-auto max-w-6xl px-6 py-8">
        <p className="text-neutral-500">Loading strategy...</p>
      </div>
    );
  }

  return (
    <div className="mx-auto max-w-6xl px-6 py-8">
      <div className="mb-6 flex items-center justify-between">
        <h1 className="text-2xl font-bold text-neutral-900">Strategy</h1>
        <Button variant="default">New Initiative</Button>
      </div>

      <SummaryBar initiatives={initiatives} />

      <div className="mt-6 flex gap-6">
        {/* Initiative list */}
        <div className="flex-1 space-y-2">
          {initiatives.length === 0 ? (
            <p className="py-12 text-center text-neutral-400">
              No initiatives yet. Create one or extract from a document.
            </p>
          ) : (
            initiatives.map((i) => (
              <InitiativeRow
                key={i.id}
                initiative={i}
                onClick={() => selectInitiative(i.id)}
              />
            ))
          )}
        </div>

        {/* Detail panel */}
        {selected && (
          <div className="w-96 shrink-0">
            <Card>
              <CardHeader>
                <div className="flex items-center gap-2">
                  <div
                    className={`h-3 w-3 rounded-full ${HEALTH_COLORS[healthColor(selected.health_score, selected.status)]}`}
                  />
                  <CardTitle className="text-lg">{selected.title}</CardTitle>
                </div>
                <div className="flex gap-2 pt-1">
                  <Pill label={selected.status} className="bg-neutral-100 text-neutral-700" />
                  <Pill
                    label={selected.priority}
                    className={PRIORITY_STYLES[selected.priority] ?? ""}
                  />
                </div>
              </CardHeader>
              <CardContent className="space-y-4">
                <p className="text-sm text-neutral-600">{selected.description}</p>

                <div>
                  <h3 className="mb-2 text-sm font-semibold text-neutral-900">Milestones</h3>
                  {selected.milestones.length === 0 ? (
                    <p className="text-sm text-neutral-400">No milestones</p>
                  ) : (
                    <ul className="space-y-1">
                      {selected.milestones.map((m) => (
                        <li key={m.id} className="flex items-center justify-between text-sm">
                          <span className="text-neutral-700">{m.title}</span>
                          <span className="text-xs text-neutral-400">{m.target_date}</span>
                        </li>
                      ))}
                    </ul>
                  )}
                </div>

                <div className="text-xs text-neutral-400">
                  {selected.start_date} — {selected.target_end_date} · {selected.time_horizon}
                </div>
              </CardContent>
            </Card>
          </div>
        )}
      </div>
    </div>
  );
}
```

Note: This is the initial list + detail view. The full Gantt timeline chart will be added in Slice 3 (Task 15) when dependencies and resource overlays are available. This gives a usable, demoable page immediately.

- [ ] **Step 4: Verify the page loads**

Run: `cd apps/web && npm run dev`
Navigate to `http://localhost:3000/strategy`
Expected: Page loads with "Strategy" header, summary bar, empty state or initiative list.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/app/\(main\)/strategy/ apps/web/src/components/nav.tsx
git commit -m "feat(web): add /strategy page with initiative list, detail panel, and nav link"
```

---

## Slice 2: Document Extraction

### Task 10: InitiativeExtractionPort and LLM Adapter

**Files:**
- Create: `apps/api/src/prescient/strategy/infrastructure/extraction_adapter.py`
- Create: `apps/api/src/prescient/strategy/application/use_cases/extract_initiatives.py`
- Test: `apps/api/tests/unit/strategy/test_extract_initiatives.py`

- [ ] **Step 1: Write failing test**

```python
# apps/api/tests/unit/strategy/test_extract_initiatives.py
"""Unit tests for ExtractInitiatives use case."""

from __future__ import annotations

from datetime import date
from uuid import uuid4

import pytest

from prescient.strategy.application.use_cases.extract_initiatives import (
    DraftInitiative,
    ExtractionContext,
    ExtractionResult,
    ExtractInitiatives,
)


class _FakeExtractionPort:
    def __init__(self, result: ExtractionResult) -> None:
        self._result = result

    async def extract_initiatives(
        self, document_text: str, context: ExtractionContext
    ) -> ExtractionResult:
        return self._result


class TestExtractInitiatives:
    @pytest.mark.asyncio
    async def test_returns_drafts_from_port(self) -> None:
        drafts = [
            DraftInitiative(
                title="Margin Expansion",
                title_confidence=0.95,
                description="Drive margin improvement",
                description_confidence=0.9,
                priority=None,
                priority_confidence=0.0,
                time_horizon=None,
                time_horizon_confidence=0.0,
                start_date=None,
                target_end_date=None,
                milestones=[],
            ),
        ]
        result = ExtractionResult(draft_initiatives=drafts, gaps=[], metadata={})
        port = _FakeExtractionPort(result)
        use_case = ExtractInitiatives(extraction_port=port)

        output = await use_case.execute(
            document_text="Some VCP content...",
            context=ExtractionContext(organization_id="org-1", existing_titles=[]),
        )

        assert len(output.draft_initiatives) == 1
        assert output.draft_initiatives[0].title == "Margin Expansion"

    @pytest.mark.asyncio
    async def test_empty_document_returns_empty(self) -> None:
        result = ExtractionResult(draft_initiatives=[], gaps=[], metadata={})
        port = _FakeExtractionPort(result)
        use_case = ExtractInitiatives(extraction_port=port)

        output = await use_case.execute(
            document_text="",
            context=ExtractionContext(organization_id="org-1", existing_titles=[]),
        )

        assert len(output.draft_initiatives) == 0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_extract_initiatives.py -v`
Expected: FAIL

- [ ] **Step 3: Implement extraction use case and types**

```python
# apps/api/src/prescient/strategy/application/use_cases/extract_initiatives.py
"""ExtractInitiatives use case — delegates to LLM extraction port."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import date
from typing import Any, Protocol


@dataclass(frozen=True)
class DraftMilestone:
    title: str
    target_date: date | None = None


@dataclass(frozen=True)
class DraftInitiative:
    title: str | None
    title_confidence: float
    description: str | None
    description_confidence: float
    priority: str | None
    priority_confidence: float
    time_horizon: str | None
    time_horizon_confidence: float
    start_date: date | None = None
    target_end_date: date | None = None
    milestones: list[DraftMilestone] = field(default_factory=list)


@dataclass(frozen=True)
class GapQuestion:
    field_name: str
    question: str
    draft_index: int


@dataclass(frozen=True)
class ExtractionContext:
    organization_id: str
    existing_titles: list[str]


@dataclass(frozen=True)
class ExtractionResult:
    draft_initiatives: list[DraftInitiative]
    gaps: list[GapQuestion]
    metadata: dict[str, Any]


class InitiativeExtractionPort(Protocol):
    async def extract_initiatives(
        self, document_text: str, context: ExtractionContext
    ) -> ExtractionResult: ...


class ExtractInitiatives:
    def __init__(self, extraction_port: InitiativeExtractionPort) -> None:
        self._port = extraction_port

    async def execute(
        self, document_text: str, context: ExtractionContext
    ) -> ExtractionResult:
        return await self._port.extract_initiatives(document_text, context)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_extract_initiatives.py -v`
Expected: PASS

- [ ] **Step 5: Implement LLM extraction adapter**

```python
# apps/api/src/prescient/strategy/infrastructure/extraction_adapter.py
"""LLM-based initiative extraction adapter."""

from __future__ import annotations

import json
from datetime import date

from anthropic import AsyncAnthropic

from prescient.strategy.application.use_cases.extract_initiatives import (
    DraftInitiative,
    DraftMilestone,
    ExtractionContext,
    ExtractionResult,
    GapQuestion,
)

_SYSTEM_PROMPT = """\
You are extracting strategic initiatives from a document. For each initiative found, return:
- title, description, priority (critical/high/medium/low), time_horizon (q1/q2/q3/q4/annual/multi_year)
- start_date and target_end_date if mentioned (ISO format)
- milestones with title and target_date if mentioned
- confidence (0.0-1.0) for each field

Return JSON: {"initiatives": [...], "gaps": [...]}
Each initiative: {"title": str, "title_confidence": float, "description": str, "description_confidence": float, "priority": str|null, "priority_confidence": float, "time_horizon": str|null, "time_horizon_confidence": float, "start_date": str|null, "target_end_date": str|null, "milestones": [{"title": str, "target_date": str|null}]}
Each gap: {"field_name": str, "question": str, "draft_index": int}

Existing initiative titles (avoid duplicates): {existing_titles}
"""


class AnthropicExtractionAdapter:
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514") -> None:
        self._client = AsyncAnthropic(api_key=api_key)
        self._model = model

    async def extract_initiatives(
        self, document_text: str, context: ExtractionContext
    ) -> ExtractionResult:
        system = _SYSTEM_PROMPT.format(existing_titles=context.existing_titles)
        response = await self._client.messages.create(
            model=self._model,
            max_tokens=4096,
            system=system,
            messages=[{"role": "user", "content": document_text}],
        )
        raw = response.content[0].text
        data = json.loads(raw)

        drafts = []
        for item in data.get("initiatives", []):
            milestones = [
                DraftMilestone(
                    title=m["title"],
                    target_date=date.fromisoformat(m["target_date"]) if m.get("target_date") else None,
                )
                for m in item.get("milestones", [])
            ]
            drafts.append(
                DraftInitiative(
                    title=item.get("title"),
                    title_confidence=item.get("title_confidence", 0.0),
                    description=item.get("description"),
                    description_confidence=item.get("description_confidence", 0.0),
                    priority=item.get("priority"),
                    priority_confidence=item.get("priority_confidence", 0.0),
                    time_horizon=item.get("time_horizon"),
                    time_horizon_confidence=item.get("time_horizon_confidence", 0.0),
                    start_date=date.fromisoformat(item["start_date"]) if item.get("start_date") else None,
                    target_end_date=date.fromisoformat(item["target_end_date"]) if item.get("target_end_date") else None,
                    milestones=milestones,
                )
            )

        gaps = [
            GapQuestion(
                field_name=g["field_name"],
                question=g["question"],
                draft_index=g["draft_index"],
            )
            for g in data.get("gaps", [])
        ]

        return ExtractionResult(
            draft_initiatives=drafts,
            gaps=gaps,
            metadata={"model": self._model, "usage": {"input_tokens": response.usage.input_tokens, "output_tokens": response.usage.output_tokens}},
        )
```

- [ ] **Step 6: Add extraction endpoint to routes**

Add to `apps/api/src/prescient/strategy/api/routes.py`:

```python
# At the top, add imports:
from prescient.config import get_settings
from prescient.strategy.application.use_cases.extract_initiatives import (
    ExtractionContext,
    ExtractionResult,
    ExtractInitiatives,
    DraftInitiative,
    GapQuestion,
)
from prescient.strategy.infrastructure.extraction_adapter import AnthropicExtractionAdapter

# Request/response models:
class ExtractRequest(BaseModel):
    document_text: str
    organization_id: str

class DraftInitiativeResponse(BaseModel):
    title: str | None
    title_confidence: float
    description: str | None
    description_confidence: float
    priority: str | None
    priority_confidence: float
    time_horizon: str | None
    time_horizon_confidence: float
    start_date: date | None
    target_end_date: date | None
    milestones: list[dict]

class ExtractionResponse(BaseModel):
    draft_initiatives: list[DraftInitiativeResponse]
    gaps: list[dict]

# Endpoint:
@router.post("/initiatives/extract")
async def extract_initiatives(
    payload: ExtractRequest,
    session: SessionDep,
) -> ExtractionResponse:
    settings = get_settings()
    adapter = AnthropicExtractionAdapter(
        api_key=settings.anthropic_api_key,
        model=settings.anthropic_model,
    )
    use_case = ExtractInitiatives(extraction_port=adapter)

    existing = await get_initiatives_for_tenant(session, payload.organization_id)
    existing_titles = [r.title for r in existing]

    result = await use_case.execute(
        document_text=payload.document_text,
        context=ExtractionContext(
            organization_id=payload.organization_id,
            existing_titles=existing_titles,
        ),
    )

    return ExtractionResponse(
        draft_initiatives=[
            DraftInitiativeResponse(
                title=d.title,
                title_confidence=d.title_confidence,
                description=d.description,
                description_confidence=d.description_confidence,
                priority=d.priority,
                priority_confidence=d.priority_confidence,
                time_horizon=d.time_horizon,
                time_horizon_confidence=d.time_horizon_confidence,
                start_date=d.start_date,
                target_end_date=d.target_end_date,
                milestones=[{"title": m.title, "target_date": str(m.target_date) if m.target_date else None} for m in d.milestones],
            )
            for d in result.draft_initiatives
        ],
        gaps=[{"field_name": g.field_name, "question": g.question, "draft_index": g.draft_index} for g in result.gaps],
    )
```

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/strategy/application/use_cases/extract_initiatives.py apps/api/src/prescient/strategy/infrastructure/extraction_adapter.py apps/api/tests/unit/strategy/test_extract_initiatives.py apps/api/src/prescient/strategy/api/routes.py
git commit -m "feat(strategy): add document-to-initiative extraction with LLM adapter"
```

---

## Slice 3: Dependencies + Resources

### Task 11: Dependency and ResourceAssignment Domain + Tables

**Files:**
- Create: `apps/api/src/prescient/strategy/domain/entities/dependency.py`
- Create: `apps/api/src/prescient/strategy/domain/entities/resource_assignment.py`
- Create: `apps/api/src/prescient/strategy/domain/entities/evidence.py`
- Create: `apps/api/src/prescient/strategy/infrastructure/tables/dependency.py`
- Create: `apps/api/src/prescient/strategy/infrastructure/tables/resource_assignment.py`
- Create: `apps/api/src/prescient/strategy/infrastructure/tables/evidence.py`

Follow the same patterns as Tasks 2 and 4. Each entity is a frozen Pydantic model. Each table uses the `strategy` schema. See the spec for exact fields.

- [ ] **Step 1: Create domain entities** (dependency.py, resource_assignment.py, evidence.py)
- [ ] **Step 2: Create ORM tables** (dependency.py, resource_assignment.py, evidence.py)
- [ ] **Step 3: Add tables to migration** — extend `20260413_strategy_schema.py` or create a follow-up migration with the three new tables (dependencies, resource_assignments, evidence)
- [ ] **Step 4: Add queries** — extend `queries.py` with `get_dependencies_for_initiative`, `get_resources_for_initiative`, `get_resource_conflicts`, `get_evidence_for_initiative`
- [ ] **Step 5: Run migration and verify**
- [ ] **Step 6: Commit**

```bash
git commit -m "feat(strategy): add dependency, resource assignment, and evidence domain + ORM + migration"
```

---

### Task 12: Dependency + Resource CRUD Use Cases

**Files:**
- Create: `apps/api/src/prescient/strategy/application/use_cases/create_dependency.py`
- Create: `apps/api/src/prescient/strategy/application/use_cases/create_resource.py`
- Create: `apps/api/src/prescient/strategy/application/use_cases/create_evidence.py`
- Test: `apps/api/tests/unit/strategy/test_create_dependency.py`
- Test: `apps/api/tests/unit/strategy/test_create_resource.py`
- Test: `apps/api/tests/unit/strategy/test_create_evidence.py`

Follow the same TDD pattern as Tasks 5-7. Each use case takes a session port and row factory, creates a row. Test with `_FakeSession` and `_FakeRow`.

- [ ] **Step 1: Write failing test for CreateDependency**
- [ ] **Step 2: Implement CreateDependency**
- [ ] **Step 3: Run tests, verify pass**
- [ ] **Step 4: Write failing test for CreateResource**
- [ ] **Step 5: Implement CreateResource** — include resource conflict check: sum allocation_pct per resource_id across active initiatives with overlapping dates, raise warning if > 100%
- [ ] **Step 6: Run tests, verify pass**
- [ ] **Step 7: Write failing test for CreateEvidence**
- [ ] **Step 8: Implement CreateEvidence**
- [ ] **Step 9: Run tests, verify pass**
- [ ] **Step 10: Commit**

```bash
git commit -m "feat(strategy): add dependency, resource, and evidence CRUD use cases with TDD"
```

---

### Task 13: Dependency + Resource + Evidence API Routes

**Files:**
- Modify: `apps/api/src/prescient/strategy/api/routes.py` — add endpoints

Add the following endpoints to the existing routes file:

- `POST /strategy/dependencies` — create dependency
- `DELETE /strategy/dependencies/{id}` — remove dependency
- `GET /strategy/initiatives/{id}/dependencies` — list upstream + downstream
- `POST /strategy/initiatives/{id}/resources` — assign resource
- `PATCH /strategy/resources/{id}` — update allocation
- `DELETE /strategy/resources/{id}` — remove
- `GET /strategy/resources/conflicts` — list active conflicts for org
- `POST /strategy/initiatives/{id}/evidence` — link evidence
- `DELETE /strategy/evidence/{id}` — unlink

Follow the same composition root pattern from Task 8.

- [ ] **Step 1: Add request/response models for dependencies, resources, evidence**
- [ ] **Step 2: Add adapter classes**
- [ ] **Step 3: Add endpoints**
- [ ] **Step 4: Verify API starts**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat(strategy): add API routes for dependencies, resources, and evidence"
```

---

### Task 14: Timeline Endpoint

**Files:**
- Modify: `apps/api/src/prescient/strategy/api/routes.py`
- Modify: `apps/api/src/prescient/strategy/queries.py`

- [ ] **Step 1: Add timeline query** — optimized query joining initiatives + milestones + dependencies + resource conflicts into a single response

```python
# Add to queries.py
async def get_timeline_data(
    session: AsyncSession,
    owner_tenant_id: str,
) -> dict:
    """Return all data needed for the timeline chart in one query batch."""
    initiatives = await get_initiatives_for_tenant(
        session, owner_tenant_id, status_filter=["active", "at_risk", "on_hold", "draft"]
    )

    initiative_ids = [i.id for i in initiatives]
    if not initiative_ids:
        return {"initiatives": [], "milestones": [], "dependencies": [], "conflicts": []}

    # Milestones for all initiatives
    from prescient.strategy.infrastructure.tables.milestone import MilestoneTable
    from prescient.strategy.infrastructure.tables.dependency import DependencyTable

    ms_result = await session.execute(
        select(MilestoneTable)
        .where(MilestoneTable.initiative_id.in_(initiative_ids))
        .order_by(MilestoneTable.sort_order)
    )
    milestones = list(ms_result.scalars().all())

    dep_result = await session.execute(
        select(DependencyTable).where(
            DependencyTable.upstream_initiative_id.in_(initiative_ids)
            | DependencyTable.downstream_initiative_id.in_(initiative_ids)
        )
    )
    dependencies = list(dep_result.scalars().all())

    return {
        "initiatives": initiatives,
        "milestones": milestones,
        "dependencies": dependencies,
    }
```

- [ ] **Step 2: Add timeline endpoint**

```python
@router.get("/timeline")
async def get_timeline(
    organization_id: str,
    session: SessionDep,
) -> dict:
    data = await get_timeline_data(session, organization_id)
    return {
        "initiatives": [_initiative_to_response(i) for i in data["initiatives"]],
        "milestones": [_milestone_to_response(m) for m in data["milestones"]],
        "dependencies": [
            {
                "id": str(d.id),
                "upstream_initiative_id": str(d.upstream_initiative_id),
                "downstream_initiative_id": str(d.downstream_initiative_id),
                "dependency_type": d.dependency_type,
            }
            for d in data["dependencies"]
        ],
    }
```

- [ ] **Step 3: Commit**

```bash
git commit -m "feat(strategy): add /strategy/timeline optimized endpoint"
```

---

### Task 15: Enhanced Timeline UI with Gantt Chart

**Files:**
- Modify: `apps/web/src/app/(main)/strategy/page.tsx` — integrate Gantt chart

- [ ] **Step 1: Update page to use timeline endpoint and render Gantt chart**

Replace the simple list with a Gantt-style view using the installed charting library. The chart should show:
- Horizontal bars per initiative, colored by health
- Diamond markers for milestones
- Dependency arrows between bars (solid for `blocks`, dashed for `informs`, dotted for `shares_resources`)
- Today line
- Click a bar → show detail panel

The exact implementation depends on the Gantt library chosen in Task 9. The engineer should:
1. Fetch from `/strategy/timeline?organization_id=...`
2. Map initiatives to Gantt tasks (start_date, target_end_date, progress from completion_pct)
3. Map milestones as sub-items or markers
4. Render dependency arrows

- [ ] **Step 2: Test in browser** — verify chart renders, bars are colored, milestones visible, click opens detail
- [ ] **Step 3: Commit**

```bash
git commit -m "feat(web): add Gantt timeline chart to /strategy page"
```

---

## Slice 4: Notifications + Cadence Engine

### Task 16: Notifications Domain + Tables + Migration

**Files:**
- Create: `apps/api/src/prescient/notifications/__init__.py`
- Create: `apps/api/src/prescient/notifications/domain/__init__.py`
- Create: `apps/api/src/prescient/notifications/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/notifications/domain/entities/notification.py`
- Create: `apps/api/src/prescient/notifications/domain/enums.py`
- Create: `apps/api/src/prescient/notifications/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/notifications/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/notifications/infrastructure/tables/notification.py`
- Create: `apps/api/src/prescient/notifications/queries.py`
- Create: `apps/api/alembic/versions/20260413_notifications_schema.py`
- Modify: `apps/api/src/prescient/shared/metadata.py` — register notification tables

- [ ] **Step 1: Create package structure** (all `__init__.py` files)

- [ ] **Step 2: Create domain enums**

```python
# apps/api/src/prescient/notifications/domain/enums.py
"""Notification context enums."""

from __future__ import annotations

from enum import StrEnum


class NotificationCategory(StrEnum):
    STRATEGY = "strategy"
    ACTION_ITEM = "action_item"
    TRIAGE = "triage"
    MONITORING = "monitoring"
    SYSTEM = "system"


class NotificationPriority(StrEnum):
    URGENT = "urgent"
    HIGH = "high"
    NORMAL = "normal"
    LOW = "low"
```

- [ ] **Step 3: Create Notification entity**

```python
# apps/api/src/prescient/notifications/domain/entities/notification.py
"""Notification domain entity."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field

from prescient.notifications.domain.enums import NotificationCategory, NotificationPriority


class Notification(BaseModel):
    model_config = ConfigDict(frozen=True, strict=True)

    id: UUID
    recipient_id: UUID
    organization_id: UUID
    category: NotificationCategory
    priority: NotificationPriority
    title: str = Field(min_length=1, max_length=256)
    body: str = Field(min_length=1)
    source_type: str
    source_id: UUID
    action_url: str | None = None
    read: bool = False
    dismissed: bool = False
    read_at: datetime | None = None
    created_at: datetime
```

- [ ] **Step 4: Create ORM table**

```python
# apps/api/src/prescient/notifications/infrastructure/tables/notification.py
"""`notifications.notifications` — unified notification persistence."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from sqlalchemy import Boolean, DateTime, Index, String, Text, func
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class NotificationTable(Base):
    __tablename__ = "notifications"
    __table_args__ = (
        Index("ix_notifications_recipient_read_created", "recipient_id", "read", "created_at"),
        {"schema": "notifications"},
    )  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    recipient_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False, index=True)
    organization_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False, index=True)
    category: Mapped[str] = mapped_column(String(32), nullable=False, index=True)
    priority: Mapped[str] = mapped_column(String(16), nullable=False)
    title: Mapped[str] = mapped_column(String(256), nullable=False)
    body: Mapped[str] = mapped_column(Text, nullable=False)
    source_type: Mapped[str] = mapped_column(String(32), nullable=False)
    source_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False)
    action_url: Mapped[str | None] = mapped_column(String(512), nullable=True)
    read: Mapped[bool] = mapped_column(Boolean(), nullable=False, server_default="false")
    dismissed: Mapped[bool] = mapped_column(Boolean(), nullable=False, server_default="false")
    read_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=func.now(), index=True)
```

- [ ] **Step 5: Register in metadata, create migration, apply**
- [ ] **Step 6: Commit**

```bash
git commit -m "feat(notifications): add domain, ORM tables, and Alembic migration"
```

---

### Task 17: Notification Use Cases + API Routes

**Files:**
- Create: `apps/api/src/prescient/notifications/application/__init__.py`
- Create: `apps/api/src/prescient/notifications/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/notifications/application/use_cases/publish_notification.py`
- Create: `apps/api/src/prescient/notifications/application/use_cases/mark_read.py`
- Create: `apps/api/src/prescient/notifications/api/__init__.py`
- Create: `apps/api/src/prescient/notifications/api/routes.py`
- Modify: `apps/api/src/prescient/main.py` — register notification router
- Test: `apps/api/tests/unit/notifications/__init__.py`
- Test: `apps/api/tests/unit/notifications/test_publish_notification.py`

- [ ] **Step 1: Write failing test for PublishNotification**
- [ ] **Step 2: Implement PublishNotification use case**
- [ ] **Step 3: Implement MarkRead use case**
- [ ] **Step 4: Implement notification routes** (`GET /notifications`, `GET /notifications/count`, `PATCH /notifications/{id}`, `POST /notifications/mark-all-read`)
- [ ] **Step 5: Register router in main.py**
- [ ] **Step 6: Run tests, verify pass**
- [ ] **Step 7: Commit**

```bash
git commit -m "feat(notifications): add publish/mark-read use cases and API routes"
```

---

### Task 18: Cadence Engine

**Files:**
- Create: `apps/api/src/prescient/strategy/application/cadence_engine.py`
- Create: `apps/api/src/prescient/strategy/infrastructure/tables/approval_request.py`
- Create: `apps/api/src/prescient/strategy/domain/entities/approval_request.py`
- Test: `apps/api/tests/unit/strategy/test_cadence_engine.py`

- [ ] **Step 1: Write failing test for cadence engine**

```python
# apps/api/tests/unit/strategy/test_cadence_engine.py
"""Unit tests for the cadence engine."""

from __future__ import annotations

from datetime import UTC, date, datetime, timedelta
from uuid import uuid4

import pytest

from prescient.strategy.application.cadence_engine import (
    CadenceEngine,
    CadenceNotification,
)
from prescient.strategy.domain.enums import InitiativeStatus, MilestoneStatus


NOW = datetime(2026, 4, 13, 12, 0, 0, tzinfo=UTC)


class _FakeInitiativeRow:
    def __init__(self, **kwargs: object) -> None:
        for k, v in kwargs.items():
            setattr(self, k, v)


class _FakeMilestoneRow:
    def __init__(self, **kwargs: object) -> None:
        for k, v in kwargs.items():
            setattr(self, k, v)


class _FakeDataPort:
    def __init__(
        self,
        initiatives: list[_FakeInitiativeRow],
        milestones: dict[str, list[_FakeMilestoneRow]],
    ) -> None:
        self._initiatives = initiatives
        self._milestones = milestones

    async def get_active_initiatives(self, org_id: str) -> list[_FakeInitiativeRow]:
        return self._initiatives

    async def get_milestones(self, initiative_id: object) -> list[_FakeMilestoneRow]:
        return self._milestones.get(str(initiative_id), [])

    async def get_dependencies_downstream(self, initiative_id: object) -> list:
        return []


class TestCadenceEngine:
    @pytest.mark.asyncio
    async def test_stale_initiative_generates_notification(self) -> None:
        stale = _FakeInitiativeRow(
            id=uuid4(),
            owner_id=uuid4(),
            status="active",
            update_cadence_days=7,
            last_updated_at=NOW - timedelta(days=15),
            health_score=None,
        )
        engine = CadenceEngine(data_port=_FakeDataPort([stale], {}), now=NOW)
        notifications = await engine.run("org-1")

        assert any(n.title.startswith("Stale") or "no update" in n.title.lower() for n in notifications)

    @pytest.mark.asyncio
    async def test_approaching_milestone_generates_notification(self) -> None:
        init_id = uuid4()
        assignee_id = uuid4()
        initiative = _FakeInitiativeRow(
            id=init_id,
            owner_id=uuid4(),
            status="active",
            update_cadence_days=7,
            last_updated_at=NOW,
            health_score=None,
        )
        milestone = _FakeMilestoneRow(
            id=uuid4(),
            initiative_id=init_id,
            title="Ship v1",
            status="pending",
            target_date=(NOW + timedelta(days=3)).date(),
            assignee_id=assignee_id,
        )
        engine = CadenceEngine(
            data_port=_FakeDataPort([initiative], {str(init_id): [milestone]}),
            now=NOW,
        )
        notifications = await engine.run("org-1")

        assert any(n.recipient_id == assignee_id for n in notifications)

    @pytest.mark.asyncio
    async def test_no_notifications_when_everything_healthy(self) -> None:
        initiative = _FakeInitiativeRow(
            id=uuid4(),
            owner_id=uuid4(),
            status="active",
            update_cadence_days=7,
            last_updated_at=NOW,
            health_score=None,
        )
        engine = CadenceEngine(data_port=_FakeDataPort([initiative], {}), now=NOW)
        notifications = await engine.run("org-1")

        assert len(notifications) == 0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_cadence_engine.py -v`
Expected: FAIL

- [ ] **Step 3: Implement cadence engine**

```python
# apps/api/src/prescient/strategy/application/cadence_engine.py
"""Cadence engine — scheduled process for staleness, risk, and conflict detection."""

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Any, Protocol
from uuid import UUID, uuid4

from prescient.strategy.domain.enums import InitiativeStatus, MilestoneStatus

_APPROACHING_DAYS = 7
_STALENESS_MULTIPLIER = 1.5


@dataclass(frozen=True)
class CadenceNotification:
    recipient_id: UUID
    title: str
    body: str
    priority: str
    source_type: str
    source_id: UUID
    action_url: str | None = None


class CadenceDataPort(Protocol):
    async def get_active_initiatives(self, org_id: str) -> list[Any]: ...
    async def get_milestones(self, initiative_id: UUID) -> list[Any]: ...
    async def get_dependencies_downstream(self, initiative_id: UUID) -> list[Any]: ...


class CadenceEngine:
    def __init__(self, data_port: CadenceDataPort, now: datetime) -> None:
        self._data_port = data_port
        self._now = now

    async def run(self, organization_id: str) -> list[CadenceNotification]:
        notifications: list[CadenceNotification] = []
        initiatives = await self._data_port.get_active_initiatives(organization_id)

        for initiative in initiatives:
            # Staleness check
            days_since = (self._now - initiative.last_updated_at).days
            threshold = int(initiative.update_cadence_days * _STALENESS_MULTIPLIER)
            if days_since > threshold:
                notifications.append(
                    CadenceNotification(
                        recipient_id=initiative.owner_id,
                        title=f"Stale: no update on initiative in {days_since} days",
                        body=f"Initiative has not been updated in {days_since} days (expected every {initiative.update_cadence_days} days).",
                        priority="high",
                        source_type="initiative",
                        source_id=initiative.id,
                        action_url=f"/strategy?initiative={initiative.id}",
                    )
                )

            # Milestone checks
            milestones = await self._data_port.get_milestones(initiative.id)
            for ms in milestones:
                if ms.status in ("completed", "deferred"):
                    continue

                days_until = (ms.target_date - self._now.date()).days

                if days_until < 0:
                    # Overdue
                    recipient = ms.assignee_id or initiative.owner_id
                    notifications.append(
                        CadenceNotification(
                            recipient_id=recipient,
                            title=f"Overdue: {ms.title}",
                            body=f"Milestone was due {abs(days_until)} days ago.",
                            priority="urgent",
                            source_type="milestone",
                            source_id=ms.id,
                            action_url=f"/strategy?initiative={initiative.id}",
                        )
                    )
                elif days_until <= _APPROACHING_DAYS:
                    # Approaching
                    recipient = ms.assignee_id or initiative.owner_id
                    notifications.append(
                        CadenceNotification(
                            recipient_id=recipient,
                            title=f"Due soon: {ms.title}",
                            body=f"Milestone is due in {days_until} days.",
                            priority="high",
                            source_type="milestone",
                            source_id=ms.id,
                            action_url=f"/strategy?initiative={initiative.id}",
                        )
                    )

        return notifications
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/api && python -m pytest tests/unit/strategy/test_cadence_engine.py -v`
Expected: PASS (all 3 tests)

- [ ] **Step 5: Add ApprovalRequest table and entity** following spec. Add to migration.

- [ ] **Step 6: Add approval endpoints** (`GET /strategy/approvals`, `POST /strategy/approvals/{id}/approve`, `POST /strategy/approvals/{id}/reject`)

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(strategy): add cadence engine with staleness/milestone detection and approval request model"
```

---

### Task 19: Notifications Bell Widget (Frontend)

**Files:**
- Create: `apps/web/src/components/notifications-bell.tsx`
- Modify: `apps/web/src/components/nav.tsx` — add bell to header

- [ ] **Step 1: Create notifications bell component**

```tsx
// apps/web/src/components/notifications-bell.tsx
"use client";

import { useEffect, useState } from "react";
import { apiFetch } from "@/lib/api-client";
import { getOrgId } from "@/lib/auth";

interface NotificationItem {
  id: string;
  category: string;
  priority: string;
  title: string;
  body: string;
  action_url: string | null;
  read: boolean;
  created_at: string;
}

interface CountResponse {
  total: number;
  by_category: Record<string, number>;
}

const PRIORITY_DOT: Record<string, string> = {
  urgent: "bg-red-500",
  high: "bg-orange-400",
  normal: "bg-blue-400",
  low: "bg-neutral-400",
};

export function NotificationsBell() {
  const [open, setOpen] = useState(false);
  const [count, setCount] = useState(0);
  const [items, setItems] = useState<NotificationItem[]>([]);

  useEffect(() => {
    const orgId = getOrgId();
    if (!orgId) return;

    apiFetch<CountResponse>(`/notifications/count?organization_id=${orgId}`)
      .then((data) => setCount(data.total))
      .catch(() => {});
  }, []);

  async function loadNotifications() {
    const orgId = getOrgId();
    if (!orgId) return;
    const data = await apiFetch<NotificationItem[]>(
      `/notifications?organization_id=${orgId}`,
    );
    setItems(data);
  }

  function toggle() {
    if (!open) loadNotifications();
    setOpen(!open);
  }

  async function markRead(id: string) {
    await apiFetch(`/notifications/${id}`, {
      method: "PATCH",
      body: JSON.stringify({ read: true }),
    });
    setItems((prev) => prev.map((n) => (n.id === id ? { ...n, read: true } : n)));
    setCount((c) => Math.max(0, c - 1));
  }

  return (
    <div className="relative">
      <button
        onClick={toggle}
        className="relative rounded-md p-1.5 text-neutral-500 transition-colors hover:bg-neutral-100 hover:text-neutral-900"
      >
        <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
          <path d="M10 2a6 6 0 00-6 6v3.586l-.707.707A1 1 0 004 14h12a1 1 0 00.707-1.707L16 11.586V8a6 6 0 00-6-6zM10 18a3 3 0 01-3-3h6a3 3 0 01-3 3z" />
        </svg>
        {count > 0 && (
          <span className="absolute -right-1 -top-1 flex h-4 w-4 items-center justify-center rounded-full bg-red-500 text-[10px] font-bold text-white">
            {count > 9 ? "9+" : count}
          </span>
        )}
      </button>

      {open && (
        <div className="absolute right-0 top-full z-50 mt-2 w-80 rounded-lg border border-neutral-200 bg-white shadow-lg">
          <div className="border-b border-neutral-100 px-4 py-2">
            <h3 className="text-sm font-semibold text-neutral-900">Notifications</h3>
          </div>
          <div className="max-h-96 overflow-y-auto">
            {items.length === 0 ? (
              <p className="px-4 py-6 text-center text-sm text-neutral-400">No notifications</p>
            ) : (
              items.map((n) => (
                <button
                  key={n.id}
                  onClick={() => {
                    markRead(n.id);
                    if (n.action_url) window.location.href = n.action_url;
                  }}
                  className={`flex w-full gap-3 px-4 py-3 text-left transition-colors hover:bg-neutral-50 ${
                    n.read ? "opacity-60" : ""
                  }`}
                >
                  <div className={`mt-1.5 h-2 w-2 shrink-0 rounded-full ${PRIORITY_DOT[n.priority] ?? PRIORITY_DOT.normal}`} />
                  <div className="min-w-0 flex-1">
                    <p className="truncate text-sm font-medium text-neutral-900">{n.title}</p>
                    <p className="truncate text-xs text-neutral-500">{n.body}</p>
                    <p className="mt-0.5 text-xs text-neutral-400">
                      {new Date(n.created_at).toLocaleDateString()}
                    </p>
                  </div>
                </button>
              ))
            )}
          </div>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Add bell to nav**

In `apps/web/src/components/nav.tsx`, import and add `<NotificationsBell />` to the header area, in the `div` with `OmniSearch` and `NavActions`:

```tsx
import { NotificationsBell } from "@/components/notifications-bell";

// In the JSX, add before OmniSearch:
<NotificationsBell />
```

- [ ] **Step 3: Verify in browser**
- [ ] **Step 4: Commit**

```bash
git commit -m "feat(web): add unified notifications bell widget to header"
```

---

## Slice 5: Copilot Integration

### Task 20: Strategy Copilot Tools

**Files:**
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/get_strategy_summary.py`
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/get_initiative_detail.py`
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/update_milestone_status.py`
- Modify: `apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py` — register new tools

- [ ] **Step 1: Create GetStrategySummaryTool**

Follow the same pattern as `SearchArtifactsTool` / `GetArtifactTool`:
- Arguments model with `target_company_id` (optional, since strategy is org-scoped)
- `execute()` queries `get_initiatives_for_tenant()`, returns summary
- Returns `ToolExecution.ok(...)` with structured records

- [ ] **Step 2: Create GetInitiativeDetailTool**

- Arguments: `initiative_id: str`
- Queries initiative + milestones + dependencies + resources + evidence
- Returns full detail as `ToolExecution.ok(...)`

- [ ] **Step 3: Create UpdateMilestoneStatusTool**

- Arguments: `milestone_id: str`, `status_update: str` (free text from assignee)
- If the update implies a change (date, scope), creates an ApprovalRequest and returns it
- If it's just a status report, updates the milestone directly
- Returns `ToolExecution.ok(...)` with what happened

- [ ] **Step 4: Register tools**

In `apps/api/src/prescient/intelligence/infrastructure/tools/__init__.py`, add imports and register:

```python
from prescient.intelligence.infrastructure.tools.get_strategy_summary import GetStrategySummaryTool
from prescient.intelligence.infrastructure.tools.get_initiative_detail import GetInitiativeDetailTool
from prescient.intelligence.infrastructure.tools.update_milestone_status import UpdateMilestoneStatusTool

# In build_tool_registry():
registry.register(GetStrategySummaryTool())
registry.register(GetInitiativeDetailTool())
registry.register(UpdateMilestoneStatusTool())
```

- [ ] **Step 5: Add to `__all__`**
- [ ] **Step 6: Write tests for each tool following existing test patterns**
- [ ] **Step 7: Commit**

```bash
git commit -m "feat(intelligence): add strategy copilot tools — summary, detail, milestone status"
```

---

### Task 21: Copilot Passive Indicators + End-of-Task Nudge

**Files:**
- Modify: `apps/api/src/prescient/intelligence/application/planner.py` — inject strategy context into system prompt
- Modify: `apps/api/src/prescient/intelligence/application/planner_stream.py` — same for streaming
- Modify: `apps/web/src/app/api/chat/route.ts` — pass notification count to copilot

This task requires understanding the planner system prompt injection pattern. The implementing engineer should:

- [ ] **Step 1: Add strategy context to planner system prompt**

Read the planner files to find where the system prompt is assembled. Add a strategy context block:

```
## Strategy Context
You have access to strategy tools. The user has {count} pending strategy items.
Top items needing attention:
{top_3_items}

IMPORTANT: Do not interrupt the user's current task with strategy items. Only mention them:
1. When the user asks about strategy directly
2. After a natural conversation pause (user says "thanks", "got it", or similar)
When offering: "By the way, you have {count} strategy items that could use your input. Want to take a look?"
Only offer ONCE per session. Never offer if strategy was already discussed.
```

- [ ] **Step 2: Add notification count fetch to copilot context setup**
- [ ] **Step 3: Update frontend chat to show passive indicators** — small badge in chat header showing pending strategy item count
- [ ] **Step 4: Test in browser** — start a chat, verify indicator appears, complete a task, verify nudge appears
- [ ] **Step 5: Commit**

```bash
git commit -m "feat(intelligence): add strategy context injection and end-of-task nudge to copilot"
```

---

## Slice 6: Briefing Integration

### Task 22: Add STRATEGY to BriefingSourcePort

**Files:**
- Modify: `apps/api/src/prescient/briefing/domain/entities/briefing_item.py` — add `STRATEGY` to `ItemSource`
- Modify: `apps/api/src/prescient/briefing/application/briefing_assembler.py` — add `get_strategy_items` to port and assembler
- Modify: `apps/api/src/prescient/briefing/infrastructure/source_adapter.py` — implement `get_strategy_items`
- Test: `apps/api/tests/unit/briefing/test_strategy_source.py`

- [ ] **Step 1: Write failing test**

```python
# apps/api/tests/unit/briefing/test_strategy_source.py
"""Unit tests for strategy items in the briefing source adapter."""

from __future__ import annotations

from prescient.briefing.domain.entities.briefing_item import ItemSource


class TestItemSourceEnum:
    def test_strategy_value_exists(self) -> None:
        assert ItemSource.STRATEGY == "strategy"
```

- [ ] **Step 2: Run test — fails because STRATEGY not in enum**

Run: `cd apps/api && python -m pytest tests/unit/briefing/test_strategy_source.py -v`
Expected: FAIL

- [ ] **Step 3: Add STRATEGY to ItemSource**

In `apps/api/src/prescient/briefing/domain/entities/briefing_item.py`, add to the `ItemSource` enum:

```python
STRATEGY = "strategy"
```

- [ ] **Step 4: Run test — passes**

- [ ] **Step 5: Add `get_strategy_items` to BriefingSourcePort**

In `apps/api/src/prescient/briefing/application/briefing_assembler.py`, add to the Protocol:

```python
async def get_strategy_items(self, *, organization_id: str) -> list[BriefingItem]: ...
```

And in the `assemble` method, add the call and merge:

```python
strategy = await self._source_port.get_strategy_items(organization_id=organization_id)

all_raw: list[BriefingItem] = [*intelligence, *kpis, *actions, *monitoring, *strategy]
```

- [ ] **Step 6: Implement `get_strategy_items` on DatabaseSourceAdapter**

In `apps/api/src/prescient/briefing/infrastructure/source_adapter.py`, add:

```python
async def get_strategy_items(self, *, organization_id: str) -> list[BriefingItem]:
    tid = self._tenant_uuid
    if not tid:
        return []

    from prescient.strategy.queries import get_initiatives_for_tenant, get_milestones_for_initiative

    initiatives = await get_initiatives_for_tenant(
        self._session, tid, status_filter=["active", "at_risk"]
    )

    items: list[BriefingItem] = []
    now = datetime.now(tz=UTC)
    today = now.date()

    for init in initiatives:
        # At-risk initiative
        if init.health_score is not None and float(init.health_score) < 0.4:
            items.append(
                BriefingItem(
                    id=str(uuid.uuid4()),
                    title=f"At risk: {init.title}",
                    summary=f"Health score dropped to {float(init.health_score):.0%}",
                    action_type=ActionType.DECIDE,
                    source=ItemSource.STRATEGY,
                    source_artifact_id=str(init.id),
                    relevance_score=0.9,
                    created_at=init.updated_at,
                )
            )

        # Stale initiative
        days_since = (now - init.last_updated_at).days
        if days_since > init.update_cadence_days * 2:
            items.append(
                BriefingItem(
                    id=str(uuid.uuid4()),
                    title=f"Stale: {init.title}",
                    summary=f"No update in {days_since} days",
                    action_type=ActionType.READ,
                    source=ItemSource.STRATEGY,
                    source_artifact_id=str(init.id),
                    relevance_score=0.65,
                    created_at=init.updated_at,
                )
            )

        # Milestones due soon
        milestones = await get_milestones_for_initiative(self._session, init.id)
        for ms in milestones:
            if ms.status in ("completed", "deferred"):
                continue
            days_until = (ms.target_date - today).days
            if days_until < 0:
                items.append(
                    BriefingItem(
                        id=str(uuid.uuid4()),
                        title=f"Overdue: {ms.title}",
                        summary=f"Milestone was due {abs(days_until)} days ago",
                        action_type=ActionType.DECIDE,
                        source=ItemSource.STRATEGY,
                        source_artifact_id=str(init.id),
                        relevance_score=0.8,
                        created_at=ms.created_at,
                    )
                )
            elif days_until <= 3:
                items.append(
                    BriefingItem(
                        id=str(uuid.uuid4()),
                        title=f"Due soon: {ms.title}",
                        summary=f"Milestone due in {days_until} days",
                        action_type=ActionType.DECIDE,
                        source=ItemSource.STRATEGY,
                        source_artifact_id=str(init.id),
                        relevance_score=0.85,
                        created_at=ms.created_at,
                    )
                )

    return items
```

- [ ] **Step 7: Run all briefing tests**

Run: `cd apps/api && python -m pytest tests/unit/briefing/ -v`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/briefing/ apps/api/tests/unit/briefing/test_strategy_source.py
git commit -m "feat(briefing): integrate strategy items into daily briefing via source adapter"
```

---

### Task 23: Run Full Test Suite

- [ ] **Step 1: Run all strategy tests**

Run: `cd apps/api && python -m pytest tests/unit/strategy/ -v`
Expected: All PASS

- [ ] **Step 2: Run all notification tests**

Run: `cd apps/api && python -m pytest tests/unit/notifications/ -v`
Expected: All PASS

- [ ] **Step 3: Run all briefing tests**

Run: `cd apps/api && python -m pytest tests/unit/briefing/ -v`
Expected: All PASS

- [ ] **Step 4: Run full test suite**

Run: `cd apps/api && python -m pytest -v`
Expected: All existing tests still pass, no regressions

- [ ] **Step 5: Verify API starts cleanly**

Run: `docker compose up api` and check logs for startup errors.

- [ ] **Step 6: Verify frontend builds**

Run: `cd apps/web && npm run build`
Expected: No TypeScript errors

- [ ] **Step 7: Final commit if any cleanup needed**

```bash
git commit -m "chore: final cleanup after strategic initiatives implementation"
```
