# Phase 3: Operating Core — KPI Management, Actions & Decisions, Daily Briefing

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the daily operating loop — KPI reporting with anomaly detection, action/decision logging, and the personalized Morning Brief that makes the platform the operator's first touchpoint each day.

**Architecture:** KPI Management adds a `kpi_values` table and ownership model to the existing documents context, with an anomaly detector that compares new values against history and uses the LLM to generate follow-up questions. Action items use the existing domain entities/tables/state machine — we add the application use cases and API routes. The Daily Briefing is a new `briefing` bounded context with a generation service that scores and selects items from multiple sources (intelligence signals, KPIs, actions, monitoring findings), personalized by role and interest profile.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Anthropic SDK (Claude), existing artifact store.

**Spec:** `docs/superpowers/specs/2026-04-12-prescient-os-redesign.md` — Modules 1, 3, 4, Section 5

**Depends on:** Phase 1 (Organizations, Artifacts, Auth) + Phase 2 (Intelligence, Onboarding, Monitoring)

---

## File Structure

### New Files (API)

```
apps/api/src/prescient/
├── kpis/                                  # NEW — KPI Management context
│   ├── __init__.py
│   ├── domain/
│   │   ├── __init__.py
│   │   ├── entities/
│   │   │   ├── __init__.py
│   │   │   ├── kpi_owner.py              # Who owns which KPI for which company
│   │   │   ├── kpi_value.py              # Point-in-time metric value
│   │   │   └── kpi_anomaly.py            # Detected anomaly with enrichment
│   │   └── errors.py
│   ├── application/
│   │   ├── __init__.py
│   │   ├── use_cases/
│   │   │   ├── __init__.py
│   │   │   ├── assign_kpi_owner.py
│   │   │   ├── report_kpi_value.py       # Submit a value + trigger anomaly check
│   │   │   └── get_kpi_series.py         # Historical values for a KPI
│   │   └── anomaly_detector.py           # Compare against history, flag deviations
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   └── tables/
│   │       ├── __init__.py
│   │       ├── kpi_owner.py
│   │       ├── kpi_value.py
│   │       └── kpi_anomaly.py
│   └── api/
│       ├── __init__.py
│       └── routes.py
├── action_items/
│   ├── application/
│   │   ├── __init__.py                   # EXISTS but empty — populate
│   │   └── use_cases/
│   │       ├── __init__.py
│   │       ├── create_action_item.py     # Create from briefing/decision
│   │       ├── list_action_items.py      # List by org, status, owner
│   │       ├── update_action_item.py     # Status transitions
│   │       └── create_decision_record.py # Log a decision + optional actions
│   └── api/
│       ├── __init__.py                   # EXISTS but empty — populate
│       └── routes.py
├── briefing/                              # NEW — Daily Briefing context
│   ├── __init__.py
│   ├── domain/
│   │   ├── __init__.py
│   │   └── entities/
│   │       ├── __init__.py
│   │       ├── briefing_item.py          # Single item in a briefing
│   │       └── daily_briefing.py         # The assembled briefing
│   ├── application/
│   │   ├── __init__.py
│   │   ├── briefing_assembler.py         # Collects items from all sources
│   │   ├── item_scorer.py               # Scores relevance per user role/interests
│   │   └── use_cases/
│   │       ├── __init__.py
│   │       └── get_daily_briefing.py     # Entry point for generating/retrieving brief
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   └── tables/
│   │       ├── __init__.py
│   │       └── briefing.py              # Persisted briefing snapshots
│   └── api/
│       ├── __init__.py
│       └── routes.py
```

### New Files (Frontend)

```
apps/web/src/
├── app/(main)/
│   ├── brief/
│   │   └── page.tsx                      # REPLACE placeholder with real briefing UI
│   ├── actions/
│   │   └── page.tsx                      # Action items list with status management
│   └── kpis/
│       └── page.tsx                      # KPI reporting dashboard
├── components/
│   ├── briefing-item.tsx                 # Single briefing item card
│   ├── action-item-row.tsx              # Action item in a table/list
│   └── kpi-report-form.tsx              # KPI value submission form
```

### New Files (Tests)

```
tests/unit/
├── kpis/
│   ├── test_kpi_domain.py
│   ├── test_anomaly_detector.py
│   ├── test_report_kpi_value.py
│   └── test_kpi_api.py
├── action_items/
│   ├── test_create_action_item.py
│   ├── test_create_decision_record.py
│   └── test_action_items_api.py
├── briefing/
│   ├── test_briefing_item.py
│   ├── test_briefing_assembler.py
│   ├── test_item_scorer.py
│   └── test_get_daily_briefing.py
```

---

## Task 1: KPI Domain Entities

**Files:**
- Create: `apps/api/src/prescient/kpis/__init__.py`
- Create: `apps/api/src/prescient/kpis/domain/__init__.py`
- Create: `apps/api/src/prescient/kpis/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/kpis/domain/entities/kpi_owner.py`
- Create: `apps/api/src/prescient/kpis/domain/entities/kpi_value.py`
- Create: `apps/api/src/prescient/kpis/domain/entities/kpi_anomaly.py`
- Create: `apps/api/src/prescient/kpis/domain/errors.py`
- Modify: `apps/api/src/prescient/shared/types.py`
- Test: `apps/api/tests/unit/kpis/__init__.py`
- Test: `apps/api/tests/unit/kpis/test_kpi_domain.py`

- [ ] **Step 1: Add new ID types**

In `apps/api/src/prescient/shared/types.py`, add:

```python
KpiOwnerId = NewType("KpiOwnerId", UUID)
KpiValueId = NewType("KpiValueId", UUID)
KpiAnomalyId = NewType("KpiAnomalyId", UUID)
```

- [ ] **Step 2: Create kpis package structure**

Create empty `__init__.py` files for the kpis context.

- [ ] **Step 3: Write failing tests**

Create `apps/api/tests/unit/kpis/test_kpi_domain.py`:

```python
from datetime import date, datetime, UTC
from decimal import Decimal
from uuid import uuid4

import pytest

from prescient.kpis.domain.entities.kpi_owner import KpiOwner, ReportingCadence
from prescient.kpis.domain.entities.kpi_value import KpiValue
from prescient.kpis.domain.entities.kpi_anomaly import KpiAnomaly, AnomalyType
from prescient.shared.types import KpiOwnerId, KpiValueId, KpiAnomalyId, OrganizationId, UserId


class TestKpiOwner:
    def test_create_owner(self) -> None:
        now = datetime.now(UTC)
        owner = KpiOwner(
            id=KpiOwnerId(uuid4()),
            organization_id=OrganizationId(uuid4()),
            kpi_id="revenue",
            owner_user_id=UserId("cfo-jane"),
            cadence=ReportingCadence.MONTHLY,
            created_at=now,
        )
        assert owner.kpi_id == "revenue"
        assert owner.cadence == ReportingCadence.MONTHLY

    def test_all_cadences(self) -> None:
        expected = {"weekly", "monthly", "quarterly"}
        assert {c.value for c in ReportingCadence} == expected


class TestKpiValue:
    def test_create_value(self) -> None:
        now = datetime.now(UTC)
        value = KpiValue(
            id=KpiValueId(uuid4()),
            organization_id=OrganizationId(uuid4()),
            kpi_id="revenue",
            period_label="Q1 2026",
            period_start=date(2026, 1, 1),
            period_end=date(2026, 3, 31),
            value=Decimal("105.2"),
            unit="USD_millions",
            commentary=None,
            reported_by=UserId("cfo-jane"),
            reported_at=now,
        )
        assert value.value == Decimal("105.2")
        assert value.period_label == "Q1 2026"


class TestKpiAnomaly:
    def test_create_anomaly(self) -> None:
        now = datetime.now(UTC)
        anomaly = KpiAnomaly(
            id=KpiAnomalyId(uuid4()),
            kpi_value_id=KpiValueId(uuid4()),
            organization_id=OrganizationId(uuid4()),
            anomaly_type=AnomalyType.DEVIATION_FROM_TREND,
            description="Revenue dropped 12% below forecast",
            severity="important",
            enrichment_question="What drove the revenue decline?",
            enrichment_response=None,
            detected_at=now,
        )
        assert anomaly.anomaly_type == AnomalyType.DEVIATION_FROM_TREND
        assert anomaly.enrichment_response is None

    def test_all_anomaly_types(self) -> None:
        expected = {"deviation_from_trend", "missed_target", "sudden_change", "stale_data"}
        assert {t.value for t in AnomalyType} == expected
```

