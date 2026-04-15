# Phase 5a: Visibility Enforcement + Triage Domain Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the AccessContext visibility enforcement layer and the triage bounded context domain model (entities, enums, tables, migration), so that all subsequent Phase 5 work (API routes, UI, seed data) has a solid foundation.

**Architecture:** A new FastAPI dependency (`get_access_context`) resolves the current user's role, scope, and visibility rules into an `AccessContext` dataclass. A new `prescient/triage/` bounded context follows the existing DDD pattern (domain entities, enums, ports, tables). An Alembic migration creates the `triage` schema with three tables.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Alembic, pytest

---

## File Structure

### New files:

```
apps/api/src/prescient/auth/access_context.py          — AccessContext dataclass + get_access_context dependency
apps/api/src/prescient/triage/__init__.py               — package init
apps/api/src/prescient/triage/domain/__init__.py        — package init
apps/api/src/prescient/triage/domain/entities/__init__.py — re-exports
apps/api/src/prescient/triage/domain/entities/triage_question.py — TriageQuestion entity
apps/api/src/prescient/triage/domain/entities/triage_item.py     — TriageItem entity
apps/api/src/prescient/triage/domain/entities/shared_item.py     — SharedItem entity
apps/api/src/prescient/triage/domain/enums.py           — TriageQuestionStatus, TriageItemType, TriageItemStatus, TriagePriority
apps/api/src/prescient/triage/domain/errors.py          — TriageStateError
apps/api/src/prescient/triage/infrastructure/__init__.py — package init
apps/api/src/prescient/triage/infrastructure/tables/__init__.py — re-exports
apps/api/src/prescient/triage/infrastructure/tables/triage_question.py — ORM table
apps/api/src/prescient/triage/infrastructure/tables/triage_item.py     — ORM table
apps/api/src/prescient/triage/infrastructure/tables/shared_item.py     — ORM table
apps/api/alembic/versions/{next}_phase5_triage_schema.py — migration
apps/api/tests/unit/auth/test_access_context.py         — unit tests for AccessContext
apps/api/tests/unit/triage/test_triage_entities.py      — unit tests for domain entities
```

### Modified files:

```
apps/api/src/prescient/shared/types.py                  — add TriageQuestionId, TriageItemId, SharedItemId type aliases
apps/api/src/prescient/shared/metadata.py               — import triage tables to register with ORM
```

---

### Task 1: Triage Domain Enums

**Files:**
- Create: `apps/api/src/prescient/triage/__init__.py`
- Create: `apps/api/src/prescient/triage/domain/__init__.py`
- Create: `apps/api/src/prescient/triage/domain/enums.py`
- Test: `apps/api/tests/unit/triage/test_triage_entities.py`

- [ ] **Step 1: Create package structure**

Create the empty `__init__.py` files for the triage bounded context:

```python
# apps/api/src/prescient/triage/__init__.py
```

```python
# apps/api/src/prescient/triage/domain/__init__.py
```

- [ ] **Step 2: Write failing tests for enums**

```python
# apps/api/tests/unit/triage/test_triage_entities.py
"""Unit tests for triage domain entities and enums."""

from __future__ import annotations

import pytest

from prescient.triage.domain.enums import (
    TriageItemStatus,
    TriageItemType,
    TriagePriority,
    TriageQuestionStatus,
)


class TestTriageQuestionStatus:
    def test_lifecycle_order(self) -> None:
        """Status values follow the expected lifecycle."""
        assert TriageQuestionStatus.SUBMITTED == "submitted"
        assert TriageQuestionStatus.CLASSIFYING == "classifying"
        assert TriageQuestionStatus.PARTIALLY_ANSWERED == "partially_answered"
        assert TriageQuestionStatus.ESCALATED == "escalated"
        assert TriageQuestionStatus.ANSWERED == "answered"
        assert TriageQuestionStatus.CLOSED == "closed"

    def test_all_statuses_are_strings(self) -> None:
        for s in TriageQuestionStatus:
            assert isinstance(s, str)


class TestTriageItemType:
    def test_all_types_present(self) -> None:
        assert TriageItemType.SPONSOR_QUESTION == "sponsor_question"
        assert TriageItemType.KPI_ANOMALY == "kpi_anomaly"
        assert TriageItemType.MONITORING_FINDING == "monitoring_finding"
        assert TriageItemType.ACTION_REVIEW == "action_review"
        assert TriageItemType.SYSTEM_ALERT == "system_alert"


class TestTriageItemStatus:
    def test_all_statuses_present(self) -> None:
        assert TriageItemStatus.PENDING == "pending"
        assert TriageItemStatus.IN_PROGRESS == "in_progress"
        assert TriageItemStatus.RESOLVED == "resolved"
        assert TriageItemStatus.DISMISSED == "dismissed"


class TestTriagePriority:
    def test_ordering(self) -> None:
        """Priority values exist."""
        assert TriagePriority.CRITICAL == "critical"
        assert TriagePriority.HIGH == "high"
        assert TriagePriority.MEDIUM == "medium"
        assert TriagePriority.LOW == "low"
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_triage_entities.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'prescient.triage.domain.enums'`

