# Live Data Pipelines Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace seed-only data loading with live Celery-based pipelines — monitor runner (SEC + NewsAPI + URL change detection), KPI CSV intake with LLM-assisted mapping, and an outbox event publisher with subscribers.

**Architecture:** All background work runs as Celery tasks on a single worker pool, scheduled by Celery Beat. The transactional outbox (already implemented) gets a drain task that dispatches domain events to registered subscriber tasks. Monitor sources use an adapter protocol for extensibility.

**Tech Stack:** Celery + Redis (existing), NewsAPI.org, SEC EDGAR (existing `EdgarClient`), Anthropic Claude (existing), PostgreSQL + Alembic, httpx for HTTP adapters.

**Worktree:** `/home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines`

**Test command:** `PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest`

**Working directory for all commands:** `/home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api`

---

## File Structure

### New files

```
apps/api/src/prescient/
  monitoring/infrastructure/tasks/
    __init__.py
    sweep.py                              # monitor.sweep + monitor.check_target Celery tasks
    adapters/
      __init__.py
      base.py                             # MonitorSourceAdapter protocol + RawFinding dataclass
      sec_filing.py                       # SEC EDGAR adapter
      news_api.py                         # NewsAPI.org adapter
      url_change.py                       # URL content change detection adapter

  kpis/
    domain/entities/intake_template.py    # IntakeTemplate, ColumnMapping, PeriodConfig
    infrastructure/
      tables/intake_template.py           # IntakeTemplateRow ORM
      repositories/intake_template_repository.py
      tasks/
        __init__.py
        check_overdue.py                  # daily overdue KPI check task
    application/use_cases/intake_csv.py   # LLM-assisted CSV mapping + batch ingestion

  shared/infrastructure/tasks/
    __init__.py
    outbox_drain.py                       # outbox drain task + subscriber registry

  events/
    __init__.py
    subscribers/
      __init__.py
      anomaly_to_triage.py               # kpi.anomaly_detected → create triage item
      target_immediate_check.py           # monitoring.target.created → immediate check

  triage/queries.py                       # published query interface for creating triage items

apps/api/alembic/versions/
  20260413_0007_live_pipelines.py         # migration: intake_templates + monitor columns

apps/api/tests/
  unit/monitoring/test_sweep.py
  unit/monitoring/test_sec_filing_adapter.py
  unit/monitoring/test_news_api_adapter.py
  unit/monitoring/test_url_change_adapter.py
  unit/kpis/test_intake_csv.py
  unit/kpis/test_intake_template.py
  unit/shared/test_outbox_drain.py
  unit/events/test_anomaly_to_triage.py
  unit/events/test_target_immediate_check.py
```

### Modified files

```
apps/api/src/prescient/celery_app.py                    # beat_schedule + expanded autodiscover
apps/api/src/prescient/config/base.py                    # newsapi_api_key setting
apps/api/src/prescient/monitoring/infrastructure/tables/monitor_target.py  # +last_checked_at, +last_content_hash
apps/api/src/prescient/monitoring/infrastructure/tables/monitor_finding.py # +content_hash
apps/api/src/prescient/kpis/application/use_cases/report_kpi_value.py     # emit outbox events
apps/api/src/prescient/monitoring/application/use_cases/add_watch_target.py # emit outbox event
apps/api/src/prescient/kpis/api/routes.py                # intake upload + confirm endpoints
apps/api/src/prescient/monitoring/api/routes.py          # GET /monitoring/runs endpoint
infrastructure/docker-compose.yml                        # worker + beat services
```

---

### Task 1: Celery Beat Infrastructure

**Files:**
- Modify: `apps/api/src/prescient/celery_app.py`
- Modify: `apps/api/src/prescient/config/base.py`
- Modify: `infrastructure/docker-compose.yml`

- [ ] **Step 1: Add `newsapi_api_key` to settings**

In `apps/api/src/prescient/config/base.py`, add after the `embedding_model` field:

```python
newsapi_api_key: str = Field(default="")
```

- [ ] **Step 2: Expand Celery autodiscover and add beat schedule**

Replace the full content of `apps/api/src/prescient/celery_app.py`:

```python
from celery import Celery
from celery.schedules import schedule, crontab

from prescient.config.base import get_settings


def create_celery_app() -> Celery:
    settings = get_settings()
    app = Celery(
        "prescient",
        broker=settings.redis_url,
        backend=settings.redis_url,
    )
    app.conf.update(
        task_serializer="json",
        accept_content=["json"],
        result_serializer="json",
        timezone="UTC",
        task_track_started=True,
        task_acks_late=True,
        worker_prefetch_multiplier=1,
        beat_schedule={
            "monitor-sweep": {
                "task": "prescient.monitoring.infrastructure.tasks.sweep.monitor_sweep",
                "schedule": schedule(run_every=1800),  # 30 minutes
            },
            "outbox-drain": {
                "task": "prescient.shared.infrastructure.tasks.outbox_drain.drain_outbox",
                "schedule": schedule(run_every=30),  # 30 seconds
            },
            "kpi-check-overdue": {
                "task": "prescient.kpis.infrastructure.tasks.check_overdue.check_overdue_kpis",
                "schedule": crontab(hour=8, minute=0),  # daily 8:00 UTC
            },
        },
    )
    app.autodiscover_tasks([
        "prescient.knowledge_mgmt.infrastructure.tasks",
        "prescient.monitoring.infrastructure.tasks",
        "prescient.kpis.infrastructure.tasks",
        "prescient.shared.infrastructure.tasks",
        "prescient.events.subscribers",
    ])
    return app


celery_app = create_celery_app()
```

- [ ] **Step 3: Add worker and beat services to docker-compose**

In `infrastructure/docker-compose.yml`, add before the `volumes:` section:

```yaml
  worker:
    build:
      context: ../apps/api
      target: dev
    container_name: prescient-worker
    env_file:
      - ../.env
    volumes:
      - ../apps/api/src:/app/src
      - ../seed:/repo/seed
    environment:
      - PRESCIENT_ENV=demo
      - DATABASE_URL=postgresql+asyncpg://prescient:prescient@postgres:5432/prescient
      - OPENSEARCH_URL=http://opensearch:9200
      - REDIS_URL=redis://redis:6379/0
      - PYTHONPATH=/repo
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: ["uv", "run", "celery", "-A", "prescient.celery_app", "worker", "--loglevel=info", "--pool=solo"]

  beat:
    build:
      context: ../apps/api
      target: dev
    container_name: prescient-beat
    env_file:
      - ../.env
    volumes:
      - ../apps/api/src:/app/src
      - ../seed:/repo/seed
    environment:
      - PRESCIENT_ENV=demo
      - DATABASE_URL=postgresql+asyncpg://prescient:prescient@postgres:5432/prescient
      - OPENSEARCH_URL=http://opensearch:9200
      - REDIS_URL=redis://redis:6379/0
      - PYTHONPATH=/repo
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: ["uv", "run", "celery", "-A", "prescient.celery_app", "beat", "--loglevel=info"]
```

- [ ] **Step 4: Create placeholder `__init__.py` files for new task packages**

Create empty `__init__.py` in each new package:

- `apps/api/src/prescient/monitoring/infrastructure/tasks/__init__.py`
- `apps/api/src/prescient/kpis/infrastructure/tasks/__init__.py`
- `apps/api/src/prescient/shared/infrastructure/__init__.py`
- `apps/api/src/prescient/shared/infrastructure/tasks/__init__.py`
- `apps/api/src/prescient/events/__init__.py`
- `apps/api/src/prescient/events/subscribers/__init__.py`
- `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/__init__.py`

- [ ] **Step 5: Verify Celery app loads without errors**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -c "from prescient.celery_app import celery_app; print(celery_app.conf.beat_schedule.keys())"`

Expected: prints the three schedule keys without import errors.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: add Celery Beat schedule and worker/beat docker services"
```

---

### Task 2: Alembic Migration

**Files:**
- Create: `apps/api/alembic/versions/20260413_0007_live_pipelines.py`
- Modify: `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_target.py`
- Modify: `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_finding.py`

- [ ] **Step 1: Add columns to MonitorTargetRow**

In `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_target.py`, add after the `created_at` column:

```python
last_checked_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
last_content_hash: Mapped[str | None] = mapped_column(String(64), nullable=True)
```

Also add `String` to the sqlalchemy import if not already present (it already is).

- [ ] **Step 2: Add content_hash to MonitorFindingRow**

In `apps/api/src/prescient/monitoring/infrastructure/tables/monitor_finding.py`, add after the `processed` column:

```python
content_hash: Mapped[str | None] = mapped_column(String(64), nullable=True, index=True)
```

- [ ] **Step 3: Create the migration file**

Create `apps/api/alembic/versions/20260413_0007_live_pipelines.py`:

```python
"""live data pipelines — intake templates + monitor columns

Revision ID: 0007_live_pipelines
Revises: 0006_km_schema
Create Date: 2026-04-13
"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB

revision = "0007_live_pipelines"
down_revision = "0006_km_schema"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # -- monitoring.monitor_targets: add sweep tracking columns --
    op.add_column(
        "monitor_targets",
        sa.Column("last_checked_at", sa.DateTime(timezone=True), nullable=True),
        schema="monitoring",
    )
    op.add_column(
        "monitor_targets",
        sa.Column("last_content_hash", sa.String(64), nullable=True),
        schema="monitoring",
    )

    # -- monitoring.monitor_findings: add dedup hash --
    op.add_column(
        "monitor_findings",
        sa.Column("content_hash", sa.String(64), nullable=True),
        schema="monitoring",
    )
    op.create_index(
        "ix_monitor_findings_content_hash",
        "monitor_findings",
        ["content_hash"],
        schema="monitoring",
    )

    # -- kpis.intake_templates --
    op.execute("CREATE SCHEMA IF NOT EXISTS kpis")
    op.create_table(
        "intake_templates",
        sa.Column("id", sa.dialects.postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("scope", sa.String(16), nullable=False, server_default="organization"),
        sa.Column("organization_id", sa.String(36), nullable=True),
        sa.Column("source_system", sa.String(64), nullable=True),
        sa.Column("report_type", sa.String(64), nullable=True),
        sa.Column("name", sa.String(256), nullable=False),
        sa.Column("header_fingerprint", sa.String(64), nullable=False, index=True),
        sa.Column("column_mappings", JSONB, nullable=False, server_default="[]"),
        sa.Column("period_config", JSONB, nullable=False, server_default="{}"),
        sa.Column("kpi_mapping_overrides", JSONB, nullable=True),
        sa.Column("created_by", sa.String(64), nullable=False),
        sa.Column("last_used_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=True),
        schema="kpis",
    )


def downgrade() -> None:
    op.drop_table("intake_templates", schema="kpis")
    op.drop_index("ix_monitor_findings_content_hash", "monitor_findings", schema="monitoring")
    op.drop_column("monitor_findings", "content_hash", schema="monitoring")
    op.drop_column("monitor_targets", "last_content_hash", schema="monitoring")
    op.drop_column("monitor_targets", "last_checked_at", schema="monitoring")
```

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: add migration for intake_templates table and monitor sweep columns"
```

---

### Task 3: Monitor Source Adapter Protocol

**Files:**
- Create: `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/base.py`
- Test: `apps/api/tests/unit/monitoring/test_adapters_base.py`

- [ ] **Step 1: Write test for RawFinding and content_hash**

Create `apps/api/tests/unit/monitoring/test_adapters_base.py`:

```python
"""Tests for monitor source adapter base types."""

from prescient.monitoring.infrastructure.tasks.adapters.base import RawFinding