- [ ] **Step 4: Implement KpiOwner entity**

Create `apps/api/src/prescient/kpis/domain/entities/kpi_owner.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field

from prescient.shared.types import KpiOwnerId, OrganizationId, UserId


class ReportingCadence(StrEnum):
    WEEKLY = "weekly"
    MONTHLY = "monthly"
    QUARTERLY = "quarterly"


class KpiOwner(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: KpiOwnerId
    organization_id: OrganizationId
    kpi_id: str = Field(min_length=1, max_length=64)
    owner_user_id: UserId
    cadence: ReportingCadence
    created_at: datetime
```

- [ ] **Step 5: Implement KpiValue entity**

Create `apps/api/src/prescient/kpis/domain/entities/kpi_value.py`:

```python
from __future__ import annotations

from datetime import date, datetime
from decimal import Decimal

from pydantic import BaseModel, ConfigDict, Field

from prescient.shared.types import KpiValueId, OrganizationId, UserId


class KpiValue(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: KpiValueId
    organization_id: OrganizationId
    kpi_id: str = Field(min_length=1, max_length=64)
    period_label: str = Field(min_length=1, max_length=64)
    period_start: date
    period_end: date
    value: Decimal
    unit: str = Field(min_length=1, max_length=32)
    commentary: str | None = None
    reported_by: UserId
    reported_at: datetime
```

- [ ] **Step 6: Implement KpiAnomaly entity**

Create `apps/api/src/prescient/kpis/domain/entities/kpi_anomaly.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict

from prescient.shared.types import KpiAnomalyId, KpiValueId, OrganizationId


class AnomalyType(StrEnum):
    DEVIATION_FROM_TREND = "deviation_from_trend"
    MISSED_TARGET = "missed_target"
    SUDDEN_CHANGE = "sudden_change"
    STALE_DATA = "stale_data"


class KpiAnomaly(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: KpiAnomalyId
    kpi_value_id: KpiValueId
    organization_id: OrganizationId
    anomaly_type: AnomalyType
    description: str
    severity: str
    enrichment_question: str | None = None
    enrichment_response: str | None = None
    detected_at: datetime
```

- [ ] **Step 7: Create domain errors**

Create `apps/api/src/prescient/kpis/domain/errors.py`:

```python
from prescient.shared.errors import NotFoundError, ValidationError


class KpiOwnerNotFoundError(NotFoundError):
    def __init__(self, identifier: str) -> None:
        super().__init__(f"KPI owner not found: {identifier}")


class KpiValueNotFoundError(NotFoundError):
    def __init__(self, identifier: str) -> None:
        super().__init__(f"KPI value not found: {identifier}")


class DuplicateKpiOwnerError(ValidationError):
    def __init__(self, kpi_id: str, user_id: str) -> None:
        super().__init__(f"KPI '{kpi_id}' is already owned by user '{user_id}'")
```

- [ ] **Step 8: Run tests**

Run: `cd apps/api && uv run python -m pytest tests/unit/kpis/test_kpi_domain.py -v`
Expected: 5 PASSED

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/kpis/ apps/api/src/prescient/shared/types.py tests/unit/kpis/
git commit -m "feat: add KPI Management domain — owner, value, anomaly entities"
```

---

## Task 2: KPI Infrastructure Tables + Migration

**Files:**
- Create: `apps/api/src/prescient/kpis/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/kpis/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/kpis/infrastructure/tables/kpi_owner.py`
- Create: `apps/api/src/prescient/kpis/infrastructure/tables/kpi_value.py`
- Create: `apps/api/src/prescient/kpis/infrastructure/tables/kpi_anomaly.py`
- Modify: `apps/api/src/prescient/shared/metadata.py`

All tables use schema "kpis".

- [ ] **Step 1: Create infrastructure package and tables**

**KpiOwnerRow** (kpis.kpi_owners):
- id: String(36) PK
- organization_id: String(36) not null, indexed
- kpi_id: String(64) not null
- owner_user_id: String(64) not null
- cadence: String(16) not null
- created_at: DateTime(timezone=True) not null
- Unique constraint on (organization_id, kpi_id)

**KpiValueRow** (kpis.kpi_values):
- id: String(36) PK
- organization_id: String(36) not null, indexed
- kpi_id: String(64) not null, indexed
- period_label: String(64) not null
- period_start: Date not null
- period_end: Date not null
- value: Numeric(precision=18, scale=4) not null
- unit: String(32) not null
- commentary: Text nullable
- reported_by: String(64) not null
- reported_at: DateTime(timezone=True) not null

**KpiAnomalyRow** (kpis.kpi_anomalies):
- id: String(36) PK
- kpi_value_id: String(36) FK → kpis.kpi_values.id, not null, indexed
- organization_id: String(36) not null, indexed
- anomaly_type: String(32) not null
- description: Text not null
- severity: String(16) not null
- enrichment_question: Text nullable
- enrichment_response: Text nullable
- detected_at: DateTime(timezone=True) not null

- [ ] **Step 2: Register tables in metadata.py and generate migration**

Add `import prescient.kpis.infrastructure.tables  # noqa: F401` to metadata.py.

Create schema: `PGPASSWORD=prescient psql -h localhost -U prescient -d prescient -c "CREATE SCHEMA IF NOT EXISTS kpis;"`

Generate: `cd apps/api && uv run python -m alembic revision --autogenerate -m "phase3_kpis"`

Review, add CREATE SCHEMA IF NOT EXISTS, run migration.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/kpis/infrastructure/ apps/api/src/prescient/shared/metadata.py apps/api/alembic/
git commit -m "feat: add KPI infrastructure tables and migration"
```

---

## Task 3: Anomaly Detector

**Files:**
- Create: `apps/api/src/prescient/kpis/application/__init__.py`
- Create: `apps/api/src/prescient/kpis/application/anomaly_detector.py`
- Test: `apps/api/tests/unit/kpis/test_anomaly_detector.py`

The anomaly detector compares a new KPI value against historical values and flags deviations.

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/kpis/test_anomaly_detector.py`:

```python
from datetime import date, datetime, UTC
from decimal import Decimal
from uuid import uuid4

import pytest

from prescient.kpis.application.anomaly_detector import (
    AnomalyDetector,
    DetectedAnomaly,
)
from prescient.kpis.domain.entities.kpi_value import KpiValue
from prescient.shared.types import KpiValueId, OrganizationId, UserId


def _make_value(val: str, period: str, start: date, end: date) -> KpiValue:
    return KpiValue(
        id=KpiValueId(uuid4()),
        organization_id=OrganizationId(uuid4()),
        kpi_id="revenue",
        period_label=period,
        period_start=start,
        period_end=end,
        value=Decimal(val),
        unit="USD_millions",
        commentary=None,
        reported_by=UserId("cfo"),
        reported_at=datetime.now(UTC),
    )


class TestAnomalyDetector:
    def test_no_anomaly_when_stable(self) -> None:
        history = [
            _make_value("100", "Q1", date(2025, 1, 1), date(2025, 3, 31)),
            _make_value("102", "Q2", date(2025, 4, 1), date(2025, 6, 30)),
            _make_value("104", "Q3", date(2025, 7, 1), date(2025, 9, 30)),
        ]
        new_value = _make_value("106", "Q4", date(2025, 10, 1), date(2025, 12, 31))
        detector = AnomalyDetector()
        result = detector.check(new_value=new_value, history=history)
        assert result is None

    def test_detects_sudden_drop(self) -> None:
        history = [
            _make_value("100", "Q1", date(2025, 1, 1), date(2025, 3, 31)),
            _make_value("102", "Q2", date(2025, 4, 1), date(2025, 6, 30)),
            _make_value("104", "Q3", date(2025, 7, 1), date(2025, 9, 30)),
        ]
        new_value = _make_value("80", "Q4", date(2025, 10, 1), date(2025, 12, 31))
        detector = AnomalyDetector()
        result = detector.check(new_value=new_value, history=history)
        assert result is not None
        assert isinstance(result, DetectedAnomaly)
        assert "drop" in result.description.lower() or "decline" in result.description.lower() or "decrease" in result.description.lower()

    def test_no_anomaly_with_empty_history(self) -> None:
        new_value = _make_value("100", "Q1", date(2025, 1, 1), date(2025, 3, 31))
        detector = AnomalyDetector()
        result = detector.check(new_value=new_value, history=[])
        assert result is None

    def test_no_anomaly_with_single_history(self) -> None:
        history = [_make_value("100", "Q1", date(2025, 1, 1), date(2025, 3, 31))]
        new_value = _make_value("95", "Q2", date(2025, 4, 1), date(2025, 6, 30))
        detector = AnomalyDetector()
        result = detector.check(new_value=new_value, history=history)
        # With only 1 data point, we can't establish a trend — no anomaly
        assert result is None
```

- [ ] **Step 2: Implement anomaly detector**

Create `apps/api/src/prescient/kpis/application/anomaly_detector.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from decimal import Decimal
from statistics import mean

from prescient.kpis.domain.entities.kpi_anomaly import AnomalyType
from prescient.kpis.domain.entities.kpi_value import KpiValue


@dataclass(frozen=True)
class DetectedAnomaly:
    anomaly_type: AnomalyType
    description: str
    severity: str
    enrichment_question: str


class AnomalyDetector:
    """Compares a new KPI value against history and flags deviations.

    Uses a simple percentage-change-from-mean approach. More sophisticated
    methods (moving average, z-score) can replace this later.
    """

    def __init__(self, threshold_pct: float = 15.0) -> None:
        self._threshold = threshold_pct

    def check(
        self,
        new_value: KpiValue,
        history: list[KpiValue],
    ) -> DetectedAnomaly | None:
        if len(history) < 2:
            return None

        historical_values = [float(v.value) for v in history]
        avg = mean(historical_values)
        if avg == 0:
            return None

        new_float = float(new_value.value)
        pct_change = ((new_float - avg) / abs(avg)) * 100

        if abs(pct_change) < self._threshold:
            return None

        direction = "increase" if pct_change > 0 else "decrease"
        severity = "important" if abs(pct_change) > 25 else "notable"

        return DetectedAnomaly(
            anomaly_type=AnomalyType.SUDDEN_CHANGE if abs(pct_change) > 25 else AnomalyType.DEVIATION_FROM_TREND,
            description=(
                f"{new_value.kpi_id} showed a {abs(pct_change):.1f}% {direction} "
                f"({new_value.period_label}: {new_value.value} {new_value.unit}) "
                f"compared to the historical average of {avg:.1f}"
            ),
            severity=severity,
            enrichment_question=(
                f"{new_value.kpi_id} for {new_value.period_label} came in at "
                f"{new_value.value} {new_value.unit}, a {abs(pct_change):.1f}% {direction} "
                f"from the historical average. What's driving this change?"
            ),
        )
```

- [ ] **Step 3: Run tests**

Run: `cd apps/api && uv run python -m pytest tests/unit/kpis/test_anomaly_detector.py -v`
Expected: 4 PASSED

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/kpis/application/ tests/unit/kpis/
git commit -m "feat: add KPI anomaly detector — flags deviations from historical trend"
```

---

## Task 4: KPI Reporting Use Cases and API

**Files:**
- Create: `apps/api/src/prescient/kpis/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/kpis/application/use_cases/assign_kpi_owner.py`
- Create: `apps/api/src/prescient/kpis/application/use_cases/report_kpi_value.py`
- Create: `apps/api/src/prescient/kpis/application/use_cases/get_kpi_series.py`
- Create: `apps/api/src/prescient/kpis/api/__init__.py`
- Create: `apps/api/src/prescient/kpis/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`
- Test: `apps/api/tests/unit/kpis/test_report_kpi_value.py`

- [ ] **Step 1: Write failing test for report_kpi_value**

Create `apps/api/tests/unit/kpis/test_report_kpi_value.py`:

```python
from datetime import date, datetime, UTC
from decimal import Decimal
from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.kpis.application.use_cases.report_kpi_value import (
    ReportKpiValue,
    ReportKpiValueCommand,
    ReportKpiValueResult,
)


class TestReportKpiValue:
    @pytest.fixture
    def mock_deps(self) -> dict:
        session_port = AsyncMock()
        session_port.save_value = AsyncMock()
        session_port.get_history = AsyncMock(return_value=[])
        session_port.save_anomaly = AsyncMock()
        return {"session_port": session_port}

    @pytest.mark.asyncio
    async def test_reports_value_and_returns_result(self, mock_deps: dict) -> None:
        use_case = ReportKpiValue(session_port=mock_deps["session_port"])
        result = await use_case.execute(ReportKpiValueCommand(
            organization_id=str(uuid4()),
            kpi_id="revenue",
            period_label="Q1 2026",
            period_start=date(2026, 1, 1),
            period_end=date(2026, 3, 31),
            value=Decimal("105.2"),
            unit="USD_millions",
            commentary="Strong Q1",
            reported_by="cfo-jane",
        ))
        assert result.value_id is not None
        assert result.anomaly is None
        mock_deps["session_port"].save_value.assert_called_once()

    @pytest.mark.asyncio
    async def test_detects_anomaly_when_deviation(self, mock_deps: dict) -> None:
        # Provide history showing stable ~100
        from prescient.kpis.domain.entities.kpi_value import KpiValue
        from prescient.shared.types import KpiValueId, OrganizationId, UserId
        history = [
            KpiValue(
                id=KpiValueId(uuid4()), organization_id=OrganizationId(uuid4()),
                kpi_id="revenue", period_label=f"Q{i}", period_start=date(2025, i*3-2, 1),
                period_end=date(2025, i*3, 28), value=Decimal("100"), unit="USD_millions",
                commentary=None, reported_by=UserId("cfo"), reported_at=datetime.now(UTC),
            ) for i in range(1, 4)
        ]
        mock_deps["session_port"].get_history.return_value = history

        use_case = ReportKpiValue(session_port=mock_deps["session_port"])
        result = await use_case.execute(ReportKpiValueCommand(
            organization_id=str(uuid4()),
            kpi_id="revenue",
            period_label="Q4 2025",
            period_start=date(2025, 10, 1),
            period_end=date(2025, 12, 31),
            value=Decimal("70"),
            unit="USD_millions",
            commentary=None,
            reported_by="cfo-jane",
        ))
        assert result.anomaly is not None
        assert result.anomaly.enrichment_question is not None
        mock_deps["session_port"].save_anomaly.assert_called_once()