- [ ] **Step 4: Implement enums**

```python
# apps/api/src/prescient/triage/domain/enums.py
"""Triage context enums."""

from __future__ import annotations

from enum import StrEnum


class TriageQuestionStatus(StrEnum):
    """Lifecycle for a sponsor question.

    submitted → classifying → partially_answered → escalated → answered → closed

    A question that is fully auto-answered skips straight to `answered`.
    A question touching only BLOCKED categories skips to `escalated` without
    a partial answer.
    """

    SUBMITTED = "submitted"
    CLASSIFYING = "classifying"
    PARTIALLY_ANSWERED = "partially_answered"
    ESCALATED = "escalated"
    ANSWERED = "answered"
    CLOSED = "closed"


class TriageItemType(StrEnum):
    """What kind of item is in the triage queue."""

    SPONSOR_QUESTION = "sponsor_question"
    KPI_ANOMALY = "kpi_anomaly"
    MONITORING_FINDING = "monitoring_finding"
    ACTION_REVIEW = "action_review"
    SYSTEM_ALERT = "system_alert"


class TriageItemStatus(StrEnum):
    """Lifecycle for a triage queue item."""

    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    RESOLVED = "resolved"
    DISMISSED = "dismissed"


class TriagePriority(StrEnum):
    """Priority levels for triage items, derived from signals."""

    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_triage_entities.py -v`
Expected: PASS (all 4 tests)

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/triage/__init__.py apps/api/src/prescient/triage/domain/__init__.py apps/api/src/prescient/triage/domain/enums.py apps/api/tests/unit/triage/test_triage_entities.py
git commit -m "feat(triage): add triage domain enums — question status, item type, priority"
```

---

### Task 2: Triage Domain Entities

**Files:**
- Create: `apps/api/src/prescient/triage/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/triage/domain/entities/triage_question.py`
- Create: `apps/api/src/prescient/triage/domain/entities/triage_item.py`
- Create: `apps/api/src/prescient/triage/domain/entities/shared_item.py`
- Create: `apps/api/src/prescient/triage/domain/errors.py`
- Modify: `apps/api/tests/unit/triage/test_triage_entities.py`

- [ ] **Step 1: Write failing tests for TriageQuestion entity**

Append to `apps/api/tests/unit/triage/test_triage_entities.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

from prescient.triage.domain.entities.triage_question import TriageQuestion
from prescient.triage.domain.entities.triage_item import TriageItem
from prescient.triage.domain.entities.shared_item import SharedItem


class TestTriageQuestion:
    def test_create_minimal(self) -> None:
        """A TriageQuestion can be created with required fields."""
        q = TriageQuestion(
            id=uuid4(),
            asker_user_id="user-001",
            asker_org_id="org-summit-001",
            target_company_org_id="org-peloton-001",
            question_text="What is driving subscriber churn?",
            categories_detected=(),
            status=TriageQuestionStatus.SUBMITTED,
            priority=TriagePriority.HIGH,
            created_at=datetime.now(tz=UTC),
        )
        assert q.status == TriageQuestionStatus.SUBMITTED
        assert q.partial_answer is None
        assert q.final_answer is None
        assert q.brainstorm_conversation_id is None

    def test_frozen(self) -> None:
        """TriageQuestion is immutable."""
        q = TriageQuestion(
            id=uuid4(),
            asker_user_id="user-001",
            asker_org_id="org-summit-001",
            target_company_org_id="org-peloton-001",
            question_text="Test?",
            categories_detected=(),
            status=TriageQuestionStatus.SUBMITTED,
            priority=TriagePriority.HIGH,
            created_at=datetime.now(tz=UTC),
        )
        with pytest.raises(Exception):
            q.status = TriageQuestionStatus.ANSWERED  # type: ignore[misc]


class TestTriageItem:
    def test_create_sponsor_question_item(self) -> None:
        item = TriageItem(
            id=uuid4(),
            organization_id="org-peloton-001",
            item_type=TriageItemType.SPONSOR_QUESTION,
            source_id="some-question-id",
            title="Subscriber churn question",
            summary="Michael Torres asks about churn drivers",
            priority=TriagePriority.HIGH,
            priority_signals=("from operating_partner",),
            status=TriageItemStatus.PENDING,
            created_at=datetime.now(tz=UTC),
        )
        assert item.item_type == TriageItemType.SPONSOR_QUESTION
        assert item.assigned_to_user_id is None
        assert item.resolved_at is None

    def test_create_kpi_anomaly_item(self) -> None:
        item = TriageItem(
            id=uuid4(),
            organization_id="org-peloton-001",
            item_type=TriageItemType.KPI_ANOMALY,
            source_id="anomaly-uuid",
            title="Churn rate spike",
            summary="Churn increased 27% MoM",
            priority=TriagePriority.HIGH,
            priority_signals=("severity: critical",),
            status=TriageItemStatus.PENDING,
            created_at=datetime.now(tz=UTC),
        )
        assert item.item_type == TriageItemType.KPI_ANOMALY