def test_raw_finding_content_hash_deterministic():
    f1 = RawFinding(
        title="10-Q filed 2026-04-10",
        finding_type="filing",
        severity="notable",
        source_url="https://sec.gov/filing/123",
    )
    f2 = RawFinding(
        title="10-Q filed 2026-04-10",
        finding_type="filing",
        severity="notable",
        source_url="https://sec.gov/filing/123",
    )
    assert f1.content_hash == f2.content_hash
    assert len(f1.content_hash) == 64  # SHA256 hex digest


def test_raw_finding_content_hash_differs_on_title():
    f1 = RawFinding(title="10-Q filed 2026-04-10", finding_type="filing", severity="notable")
    f2 = RawFinding(title="10-K filed 2026-04-10", finding_type="filing", severity="notable")
    assert f1.content_hash != f2.content_hash


def test_raw_finding_content_hash_none_url():
    f = RawFinding(title="Something happened", finding_type="custom", severity="info")
    assert f.source_url is None
    assert len(f.content_hash) == 64
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_adapters_base.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement adapter protocol and RawFinding**

Create `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/base.py`:

```python
"""Monitor source adapter protocol and shared types."""

from __future__ import annotations

import hashlib
from dataclasses import dataclass, field
from datetime import datetime
from typing import Protocol

from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow


@dataclass
class RawFinding:
    """A finding discovered by a monitor source adapter before persistence."""

    title: str
    finding_type: str  # matches FindingType values
    severity: str  # matches FindingSeverity values
    summary: str | None = None
    source_url: str | None = None
    occurred_at: datetime | None = None
    raw_payload: dict | None = None

    @property
    def content_hash(self) -> str:
        """Deterministic hash for deduplication."""
        parts = f"{self.source_url or ''}|{self.title}|{self.finding_type}"
        return hashlib.sha256(parts.encode()).hexdigest()


# Default source adapters per target type
DEFAULT_SOURCES: dict[str, list[str]] = {
    "COMPANY": ["sec_filing", "news_api"],
    "PRODUCT": ["news_api", "url_change"],
    "MARKET": ["news_api"],
    "PERSON": ["news_api"],
    "TOPIC": ["news_api"],
}


class MonitorSourceAdapter(Protocol):
    """Protocol for all monitor source adapters.

    Each adapter checks a single external source for changes relevant to
    a watch target since the last check time.
    """

    async def check(
        self, target: MonitorTargetRow, since: datetime | None
    ) -> list[RawFinding]: ...
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_adapters_base.py -v`

Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add MonitorSourceAdapter protocol and RawFinding dataclass"
```

---

### Task 4: SEC Filing Adapter

**Files:**
- Create: `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/sec_filing.py`
- Test: `apps/api/tests/unit/monitoring/test_sec_filing_adapter.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/monitoring/test_sec_filing_adapter.py`:

```python
"""Tests for SEC filing monitor adapter."""

from __future__ import annotations

from datetime import date, datetime, timezone
from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.monitoring.infrastructure.tasks.adapters.sec_filing import SecFilingAdapter


def _make_target(cik: str = "0001868726", company_name: str = "Echelon") -> MagicMock:
    target = MagicMock()
    target.id = uuid4()
    target.config = {
        "cik": cik,
        "company_name": company_name,
        "sources": ["sec_filing"],
    }
    return target


def _make_filing_ref(form: str, filing_date: date, accession: str) -> MagicMock:
    ref = MagicMock()
    ref.form = form
    ref.filing_date = filing_date
    ref.accession_number = accession
    return ref


@pytest.mark.asyncio
async def test_returns_findings_for_new_filings():
    client = AsyncMock()
    client.list_filings.return_value = [
        _make_filing_ref("10-Q", date(2026, 4, 10), "0001868726-26-000123"),
    ]
    adapter = SecFilingAdapter(edgar_client=client)
    since = datetime(2026, 4, 1, tzinfo=timezone.utc)

    findings = await adapter.check(_make_target(), since)

    assert len(findings) == 1
    assert findings[0].finding_type == "filing"
    assert "10-Q" in findings[0].title
    assert findings[0].severity == "notable"


@pytest.mark.asyncio
async def test_filters_filings_before_since():
    client = AsyncMock()
    client.list_filings.return_value = [
        _make_filing_ref("10-Q", date(2026, 3, 15), "0001868726-26-000100"),  # before since
        _make_filing_ref("10-K", date(2026, 4, 10), "0001868726-26-000123"),  # after since
    ]
    adapter = SecFilingAdapter(edgar_client=client)
    since = datetime(2026, 4, 1, tzinfo=timezone.utc)

    findings = await adapter.check(_make_target(), since)

    assert len(findings) == 1
    assert "10-K" in findings[0].title


@pytest.mark.asyncio
async def test_returns_empty_on_no_cik():
    client = AsyncMock()
    target = _make_target()
    target.config = {"company_name": "NoCIK"}
    adapter = SecFilingAdapter(edgar_client=client)

    findings = await adapter.check(target, None)

    assert findings == []
    client.list_filings.assert_not_called()


@pytest.mark.asyncio
async def test_returns_empty_on_client_error():
    client = AsyncMock()
    client.list_filings.side_effect = Exception("network error")
    adapter = SecFilingAdapter(edgar_client=client)

    findings = await adapter.check(_make_target(), None)

    assert findings == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_sec_filing_adapter.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement SecFilingAdapter**

Create `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/sec_filing.py`:

```python
"""SEC EDGAR filing adapter — checks for new filings since last check."""

from __future__ import annotations

import logging
from datetime import datetime, timezone

from prescient.monitoring.infrastructure.tasks.adapters.base import (
    MonitorSourceAdapter,
    RawFinding,
)
from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow

logger = logging.getLogger(__name__)

# Filing types considered important get higher severity
_IMPORTANT_FORMS = {"8-K", "DEF 14A", "SC 13D", "SC 13G"}


class SecFilingAdapter:
    """Checks SEC EDGAR for new filings for a company identified by CIK."""

    def __init__(self, edgar_client: object) -> None:
        self._edgar = edgar_client

    async def check(
        self, target: MonitorTargetRow, since: datetime | None
    ) -> list[RawFinding]:
        cik = (target.config or {}).get("cik")
        if not cik:
            logger.debug("Target %s has no CIK, skipping SEC check", target.id)
            return []

        company_name = (target.config or {}).get("company_name", "Unknown")

        try:
            filing_refs = await self._edgar.list_filings(
                cik=cik,
                form_counts={"10-K": 3, "10-Q": 4, "8-K": 5},
            )
        except Exception:
            logger.exception("Failed to list filings for CIK %s", cik)
            return []

        findings: list[RawFinding] = []
        for ref in filing_refs:
            filing_dt = datetime.combine(
                ref.filing_date, datetime.min.time(), tzinfo=timezone.utc
            )
            if since and filing_dt <= since:
                continue

            severity = "important" if ref.form in _IMPORTANT_FORMS else "notable"
            findings.append(
                RawFinding(
                    title=f"{ref.form} filed {ref.filing_date} — {company_name}",
                    finding_type="filing",
                    severity=severity,
                    source_url=f"https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK={cik}&type={ref.form}",
                    occurred_at=filing_dt,
                    raw_payload={
                        "form": ref.form,
                        "filing_date": str(ref.filing_date),
                        "accession_number": ref.accession_number,
                        "cik": cik,
                    },
                )
            )

        return findings
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_sec_filing_adapter.py -v`

Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add SEC filing monitor adapter"
```

---

### Task 5: NewsAPI Adapter

**Files:**
- Create: `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/news_api.py`
- Test: `apps/api/tests/unit/monitoring/test_news_api_adapter.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/monitoring/test_news_api_adapter.py`:

```python
"""Tests for NewsAPI monitor adapter."""

from __future__ import annotations

from datetime import datetime, timezone
from unittest.mock import AsyncMock, MagicMock, patch
from uuid import uuid4

import pytest

from prescient.monitoring.infrastructure.tasks.adapters.news_api import NewsApiAdapter


def _make_target(
    company_name: str = "Echelon",
    news_keywords: list[str] | None = None,
) -> MagicMock:
    target = MagicMock()
    target.id = uuid4()
    target.config = {
        "company_name": company_name,
        "sources": ["news_api"],
    }
    if news_keywords:
        target.config["news_keywords"] = news_keywords
    return target


def _news_response(articles: list[dict]) -> dict:
    return {"status": "ok", "totalResults": len(articles), "articles": articles}


def _article(title: str, url: str, published_at: str, description: str = "") -> dict:
    return {
        "title": title,
        "url": url,
        "publishedAt": published_at,
        "description": description,
        "source": {"name": "Test Source"},
    }


@pytest.mark.asyncio
async def test_returns_findings_from_articles():
    response = _news_response([
        _article(
            "Echelon launches new bike",
            "https://example.com/1",
            "2026-04-10T12:00:00Z",
            "Echelon announces a new product.",
        ),
    ])
    mock_client = AsyncMock()
    mock_resp = AsyncMock()
    mock_resp.status_code = 200
    mock_resp.json.return_value = response
    mock_resp.raise_for_status = MagicMock()
    mock_client.get.return_value = mock_resp

    adapter = NewsApiAdapter(api_key="test-key", http_client=mock_client)
    findings = await adapter.check(_make_target(), since=None)

    assert len(findings) == 1
    assert findings[0].finding_type == "new_document"
    assert "Echelon launches new bike" in findings[0].title
    assert findings[0].source_url == "https://example.com/1"


@pytest.mark.asyncio
async def test_severity_classification_exec_move():
    response = _news_response([
        _article(
            "Echelon CEO resigns",
            "https://example.com/2",
            "2026-04-10T12:00:00Z",
            "The CEO has stepped down.",
        ),
    ])
    mock_client = AsyncMock()
    mock_resp = AsyncMock()
    mock_resp.status_code = 200
    mock_resp.json.return_value = response
    mock_resp.raise_for_status = MagicMock()
    mock_client.get.return_value = mock_resp

    adapter = NewsApiAdapter(api_key="test-key", http_client=mock_client)
    findings = await adapter.check(_make_target(), since=None)

    assert findings[0].severity == "important"


@pytest.mark.asyncio
async def test_uses_news_keywords_in_query():
    mock_client = AsyncMock()
    mock_resp = AsyncMock()
    mock_resp.status_code = 200
    mock_resp.json.return_value = _news_response([])
    mock_resp.raise_for_status = MagicMock()
    mock_client.get.return_value = mock_resp

    adapter = NewsApiAdapter(api_key="test-key", http_client=mock_client)
    await adapter.check(
        _make_target(news_keywords=["connected fitness", "home gym"]),
        since=None,
    )

    call_kwargs = mock_client.get.call_args
    params = call_kwargs.kwargs.get("params", call_kwargs[1].get("params", {}))
    assert "connected fitness" in params["q"] or "home gym" in params["q"]


@pytest.mark.asyncio
async def test_returns_empty_on_no_api_key():
    adapter = NewsApiAdapter(api_key="", http_client=AsyncMock())
    findings = await adapter.check(_make_target(), since=None)
    assert findings == []


@pytest.mark.asyncio
async def test_returns_empty_on_api_error():
    mock_client = AsyncMock()
    mock_client.get.side_effect = Exception("timeout")

    adapter = NewsApiAdapter(api_key="test-key", http_client=mock_client)
    findings = await adapter.check(_make_target(), since=None)
    assert findings == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_news_api_adapter.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement NewsApiAdapter**

Create `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/news_api.py`:

```python
"""NewsAPI.org adapter — searches for news articles relevant to a watch target."""

from __future__ import annotations

import logging
import re
from datetime import datetime, timezone

from prescient.monitoring.infrastructure.tasks.adapters.base import (
    RawFinding,
)
from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow

logger = logging.getLogger(__name__)

_NEWSAPI_URL = "https://newsapi.org/v2/everything"

# Keywords that signal higher-severity findings
_IMPORTANT_PATTERNS = re.compile(
    r"(CEO|CFO|CTO|COO|president)\s+(resign|step.?down|fired|replaced|depart|hire|appoint)"
    r"|acqui(re|sition|red)|merger|layoff|bankrupt|lawsuit|SEC\s+investig",
    re.IGNORECASE,
)
_NOTABLE_PATTERNS = re.compile(
    r"launch|partnership|funding|IPO|expansion|restructur|recall|breach",
    re.IGNORECASE,
)


def _classify_severity(title: str, description: str) -> str:
    """Heuristic severity classification based on headline and description text."""
    text = f"{title} {description}"
    if _IMPORTANT_PATTERNS.search(text):
        return "important"
    if _NOTABLE_PATTERNS.search(text):
        return "notable"
    return "info"


class NewsApiAdapter:
    """Searches NewsAPI.org for articles relevant to a watch target."""

    def __init__(self, api_key: str, http_client: object) -> None:
        self._api_key = api_key
        self._http = http_client

    async def check(
        self, target: MonitorTargetRow, since: datetime | None
    ) -> list[RawFinding]:
        if not self._api_key:
            logger.warning("No NewsAPI key configured, skipping news check")
            return []

        config = target.config or {}
        company_name = config.get("company_name", "")
        keywords = config.get("news_keywords", [])

        # Build query: company name OR any keyword
        query_parts = [f'"{company_name}"'] if company_name else []
        for kw in keywords:
            query_parts.append(f'"{kw}"')
        if not query_parts:
            logger.debug("Target %s has no company_name or keywords, skipping", target.id)
            return []

        query = " OR ".join(query_parts)

        params: dict[str, str] = {
            "q": query,
            "sortBy": "publishedAt",
            "pageSize": "20",
            "apiKey": self._api_key,
        }
        if since:
            params["from"] = since.strftime("%Y-%m-%dT%H:%M:%SZ")

        try:
            response = await self._http.get(_NEWSAPI_URL, params=params)
            response.raise_for_status()
            data = response.json()
        except Exception:
            logger.exception("NewsAPI request failed for target %s", target.id)
            return []

        findings: list[RawFinding] = []
        for article in data.get("articles", []):
            title = article.get("title", "")
            description = article.get("description", "") or ""
            url = article.get("url", "")
            published = article.get("publishedAt", "")

            occurred_at = None
            if published:
                try:
                    occurred_at = datetime.fromisoformat(published.replace("Z", "+00:00"))
                except ValueError:
                    pass

            severity = _classify_severity(title, description)

            findings.append(
                RawFinding(
                    title=title,
                    finding_type="new_document",
                    severity=severity,
                    summary=description[:500] if description else None,
                    source_url=url,
                    occurred_at=occurred_at,
                    raw_payload={
                        "source_name": article.get("source", {}).get("name"),
                        "author": article.get("author"),
                    },
                )
            )

        return findings
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_news_api_adapter.py -v`

Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add NewsAPI monitor adapter with severity heuristics"
```

---

### Task 6: URL Change Adapter

**Files:**
- Create: `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/url_change.py`
- Test: `apps/api/tests/unit/monitoring/test_url_change_adapter.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/monitoring/test_url_change_adapter.py`:

```python
"""Tests for URL change detection adapter."""

from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.monitoring.infrastructure.tasks.adapters.url_change import UrlChangeAdapter


def _make_target(url: str = "https://example.com/pricing", last_hash: str | None = None) -> MagicMock:
    target = MagicMock()
    target.id = uuid4()
    target.config = {"url": url, "sources": ["url_change"]}
    target.last_content_hash = last_hash
    return target


@pytest.mark.asyncio
async def test_detects_change_from_previous_hash():
    mock_client = AsyncMock()
    mock_resp = AsyncMock()
    mock_resp.status_code = 200
    mock_resp.text = "new content here"
    mock_resp.raise_for_status = MagicMock()
    mock_client.get.return_value = mock_resp

    adapter = UrlChangeAdapter(http_client=mock_client)
    findings = await adapter.check(_make_target(last_hash="oldhash"), since=None)

    assert len(findings) == 1
    assert findings[0].finding_type == "custom"
    assert "changed" in findings[0].title.lower()


@pytest.mark.asyncio
async def test_no_finding_when_hash_matches():
    import hashlib

    content = "same content"
    content_hash = hashlib.sha256(content.encode()).hexdigest()

    mock_client = AsyncMock()
    mock_resp = AsyncMock()
    mock_resp.status_code = 200
    mock_resp.text = content
    mock_resp.raise_for_status = MagicMock()
    mock_client.get.return_value = mock_resp

    adapter = UrlChangeAdapter(http_client=mock_client)
    findings = await adapter.check(_make_target(last_hash=content_hash), since=None)

    assert findings == []


@pytest.mark.asyncio
async def test_first_check_no_previous_hash():
    mock_client = AsyncMock()
    mock_resp = AsyncMock()
    mock_resp.status_code = 200
    mock_resp.text = "initial content"
    mock_resp.raise_for_status = MagicMock()
    mock_client.get.return_value = mock_resp

    adapter = UrlChangeAdapter(http_client=mock_client)
    findings = await adapter.check(_make_target(last_hash=None), since=None)

    # First check — no previous hash to compare, so no finding (just record baseline)
    assert findings == []
    # But new_hash should be set on the adapter result
    assert adapter.last_computed_hash is not None


@pytest.mark.asyncio
async def test_no_url_in_config():
    target = MagicMock()
    target.id = uuid4()
    target.config = {"sources": ["url_change"]}
    target.last_content_hash = None

    adapter = UrlChangeAdapter(http_client=AsyncMock())
    findings = await adapter.check(target, since=None)
    assert findings == []


@pytest.mark.asyncio
async def test_returns_empty_on_http_error():
    mock_client = AsyncMock()
    mock_client.get.side_effect = Exception("connection refused")

    adapter = UrlChangeAdapter(http_client=mock_client)
    findings = await adapter.check(_make_target(), since=None)
    assert findings == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_url_change_adapter.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement UrlChangeAdapter**

Create `apps/api/src/prescient/monitoring/infrastructure/tasks/adapters/url_change.py`:

```python
"""URL change detection adapter — detects content changes at a watched URL."""

from __future__ import annotations

import hashlib
import logging
from datetime import datetime

from prescient.monitoring.infrastructure.tasks.adapters.base import RawFinding
from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow

logger = logging.getLogger(__name__)


class UrlChangeAdapter:
    """Fetches a URL, hashes the content, and compares to the previous hash.

    On first check (no previous hash), records a baseline without generating
    a finding. On subsequent checks, generates a finding if content changed.
    """

    def __init__(self, http_client: object) -> None:
        self._http = http_client
        self.last_computed_hash: str | None = None

    async def check(
        self, target: MonitorTargetRow, since: datetime | None
    ) -> list[RawFinding]:
        url = (target.config or {}).get("url")
        if not url:
            logger.debug("Target %s has no URL, skipping change check", target.id)
            return []

        try:
            response = await self._http.get(url)
            response.raise_for_status()
            content = response.text
        except Exception:
            logger.exception("Failed to fetch URL %s for target %s", url, target.id)
            return []

        new_hash = hashlib.sha256(content.encode()).hexdigest()
        self.last_computed_hash = new_hash

        previous_hash = getattr(target, "last_content_hash", None)
        if previous_hash is None:
            # First check — record baseline, no finding
            return []

        if new_hash == previous_hash:
            return []

        return [
            RawFinding(
                title=f"Content changed at {url}",
                finding_type="custom",
                severity="notable",
                summary=f"The content at {url} has changed since last check.",
                source_url=url,
                raw_payload={"previous_hash": previous_hash, "new_hash": new_hash},
            )
        ]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_url_change_adapter.py -v`

Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add URL change detection monitor adapter"
```

---

### Task 7: Monitor Sweep Task

**Files:**
- Create: `apps/api/src/prescient/monitoring/infrastructure/tasks/sweep.py`
- Test: `apps/api/tests/unit/monitoring/test_sweep.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/monitoring/test_sweep.py`:

```python
"""Tests for the monitor sweep and check_target logic."""

from __future__ import annotations

from datetime import datetime, timezone
from unittest.mock import AsyncMock, MagicMock, patch
from uuid import uuid4

import pytest

from prescient.monitoring.infrastructure.tasks.adapters.base import RawFinding
from prescient.monitoring.infrastructure.tasks.sweep import (
    _check_target_async,
    _resolve_adapters,
)


def _make_target(
    target_type: str = "COMPANY",
    config: dict | None = None,
    last_checked_at: datetime | None = None,
) -> MagicMock:
    target = MagicMock()
    target.id = uuid4()
    target.owner_tenant_id = uuid4()
    target.target_type = target_type
    target.target_id = uuid4()
    target.config = config or {"company_name": "Test", "cik": "0001234567", "sources": ["sec_filing"]}
    target.last_checked_at = last_checked_at
    target.last_content_hash = None
    return target


def test_resolve_adapters_from_config():
    adapters = _resolve_adapters(
        config={"sources": ["sec_filing", "news_api"]},
        target_type="COMPANY",
        settings=MagicMock(newsapi_api_key="key"),
    )
    assert len(adapters) == 2


def test_resolve_adapters_defaults_by_target_type():
    adapters = _resolve_adapters(
        config={"company_name": "Test"},
        target_type="COMPANY",
        settings=MagicMock(newsapi_api_key="key"),
    )
    # COMPANY default: sec_filing + news_api
    assert len(adapters) == 2


def test_resolve_adapters_skips_unknown():
    adapters = _resolve_adapters(
        config={"sources": ["unknown_type"]},
        target_type="COMPANY",
        settings=MagicMock(newsapi_api_key=""),
    )
    assert len(adapters) == 0


@pytest.mark.asyncio
async def test_check_target_creates_findings_and_deduplicates():
    target = _make_target()

    raw_finding = RawFinding(
        title="10-Q filed 2026-04-10",
        finding_type="filing",
        severity="notable",
        source_url="https://sec.gov/filing/123",
    )

    mock_adapter = AsyncMock()
    mock_adapter.check.return_value = [raw_finding]
    mock_adapter.last_computed_hash = None

    mock_session = AsyncMock()
    # No existing findings with this hash
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = None
    mock_session.execute.return_value = mock_result

    new_findings = await _check_target_async(
        target=target,
        adapters=[mock_adapter],
        session=mock_session,
    )

    assert len(new_findings) == 1
    # session.add should have been called for MonitorRunRow and MonitorFindingRow
    assert mock_session.add.call_count >= 2


@pytest.mark.asyncio
async def test_check_target_skips_duplicate_finding():
    target = _make_target()

    raw_finding = RawFinding(
        title="Already seen",
        finding_type="filing",
        severity="notable",
    )

    mock_adapter = AsyncMock()
    mock_adapter.check.return_value = [raw_finding]
    mock_adapter.last_computed_hash = None

    mock_session = AsyncMock()
    # Existing finding with same hash
    mock_result = MagicMock()
    mock_result.scalar_one_or_none.return_value = MagicMock()  # existing row
    mock_session.execute.return_value = mock_result

    new_findings = await _check_target_async(
        target=target,
        adapters=[mock_adapter],
        session=mock_session,
    )

    assert len(new_findings) == 0
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_sweep.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement sweep tasks**