```

- [ ] **Step 2: Implement report_kpi_value use case**

Create `apps/api/src/prescient/kpis/application/use_cases/report_kpi_value.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import date, datetime, UTC
from decimal import Decimal
from typing import Protocol
from uuid import uuid4

from prescient.kpis.application.anomaly_detector import AnomalyDetector, DetectedAnomaly
from prescient.kpis.domain.entities.kpi_value import KpiValue
from prescient.shared.types import KpiValueId, OrganizationId, UserId


class KpiSessionPort(Protocol):
    async def save_value(self, *, row_kwargs: dict) -> None: ...
    async def get_history(self, *, organization_id: str, kpi_id: str, limit: int) -> list[KpiValue]: ...
    async def save_anomaly(self, *, row_kwargs: dict) -> None: ...


@dataclass(frozen=True)
class ReportKpiValueCommand:
    organization_id: str
    kpi_id: str
    period_label: str
    period_start: date
    period_end: date
    value: Decimal
    unit: str
    commentary: str | None = None
    reported_by: str = "system"


@dataclass(frozen=True)
class ReportKpiValueResult:
    value_id: str
    anomaly: DetectedAnomaly | None


class ReportKpiValue:
    def __init__(self, session_port: KpiSessionPort) -> None:
        self._port = session_port

    async def execute(self, command: ReportKpiValueCommand) -> ReportKpiValueResult:
        now = datetime.now(UTC)
        value_id = str(uuid4())

        # Build domain entity for anomaly check
        kpi_value = KpiValue(
            id=KpiValueId(uuid4()),
            organization_id=OrganizationId(uuid4()),
            kpi_id=command.kpi_id,
            period_label=command.period_label,
            period_start=command.period_start,
            period_end=command.period_end,
            value=command.value,
            unit=command.unit,
            commentary=command.commentary,
            reported_by=UserId(command.reported_by),
            reported_at=now,
        )

        # Persist the value
        await self._port.save_value(row_kwargs={
            "id": value_id,
            "organization_id": command.organization_id,
            "kpi_id": command.kpi_id,
            "period_label": command.period_label,
            "period_start": command.period_start,
            "period_end": command.period_end,
            "value": command.value,
            "unit": command.unit,
            "commentary": command.commentary,
            "reported_by": command.reported_by,
            "reported_at": now,
        })

        # Check for anomalies
        history = await self._port.get_history(
            organization_id=command.organization_id,
            kpi_id=command.kpi_id,
            limit=8,
        )

        detector = AnomalyDetector()
        anomaly = detector.check(new_value=kpi_value, history=history)

        if anomaly is not None:
            await self._port.save_anomaly(row_kwargs={
                "id": str(uuid4()),
                "kpi_value_id": value_id,
                "organization_id": command.organization_id,
                "anomaly_type": anomaly.anomaly_type.value,
                "description": anomaly.description,
                "severity": anomaly.severity,
                "enrichment_question": anomaly.enrichment_question,
                "enrichment_response": None,
                "detected_at": now,
            })

        return ReportKpiValueResult(value_id=value_id, anomaly=anomaly)
```

- [ ] **Step 3: Implement assign_kpi_owner and get_kpi_series use cases**

Create `apps/api/src/prescient/kpis/application/use_cases/assign_kpi_owner.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from typing import Protocol
from uuid import uuid4


class KpiOwnerPort(Protocol):
    async def save_owner(self, *, row_kwargs: dict) -> None: ...


@dataclass(frozen=True)
class AssignKpiOwnerCommand:
    organization_id: str
    kpi_id: str
    owner_user_id: str
    cadence: str


@dataclass(frozen=True)
class AssignKpiOwnerResult:
    owner_id: str


class AssignKpiOwner:
    def __init__(self, port: KpiOwnerPort) -> None:
        self._port = port

    async def execute(self, command: AssignKpiOwnerCommand) -> AssignKpiOwnerResult:
        owner_id = str(uuid4())
        await self._port.save_owner(row_kwargs={
            "id": owner_id,
            "organization_id": command.organization_id,
            "kpi_id": command.kpi_id,
            "owner_user_id": command.owner_user_id,
            "cadence": command.cadence,
            "created_at": datetime.now(UTC),
        })
        return AssignKpiOwnerResult(owner_id=owner_id)
```

Create `apps/api/src/prescient/kpis/application/use_cases/get_kpi_series.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol

from prescient.kpis.domain.entities.kpi_value import KpiValue


class KpiQueryPort(Protocol):
    async def get_history(self, *, organization_id: str, kpi_id: str, limit: int) -> list[KpiValue]: ...


@dataclass(frozen=True)
class GetKpiSeriesQuery:
    organization_id: str
    kpi_id: str
    limit: int = 12


class GetKpiSeries:
    def __init__(self, port: KpiQueryPort) -> None:
        self._port = port

    async def execute(self, query: GetKpiSeriesQuery) -> list[KpiValue]:
        return await self._port.get_history(
            organization_id=query.organization_id,
            kpi_id=query.kpi_id,
            limit=query.limit,
        )
```

- [ ] **Step 4: Implement KPI API routes**

Create `apps/api/src/prescient/kpis/api/routes.py` with endpoints:
- `POST /kpis/owners` — assign KPI owner (201)
- `POST /kpis/values` — report a KPI value, returns anomaly if detected (201)
- `GET /kpis/values?organization_id=X&kpi_id=Y` — get historical series

Wire `kpis_router` into `main.py`.

- [ ] **Step 5: Run tests and commit**

Run: `cd apps/api && uv run python -m pytest tests/unit/kpis/ -v`

```bash
git add apps/api/src/prescient/kpis/ apps/api/src/prescient/main.py apps/api/alembic/ tests/unit/kpis/
git commit -m "feat: add KPI Management — reporting, anomaly detection, API endpoints"
```

---

## Task 5: Action Items Application Layer and API

**Files:**
- Create: `apps/api/src/prescient/action_items/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/action_items/application/use_cases/create_action_item.py`
- Create: `apps/api/src/prescient/action_items/application/use_cases/list_action_items.py`
- Create: `apps/api/src/prescient/action_items/application/use_cases/update_action_item.py`
- Create: `apps/api/src/prescient/action_items/application/use_cases/create_decision_record.py`
- Create: `apps/api/src/prescient/action_items/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`
- Test: `apps/api/tests/unit/action_items/__init__.py`
- Test: `apps/api/tests/unit/action_items/test_create_action_item.py`
- Test: `apps/api/tests/unit/action_items/test_create_decision_record.py`

The action_items context has existing domain entities, state machine, and tables. The application and API layers are empty stubs. Read the existing code first:
- `apps/api/src/prescient/action_items/domain/entities/action_item.py`
- `apps/api/src/prescient/action_items/domain/enums.py`
- `apps/api/src/prescient/action_items/domain/state/action_item_state_machine.py`
- `apps/api/src/prescient/action_items/domain/ports.py`
- `apps/api/src/prescient/action_items/infrastructure/tables/action_item.py`

- [ ] **Step 1: Write failing test for create_action_item**

Create `apps/api/tests/unit/action_items/test_create_action_item.py`:

```python
from datetime import date
from unittest.mock import AsyncMock
from uuid import uuid4