class TestSharedItem:
    def test_create_shared_item(self) -> None:
        item = SharedItem(
            id=uuid4(),
            artifact_id="artifact-001",
            sponsor_org_id="org-summit-001",
            shared_by_user_id="sarah-chen-001",
            shared_at=datetime.now(tz=UTC),
        )
        assert item.triage_question_id is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_triage_entities.py::TestTriageQuestion -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement TriageQuestion entity**

```python
# apps/api/src/prescient/triage/domain/entities/triage_question.py
"""TriageQuestion domain entity — a question asked by a sponsor."""

from __future__ import annotations

from datetime import datetime
from typing import Any
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field

from prescient.triage.domain.enums import TriagePriority, TriageQuestionStatus


class TriageQuestion(BaseModel):
    model_config = ConfigDict(frozen=True, strict=True)

    id: UUID
    asker_user_id: str
    asker_org_id: str
    target_company_org_id: str
    question_text: str = Field(min_length=1)
    categories_detected: tuple[str, ...] = Field(default_factory=tuple)
    status: TriageQuestionStatus
    partial_answer: str | None = None
    partial_citations: tuple[dict[str, Any], ...] | None = None
    escalation_reason: str | None = None
    final_answer: str | None = None
    final_citations: tuple[dict[str, Any], ...] | None = None
    brainstorm_conversation_id: UUID | None = None
    priority: TriagePriority
    created_at: datetime
    answered_at: datetime | None = None
```

- [ ] **Step 4: Implement TriageItem entity**

```python
# apps/api/src/prescient/triage/domain/entities/triage_item.py
"""TriageItem domain entity — a general-purpose attention queue entry."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field

from prescient.triage.domain.enums import (
    TriageItemStatus,
    TriageItemType,
    TriagePriority,
)


class TriageItem(BaseModel):
    model_config = ConfigDict(frozen=True, strict=True)

    id: UUID
    organization_id: str
    item_type: TriageItemType
    source_id: str
    title: str = Field(min_length=1, max_length=256)
    summary: str = Field(min_length=1)
    priority: TriagePriority
    priority_signals: tuple[str, ...] = Field(default_factory=tuple)
    status: TriageItemStatus
    assigned_to_user_id: str | None = None
    created_at: datetime
    resolved_at: datetime | None = None
```

- [ ] **Step 5: Implement SharedItem entity**

```python
# apps/api/src/prescient/triage/domain/entities/shared_item.py
"""SharedItem domain entity — tracks what operators have shared with sponsors."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict


class SharedItem(BaseModel):
    model_config = ConfigDict(frozen=True, strict=True)

    id: UUID
    artifact_id: str
    sponsor_org_id: str
    shared_by_user_id: str
    shared_at: datetime
    triage_question_id: UUID | None = None
```

- [ ] **Step 6: Create entities __init__.py and errors**

```python
# apps/api/src/prescient/triage/domain/entities/__init__.py
"""Triage domain entities."""

from prescient.triage.domain.entities.shared_item import SharedItem
from prescient.triage.domain.entities.triage_item import TriageItem
from prescient.triage.domain.entities.triage_question import TriageQuestion

__all__ = ["SharedItem", "TriageItem", "TriageQuestion"]
```

```python
# apps/api/src/prescient/triage/domain/errors.py
"""Triage domain errors."""

from __future__ import annotations

from prescient.shared.errors import DomainError


class TriageStateError(DomainError):
    """Raised when a triage state transition is invalid."""
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_triage_entities.py -v`
Expected: PASS (all tests)

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/triage/domain/ apps/api/tests/unit/triage/
git commit -m "feat(triage): add TriageQuestion, TriageItem, SharedItem domain entities"
```

---

### Task 3: Triage SQLAlchemy Tables

**Files:**
- Create: `apps/api/src/prescient/triage/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/triage/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/triage/infrastructure/tables/triage_question.py`
- Create: `apps/api/src/prescient/triage/infrastructure/tables/triage_item.py`
- Create: `apps/api/src/prescient/triage/infrastructure/tables/shared_item.py`
- Modify: `apps/api/src/prescient/shared/metadata.py`

- [ ] **Step 1: Create infrastructure package structure**

```python
# apps/api/src/prescient/triage/infrastructure/__init__.py
```

```python
# apps/api/src/prescient/triage/infrastructure/tables/__init__.py
"""Triage ORM tables."""

from prescient.triage.infrastructure.tables.shared_item import SharedItemTable
from prescient.triage.infrastructure.tables.triage_item import TriageItemTable
from prescient.triage.infrastructure.tables.triage_question import TriageQuestionTable