Create `apps/api/src/prescient/monitoring/infrastructure/tasks/sweep.py`:

```python
"""Monitor sweep — Celery tasks for checking watch targets."""

from __future__ import annotations

import asyncio
import logging
from datetime import datetime, timezone
from uuid import uuid4

from celery import shared_task
from sqlalchemy import select

from prescient.monitoring.infrastructure.tasks.adapters.base import (
    DEFAULT_SOURCES,
    RawFinding,
)
from prescient.monitoring.infrastructure.tables.monitor_finding import MonitorFindingRow
from prescient.monitoring.infrastructure.tables.monitor_run import MonitorRunRow
from prescient.monitoring.infrastructure.tables.monitor_target import MonitorTargetRow
from prescient.shared.events import DomainEvent
from prescient.shared.outbox import append_event

logger = logging.getLogger(__name__)


def _resolve_adapters(
    config: dict, target_type: str, settings: object
) -> list[object]:
    """Resolve source adapter instances from config or target type defaults."""
    from prescient.monitoring.infrastructure.tasks.adapters.news_api import NewsApiAdapter
    from prescient.monitoring.infrastructure.tasks.adapters.sec_filing import SecFilingAdapter
    from prescient.monitoring.infrastructure.tasks.adapters.url_change import UrlChangeAdapter

    source_names = config.get("sources") or DEFAULT_SOURCES.get(target_type, [])

    adapter_map: dict[str, object | None] = {}

    for name in source_names:
        if name == "sec_filing":
            from seed.edgar import EdgarClient

            adapter_map[name] = SecFilingAdapter(edgar_client=EdgarClient())
        elif name == "news_api":
            api_key = getattr(settings, "newsapi_api_key", "")
            if api_key:
                import httpx

                adapter_map[name] = NewsApiAdapter(
                    api_key=api_key, http_client=httpx.AsyncClient(timeout=30)
                )
        elif name == "url_change":
            import httpx

            adapter_map[name] = UrlChangeAdapter(http_client=httpx.AsyncClient(timeout=30))
        else:
            logger.warning("Unknown source adapter: %s", name)

    return [a for a in adapter_map.values() if a is not None]


async def _check_target_async(
    *,
    target: MonitorTargetRow,
    adapters: list[object],
    session: object,
) -> list[RawFinding]:
    """Run all adapters for a target, deduplicate, and persist findings.

    Returns the list of new (non-duplicate) findings.
    """
    run_id = uuid4()
    run_row = MonitorRunRow(
        id=run_id,
        source_id=target.id,
        status="running",
    )
    session.add(run_row)

    all_raw: list[RawFinding] = []
    last_hash: str | None = None

    for adapter in adapters:
        try:
            findings = await adapter.check(target, target.last_checked_at)
            all_raw.extend(findings)
            # Capture URL change adapter hash if present
            if hasattr(adapter, "last_computed_hash") and adapter.last_computed_hash:
                last_hash = adapter.last_computed_hash
        except Exception:
            logger.exception("Adapter %s failed for target %s", type(adapter).__name__, target.id)

    # Deduplicate against existing findings
    new_findings: list[RawFinding] = []
    for raw in all_raw:
        existing = await session.execute(
            select(MonitorFindingRow).where(
                MonitorFindingRow.target_id == target.id,
                MonitorFindingRow.content_hash == raw.content_hash,
            )
        )
        if existing.scalar_one_or_none() is not None:
            continue

        finding_id = uuid4()
        session.add(
            MonitorFindingRow(
                id=finding_id,
                run_id=run_id,
                target_id=target.id,
                owner_tenant_id=target.owner_tenant_id,
                finding_type=raw.finding_type,
                title=raw.title[:256],
                summary=raw.summary,
                severity=raw.severity,
                occurred_at=raw.occurred_at,
                source_url=raw.source_url,
                raw_payload=raw.raw_payload,
                content_hash=raw.content_hash,
            )
        )

        await append_event(
            session,
            DomainEvent(
                aggregate_id=str(finding_id),
                aggregate_type="monitoring.finding",
                event_type="monitoring.finding.discovered",
                payload={
                    "finding_id": str(finding_id),
                    "target_id": str(target.id),
                    "owner_tenant_id": str(target.owner_tenant_id),
                    "finding_type": raw.finding_type,
                    "title": raw.title,
                    "severity": raw.severity,
                    "source_url": raw.source_url,
                },
            ),
        )
        new_findings.append(raw)

    # Update run and target
    run_row.status = "completed"
    run_row.items_found = len(new_findings)
    run_row.finished_at = datetime.now(timezone.utc)

    target.last_checked_at = datetime.now(timezone.utc)
    if last_hash:
        target.last_content_hash = last_hash

    return new_findings


@shared_task(name="prescient.monitoring.infrastructure.tasks.sweep.monitor_sweep")
def monitor_sweep() -> dict:
    """Sweep all active watch targets and dispatch individual check tasks."""
    from prescient.db import SessionLocal

    async def _sweep() -> int:
        async with SessionLocal() as session:
            result = await session.execute(
                select(MonitorTargetRow).where(MonitorTargetRow.active.is_(True))
            )
            targets = list(result.scalars().all())

        count = 0
        for target in targets:
            check_target.delay(str(target.id))
            count += 1
        return count

    dispatched = asyncio.get_event_loop().run_until_complete(_sweep())
    return {"dispatched": dispatched}


@shared_task(
    name="prescient.monitoring.infrastructure.tasks.sweep.check_target",
    soft_time_limit=300,
    time_limit=360,
)
def check_target(target_id: str) -> dict:
    """Check a single watch target against its configured source adapters."""
    from prescient.config import get_settings
    from prescient.db import SessionLocal

    async def _run() -> dict:
        settings = get_settings()
        async with SessionLocal() as session:
            result = await session.execute(
                select(MonitorTargetRow).where(
                    MonitorTargetRow.id == target_id,
                    MonitorTargetRow.active.is_(True),
                )
            )
            target = result.scalar_one_or_none()
            if target is None:
                return {"error": f"Target {target_id} not found or inactive"}

            adapters = _resolve_adapters(
                config=target.config or {},
                target_type=target.target_type,
                settings=settings,
            )

            new_findings = await _check_target_async(
                target=target,
                adapters=adapters,
                session=session,
            )
            await session.commit()
            return {"target_id": target_id, "new_findings": len(new_findings)}

    return asyncio.get_event_loop().run_until_complete(_run())
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/monitoring/test_sweep.py -v`

Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add monitor sweep and check_target Celery tasks"
```

---

### Task 8: Monitor Runs API Endpoint

**Files:**
- Modify: `apps/api/src/prescient/monitoring/api/routes.py`

- [ ] **Step 1: Add GET /monitoring/runs endpoint**

In `apps/api/src/prescient/monitoring/api/routes.py`, add the following imports at the top (alongside existing ones):

```python
from prescient.monitoring.infrastructure.tables.monitor_run import MonitorRunRow
```

Add a response model after `WatchTargetResponse`:

```python
class MonitorRunResponse(BaseModel):
    id: str
    source_id: str
    started_at: datetime
    finished_at: datetime | None
    status: str
    items_found: int
    error: str | None
```

Add `datetime` to the imports from the standard library.

Add the route after the `remove_watch_target` route:

```python
@router.get("/runs", response_model=list[MonitorRunResponse])
async def list_monitor_runs(
    target_id: str,
    limit: int = 20,
    session: SessionDep = None,  # type: ignore[assignment]
) -> list[MonitorRunResponse]:
    """List execution history for a watch target."""
    stmt = (
        select(MonitorRunRow)
        .where(MonitorRunRow.source_id == target_id)
        .order_by(MonitorRunRow.started_at.desc())
        .limit(limit)
    )
    result = await session.execute(stmt)
    rows = list(result.scalars().all())
    return [
        MonitorRunResponse(
            id=str(r.id),
            source_id=str(r.source_id),
            started_at=r.started_at,
            finished_at=r.finished_at,
            status=r.status,
            items_found=r.items_found,
            error=r.error,
        )
        for r in rows
    ]
```

- [ ] **Step 2: Commit**

```bash
git add -A
git commit -m "feat: add GET /monitoring/runs endpoint for execution history"
```

---

### Task 9: Outbox Drain Task

**Files:**
- Create: `apps/api/src/prescient/shared/infrastructure/tasks/outbox_drain.py`
- Test: `apps/api/tests/unit/shared/test_outbox_drain.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/shared/test_outbox_drain.py`:

```python
"""Tests for the outbox drain task."""

from __future__ import annotations

from datetime import datetime, timezone
from unittest.mock import MagicMock, patch
from uuid import uuid4

import pytest

from prescient.shared.infrastructure.tasks.outbox_drain import (
    EVENT_SUBSCRIBERS,
    _dispatch_event,
)


def test_dispatch_event_calls_matching_subscribers():
    handler_a = MagicMock()
    handler_b = MagicMock()
    handler_a.delay = MagicMock()
    handler_b.delay = MagicMock()

    registry = {
        "kpi.anomaly_detected": [handler_a],
        "monitoring.target.created": [handler_b],
    }

    event_row = MagicMock()
    event_row.event_id = uuid4()
    event_row.event_type = "kpi.anomaly_detected"
    event_row.payload = {"anomaly_id": "abc"}
    event_row.correlation_id = None
    event_row.causation_id = None

    _dispatch_event(event_row, registry)

    handler_a.delay.assert_called_once()
    handler_b.delay.assert_not_called()


def test_dispatch_event_no_subscribers_is_noop():
    event_row = MagicMock()
    event_row.event_type = "unknown.event"
    event_row.payload = {}

    # Should not raise
    _dispatch_event(event_row, {})


def test_subscriber_registry_has_expected_keys():
    assert "kpi.anomaly_detected" in EVENT_SUBSCRIBERS
    assert "monitoring.target.created" in EVENT_SUBSCRIBERS
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/shared/test_outbox_drain.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement outbox drain**

Create `apps/api/src/prescient/shared/infrastructure/tasks/outbox_drain.py`:

```python
"""Outbox drain — polls unpublished domain events and dispatches to subscribers."""

from __future__ import annotations

import asyncio
import logging
from datetime import datetime, timezone

from celery import shared_task
from sqlalchemy import select, update

from prescient.shared.outbox import DomainEventRow

logger = logging.getLogger(__name__)

_BATCH_SIZE = 100


def _dispatch_event(
    event_row: DomainEventRow,
    registry: dict[str, list[object]],
) -> None:
    """Dispatch a domain event to all registered subscriber tasks."""
    handlers = registry.get(event_row.event_type, [])
    if not handlers:
        logger.debug("No subscribers for event type %s", event_row.event_type)
        return

    for handler in handlers:
        try:
            handler.delay(event_row.payload)
        except Exception:
            logger.exception(
                "Failed to dispatch event %s to handler %s",
                event_row.event_id,
                getattr(handler, "name", handler),
            )


def _get_subscriber_registry() -> dict[str, list[object]]:
    """Build subscriber registry. Lazy import to avoid circular imports."""
    from prescient.events.subscribers.anomaly_to_triage import create_triage_from_anomaly
    from prescient.events.subscribers.target_immediate_check import run_immediate_target_check

    return {
        "kpi.anomaly_detected": [create_triage_from_anomaly],
        "monitoring.target.created": [run_immediate_target_check],
    }


# Eagerly-importable registry for tests
EVENT_SUBSCRIBERS = _get_subscriber_registry()


@shared_task(
    name="prescient.shared.infrastructure.tasks.outbox_drain.drain_outbox",
    soft_time_limit=30,
    time_limit=45,
)
def drain_outbox() -> dict:
    """Poll unpublished domain events and dispatch to subscribers."""
    from prescient.db import SessionLocal

    registry = _get_subscriber_registry()

    async def _drain() -> int:
        async with SessionLocal() as session:
            result = await session.execute(
                select(DomainEventRow)
                .where(DomainEventRow.published_at.is_(None))
                .order_by(DomainEventRow.occurred_at.asc())
                .limit(_BATCH_SIZE)
            )
            events = list(result.scalars().all())

            if not events:
                return 0

            for event_row in events:
                _dispatch_event(event_row, registry)

            # Mark all as published
            event_ids = [e.event_id for e in events]
            await session.execute(
                update(DomainEventRow)
                .where(DomainEventRow.event_id.in_(event_ids))
                .values(published_at=datetime.now(timezone.utc))
            )
            await session.commit()
            return len(events)

    count = asyncio.get_event_loop().run_until_complete(_drain())
    return {"drained": count}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/shared/test_outbox_drain.py -v`

Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add outbox drain Celery task with subscriber registry"
```

---

### Task 10: Event Subscriber — Anomaly to Triage

**Files:**
- Create: `apps/api/src/prescient/triage/queries.py`
- Create: `apps/api/src/prescient/events/subscribers/anomaly_to_triage.py`
- Test: `apps/api/tests/unit/events/test_anomaly_to_triage.py`

- [ ] **Step 1: Create triage published query interface**

Create `apps/api/src/prescient/triage/queries.py`:

```python
"""Published query interface for the triage bounded context.

Other contexts use these functions instead of importing triage tables directly.
"""

from __future__ import annotations

from uuid import UUID, uuid4

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.triage.domain.enums import (
    TriageItemStatus,
    TriageItemType,
    TriagePriority,
)
from prescient.triage.infrastructure.tables.triage_item import TriageItemTable
from prescient.shared.types import utcnow


async def create_triage_item(
    session: AsyncSession,
    *,
    organization_id: str,
    item_type: TriageItemType,
    source_id: str,
    title: str,
    summary: str,
    priority: TriagePriority,
    priority_signals: list[str] | None = None,
) -> UUID:
    """Create a triage item and return its ID. Caller must commit."""
    item_id = uuid4()
    session.add(
        TriageItemTable(
            id=item_id,
            organization_id=organization_id,
            item_type=item_type.value,
            source_id=source_id,
            title=title,
            summary=summary[:500],
            priority=priority.value,
            priority_signals=priority_signals or [],
            status=TriageItemStatus.PENDING.value,
            assigned_to_user_id=None,
            created_at=utcnow(),
            resolved_at=None,
        )
    )
    return item_id
```

- [ ] **Step 2: Write tests for anomaly subscriber**

Create `apps/api/tests/unit/events/test_anomaly_to_triage.py`:

```python
"""Tests for anomaly-to-triage event subscriber."""

from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from prescient.events.subscribers.anomaly_to_triage import _handle_anomaly_event


@pytest.mark.asyncio
async def test_creates_triage_item_from_anomaly_payload():
    payload = {
        "anomaly_id": "anom-001",
        "organization_id": "org-peloton-001",
        "kpi_id": "revenue",
        "anomaly_type": "sudden_change",
        "severity": "important",
        "description": "Revenue dropped 28% vs trailing average",
        "enrichment_question": "Revenue declined significantly. What drove this change?",
    }

    mock_session = AsyncMock()

    with patch(
        "prescient.events.subscribers.anomaly_to_triage.SessionLocal"
    ) as mock_session_local:
        mock_ctx = AsyncMock()
        mock_ctx.__aenter__.return_value = mock_session
        mock_ctx.__aexit__.return_value = None
        mock_session_local.return_value = mock_ctx

        with patch(
            "prescient.events.subscribers.anomaly_to_triage.create_triage_item",
            new_callable=AsyncMock,
        ) as mock_create:
            from uuid import uuid4

            mock_create.return_value = uuid4()
            await _handle_anomaly_event(payload)
            mock_create.assert_called_once()
            call_kwargs = mock_create.call_args.kwargs
            assert call_kwargs["organization_id"] == "org-peloton-001"
            assert call_kwargs["source_id"] == "anom-001"
            assert call_kwargs["item_type"].value == "kpi_anomaly"


@pytest.mark.asyncio
async def test_maps_severity_to_priority():
    from prescient.events.subscribers.anomaly_to_triage import _severity_to_priority
    from prescient.triage.domain.enums import TriagePriority

    assert _severity_to_priority("important") == TriagePriority.HIGH
    assert _severity_to_priority("notable") == TriagePriority.MEDIUM
    assert _severity_to_priority("info") == TriagePriority.LOW
    assert _severity_to_priority("critical") == TriagePriority.CRITICAL
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/events/test_anomaly_to_triage.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement anomaly-to-triage subscriber**

Create `apps/api/src/prescient/events/subscribers/anomaly_to_triage.py`:

```python
"""Event subscriber: kpi.anomaly_detected -> create triage item."""

from __future__ import annotations

import asyncio
import logging

from celery import shared_task

from prescient.triage.domain.enums import TriageItemType, TriagePriority

logger = logging.getLogger(__name__)

_SEVERITY_TO_PRIORITY: dict[str, TriagePriority] = {
    "critical": TriagePriority.CRITICAL,
    "important": TriagePriority.HIGH,
    "notable": TriagePriority.MEDIUM,
    "info": TriagePriority.LOW,
}


def _severity_to_priority(severity: str) -> TriagePriority:
    return _SEVERITY_TO_PRIORITY.get(severity, TriagePriority.MEDIUM)


async def _handle_anomaly_event(payload: dict) -> None:
    from prescient.db import SessionLocal
    from prescient.triage.queries import create_triage_item

    anomaly_id = payload.get("anomaly_id", "")
    organization_id = payload.get("organization_id", "")
    kpi_id = payload.get("kpi_id", "")
    severity = payload.get("severity", "info")
    description = payload.get("description", "")
    enrichment_question = payload.get("enrichment_question", "")

    async with SessionLocal() as session:
        await create_triage_item(
            session,
            organization_id=organization_id,
            item_type=TriageItemType.KPI_ANOMALY,
            source_id=anomaly_id,
            title=f"KPI anomaly: {kpi_id}",
            summary=enrichment_question or description,
            priority=_severity_to_priority(severity),
            priority_signals=[f"severity:{severity}", f"kpi:{kpi_id}"],
        )
        await session.commit()
        logger.info("Created triage item for KPI anomaly %s", anomaly_id)


@shared_task(
    name="prescient.events.subscribers.anomaly_to_triage.create_triage_from_anomaly",
    max_retries=3,
    default_retry_delay=10,
)
def create_triage_from_anomaly(payload: dict) -> dict:
    """Celery task: create a triage item from a KPI anomaly event."""
    asyncio.get_event_loop().run_until_complete(_handle_anomaly_event(payload))
    return {"anomaly_id": payload.get("anomaly_id"), "status": "created"}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/events/test_anomaly_to_triage.py -v`

Expected: 2 passed

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: add anomaly-to-triage event subscriber and triage query interface"
```

---

### Task 11: Event Subscriber — Immediate Target Check

**Files:**
- Create: `apps/api/src/prescient/events/subscribers/target_immediate_check.py`
- Test: `apps/api/tests/unit/events/test_target_immediate_check.py`

- [ ] **Step 1: Write test**

Create `apps/api/tests/unit/events/test_target_immediate_check.py`:

```python
"""Tests for immediate target check subscriber."""

from __future__ import annotations

from unittest.mock import patch, MagicMock
from uuid import uuid4

from prescient.events.subscribers.target_immediate_check import run_immediate_target_check


def test_dispatches_check_target_task():
    target_id = str(uuid4())
    payload = {"target_id": target_id}

    with patch(
        "prescient.events.subscribers.target_immediate_check.check_target"
    ) as mock_check:
        mock_check.delay = MagicMock()
        run_immediate_target_check(payload)
        mock_check.delay.assert_called_once_with(target_id)


def test_handles_missing_target_id():
    # Should not raise
    with patch(
        "prescient.events.subscribers.target_immediate_check.check_target"
    ) as mock_check:
        mock_check.delay = MagicMock()
        result = run_immediate_target_check({})
        mock_check.delay.assert_not_called()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/events/test_target_immediate_check.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement immediate check subscriber**

Create `apps/api/src/prescient/events/subscribers/target_immediate_check.py`:

```python
"""Event subscriber: monitoring.target.created -> immediate check."""

from __future__ import annotations

import logging

from celery import shared_task

logger = logging.getLogger(__name__)


@shared_task(
    name="prescient.events.subscribers.target_immediate_check.run_immediate_target_check",
    max_retries=2,
    default_retry_delay=5,
)
def run_immediate_target_check(payload: dict) -> dict:
    """Dispatch an immediate check for a newly created watch target."""
    from prescient.monitoring.infrastructure.tasks.sweep import check_target

    target_id = payload.get("target_id")
    if not target_id:
        logger.warning("No target_id in event payload, skipping immediate check")
        return {"status": "skipped", "reason": "no target_id"}

    check_target.delay(target_id)
    logger.info("Dispatched immediate check for new target %s", target_id)
    return {"target_id": target_id, "status": "dispatched"}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/events/test_target_immediate_check.py -v`

Expected: 2 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add immediate target check event subscriber"
```

---

### Task 12: Wire Outbox Events into Existing Use Cases

**Files:**
- Modify: `apps/api/src/prescient/kpis/application/use_cases/report_kpi_value.py`
- Modify: `apps/api/src/prescient/monitoring/application/use_cases/add_watch_target.py`
- Modify: `apps/api/src/prescient/kpis/api/routes.py`
- Modify: `apps/api/src/prescient/monitoring/api/routes.py`

- [ ] **Step 1: Add outbox event emission to ReportKpiValue**

In `apps/api/src/prescient/kpis/application/use_cases/report_kpi_value.py`, add to imports:

```python
from prescient.shared.events import DomainEvent
```

Add an `outbox_port` to the `__init__` and the `execute` method. Change the `__init__` signature:

```python
class OutboxPort(Protocol):
    """Port for appending domain events."""

    async def append(self, event: DomainEvent) -> None: ...
```

Add `OutboxPort` after the existing port definitions. Then modify the `__init__`:

```python
def __init__(
    self,
    session: KpiValueSessionPort,
    query: KpiValueQueryPort,
    kpi_value_row_factory: Any,
    kpi_anomaly_row_factory: Any,
    detector: AnomalyDetector | None = None,
    outbox: OutboxPort | None = None,
) -> None:
    self._session = session
    self._query = query
    self._kpi_value_row_factory = kpi_value_row_factory
    self._kpi_anomaly_row_factory = kpi_anomaly_row_factory
    self._detector = detector or AnomalyDetector()
    self._outbox = outbox
```

At the end of the `execute` method, after the anomaly detection block and before the return, add:

```python
if self._outbox is not None:
    await self._outbox.append(
        DomainEvent(
            aggregate_id=command.value_id,
            aggregate_type="kpi.value",
            event_type="kpi.value_reported",
            payload={
                "value_id": command.value_id,
                "organization_id": command.organization_id,
                "kpi_id": command.kpi_id,
                "period_label": command.period_label,
                "value": str(command.value),
            },
        )
    )
    if detected is not None:
        await self._outbox.append(
            DomainEvent(
                aggregate_id=anomaly_id,
                aggregate_type="kpi.anomaly",
                event_type="kpi.anomaly_detected",
                payload={
                    "anomaly_id": anomaly_id,
                    "organization_id": command.organization_id,
                    "kpi_id": command.kpi_id,
                    "anomaly_type": detected.anomaly_type.value,
                    "severity": detected.severity,
                    "description": detected.description,
                    "enrichment_question": detected.enrichment_question,
                },
            )
        )