import pytest

from prescient.action_items.application.use_cases.create_action_item import (
    CreateActionItem,
    CreateActionItemCommand,
)


class TestCreateActionItem:
    @pytest.fixture
    def mock_port(self) -> AsyncMock:
        port = AsyncMock()
        port.save = AsyncMock()
        return port

    @pytest.mark.asyncio
    async def test_creates_action_item(self, mock_port: AsyncMock) -> None:
        use_case = CreateActionItem(port=mock_port)
        result = await use_case.execute(CreateActionItemCommand(
            organization_id=str(uuid4()),
            title="Accelerate Q3 hiring plan",
            description="Based on competitor expansion signal, accelerate hiring.",
            owner_name="VP Engineering",
            owner_role="functional_leader",
            priority="high",
            due_date=date(2026, 6, 30),
            created_by="ryan",
            originating_artifact_id=str(uuid4()),
        ))
        assert result.action_item_id is not None
        mock_port.save.assert_called_once()
```

- [ ] **Step 2: Implement create_action_item**

Create `apps/api/src/prescient/action_items/application/use_cases/create_action_item.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import date, datetime, UTC
from typing import Protocol
from uuid import uuid4


class ActionItemSavePort(Protocol):
    async def save(self, *, row_kwargs: dict) -> None: ...


@dataclass(frozen=True)
class CreateActionItemCommand:
    organization_id: str
    title: str
    description: str
    owner_name: str
    owner_role: str | None = None
    priority: str = "medium"
    due_date: date | None = None
    created_by: str = "system"
    originating_artifact_id: str | None = None


@dataclass(frozen=True)
class CreateActionItemResult:
    action_item_id: str


class CreateActionItem:
    def __init__(self, port: ActionItemSavePort) -> None:
        self._port = port

    async def execute(self, command: CreateActionItemCommand) -> CreateActionItemResult:
        item_id = str(uuid4())
        now = datetime.now(UTC)
        await self._port.save(row_kwargs={
            "id": item_id,
            "owner_tenant_id": command.organization_id,
            "title": command.title,
            "description": command.description,
            "owner_name": command.owner_name,
            "owner_role": command.owner_role,
            "priority": command.priority,
            "status": "pending_review",
            "due_date": command.due_date,
            "citations": [],
            "created_by": command.created_by,
            "created_at": now,
        })
        return CreateActionItemResult(action_item_id=item_id)
```

- [ ] **Step 3: Implement list_action_items and update_action_item**

Create `apps/api/src/prescient/action_items/application/use_cases/list_action_items.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class ActionItemSummary:
    id: str
    title: str
    owner_name: str
    priority: str
    status: str
    due_date: str | None


class ActionItemListPort(Protocol):
    async def list_by_org(self, *, organization_id: str, status: str | None) -> list[ActionItemSummary]: ...


@dataclass(frozen=True)
class ListActionItemsQuery:
    organization_id: str
    status: str | None = None


class ListActionItems:
    def __init__(self, port: ActionItemListPort) -> None:
        self._port = port

    async def execute(self, query: ListActionItemsQuery) -> list[ActionItemSummary]:
        return await self._port.list_by_org(
            organization_id=query.organization_id,
            status=query.status,
        )
```

Create `apps/api/src/prescient/action_items/application/use_cases/update_action_item.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from typing import Protocol


class ActionItemUpdatePort(Protocol):
    async def update_status(self, *, item_id: str, status: str, approved_by: str | None, approved_at: datetime | None) -> None: ...


@dataclass(frozen=True)
class UpdateActionItemCommand:
    action_item_id: str
    status: str
    approved_by: str | None = None


class UpdateActionItem:
    def __init__(self, port: ActionItemUpdatePort) -> None:
        self._port = port

    async def execute(self, command: UpdateActionItemCommand) -> None:
        approved_at = datetime.now(UTC) if command.approved_by else None
        await self._port.update_status(
            item_id=command.action_item_id,
            status=command.status,
            approved_by=command.approved_by,
            approved_at=approved_at,
        )
```

- [ ] **Step 4: Implement create_decision_record**

Create `apps/api/src/prescient/action_items/application/use_cases/create_decision_record.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from typing import Protocol
from uuid import uuid4


class DecisionRecordPort(Protocol):
    async def save_artifact(self, *, row_kwargs: dict) -> None: ...
    async def save_version(self, *, row_kwargs: dict) -> None: ...


@dataclass(frozen=True)
class CreateDecisionRecordCommand:
    organization_id: str
    title: str
    context: str
    decision: str
    rationale: str
    created_by: str
    originating_signal_id: str | None = None


@dataclass(frozen=True)
class CreateDecisionRecordResult:
    artifact_id: str
    version_id: str


class CreateDecisionRecord:
    def __init__(self, port: DecisionRecordPort) -> None:
        self._port = port

    async def execute(self, command: CreateDecisionRecordCommand) -> CreateDecisionRecordResult:
        now = datetime.now(UTC)
        artifact_id = str(uuid4())
        version_id = str(uuid4())

        await self._port.save_artifact(row_kwargs={
            "id": artifact_id,
            "organization_id": command.organization_id,
            "artifact_type": "decision_record",
            "title": command.title,
            "visibility": "shared",
            "active_version_id": version_id,
            "created_at": now,
            "updated_at": now,
        })

        blocks = [
            {"block_id": "context", "heading": "Context", "body": command.context, "order": 0},
            {"block_id": "decision", "heading": "Decision", "body": command.decision, "order": 1},
            {"block_id": "rationale", "heading": "Rationale", "body": command.rationale, "order": 2},
        ]

        await self._port.save_version(row_kwargs={
            "id": version_id,
            "artifact_id": artifact_id,
            "version_number": 1,
            "state": "draft",
            "confidence_label": "high",
            "blocks": blocks,
            "citations": [],
            "summary": f"Decision: {command.title}",
            "created_by": command.created_by,
            "created_at": now,
        })

        return CreateDecisionRecordResult(artifact_id=artifact_id, version_id=version_id)