__all__ = ["SharedItemTable", "TriageItemTable", "TriageQuestionTable"]
```

- [ ] **Step 2: Implement TriageQuestionTable**

```python
# apps/api/src/prescient/triage/infrastructure/tables/triage_question.py
"""`triage.triage_questions` — sponsor questions with classification and answers."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from sqlalchemy import DateTime, ForeignKey, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class TriageQuestionTable(Base):
    __tablename__ = "triage_questions"
    __table_args__ = {"schema": "triage"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    asker_user_id: Mapped[str] = mapped_column(
        String(64),
        ForeignKey("organizations.users.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    asker_org_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("organizations.organizations.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    target_company_org_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("organizations.organizations.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    question_text: Mapped[str] = mapped_column(Text, nullable=False)
    categories_detected: Mapped[list[str]] = mapped_column(
        JSONB, nullable=False, server_default="[]"
    )
    status: Mapped[str] = mapped_column(String(32), nullable=False, index=True)
    partial_answer: Mapped[str | None] = mapped_column(Text, nullable=True)
    partial_citations: Mapped[list[dict] | None] = mapped_column(JSONB, nullable=True)
    escalation_reason: Mapped[str | None] = mapped_column(Text, nullable=True)
    final_answer: Mapped[str | None] = mapped_column(Text, nullable=True)
    final_citations: Mapped[list[dict] | None] = mapped_column(JSONB, nullable=True)
    brainstorm_conversation_id: Mapped[UUID | None] = mapped_column(
        PgUUID(as_uuid=True),
        ForeignKey("intelligence.copilot_conversations.id", ondelete="SET NULL"),
        nullable=True,
    )
    priority: Mapped[str] = mapped_column(String(16), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    answered_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
```

- [ ] **Step 3: Implement TriageItemTable**

```python
# apps/api/src/prescient/triage/infrastructure/tables/triage_item.py
"""`triage.triage_items` — general-purpose attention queue entries."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from sqlalchemy import DateTime, ForeignKey, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class TriageItemTable(Base):
    __tablename__ = "triage_items"
    __table_args__ = {"schema": "triage"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    organization_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("organizations.organizations.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    item_type: Mapped[str] = mapped_column(String(32), nullable=False, index=True)
    source_id: Mapped[str] = mapped_column(String(64), nullable=False)
    title: Mapped[str] = mapped_column(String(256), nullable=False)
    summary: Mapped[str] = mapped_column(Text, nullable=False)
    priority: Mapped[str] = mapped_column(String(16), nullable=False, index=True)
    priority_signals: Mapped[list[str]] = mapped_column(
        JSONB, nullable=False, server_default="[]"
    )
    status: Mapped[str] = mapped_column(
        String(16), nullable=False, server_default="pending", index=True
    )
    assigned_to_user_id: Mapped[str | None] = mapped_column(
        String(64),
        ForeignKey("organizations.users.id", ondelete="SET NULL"),
        nullable=True,
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    resolved_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
```

- [ ] **Step 4: Implement SharedItemTable**

```python
# apps/api/src/prescient/triage/infrastructure/tables/shared_item.py
"""`triage.shared_items` — tracks artifacts shared with sponsor orgs."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from sqlalchemy import DateTime, ForeignKey, String, func
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class SharedItemTable(Base):
    __tablename__ = "shared_items"
    __table_args__ = {"schema": "triage"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    artifact_id: Mapped[str] = mapped_column(
        String(36),
        nullable=False,
        index=True,
    )
    sponsor_org_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("organizations.organizations.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    shared_by_user_id: Mapped[str] = mapped_column(
        String(64),
        ForeignKey("organizations.users.id", ondelete="CASCADE"),
        nullable=False,
    )
    shared_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    triage_question_id: Mapped[UUID | None] = mapped_column(
        PgUUID(as_uuid=True),
        ForeignKey("triage.triage_questions.id", ondelete="SET NULL"),
        nullable=True,
    )
```

- [ ] **Step 5: Register triage tables with ORM metadata**

Open `apps/api/src/prescient/shared/metadata.py` and add the triage table imports so they are registered with the SQLAlchemy Base metadata. Follow the existing import pattern — find the last context import and add after it:

```python
import prescient.triage.infrastructure.tables  # noqa: F401
```

- [ ] **Step 6: Verify tables can be imported**

Run: `cd apps/api && uv run python -c "from prescient.triage.infrastructure.tables import TriageQuestionTable, TriageItemTable, SharedItemTable; print('OK')"`
Expected: prints `OK`

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/triage/infrastructure/ apps/api/src/prescient/shared/metadata.py
git commit -m "feat(triage): add SQLAlchemy ORM tables for triage schema"
```

---

### Task 4: Alembic Migration

**Files:**
- Create: `apps/api/alembic/versions/{next}_phase5_triage_schema.py`

- [ ] **Step 1: Find the current Alembic head revision**

Run: `cd apps/api && uv run alembic heads`

Note the revision ID — you'll use it as `down_revision` in the new migration.

- [ ] **Step 2: Generate migration stub**

Run: `cd apps/api && uv run alembic revision -m "phase5_triage_schema"`

This creates a new migration file. Note the file path.

- [ ] **Step 3: Write the migration**

Replace the generated migration content with:

```python
"""phase5_triage_schema

Create the triage schema with triage_questions, triage_items, and shared_items tables.
"""

from __future__ import annotations

from collections.abc import Sequence

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

# revision identifiers — filled in by alembic revision
revision: str = "<GENERATED>"
down_revision: str | None = "<PREVIOUS_HEAD>"
branch_labels: str | Sequence[str] | None = None
depends_on: str | Sequence[str] | None = None


def upgrade() -> None:
    op.execute("CREATE SCHEMA IF NOT EXISTS triage")

    # triage.triage_questions
    op.create_table(
        "triage_questions",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("asker_user_id", sa.String(64), nullable=False),
        sa.Column("asker_org_id", sa.String(36), nullable=False),
        sa.Column("target_company_org_id", sa.String(36), nullable=False),
        sa.Column("question_text", sa.Text(), nullable=False),
        sa.Column(
            "categories_detected",
            postgresql.JSONB(astext_type=sa.Text()),
            server_default="[]",
            nullable=False,
        ),
        sa.Column("status", sa.String(32), nullable=False),
        sa.Column("partial_answer", sa.Text(), nullable=True),
        sa.Column(
            "partial_citations",
            postgresql.JSONB(astext_type=sa.Text()),
            nullable=True,
        ),
        sa.Column("escalation_reason", sa.Text(), nullable=True),
        sa.Column("final_answer", sa.Text(), nullable=True),
        sa.Column(
            "final_citations",
            postgresql.JSONB(astext_type=sa.Text()),
            nullable=True,
        ),
        sa.Column("brainstorm_conversation_id", postgresql.UUID(as_uuid=True), nullable=True),
        sa.Column("priority", sa.String(16), nullable=False),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.Column("answered_at", sa.DateTime(timezone=True), nullable=True),
        sa.PrimaryKeyConstraint("id"),
        sa.ForeignKeyConstraint(
            ["asker_user_id"],
            ["organizations.users.id"],
            ondelete="CASCADE",
        ),
        sa.ForeignKeyConstraint(
            ["asker_org_id"],
            ["organizations.organizations.id"],
            ondelete="CASCADE",
        ),
        sa.ForeignKeyConstraint(
            ["target_company_org_id"],
            ["organizations.organizations.id"],
            ondelete="CASCADE",
        ),
        sa.ForeignKeyConstraint(
            ["brainstorm_conversation_id"],
            ["intelligence.copilot_conversations.id"],
            ondelete="SET NULL",
        ),
        schema="triage",
    )
    op.create_index(
        "ix_triage_questions_asker_user_id",
        "triage_questions",
        ["asker_user_id"],
        schema="triage",
    )
    op.create_index(
        "ix_triage_questions_asker_org_id",
        "triage_questions",
        ["asker_org_id"],
        schema="triage",
    )
    op.create_index(
        "ix_triage_questions_target_company_org_id",
        "triage_questions",
        ["target_company_org_id"],
        schema="triage",
    )
    op.create_index(
        "ix_triage_questions_status",
        "triage_questions",
        ["status"],
        schema="triage",
    )

    # triage.triage_items
    op.create_table(
        "triage_items",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("organization_id", sa.String(36), nullable=False),
        sa.Column("item_type", sa.String(32), nullable=False),
        sa.Column("source_id", sa.String(64), nullable=False),
        sa.Column("title", sa.String(256), nullable=False),
        sa.Column("summary", sa.Text(), nullable=False),
        sa.Column("priority", sa.String(16), nullable=False),
        sa.Column(
            "priority_signals",
            postgresql.JSONB(astext_type=sa.Text()),
            server_default="[]",
            nullable=False,
        ),
        sa.Column("status", sa.String(16), server_default="pending", nullable=False),
        sa.Column("assigned_to_user_id", sa.String(64), nullable=True),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.Column("resolved_at", sa.DateTime(timezone=True), nullable=True),
        sa.PrimaryKeyConstraint("id"),
        sa.ForeignKeyConstraint(
            ["organization_id"],
            ["organizations.organizations.id"],
            ondelete="CASCADE",
        ),
        sa.ForeignKeyConstraint(
            ["assigned_to_user_id"],
            ["organizations.users.id"],
            ondelete="SET NULL",
        ),
        schema="triage",
    )
    op.create_index(
        "ix_triage_items_organization_id",
        "triage_items",
        ["organization_id"],
        schema="triage",
    )
    op.create_index(
        "ix_triage_items_item_type",
        "triage_items",
        ["item_type"],
        schema="triage",
    )
    op.create_index(
        "ix_triage_items_priority",
        "triage_items",
        ["priority"],
        schema="triage",
    )
    op.create_index(
        "ix_triage_items_status",
        "triage_items",
        ["status"],
        schema="triage",
    )

    # triage.shared_items
    op.create_table(
        "shared_items",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("artifact_id", sa.String(36), nullable=False),
        sa.Column("sponsor_org_id", sa.String(36), nullable=False),
        sa.Column("shared_by_user_id", sa.String(64), nullable=False),
        sa.Column(
            "shared_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.Column("triage_question_id", postgresql.UUID(as_uuid=True), nullable=True),
        sa.PrimaryKeyConstraint("id"),
        sa.ForeignKeyConstraint(
            ["sponsor_org_id"],
            ["organizations.organizations.id"],
            ondelete="CASCADE",
        ),
        sa.ForeignKeyConstraint(
            ["shared_by_user_id"],
            ["organizations.users.id"],
            ondelete="CASCADE",
        ),
        sa.ForeignKeyConstraint(
            ["triage_question_id"],
            ["triage.triage_questions.id"],
            ondelete="SET NULL",
        ),
        schema="triage",
    )
    op.create_index(
        "ix_shared_items_artifact_id",
        "shared_items",
        ["artifact_id"],
        schema="triage",
    )
    op.create_index(
        "ix_shared_items_sponsor_org_id",
        "shared_items",
        ["sponsor_org_id"],
        schema="triage",
    )


def downgrade() -> None:
    op.drop_index("ix_shared_items_sponsor_org_id", table_name="shared_items", schema="triage")
    op.drop_index("ix_shared_items_artifact_id", table_name="shared_items", schema="triage")
    op.drop_table("shared_items", schema="triage")

    op.drop_index("ix_triage_items_status", table_name="triage_items", schema="triage")
    op.drop_index("ix_triage_items_priority", table_name="triage_items", schema="triage")
    op.drop_index("ix_triage_items_item_type", table_name="triage_items", schema="triage")
    op.drop_index("ix_triage_items_organization_id", table_name="triage_items", schema="triage")
    op.drop_table("triage_items", schema="triage")

    op.drop_index("ix_triage_questions_status", table_name="triage_questions", schema="triage")
    op.drop_index(
        "ix_triage_questions_target_company_org_id",
        table_name="triage_questions",
        schema="triage",
    )
    op.drop_index("ix_triage_questions_asker_org_id", table_name="triage_questions", schema="triage")
    op.drop_index(
        "ix_triage_questions_asker_user_id", table_name="triage_questions", schema="triage"
    )
    op.drop_table("triage_questions", schema="triage")

    op.execute("DROP SCHEMA IF EXISTS triage")
```

- [ ] **Step 4: Run the migration against Docker Postgres**

Run: `cd infrastructure && docker compose run --rm migrate`
Expected: Migration applies successfully with no errors.

- [ ] **Step 5: Verify tables exist**

Run: `docker exec prescient-postgres psql -U prescient -d prescient -c "\dt triage.*"`
Expected: Shows `triage_questions`, `triage_items`, and `shared_items` tables.

- [ ] **Step 6: Commit**

```bash
git add apps/api/alembic/versions/*phase5_triage_schema*
git commit -m "feat(triage): add Alembic migration for triage schema — 3 tables"
```

---

### Task 5: AccessContext Dataclass and Dependency

**Files:**
- Create: `apps/api/src/prescient/auth/access_context.py`
- Test: `apps/api/tests/unit/auth/test_access_context.py`

- [ ] **Step 1: Write failing tests for AccessContext**

```python
# apps/api/tests/unit/auth/test_access_context.py
"""Unit tests for AccessContext and visibility resolution."""

from __future__ import annotations

import pytest

from prescient.auth.access_context import AccessContext
from prescient.organizations.domain.entities.user import UserRole
from prescient.organizations.domain.entities.membership import MembershipScope
from prescient.organizations.domain.entities.visibility_rule import DataCategory


class TestAccessContext:
    def test_operator_is_not_sponsor(self) -> None:
        ctx = AccessContext(
            user_id="sarah-chen-001",
            org_id="org-peloton-001",
            role=UserRole.OPERATOR,
            scope=MembershipScope.FULL,
            visible_categories=frozenset(),
            gated_categories=frozenset(),
            blocked_categories=frozenset(),
            company_org_ids=("org-peloton-001",),
        )
        assert ctx.is_sponsor is False

    def test_operating_partner_is_sponsor(self) -> None:
        ctx = AccessContext(
            user_id="michael-torres-001",
            org_id="org-summit-001",
            role=UserRole.OPERATING_PARTNER,
            scope=MembershipScope.FULL,
            visible_categories=frozenset({DataCategory.KPI_METRICS}),
            gated_categories=frozenset({
                DataCategory.NARRATIVE,
                DataCategory.DECISIONS,
                DataCategory.ACTION_ITEMS,
                DataCategory.INTELLIGENCE_SIGNALS,
                DataCategory.BRIEFINGS,
            }),
            blocked_categories=frozenset(),
            company_org_ids=("org-peloton-001",),
        )
        assert ctx.is_sponsor is True
        assert DataCategory.KPI_METRICS in ctx.visible_categories
        assert DataCategory.NARRATIVE in ctx.gated_categories

    def test_board_member_is_sponsor(self) -> None:
        ctx = AccessContext(
            user_id="board-member-001",
            org_id="org-summit-001",
            role=UserRole.BOARD_MEMBER,
            scope=MembershipScope.READ_ONLY,
            visible_categories=frozenset({DataCategory.KPI_METRICS}),
            gated_categories=frozenset(),
            blocked_categories=frozenset(),
            company_org_ids=("org-peloton-001",),
        )
        assert ctx.is_sponsor is True

    def test_pe_analyst_is_sponsor(self) -> None:
        ctx = AccessContext(
            user_id="analyst-001",
            org_id="org-summit-001",
            role=UserRole.PE_ANALYST,
            scope=MembershipScope.KPI_ONLY,
            visible_categories=frozenset({DataCategory.KPI_METRICS}),
            gated_categories=frozenset(),
            blocked_categories=frozenset(),
            company_org_ids=("org-peloton-001",),
        )
        assert ctx.is_sponsor is True

    def test_can_see_category_visible(self) -> None:
        ctx = AccessContext(
            user_id="michael-torres-001",
            org_id="org-summit-001",
            role=UserRole.OPERATING_PARTNER,
            scope=MembershipScope.FULL,
            visible_categories=frozenset({DataCategory.KPI_METRICS}),
            gated_categories=frozenset({DataCategory.NARRATIVE}),
            blocked_categories=frozenset(),
            company_org_ids=("org-peloton-001",),
        )
        assert ctx.can_see_category(DataCategory.KPI_METRICS) is True
        assert ctx.can_see_category(DataCategory.NARRATIVE) is False
        assert ctx.can_see_category(DataCategory.BRIEFINGS) is False

    def test_operator_can_see_all_categories(self) -> None:
        ctx = AccessContext(
            user_id="sarah-chen-001",
            org_id="org-peloton-001",
            role=UserRole.OPERATOR,
            scope=MembershipScope.FULL,
            visible_categories=frozenset(),
            gated_categories=frozenset(),
            blocked_categories=frozenset(),
            company_org_ids=("org-peloton-001",),
        )
        assert ctx.can_see_category(DataCategory.KPI_METRICS) is True
        assert ctx.can_see_category(DataCategory.NARRATIVE) is True
        assert ctx.can_see_category(DataCategory.DECISIONS) is True
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/auth/test_access_context.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'prescient.auth.access_context'`

- [ ] **Step 3: Implement AccessContext**

```python
# apps/api/src/prescient/auth/access_context.py
"""AccessContext — resolved visibility context for the current request.

Carries the user's role, scope, and visibility rules so that routes can
filter data without querying the organization tables again. Built by
`get_access_context()` which is a FastAPI dependency.
"""

from __future__ import annotations

from dataclasses import dataclass

from fastapi import Depends, Request
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.auth.base import CurrentUser, get_current_user
from prescient.db import get_session
from prescient.organizations.domain.entities.membership import MembershipScope
from prescient.organizations.domain.entities.user import UserRole
from prescient.organizations.domain.entities.visibility_rule import DataCategory, GateType
from prescient.organizations.infrastructure.tables.membership import MembershipTable
from prescient.organizations.infrastructure.tables.portfolio_link import PortfolioLinkTable
from prescient.organizations.infrastructure.tables.visibility_rule import VisibilityRuleTable

_SPONSOR_ROLES: frozenset[UserRole] = frozenset({
    UserRole.OPERATING_PARTNER,
    UserRole.PE_ANALYST,
    UserRole.BOARD_MEMBER,
})


@dataclass(frozen=True, slots=True)
class AccessContext:
    """Resolved visibility context for the current request."""

    user_id: str
    org_id: str
    role: UserRole
    scope: MembershipScope
    visible_categories: frozenset[DataCategory]
    gated_categories: frozenset[DataCategory]
    blocked_categories: frozenset[DataCategory]
    company_org_ids: tuple[str, ...]

    @property
    def is_sponsor(self) -> bool:
        return self.role in _SPONSOR_ROLES

    def can_see_category(self, category: DataCategory) -> bool:
        """Return True if the user can see data in this category.

        Operators can see everything. Sponsors can only see AUTOMATIC categories.
        """
        if not self.is_sponsor:
            return True
        return category in self.visible_categories


async def get_access_context(
    current_user: CurrentUser = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
) -> AccessContext:
    """FastAPI dependency that resolves the full access context.

    For operators, returns full access. For sponsors, resolves portfolio
    links and visibility rules.
    """
    # Look up membership to get role, scope, and org_id
    membership_stmt = (
        select(MembershipTable)
        .where(
            MembershipTable.user_id == current_user.user_id,
            MembershipTable.revoked_at.is_(None),
        )
        .limit(1)
    )
    membership_result = await session.execute(membership_stmt)
    membership_row = membership_result.scalar_one_or_none()

    if membership_row is None:
        # Fallback for users without membership rows (legacy demo setup)
        return AccessContext(
            user_id=current_user.user_id,
            org_id=str(current_user.fund_id),
            role=UserRole.OPERATOR,
            scope=MembershipScope.FULL,
            visible_categories=frozenset(),
            gated_categories=frozenset(),
            blocked_categories=frozenset(),
            company_org_ids=(),
        )

    role = UserRole(membership_row.role)
    scope = MembershipScope(membership_row.scope)
    org_id = membership_row.organization_id

    if role not in _SPONSOR_ROLES:
        # Operators see everything — no visibility restrictions
        return AccessContext(
            user_id=current_user.user_id,
            org_id=org_id,
            role=role,
            scope=scope,
            visible_categories=frozenset(),
            gated_categories=frozenset(),
            blocked_categories=frozenset(),
            company_org_ids=(org_id,),
        )

    # Sponsor: resolve portfolio links and visibility rules
    links_stmt = select(PortfolioLinkTable).where(
        PortfolioLinkTable.sponsor_org_id == org_id,
        PortfolioLinkTable.status == "active",
    )
    links_result = await session.execute(links_stmt)
    links = list(links_result.scalars().all())

    company_org_ids = tuple(link.company_org_id for link in links)
    link_ids = [link.id for link in links]

    visible: set[DataCategory] = set()
    gated: set[DataCategory] = set()
    blocked: set[DataCategory] = set()

    if link_ids:
        rules_stmt = select(VisibilityRuleTable).where(
            VisibilityRuleTable.portfolio_link_id.in_(link_ids),
            VisibilityRuleTable.direction.in_(["company_to_sponsor", "bidirectional"]),
        )
        rules_result = await session.execute(rules_stmt)
        rules = list(rules_result.scalars().all())

        for rule in rules:
            category = DataCategory(rule.data_category)
            gate = GateType(rule.gate_type)
            if gate == GateType.AUTOMATIC:
                visible.add(category)
            elif gate == GateType.OPERATOR_APPROVED:
                gated.add(category)
            elif gate == GateType.BLOCKED:
                blocked.add(category)

    return AccessContext(
        user_id=current_user.user_id,
        org_id=org_id,
        role=role,
        scope=scope,
        visible_categories=frozenset(visible),
        gated_categories=frozenset(gated),
        blocked_categories=frozenset(blocked),
        company_org_ids=company_org_ids,
    )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/auth/test_access_context.py -v`
Expected: PASS (all 7 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/auth/access_context.py apps/api/tests/unit/auth/test_access_context.py
git commit -m "feat(auth): add AccessContext with visibility rule resolution"
```

---

### Task 6: Add Shared Type Aliases

**Files:**
- Modify: `apps/api/src/prescient/shared/types.py`

- [ ] **Step 1: Add triage type aliases**

Open `apps/api/src/prescient/shared/types.py` and add these type aliases alongside the existing ones (after the last `NewType` definition):

```python
TriageQuestionId = NewType("TriageQuestionId", UUID)
TriageItemId = NewType("TriageItemId", UUID)
SharedItemId = NewType("SharedItemId", UUID)
```

- [ ] **Step 2: Verify imports work**

Run: `cd apps/api && uv run python -c "from prescient.shared.types import TriageQuestionId, TriageItemId, SharedItemId; print('OK')"`
Expected: prints `OK`

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/shared/types.py
git commit -m "feat(shared): add TriageQuestionId, TriageItemId, SharedItemId type aliases"
```

---

### Task 7: Run Full Test Suite

**Files:** None (verification only)

- [ ] **Step 1: Run all existing unit tests**

Run: `cd apps/api && uv run pytest tests/unit/ -v --tb=short`
Expected: All existing tests PASS, plus the new triage and access_context tests.

- [ ] **Step 2: Run type checking**

Run: `cd apps/api && uv run mypy src/prescient/triage/ src/prescient/auth/access_context.py --ignore-missing-imports`
Expected: No type errors (or only pre-existing ones unrelated to new code).

- [ ] **Step 3: Run linting**

Run: `cd apps/api && uv run ruff check src/prescient/triage/ src/prescient/auth/access_context.py`
Expected: No lint errors.

- [ ] **Step 4: If any failures, fix and commit**

Fix any issues found in steps 1-3, then:

```bash
git add -A
git commit -m "fix: resolve test/lint issues from Phase 5a"
```

---