```

- [ ] **Step 2: Wire outbox adapter in KPI routes**

In `apps/api/src/prescient/kpis/api/routes.py`, add after `_SessionAdapter`:

```python
class _OutboxAdapter:
    """Adapts AsyncSession to OutboxPort for domain event emission."""

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def append(self, event: object) -> None:
        from prescient.shared.outbox import append_event
        await append_event(self._session, event)
```

In the `report_kpi_value` route, update the `ReportKpiValue` construction to pass the outbox:

```python
use_case = ReportKpiValue(
    session=_SessionAdapter(session),
    query=query_adapter,
    kpi_value_row_factory=KpiValueRow,
    kpi_anomaly_row_factory=KpiAnomalyRow,
    outbox=_OutboxAdapter(session),
)
```

- [ ] **Step 3: Add outbox event emission to AddWatchTarget**

In `apps/api/src/prescient/monitoring/application/use_cases/add_watch_target.py`, add to imports:

```python
from prescient.shared.events import DomainEvent
```

Add an `OutboxPort` protocol:

```python
class OutboxPort(Protocol):
    async def append(self, event: DomainEvent) -> None: ...
```

Update `__init__`:

```python
def __init__(self, session: WatchTargetSessionPort, row_factory: Any, outbox: OutboxPort | None = None) -> None:
    self._session = session
    self._row_factory = row_factory
    self._outbox = outbox
```

At the end of `execute`, before the return:

```python
if self._outbox is not None:
    await self._outbox.append(
        DomainEvent(
            aggregate_id=command.id,
            aggregate_type="monitoring.target",
            event_type="monitoring.target.created",
            payload={
                "target_id": command.id,
                "owner_tenant_id": command.owner_tenant_id,
                "target_type": command.target_type,
            },
        )
    )
```

- [ ] **Step 4: Wire outbox adapter in monitoring routes**

In `apps/api/src/prescient/monitoring/api/routes.py`, add:

```python
class _OutboxAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def append(self, event: object) -> None:
        from prescient.shared.outbox import append_event
        await append_event(self._session, event)
```

Update the `add_watch_target` route to pass the outbox:

```python
use_case = AddWatchTarget(
    session=_SessionAdapter(session),
    row_factory=MonitorTargetRow,
    outbox=_OutboxAdapter(session),
)
```

- [ ] **Step 5: Run existing tests to ensure nothing broke**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/ -q --tb=short --ignore=tests/artifacts/test_artifact_domain.py 2>&1 | tail -10`

Expected: all previously passing tests still pass (outbox is optional so existing callers are unaffected)

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: wire outbox events into ReportKpiValue and AddWatchTarget use cases"
```

---

### Task 13: KPI Intake Template Domain Entity

**Files:**
- Create: `apps/api/src/prescient/kpis/domain/entities/intake_template.py`
- Create: `apps/api/src/prescient/kpis/infrastructure/tables/intake_template.py`
- Create: `apps/api/src/prescient/kpis/infrastructure/repositories/intake_template_repository.py`
- Test: `apps/api/tests/unit/kpis/test_intake_template.py`

- [ ] **Step 1: Write tests for domain entities and fingerprinting**

Create `apps/api/tests/unit/kpis/test_intake_template.py`:

```python
"""Tests for KPI intake template domain model."""

from prescient.kpis.domain.entities.intake_template import (
    ColumnMapping,
    PeriodConfig,
    compute_header_fingerprint,
)


def test_fingerprint_deterministic():
    headers = ["Net Rev ($M)", "Quarter", "EBITDA"]
    assert compute_header_fingerprint(headers) == compute_header_fingerprint(headers)


def test_fingerprint_order_independent():
    h1 = ["Net Rev ($M)", "Quarter", "EBITDA"]
    h2 = ["EBITDA", "Quarter", "Net Rev ($M)"]
    assert compute_header_fingerprint(h1) == compute_header_fingerprint(h2)


def test_fingerprint_case_insensitive():
    h1 = ["Revenue", "Quarter"]
    h2 = ["revenue", "QUARTER"]
    assert compute_header_fingerprint(h1) == compute_header_fingerprint(h2)


def test_fingerprint_trims_whitespace():
    h1 = ["  Revenue ", "Quarter  "]
    h2 = ["Revenue", "Quarter"]
    assert compute_header_fingerprint(h1) == compute_header_fingerprint(h2)


def test_column_mapping_creation():
    m = ColumnMapping(
        source_column="Net Rev ($M)",
        target_kpi_id="revenue",
        unit="USD_MILLIONS",
        transform="multiply_1000",
        skip=False,
    )
    assert m.target_kpi_id == "revenue"
    assert m.transform == "multiply_1000"


def test_period_config_creation():
    p = PeriodConfig(
        period_column="Quarter",
        period_format="quarter_label",
        fy_end_month=6,
    )
    assert p.fy_end_month == 6
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/kpis/test_intake_template.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement domain entities**

Create `apps/api/src/prescient/kpis/domain/entities/intake_template.py`:

```python
"""KPI intake template domain entities."""

from __future__ import annotations

import hashlib
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict


class ColumnMapping(BaseModel):
    model_config = ConfigDict(frozen=True)

    source_column: str
    target_kpi_id: str
    unit: str
    transform: str | None = None
    skip: bool = False


class PeriodConfig(BaseModel):
    model_config = ConfigDict(frozen=True)

    period_column: str
    period_format: str  # "quarter_label", "date_range", "month"
    fy_end_month: int = 12


class IntakeTemplate(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: UUID
    scope: str  # "system" or "organization"
    organization_id: str | None = None
    source_system: str | None = None
    report_type: str | None = None
    name: str
    header_fingerprint: str
    column_mappings: list[ColumnMapping]
    period_config: PeriodConfig
    kpi_mapping_overrides: dict | None = None
    created_by: str
    last_used_at: datetime | None = None
    created_at: datetime | None = None


def compute_header_fingerprint(headers: list[str]) -> str:
    """SHA256 of sorted, lowercased, trimmed headers joined by pipe."""
    normalized = sorted(h.strip().lower() for h in headers)
    joined = "|".join(normalized)
    return hashlib.sha256(joined.encode()).hexdigest()
```

- [ ] **Step 4: Create ORM table**

Create `apps/api/src/prescient/kpis/infrastructure/tables/intake_template.py`:

```python
"""`kpis.intake_templates` table."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from sqlalchemy import DateTime, String, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class IntakeTemplateRow(Base):
    __tablename__ = "intake_templates"
    __table_args__ = {"schema": "kpis"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    scope: Mapped[str] = mapped_column(String(16), nullable=False, server_default="organization")
    organization_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    source_system: Mapped[str | None] = mapped_column(String(64), nullable=True)
    report_type: Mapped[str | None] = mapped_column(String(64), nullable=True)
    name: Mapped[str] = mapped_column(String(256), nullable=False)
    header_fingerprint: Mapped[str] = mapped_column(String(64), nullable=False, index=True)
    column_mappings: Mapped[list] = mapped_column(JSONB, nullable=False, server_default="[]")
    period_config: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="{}")
    kpi_mapping_overrides: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    created_by: Mapped[str] = mapped_column(String(64), nullable=False)
    last_used_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    updated_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

- [ ] **Step 5: Create repository**

Create `apps/api/src/prescient/kpis/infrastructure/repositories/__init__.py` (empty) and `apps/api/src/prescient/kpis/infrastructure/repositories/intake_template_repository.py`:

```python
"""Repository for KPI intake templates."""

from __future__ import annotations

from datetime import datetime, timezone

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.kpis.infrastructure.tables.intake_template import IntakeTemplateRow


class IntakeTemplateRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def find_by_fingerprint(
        self, fingerprint: str, organization_id: str | None = None
    ) -> IntakeTemplateRow | None:
        """Find template by fingerprint. Checks org-scoped first, then system."""
        if organization_id:
            result = await self._session.execute(
                select(IntakeTemplateRow).where(
                    IntakeTemplateRow.header_fingerprint == fingerprint,
                    IntakeTemplateRow.scope == "organization",
                    IntakeTemplateRow.organization_id == organization_id,
                )
            )
            row = result.scalar_one_or_none()
            if row:
                return row

        # Check system templates
        result = await self._session.execute(
            select(IntakeTemplateRow).where(
                IntakeTemplateRow.header_fingerprint == fingerprint,
                IntakeTemplateRow.scope == "system",
            )
        )
        return result.scalar_one_or_none()

    async def list_for_org(self, organization_id: str) -> list[IntakeTemplateRow]:
        """List templates visible to an org (org-scoped + all system)."""
        result = await self._session.execute(
            select(IntakeTemplateRow).where(
                (IntakeTemplateRow.organization_id == organization_id)
                | (IntakeTemplateRow.scope == "system")
            ).order_by(IntakeTemplateRow.last_used_at.desc().nullsfirst())
        )
        return list(result.scalars().all())

    async def save(self, row: IntakeTemplateRow) -> None:
        self._session.add(row)

    async def delete(self, template_id: str) -> bool:
        result = await self._session.execute(
            select(IntakeTemplateRow).where(IntakeTemplateRow.id == template_id)
        )
        row = result.scalar_one_or_none()
        if row is None:
            return False
        await self._session.delete(row)
        return True

    async def touch_last_used(self, template_id: str) -> None:
        result = await self._session.execute(
            select(IntakeTemplateRow).where(IntakeTemplateRow.id == template_id)
        )
        row = result.scalar_one_or_none()
        if row:
            row.last_used_at = datetime.now(timezone.utc)
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/kpis/test_intake_template.py -v`

Expected: 6 passed

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: add KPI intake template domain model, ORM table, and repository"
```

---

### Task 14: KPI CSV Intake Use Case

**Files:**
- Create: `apps/api/src/prescient/kpis/application/use_cases/intake_csv.py`
- Test: `apps/api/tests/unit/kpis/test_intake_csv.py`

- [ ] **Step 1: Write tests**

Create `apps/api/tests/unit/kpis/test_intake_csv.py`:

```python
"""Tests for KPI CSV intake use case."""

from __future__ import annotations

import csv
import io
from unittest.mock import AsyncMock, MagicMock

import pytest

from prescient.kpis.application.use_cases.intake_csv import (
    IntakeCsvResult,
    parse_csv_headers_and_sample,
    apply_mapping_to_rows,
)
from prescient.kpis.domain.entities.intake_template import ColumnMapping, PeriodConfig


def _csv_bytes(rows: list[list[str]]) -> bytes:
    buf = io.StringIO()
    writer = csv.writer(buf)
    for row in rows:
        writer.writerow(row)
    return buf.getvalue().encode()


def test_parse_headers_and_sample():
    data = _csv_bytes([
        ["Quarter", "Revenue ($M)", "EBITDA ($M)"],
        ["Q1 FY2025", "100.5", "20.3"],
        ["Q2 FY2025", "105.2", "22.1"],
    ])
    headers, sample = parse_csv_headers_and_sample(data, max_rows=5)
    assert headers == ["Quarter", "Revenue ($M)", "EBITDA ($M)"]
    assert len(sample) == 2


def test_apply_mapping_to_rows():
    rows = [
        {"Quarter": "Q1 FY2025", "Revenue ($M)": "100.5", "EBITDA ($M)": "20.3"},
        {"Quarter": "Q2 FY2025", "Revenue ($M)": "105.2", "EBITDA ($M)": "22.1"},
    ]
    mappings = [
        ColumnMapping(source_column="Revenue ($M)", target_kpi_id="revenue", unit="USD_MILLIONS", skip=False),
        ColumnMapping(source_column="EBITDA ($M)", target_kpi_id="ebitda", unit="USD_MILLIONS", skip=False),
    ]
    period_config = PeriodConfig(period_column="Quarter", period_format="quarter_label", fy_end_month=6)

    kpi_rows = apply_mapping_to_rows(rows, mappings, period_config)

    assert len(kpi_rows) == 4  # 2 rows x 2 KPIs
    assert kpi_rows[0]["kpi_id"] == "revenue"
    assert kpi_rows[0]["period_label"] == "Q1 FY2025"


def test_apply_mapping_skips_skip_columns():
    rows = [{"Quarter": "Q1", "Revenue": "100", "Notes": "ignore"}]
    mappings = [
        ColumnMapping(source_column="Revenue", target_kpi_id="revenue", unit="USD", skip=False),
        ColumnMapping(source_column="Notes", target_kpi_id="", unit="", skip=True),
    ]
    period_config = PeriodConfig(period_column="Quarter", period_format="quarter_label", fy_end_month=12)

    kpi_rows = apply_mapping_to_rows(rows, mappings, period_config)
    assert len(kpi_rows) == 1


def test_apply_mapping_handles_transform_multiply():
    rows = [{"Q": "Q1", "Rev": "100"}]
    mappings = [
        ColumnMapping(source_column="Rev", target_kpi_id="revenue", unit="USD", transform="multiply_1000", skip=False),
    ]
    period_config = PeriodConfig(period_column="Q", period_format="quarter_label", fy_end_month=12)

    kpi_rows = apply_mapping_to_rows(rows, mappings, period_config)
    assert float(kpi_rows[0]["value"]) == 100000.0
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/kpis/test_intake_csv.py -v`

Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement CSV intake use case**

Create `apps/api/src/prescient/kpis/application/use_cases/intake_csv.py`:

```python
"""KPI CSV intake — parse, map, and ingest KPI values from CSV files."""

from __future__ import annotations

import csv
import io
import logging
from dataclasses import dataclass, field
from decimal import Decimal, InvalidOperation

from prescient.kpis.domain.entities.intake_template import ColumnMapping, PeriodConfig

logger = logging.getLogger(__name__)

_TRANSFORMS: dict[str, callable] = {
    "multiply_1000": lambda v: v * 1000,
    "divide_100": lambda v: v / 100,
    "negate": lambda v: -v,
}


def parse_csv_headers_and_sample(
    data: bytes, max_rows: int = 5
) -> tuple[list[str], list[dict[str, str]]]:
    """Parse CSV bytes and return headers + sample rows as dicts."""
    text = data.decode("utf-8-sig")  # handle BOM
    reader = csv.DictReader(io.StringIO(text))
    headers = reader.fieldnames or []
    sample: list[dict[str, str]] = []
    for i, row in enumerate(reader):
        if i >= max_rows:
            break
        sample.append(dict(row))
    return list(headers), sample


def apply_mapping_to_rows(
    rows: list[dict[str, str]],
    mappings: list[ColumnMapping],
    period_config: PeriodConfig,
) -> list[dict]:
    """Apply column mappings to CSV rows, producing KPI value dicts.

    Returns a list of dicts with keys: kpi_id, period_label, value, unit.
    """
    active_mappings = [m for m in mappings if not m.skip]
    result: list[dict] = []

    for row in rows:
        period_label = row.get(period_config.period_column, "")
        if not period_label:
            continue

        for mapping in active_mappings:
            raw_value = row.get(mapping.source_column, "").strip()
            if not raw_value:
                continue

            # Clean common formatting
            clean = raw_value.replace(",", "").replace("$", "").replace("%", "").strip("()")

            try:
                value = Decimal(clean)
            except (InvalidOperation, ValueError):
                logger.warning("Could not parse value '%s' for column '%s'", raw_value, mapping.source_column)
                continue

            # Apply transform
            if mapping.transform and mapping.transform in _TRANSFORMS:
                value = Decimal(str(_TRANSFORMS[mapping.transform](float(value))))

            result.append({
                "kpi_id": mapping.target_kpi_id,
                "period_label": period_label.strip(),
                "value": value,
                "unit": mapping.unit,
            })

    return result


LLM_MAPPING_PROMPT = """You are analyzing a CSV file to map its columns to known KPI definitions.

## KPI Definitions
{kpi_definitions}

## CSV Headers
{headers}

## Sample Rows (first 5)
{sample_rows}

## Instructions
Map each CSV column to a KPI definition. Return valid JSON with this structure:
{{
  "column_mappings": [
    {{
      "source_column": "exact column header",
      "target_kpi_id": "matching kpi id from definitions",
      "unit": "unit string (e.g. USD_MILLIONS, percent, count)",
      "transform": null or "multiply_1000" or "divide_100" or "negate",
      "skip": false,
      "uncertain": false
    }}
  ],
  "period_config": {{
    "period_column": "column that contains the time period",
    "period_format": "quarter_label" or "date_range" or "month",
    "fy_end_month": 12
  }},
  "source_system": null or "quickbooks" or "xero" or "stripe" or "sage" etc,
  "report_type": null or "profit_and_loss" or "balance_sheet" or "mrr_summary" etc
}}

If a column doesn't map to any KPI, set skip=true.
If you're unsure about a mapping, set uncertain=true with your best guess.
Only return the JSON object, no other text.
"""


@dataclass
class IntakeCsvResult:
    """Result of a CSV intake operation."""

    values_ingested: int = 0
    anomalies_detected: int = 0
    rows_skipped: int = 0
    skip_reasons: list[str] = field(default_factory=list)
    errors: list[str] = field(default_factory=list)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/unit/kpis/test_intake_csv.py -v`

Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add KPI CSV intake parsing, mapping, and LLM prompt"
```

---

### Task 15: KPI Intake API Endpoints

**Files:**
- Modify: `apps/api/src/prescient/kpis/api/routes.py`

- [ ] **Step 1: Add intake endpoints to KPI routes**

In `apps/api/src/prescient/kpis/api/routes.py`, add these imports:

```python
import json

from fastapi import HTTPException, UploadFile

from prescient.kpis.application.use_cases.intake_csv import (
    IntakeCsvResult,
    LLM_MAPPING_PROMPT,
    apply_mapping_to_rows,
    parse_csv_headers_and_sample,
)
from prescient.kpis.domain.entities.intake_template import (
    ColumnMapping,
    PeriodConfig,
    compute_header_fingerprint,
)
from prescient.kpis.infrastructure.repositories.intake_template_repository import (
    IntakeTemplateRepository,
)
from prescient.kpis.infrastructure.tables.intake_template import IntakeTemplateRow
```

Add these response models:

```python
class UploadAnalysisResponse(BaseModel):
    template_id: str | None
    template_name: str | None
    proposed_mappings: list[dict]
    proposed_period_config: dict
    source_system: str | None
    report_type: str | None
    headers: list[str]
    sample_rows: list[dict]
    fingerprint: str


class ConfirmIntakeBody(BaseModel):
    organization_id: str
    fingerprint: str
    column_mappings: list[dict]
    period_config: dict
    template_name: str
    source_system: str | None = None
    report_type: str | None = None
    save_as_system: bool = False
    csv_data: str  # base64 or raw CSV text


class IntakeResultResponse(BaseModel):
    values_ingested: int
    anomalies_detected: int
    rows_skipped: int
    skip_reasons: list[str]
    errors: list[str]


class IntakeTemplateResponse(BaseModel):
    id: str
    scope: str
    name: str
    source_system: str | None
    report_type: str | None
    header_fingerprint: str
    last_used_at: datetime | None
```

Add the routes at the bottom of the file:

```python
@router.post("/intake/upload", response_model=UploadAnalysisResponse)
async def upload_csv_for_mapping(
    file: UploadFile,
    organization_id: str = Form(...),
    session: SessionDep = None,  # type: ignore[assignment]
) -> UploadAnalysisResponse:
    """Upload a CSV file and get proposed column mappings."""
    content = await file.read()
    if len(content) > 10 * 1024 * 1024:  # 10MB
        raise HTTPException(status_code=413, detail="File too large (max 10 MB)")

    headers, sample = parse_csv_headers_and_sample(content)
    if not headers:
        raise HTTPException(status_code=422, detail="Could not parse CSV headers")

    fingerprint = compute_header_fingerprint(headers)

    # Check for existing template
    repo = IntakeTemplateRepository(session)
    existing = await repo.find_by_fingerprint(fingerprint, organization_id)

    if existing is not None:
        return UploadAnalysisResponse(
            template_id=str(existing.id),
            template_name=existing.name,
            proposed_mappings=existing.column_mappings,
            proposed_period_config=existing.period_config,
            source_system=existing.source_system,
            report_type=existing.report_type,
            headers=headers,
            sample_rows=sample,
            fingerprint=fingerprint,
        )

    # No template match — call LLM for mapping proposal
    from prescient.config import get_settings

    settings = get_settings()
    proposed_mappings: list[dict] = []
    proposed_period: dict = {}
    source_system: str | None = None
    report_type: str | None = None

    if settings.anthropic_api_key:
        import anthropic

        client = anthropic.AsyncAnthropic(api_key=settings.anthropic_api_key)

        # Format KPI definitions from database
        from prescient.kpis.queries import get_distinct_kpi_ids

        kpi_ids = await get_distinct_kpi_ids(session, organization_id)
        kpi_defs_text = "\n".join(f"- {kid}" for kid in kpi_ids) if kpi_ids else "No KPI definitions found."

        sample_text = "\n".join(
            ", ".join(f"{k}: {v}" for k, v in row.items()) for row in sample
        )

        prompt = LLM_MAPPING_PROMPT.format(
            kpi_definitions=kpi_defs_text,
            headers=", ".join(headers),
            sample_rows=sample_text,
        )

        response = await client.messages.create(
            model=settings.anthropic_model,
            max_tokens=2000,
            messages=[{"role": "user", "content": prompt}],
        )
        try:
            raw = response.content[0].text.strip()
            # Handle markdown code fences
            if raw.startswith("```"):
                raw = raw.split("\n", 1)[1].rsplit("```", 1)[0]
            llm_result = json.loads(raw)
            proposed_mappings = llm_result.get("column_mappings", [])
            proposed_period = llm_result.get("period_config", {})
            source_system = llm_result.get("source_system")
            report_type = llm_result.get("report_type")
        except (json.JSONDecodeError, IndexError, KeyError):
            pass  # Fall through with empty proposals

    return UploadAnalysisResponse(
        template_id=None,
        template_name=None,
        proposed_mappings=proposed_mappings,
        proposed_period_config=proposed_period,
        source_system=source_system,
        report_type=report_type,
        headers=headers,
        sample_rows=sample,
        fingerprint=fingerprint,
    )