```

- [ ] **Step 5: Implement action items API routes**

Create `apps/api/src/prescient/action_items/api/routes.py` with:
- `POST /actions/items` — create action item (201)
- `GET /actions/items?organization_id=X&status=Y` — list action items
- `PATCH /actions/items/{item_id}` — update status
- `POST /actions/decisions` — create decision record (201)

Wire `actions_router` into `main.py`.

- [ ] **Step 6: Run tests and commit**

```bash
git add apps/api/src/prescient/action_items/ apps/api/src/prescient/main.py tests/unit/action_items/
git commit -m "feat: add Action Items API — CRUD, decision records, status management"
```

---

## Task 6: Briefing Domain Entities

**Files:**
- Create: `apps/api/src/prescient/briefing/__init__.py`
- Create: `apps/api/src/prescient/briefing/domain/__init__.py`
- Create: `apps/api/src/prescient/briefing/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/briefing/domain/entities/briefing_item.py`
- Create: `apps/api/src/prescient/briefing/domain/entities/daily_briefing.py`
- Test: `apps/api/tests/unit/briefing/__init__.py`
- Test: `apps/api/tests/unit/briefing/test_briefing_item.py`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/briefing/test_briefing_item.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.briefing.domain.entities.briefing_item import (
    BriefingItem,
    ActionType,
    ItemSource,
)
from prescient.briefing.domain.entities.daily_briefing import DailyBriefing


class TestBriefingItem:
    def test_create_item(self) -> None:
        item = BriefingItem(
            id=str(uuid4()),
            title="Competitor K18 announced new product line",
            summary="K18 launched a salon-exclusive bond repair treatment targeting Olaplex's core market.",
            action_type=ActionType.READ,
            source=ItemSource.INTELLIGENCE,
            source_artifact_id=str(uuid4()),
            relevance_score=0.85,
            created_at=datetime.now(UTC),
        )
        assert item.action_type == ActionType.READ
        assert item.relevance_score == 0.85

    def test_all_action_types(self) -> None:
        expected = {"read", "decide", "fyi"}
        assert {t.value for t in ActionType} == expected

    def test_all_sources(self) -> None:
        expected = {"intelligence", "kpi", "action_item", "monitoring", "calendar"}
        assert {s.value for s in ItemSource} == expected


class TestDailyBriefing:
    def test_create_briefing(self) -> None:
        now = datetime.now(UTC)
        items = [
            BriefingItem(
                id=str(uuid4()), title="Item 1", summary="Summary 1",
                action_type=ActionType.DECIDE, source=ItemSource.KPI,
                source_artifact_id=None, relevance_score=0.9, created_at=now,
            ),
            BriefingItem(
                id=str(uuid4()), title="Item 2", summary="Summary 2",
                action_type=ActionType.FYI, source=ItemSource.INTELLIGENCE,
                source_artifact_id=None, relevance_score=0.6, created_at=now,
            ),
        ]
        briefing = DailyBriefing(
            id=str(uuid4()),
            user_id="ryan",
            organization_id=str(uuid4()),
            date=now.date(),
            items=tuple(items),
            is_monday=False,
            generated_at=now,
        )
        assert len(briefing.items) == 2
        assert briefing.top_items(limit=1) == (items[0],)

    def test_empty_briefing(self) -> None:
        now = datetime.now(UTC)
        briefing = DailyBriefing(
            id=str(uuid4()), user_id="ryan", organization_id=str(uuid4()),
            date=now.date(), items=(), is_monday=False, generated_at=now,
        )
        assert briefing.is_empty is True
```

- [ ] **Step 2: Implement briefing entities**

Create `apps/api/src/prescient/briefing/domain/entities/briefing_item.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict


class ActionType(StrEnum):
    READ = "read"
    DECIDE = "decide"
    FYI = "fyi"


class ItemSource(StrEnum):
    INTELLIGENCE = "intelligence"
    KPI = "kpi"
    ACTION_ITEM = "action_item"
    MONITORING = "monitoring"
    CALENDAR = "calendar"


class BriefingItem(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: str
    title: str
    summary: str
    action_type: ActionType
    source: ItemSource
    source_artifact_id: str | None = None
    relevance_score: float
    created_at: datetime
```

Create `apps/api/src/prescient/briefing/domain/entities/daily_briefing.py`:

```python
from __future__ import annotations

from datetime import date, datetime

from pydantic import BaseModel, ConfigDict

from prescient.briefing.domain.entities.briefing_item import BriefingItem


class DailyBriefing(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: str
    user_id: str
    organization_id: str
    date: date
    items: tuple[BriefingItem, ...] = ()
    is_monday: bool = False
    generated_at: datetime

    def top_items(self, limit: int = 5) -> tuple[BriefingItem, ...]:
        sorted_items = sorted(self.items, key=lambda i: i.relevance_score, reverse=True)
        return tuple(sorted_items[:limit])

    @property
    def is_empty(self) -> bool:
        return len(self.items) == 0
```

- [ ] **Step 3: Run tests and commit**

Run: `cd apps/api && uv run python -m pytest tests/unit/briefing/test_briefing_item.py -v`

```bash
git add apps/api/src/prescient/briefing/ tests/unit/briefing/
git commit -m "feat: add Daily Briefing domain — BriefingItem and DailyBriefing entities"
```

---

## Task 7: Briefing Assembler and Item Scorer

**Files:**
- Create: `apps/api/src/prescient/briefing/application/__init__.py`
- Create: `apps/api/src/prescient/briefing/application/item_scorer.py`
- Create: `apps/api/src/prescient/briefing/application/briefing_assembler.py`
- Test: `apps/api/tests/unit/briefing/test_item_scorer.py`
- Test: `apps/api/tests/unit/briefing/test_briefing_assembler.py`

- [ ] **Step 1: Write failing test for item scorer**

Create `apps/api/tests/unit/briefing/test_item_scorer.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.briefing.application.item_scorer import ItemScorer
from prescient.briefing.domain.entities.briefing_item import ActionType, BriefingItem, ItemSource


class TestItemScorer:
    def test_boosts_decide_items(self) -> None:
        scorer = ItemScorer(role="operator", interests=["supply chain"])
        decide_item = BriefingItem(
            id=str(uuid4()), title="Revenue decision needed", summary="...",
            action_type=ActionType.DECIDE, source=ItemSource.KPI,
            source_artifact_id=None, relevance_score=0.5, created_at=datetime.now(UTC),
        )
        fyi_item = BriefingItem(
            id=str(uuid4()), title="FYI market news", summary="...",
            action_type=ActionType.FYI, source=ItemSource.INTELLIGENCE,
            source_artifact_id=None, relevance_score=0.5, created_at=datetime.now(UTC),
        )
        scored_decide = scorer.score(decide_item)
        scored_fyi = scorer.score(fyi_item)
        assert scored_decide.relevance_score > scored_fyi.relevance_score

    def test_boosts_matching_interests(self) -> None:
        scorer = ItemScorer(role="operator", interests=["supply chain"])
        matching = BriefingItem(
            id=str(uuid4()), title="Supply chain disruption alert", summary="Supply chain issue...",
            action_type=ActionType.READ, source=ItemSource.MONITORING,
            source_artifact_id=None, relevance_score=0.5, created_at=datetime.now(UTC),
        )
        unrelated = BriefingItem(
            id=str(uuid4()), title="New patent filing", summary="Patent info...",
            action_type=ActionType.READ, source=ItemSource.MONITORING,
            source_artifact_id=None, relevance_score=0.5, created_at=datetime.now(UTC),
        )
        assert scorer.score(matching).relevance_score > scorer.score(unrelated).relevance_score
```

- [ ] **Step 2: Implement item scorer**

Create `apps/api/src/prescient/briefing/application/item_scorer.py`:

```python
from __future__ import annotations

from prescient.briefing.domain.entities.briefing_item import ActionType, BriefingItem


class ItemScorer:
    """Scores briefing items based on user role and interests.

    Simple heuristic scorer — weights action type and interest keyword matching.
    Can be replaced with a more sophisticated model later.
    """

    _ACTION_WEIGHTS: dict[ActionType, float] = {
        ActionType.DECIDE: 0.3,
        ActionType.READ: 0.1,
        ActionType.FYI: 0.0,
    }

    def __init__(self, role: str, interests: list[str] | None = None) -> None:
        self._role = role
        self._interests = [i.lower() for i in (interests or [])]

    def score(self, item: BriefingItem) -> BriefingItem:
        base = item.relevance_score
        boost = self._ACTION_WEIGHTS.get(item.action_type, 0.0)
        interest_boost = self._interest_match(item)
        final = min(base + boost + interest_boost, 1.0)
        return item.model_copy(update={"relevance_score": final})

    def _interest_match(self, item: BriefingItem) -> float:
        if not self._interests:
            return 0.0
        text = f"{item.title} {item.summary}".lower()
        for interest in self._interests:
            if interest in text:
                return 0.15
        return 0.0
```

- [ ] **Step 3: Write failing test for briefing assembler**

Create `apps/api/tests/unit/briefing/test_briefing_assembler.py`:

```python
from datetime import date, datetime, UTC
from unittest.mock import AsyncMock
from uuid import uuid4

import pytest

from prescient.briefing.application.briefing_assembler import (
    BriefingAssembler,
    BriefingSourcePort,
)
from prescient.briefing.domain.entities.briefing_item import ActionType, BriefingItem, ItemSource


def _make_item(title: str, source: ItemSource, score: float) -> BriefingItem:
    return BriefingItem(
        id=str(uuid4()), title=title, summary=f"Summary of {title}",
        action_type=ActionType.READ, source=source, source_artifact_id=None,
        relevance_score=score, created_at=datetime.now(UTC),
    )


class TestBriefingAssembler:
    @pytest.mark.asyncio
    async def test_assembles_from_multiple_sources(self) -> None:
        source_port = AsyncMock(spec=BriefingSourcePort)
        source_port.get_intelligence_items.return_value = [
            _make_item("Intel signal", ItemSource.INTELLIGENCE, 0.7),
        ]
        source_port.get_kpi_items.return_value = [
            _make_item("Revenue anomaly", ItemSource.KPI, 0.8),
        ]
        source_port.get_action_items.return_value = [
            _make_item("Overdue action", ItemSource.ACTION_ITEM, 0.6),
        ]
        source_port.get_monitoring_items.return_value = []

        assembler = BriefingAssembler(source_port=source_port)
        briefing = await assembler.assemble(
            user_id="ryan",
            organization_id=str(uuid4()),
            role="operator",
            interests=["revenue"],
            briefing_date=date(2026, 4, 14),
        )

        assert len(briefing.items) == 3
        # Items should be scored and sorted by relevance
        assert briefing.items[0].relevance_score >= briefing.items[1].relevance_score

    @pytest.mark.asyncio
    async def test_limits_to_max_items(self) -> None:
        source_port = AsyncMock(spec=BriefingSourcePort)
        source_port.get_intelligence_items.return_value = [
            _make_item(f"Intel {i}", ItemSource.INTELLIGENCE, 0.5) for i in range(10)
        ]
        source_port.get_kpi_items.return_value = []
        source_port.get_action_items.return_value = []
        source_port.get_monitoring_items.return_value = []

        assembler = BriefingAssembler(source_port=source_port, max_items=5)
        briefing = await assembler.assemble(
            user_id="ryan", organization_id=str(uuid4()),
            role="operator", interests=[], briefing_date=date(2026, 4, 14),
        )
        assert len(briefing.items) <= 5
```

- [ ] **Step 4: Implement briefing assembler**

Create `apps/api/src/prescient/briefing/application/briefing_assembler.py`:

```python
from __future__ import annotations

from datetime import date, datetime, UTC
from typing import Protocol
from uuid import uuid4

from prescient.briefing.application.item_scorer import ItemScorer
from prescient.briefing.domain.entities.briefing_item import BriefingItem
from prescient.briefing.domain.entities.daily_briefing import DailyBriefing


class BriefingSourcePort(Protocol):
    async def get_intelligence_items(self, *, organization_id: str) -> list[BriefingItem]: ...
    async def get_kpi_items(self, *, organization_id: str) -> list[BriefingItem]: ...
    async def get_action_items(self, *, organization_id: str) -> list[BriefingItem]: ...
    async def get_monitoring_items(self, *, organization_id: str) -> list[BriefingItem]: ...


class BriefingAssembler:
    def __init__(self, source_port: BriefingSourcePort, max_items: int = 7) -> None:
        self._source = source_port
        self._max = max_items

    async def assemble(
        self,
        user_id: str,
        organization_id: str,
        role: str,
        interests: list[str],
        briefing_date: date,
    ) -> DailyBriefing:
        # Gather items from all sources
        all_items: list[BriefingItem] = []
        all_items.extend(await self._source.get_intelligence_items(organization_id=organization_id))
        all_items.extend(await self._source.get_kpi_items(organization_id=organization_id))
        all_items.extend(await self._source.get_action_items(organization_id=organization_id))
        all_items.extend(await self._source.get_monitoring_items(organization_id=organization_id))

        # Score each item
        scorer = ItemScorer(role=role, interests=interests)
        scored = [scorer.score(item) for item in all_items]

        # Sort by score descending, take top N
        scored.sort(key=lambda i: i.relevance_score, reverse=True)
        top = scored[: self._max]

        is_monday = briefing_date.weekday() == 0

        return DailyBriefing(
            id=str(uuid4()),
            user_id=user_id,
            organization_id=organization_id,
            date=briefing_date,
            items=tuple(top),
            is_monday=is_monday,
            generated_at=datetime.now(UTC),
        )
```

- [ ] **Step 5: Run tests and commit**

Run: `cd apps/api && uv run python -m pytest tests/unit/briefing/ -v`

```bash
git add apps/api/src/prescient/briefing/application/ tests/unit/briefing/
git commit -m "feat: add briefing assembler and item scorer — personalized daily briefing generation"
```

---

## Task 8: Briefing API and Use Case

**Files:**
- Create: `apps/api/src/prescient/briefing/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/briefing/application/use_cases/get_daily_briefing.py`
- Create: `apps/api/src/prescient/briefing/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/briefing/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/briefing/infrastructure/tables/briefing.py`
- Create: `apps/api/src/prescient/briefing/api/__init__.py`
- Create: `apps/api/src/prescient/briefing/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`
- Modify: `apps/api/src/prescient/shared/metadata.py`
- Test: `apps/api/tests/unit/briefing/test_get_daily_briefing.py`

- [ ] **Step 1: Create briefing table**

Create `apps/api/src/prescient/briefing/infrastructure/tables/briefing.py`:

```python
from __future__ import annotations

from datetime import date, datetime
from uuid import uuid4

from sqlalchemy import Boolean, Date, DateTime, String
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class BriefingRow(Base):
    __tablename__ = "briefings"
    __table_args__ = {"schema": "briefing"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    user_id: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    briefing_date: Mapped[date] = mapped_column(Date, nullable=False)
    items: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="[]")
    is_monday: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default="false")
    generated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 2: Implement get_daily_briefing use case**

Create `apps/api/src/prescient/briefing/application/use_cases/get_daily_briefing.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import date

from prescient.briefing.application.briefing_assembler import BriefingAssembler, BriefingSourcePort
from prescient.briefing.domain.entities.daily_briefing import DailyBriefing


@dataclass(frozen=True)
class GetDailyBriefingQuery:
    user_id: str
    organization_id: str
    role: str
    interests: list[str]
    briefing_date: date


class GetDailyBriefing:
    def __init__(self, source_port: BriefingSourcePort, max_items: int = 7) -> None:
        self._assembler = BriefingAssembler(source_port=source_port, max_items=max_items)

    async def execute(self, query: GetDailyBriefingQuery) -> DailyBriefing:
        return await self._assembler.assemble(
            user_id=query.user_id,
            organization_id=query.organization_id,
            role=query.role,
            interests=query.interests,
            briefing_date=query.briefing_date,
        )