@router.post("/intake/confirm", response_model=IntakeResultResponse)
async def confirm_intake(
    body: ConfirmIntakeBody,
    session: SessionDep = None,  # type: ignore[assignment]
) -> IntakeResultResponse:
    """Confirm column mappings and ingest CSV data."""
    import base64

    try:
        csv_bytes = base64.b64decode(body.csv_data)
    except Exception:
        csv_bytes = body.csv_data.encode("utf-8")

    headers, _ = parse_csv_headers_and_sample(csv_bytes, max_rows=0)
    text = csv_bytes.decode("utf-8-sig")
    import csv as csv_mod
    import io

    reader = csv_mod.DictReader(io.StringIO(text))
    all_rows = list(reader)

    mappings = [ColumnMapping(**m) for m in body.column_mappings if not m.get("skip", False)]
    period_config = PeriodConfig(**body.period_config)

    kpi_rows = apply_mapping_to_rows(all_rows, mappings, period_config)

    # Save template
    repo = IntakeTemplateRepository(session)
    template_row = IntakeTemplateRow(
        id=uuid.uuid4(),
        scope="system" if body.save_as_system else "organization",
        organization_id=None if body.save_as_system else body.organization_id,
        source_system=body.source_system,
        report_type=body.report_type,
        name=body.template_name,
        header_fingerprint=body.fingerprint,
        column_mappings=[m.dict() for m in mappings],
        period_config=period_config.dict(),
        created_by="system",
    )
    existing = await repo.find_by_fingerprint(body.fingerprint, body.organization_id)
    if existing:
        await repo.touch_last_used(str(existing.id))
    else:
        await repo.save(template_row)

    # Ingest values
    result = IntakeCsvResult()
    query_adapter = _KpiValueQueryAdapter(session)
    session_adapter = _SessionAdapter(session)

    for kpi_row in kpi_rows:
        try:
            use_case = ReportKpiValue(
                session=session_adapter,
                query=query_adapter,
                kpi_value_row_factory=KpiValueRow,
                kpi_anomaly_row_factory=KpiAnomalyRow,
            )
            report_result = await use_case.execute(
                ReportKpiValueCommand(
                    organization_id=body.organization_id,
                    kpi_id=kpi_row["kpi_id"],
                    period_label=kpi_row["period_label"],
                    period_start=date.today(),  # simplified — real impl parses from period_label
                    period_end=date.today(),
                    value=kpi_row["value"],
                    unit=kpi_row["unit"],
                    reported_by="csv_intake",
                )
            )
            result.values_ingested += 1
            if report_result.anomaly is not None:
                result.anomalies_detected += 1
        except Exception as exc:
            result.errors.append(f"{kpi_row['kpi_id']} / {kpi_row['period_label']}: {exc}")
            result.rows_skipped += 1

    await session.commit()

    return IntakeResultResponse(
        values_ingested=result.values_ingested,
        anomalies_detected=result.anomalies_detected,
        rows_skipped=result.rows_skipped,
        skip_reasons=result.skip_reasons,
        errors=result.errors,
    )


@router.get("/intake/templates", response_model=list[IntakeTemplateResponse])
async def list_intake_templates(
    organization_id: str,
    session: SessionDep = None,  # type: ignore[assignment]
) -> list[IntakeTemplateResponse]:
    """List intake templates visible to an organization."""
    repo = IntakeTemplateRepository(session)
    rows = await repo.list_for_org(organization_id)
    return [
        IntakeTemplateResponse(
            id=str(r.id),
            scope=r.scope,
            name=r.name,
            source_system=r.source_system,
            report_type=r.report_type,
            header_fingerprint=r.header_fingerprint,
            last_used_at=r.last_used_at,
        )
        for r in rows
    ]


@router.delete("/intake/templates/{template_id}", status_code=204)
async def delete_intake_template(
    template_id: str,
    session: SessionDep = None,  # type: ignore[assignment]
) -> Response:
    """Delete an intake template."""
    from fastapi import Response

    repo = IntakeTemplateRepository(session)
    deleted = await repo.delete(template_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Template not found")
    await session.commit()
    return Response(status_code=204)
```

Also add the missing `Form` and `Response` imports if not already present, and `uuid` import.

- [ ] **Step 2: Run all tests to verify nothing broke**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/ -q --tb=short --ignore=tests/artifacts/test_artifact_domain.py 2>&1 | tail -10`

Expected: all previously passing tests still pass + new tests pass

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "feat: add KPI intake CSV upload and confirm API endpoints"
```

---

### Task 16: KPI Overdue Check Task

**Files:**
- Create: `apps/api/src/prescient/kpis/infrastructure/tasks/check_overdue.py`

- [ ] **Step 1: Implement the overdue check task**

Create `apps/api/src/prescient/kpis/infrastructure/tasks/check_overdue.py`:

```python
"""Check for overdue KPI reports — runs daily via Celery Beat."""

from __future__ import annotations

import asyncio
import logging

from celery import shared_task

logger = logging.getLogger(__name__)


@shared_task(
    name="prescient.kpis.infrastructure.tasks.check_overdue.check_overdue_kpis",
    soft_time_limit=60,
    time_limit=90,
)
def check_overdue_kpis() -> dict:
    """Check for KPI owners who haven't reported on their cadence.

    Stub implementation — logs a message. Full implementation will query
    kpi_owners table, compare last reported_at against cadence, and create
    triage items for overdue reports.
    """
    logger.info("Running overdue KPI check (stub)")
    return {"status": "stub", "message": "Full implementation pending KPI owner cadence tracking"}
```

- [ ] **Step 2: Commit**

```bash
git add -A
git commit -m "feat: add stub overdue KPI check Celery task"
```

---

### Task 17: Migrate KM Ingestion from BackgroundTasks to Celery

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingest_celery.py`
- Modify: `apps/api/src/prescient/knowledge_mgmt/api/routes.py`

- [ ] **Step 1: Create Celery wrapper for ingestion**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingest_celery.py`:

```python
"""Celery task wrapper for the KM ingestion pipeline."""

from __future__ import annotations

import asyncio
import logging

from celery import shared_task

logger = logging.getLogger(__name__)


@shared_task(
    name="prescient.knowledge_mgmt.infrastructure.tasks.ingest_celery.run_ingestion",
    soft_time_limit=300,
    time_limit=360,
    max_retries=2,
    default_retry_delay=30,
)
def run_ingestion(source_id: str, tenant_id: str) -> dict:
    """Run the KM ingestion pipeline as a Celery task."""
    from prescient.config import get_settings
    from prescient.db import SessionLocal
    from prescient.knowledge_mgmt.infrastructure.repositories.chunk_repository import ChunkRepository
    from prescient.knowledge_mgmt.infrastructure.repositories.entity_repository import EntityRepository
    from prescient.knowledge_mgmt.infrastructure.repositories.source_repository import SourceRepository
    from prescient.knowledge_mgmt.infrastructure.services.chunker import SemanticChunker
    from prescient.knowledge_mgmt.infrastructure.services.embedding import get_embed_fn
    from prescient.knowledge_mgmt.infrastructure.services.enrichment import EnrichmentService
    from prescient.knowledge_mgmt.infrastructure.services.file_storage import LocalFileStorage
    from prescient.knowledge_mgmt.infrastructure.services.search import KMSearchService
    from prescient.knowledge_mgmt.infrastructure.services.text_extractor import TextExtractor
    from prescient.knowledge_mgmt.infrastructure.tasks.ingestion import run_ingestion_pipeline

    async def _run() -> None:
        settings = get_settings()
        async with SessionLocal() as session:
            from sqlalchemy import func as sa_func, select as sa_select
            await session.execute(
                sa_select(sa_func.set_config("app.current_fund_id", tenant_id, True))
            )

            source_repo = SourceRepository(session)
            chunk_repo = ChunkRepository(session)
            entity_repo = EntityRepository(session)
            file_storage = LocalFileStorage(base_dir=settings.file_storage_path)

            if not settings.anthropic_api_key:
                logger.warning("No Anthropic API key — skipping enrichment for %s", source_id)
                await source_repo.update_status(source_id, "ready", meta={"skipped_enrichment": True})
                await session.commit()
                return

            import anthropic
            anthropic_client = anthropic.AsyncAnthropic(api_key=settings.anthropic_api_key)
            enrichment = EnrichmentService(
                anthropic_client=anthropic_client,
                model=settings.anthropic_model,
            )

            from opensearchpy import AsyncOpenSearch
            os_client = AsyncOpenSearch(
                hosts=[settings.opensearch_url],
                use_ssl=False,
                verify_certs=False,
            )
            search_service = KMSearchService(
                opensearch=os_client,
                embed_fn=get_embed_fn(),
            )

            await run_ingestion_pipeline(
                source_id=source_id,
                tenant_id=tenant_id,
                session=session,
                source_repo=source_repo,
                chunk_repo=chunk_repo,
                entity_repo=entity_repo,
                file_storage=file_storage,
                extractor=TextExtractor(),
                chunker=SemanticChunker(),
                enrichment=enrichment,
                search_service=search_service,
            )

    asyncio.get_event_loop().run_until_complete(_run())
    return {"source_id": source_id, "status": "completed"}
```

- [ ] **Step 2: Update KM routes to use Celery instead of BackgroundTasks**

In `apps/api/src/prescient/knowledge_mgmt/api/routes.py`, replace the `_trigger_ingestion` function and update the routes to use the Celery task.

Replace the `_trigger_ingestion` function with:

```python
def _trigger_ingestion_celery(source_id: str, tenant_id: str) -> None:
    """Trigger the ingestion pipeline via Celery."""
    from prescient.knowledge_mgmt.infrastructure.tasks.ingest_celery import run_ingestion

    run_ingestion.delay(source_id, tenant_id)
```

In `upload_source`, replace:
```python
background_tasks.add_task(_trigger_ingestion, source.id, str(ctx.user.fund_id))
```
with:
```python
_trigger_ingestion_celery(source.id, str(ctx.user.fund_id))
```

Remove `BackgroundTasks` from the `upload_source` parameter list and the import.

Do the same for `create_authored_source`.

Remove the old `_trigger_ingestion` function entirely.

- [ ] **Step 3: Run all tests**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/ -q --tb=short --ignore=tests/artifacts/test_artifact_domain.py 2>&1 | tail -10`

Expected: all tests pass

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: migrate KM ingestion from BackgroundTasks to Celery task"
```

---

### Task 18: Final Integration Test — Full Sweep

**Files:**
- All previously created files

- [ ] **Step 1: Run the full test suite**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -m pytest tests/ -q --tb=short --ignore=tests/artifacts/test_artifact_domain.py`

Expected: all new tests pass, pre-existing failures unchanged

- [ ] **Step 2: Verify Celery app loads with all tasks**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -c "from prescient.celery_app import celery_app; print('Beat schedule:', list(celery_app.conf.beat_schedule.keys()))"`

Expected: prints `['monitor-sweep', 'outbox-drain', 'kpi-check-overdue']`

- [ ] **Step 3: Verify imports are clean**

Run: `cd /home/rhallman/Projects/prescient_os/.worktrees/live-data-pipelines/apps/api && PYTHONPATH=src /home/rhallman/Projects/prescient_os/apps/api/.venv/bin/python -c "
from prescient.monitoring.infrastructure.tasks.sweep import monitor_sweep, check_target
from prescient.shared.infrastructure.tasks.outbox_drain import drain_outbox
from prescient.events.subscribers.anomaly_to_triage import create_triage_from_anomaly
from prescient.events.subscribers.target_immediate_check import run_immediate_target_check
from prescient.kpis.application.use_cases.intake_csv import parse_csv_headers_and_sample
print('All imports OK')
"`

Expected: `All imports OK`

- [ ] **Step 4: Final commit if any loose changes**

```bash
git status
# If any uncommitted changes:
git add -A
git commit -m "chore: final integration cleanup for live data pipelines"
```