```

- [ ] **Step 3: Implement briefing API route**

Create `apps/api/src/prescient/briefing/api/routes.py`:

```python
from __future__ import annotations

from datetime import date, datetime, UTC
from uuid import uuid4

from pydantic import BaseModel
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.briefing.application.use_cases.get_daily_briefing import GetDailyBriefing, GetDailyBriefingQuery
from prescient.briefing.domain.entities.briefing_item import ActionType, BriefingItem, ItemSource
from prescient.db import get_session

router = APIRouter(prefix="/briefing", tags=["briefing"])


class BriefingItemResponse(BaseModel):
    id: str
    title: str
    summary: str
    action_type: str
    source: str
    source_artifact_id: str | None
    relevance_score: float


class DailyBriefingResponse(BaseModel):
    id: str
    user_id: str
    organization_id: str
    date: str
    items: list[BriefingItemResponse]
    is_monday: bool


class _StubSourceAdapter:
    """Stub source adapter — returns empty lists until real sources are wired."""

    async def get_intelligence_items(self, *, organization_id: str) -> list[BriefingItem]:
        return []

    async def get_kpi_items(self, *, organization_id: str) -> list[BriefingItem]:
        return []

    async def get_action_items(self, *, organization_id: str) -> list[BriefingItem]:
        return []

    async def get_monitoring_items(self, *, organization_id: str) -> list[BriefingItem]:
        return []


@router.get("/today", response_model=DailyBriefingResponse)
async def get_todays_briefing(
    user_id: str,
    organization_id: str,
    role: str = "operator",
    session: AsyncSession = Depends(get_session),
) -> DailyBriefingResponse:
    source_adapter = _StubSourceAdapter()
    use_case = GetDailyBriefing(source_port=source_adapter)
    today = date.today()
    briefing = await use_case.execute(GetDailyBriefingQuery(
        user_id=user_id,
        organization_id=organization_id,
        role=role,
        interests=[],
        briefing_date=today,
    ))
    return DailyBriefingResponse(
        id=briefing.id,
        user_id=briefing.user_id,
        organization_id=briefing.organization_id,
        date=str(briefing.date),
        items=[
            BriefingItemResponse(
                id=i.id, title=i.title, summary=i.summary,
                action_type=i.action_type.value, source=i.source.value,
                source_artifact_id=i.source_artifact_id, relevance_score=i.relevance_score,
            ) for i in briefing.items
        ],
        is_monday=briefing.is_monday,
    )
```

- [ ] **Step 4: Register tables, migration, wire router**

Add to metadata.py: `import prescient.briefing.infrastructure.tables  # noqa: F401`

Create schema + migration for briefing.

Wire `briefing_router` into main.py.

- [ ] **Step 5: Run tests and commit**

```bash
git add apps/api/src/prescient/briefing/ apps/api/src/prescient/main.py apps/api/src/prescient/shared/metadata.py apps/api/alembic/ tests/unit/briefing/
git commit -m "feat: add Daily Briefing API — personalized morning brief with stub sources"
```

---

## Task 9: Frontend — Morning Brief, Actions, KPIs Pages

**Files:**
- Modify: `apps/web/src/app/(main)/brief/page.tsx`
- Create: `apps/web/src/app/(main)/actions/page.tsx`
- Create: `apps/web/src/app/(main)/kpis/page.tsx`
- Create: `apps/web/src/components/briefing-item.tsx`
- Create: `apps/web/src/components/action-item-row.tsx`
- Create: `apps/web/src/components/kpi-report-form.tsx`
- Modify: `apps/web/src/components/nav.tsx`

- [ ] **Step 1: Read existing frontend patterns**

Read the current brief page, nav, auth, and shadcn components.

- [ ] **Step 2: Replace morning brief placeholder with real UI**

Replace `apps/web/src/app/(main)/brief/page.tsx` with a client component that:
- Fetches from `GET /briefing/today?user_id=X&organization_id=X`
- Shows date and "Good morning" header
- Monday variant shows "Here's your week ahead" subheader
- Lists briefing items as cards with action type badges (Read=blue, Decide=orange, FYI=gray)
- Each item shows title, summary, source badge, and relevance score indicator
- Empty state: "Nothing urgent today — enjoy the quiet."

- [ ] **Step 3: Create BriefingItem component**

Create `apps/web/src/components/briefing-item.tsx` — card with action type badge, title, summary, source indicator.

- [ ] **Step 4: Create actions page**

Create `apps/web/src/app/(main)/actions/page.tsx`:
- Fetches from `GET /actions/items?organization_id=X`
- Table/list of action items with: title, owner, priority badge, status badge, due date
- Status management: approve/reject/defer buttons for pending items

- [ ] **Step 5: Create KPI reporting page**

Create `apps/web/src/app/(main)/kpis/page.tsx`:
- Form to submit a KPI value (kpi_id dropdown, period, value, commentary)
- Shows anomaly alert if detected after submission
- Lists recent KPI values

- [ ] **Step 6: Update nav to activate Actions and Board Prep links**

Update `apps/web/src/components/nav.tsx` — change "Actions" and KPI links from placeholder spans to real `<Link>` elements.

- [ ] **Step 7: Verify frontend compiles**

Run: `cd apps/web && npx tsc --noEmit`

- [ ] **Step 8: Commit**

```bash
git add apps/web/src/
git commit -m "feat: add Morning Brief, Actions, and KPI pages — operator daily workflow"
```

---

## Task 10: Lint, Typecheck, and Final Verification

- [ ] **Step 1: Run backend linter**

Run: `cd apps/api && uv run python -m ruff check src/prescient/kpis/ src/prescient/briefing/ src/prescient/action_items/`

- [ ] **Step 2: Run backend type checker**

Run: `cd apps/api && uv run python -m mypy src/prescient/kpis/ src/prescient/briefing/`

- [ ] **Step 3: Run frontend type check**

Run: `cd apps/web && npx tsc --noEmit`

- [ ] **Step 4: Run all Phase 3 tests**

Run: `cd apps/api && uv run python -m pytest tests/unit/kpis/ tests/unit/briefing/ tests/unit/action_items/ -v`

- [ ] **Step 5: Fix any issues and commit**

```bash
git commit -m "fix: address lint and typecheck issues from Phase 3"
```

---

## Summary

Phase 3 delivers:
- **KPI Management** — KpiOwner (cadence-based assignment), KpiValue (point-in-time metrics), KpiAnomaly (deviation detection), anomaly detector that compares new values against history, enrichment question generation. API: assign owner, report value (returns anomaly if detected), get series.
- **Action & Decision Log** — Application layer for existing action items context: create, list, update status. Decision records persisted as artifacts with context/decision/rationale blocks. API: CRUD for action items, create decision records.
- **Daily Briefing** — BriefingItem (action-typed: Read/Decide/FYI), DailyBriefing aggregate, ItemScorer (role + interest-based personalization), BriefingAssembler (collects from intelligence/KPI/actions/monitoring sources), stub source adapter for MVP. API: get today's briefing.
- **Frontend** — Morning Brief page (real UI replacing placeholder), Actions page (list + status management), KPI reporting page (submit values, anomaly alerts). Nav updated with active links.
- **Infrastructure** — Alembic migrations for kpis and briefing schemas.

Next: **Phase 4 — Reporting & Sponsors: Board Prep View, Sponsor Access, Triage Flow**
