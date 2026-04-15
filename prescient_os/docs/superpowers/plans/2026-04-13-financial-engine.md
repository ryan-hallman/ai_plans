# Board Prep Financial Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add structured financial slide types (P&L, segments, cash flow, KPI trends, margins, pipeline, budget) to the board deck editor, backed by new pipeline and budget data models with seed data.

**Architecture:** Three layers built bottom-up: (1) Pipeline and budget database tables + seed data, (2) typed slide system in the assembler that queries data directly instead of calling the LLM, (3) frontend slide renderer registry with dedicated components for tables and charts via Recharts.

**Tech Stack:** FastAPI, SQLAlchemy, Alembic (backend), Next.js 15, React 19, TypeScript, Zustand, Recharts, Tailwind (frontend), PostgreSQL.

**Worktree:** `.worktrees/financial-engine` on branch `feature/board-prep-financial-engine`

---

## File Structure

### Backend — New Files

| File | Responsibility |
|------|---------------|
| `apps/api/alembic/versions/20260413_pipeline_budget_schema.py` | Migration: pipeline.deals, budget.budget_plans, budget.budget_line_items |
| `apps/api/src/prescient/pipeline/__init__.py` | Pipeline bounded context package |
| `apps/api/src/prescient/pipeline/infrastructure/__init__.py` | |
| `apps/api/src/prescient/pipeline/infrastructure/tables/__init__.py` | |
| `apps/api/src/prescient/pipeline/infrastructure/tables/deal.py` | SQLAlchemy model for pipeline.deals |
| `apps/api/src/prescient/pipeline/api/__init__.py` | |
| `apps/api/src/prescient/pipeline/api/routes.py` | CRUD endpoints for deals |
| `apps/api/src/prescient/budget/__init__.py` | Budget bounded context package |
| `apps/api/src/prescient/budget/infrastructure/__init__.py` | |
| `apps/api/src/prescient/budget/infrastructure/tables/__init__.py` | |
| `apps/api/src/prescient/budget/infrastructure/tables/budget_plan.py` | SQLAlchemy model for budget.budget_plans |
| `apps/api/src/prescient/budget/infrastructure/tables/budget_line_item.py` | SQLAlchemy model for budget.budget_line_items |
| `apps/api/src/prescient/budget/api/__init__.py` | |
| `apps/api/src/prescient/budget/api/routes.py` | CRUD endpoints for budget plans + line items |
| `apps/api/src/prescient/board_prep/application/data_slides.py` | Assembler functions for each data slide type |
| `seed/peloton/pipeline_deals.py` | Seed pipeline deals for Peloton |
| `seed/peloton/budget.py` | Seed budget plan + line items for Peloton |

### Backend — Modified Files

| File | Change |
|------|--------|
| `apps/api/src/prescient/main.py` | Register pipeline and budget routers |
| `apps/api/src/prescient/board_prep/api/routes.py` | Update SlidePayload with slide_type + data fields |
| `apps/api/src/prescient/board_prep/domain/entities/board_deck.py` | Add slide_type + data to BoardSection |
| `apps/api/src/prescient/board_prep/application/assembler.py` | Generate typed slides for financial sections |
| `seed/loader.py` | Add pipeline + budget seed loading |

### Frontend — New Files

| File | Responsibility |
|------|---------------|
| `apps/web/src/components/board-prep/slide-renderer.tsx` | Registry: switches on slide_type to render correct component |
| `apps/web/src/components/board-prep/slides/pl-summary-slide.tsx` | P&L table renderer |
| `apps/web/src/components/board-prep/slides/cash-flow-slide.tsx` | Cash flow table renderer |
| `apps/web/src/components/board-prep/slides/segment-breakdown-slide.tsx` | Segment bar chart + table |
| `apps/web/src/components/board-prep/slides/kpi-trends-slide.tsx` | KPI trend line/bar charts |
| `apps/web/src/components/board-prep/slides/margin-analysis-slide.tsx` | Margin trend charts + table |
| `apps/web/src/components/board-prep/slides/pipeline-slide.tsx` | Pipeline funnel + deal table |
| `apps/web/src/components/board-prep/slides/budget-vs-actual-slide.tsx` | Budget table with variance coloring |
| `apps/web/src/components/board-prep/slides/financial-table.tsx` | Shared table component for P&L, cash flow, budget |

### Frontend — Modified Files

| File | Change |
|------|--------|
| `apps/web/src/stores/deck-store.ts` | Add slide_type + data to Slide interface |
| `apps/web/src/components/board-prep/slide-canvas.tsx` | Use SlideRenderer instead of directly rendering SlideContent |

---

## Task 1: Pipeline & Budget Database Schema

**Files:**
- Create: `apps/api/alembic/versions/20260413_pipeline_budget_schema.py`
- Create: `apps/api/src/prescient/pipeline/__init__.py`
- Create: `apps/api/src/prescient/pipeline/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/pipeline/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/pipeline/infrastructure/tables/deal.py`
- Create: `apps/api/src/prescient/budget/__init__.py`
- Create: `apps/api/src/prescient/budget/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/budget/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/budget/infrastructure/tables/budget_plan.py`
- Create: `apps/api/src/prescient/budget/infrastructure/tables/budget_line_item.py`

- [ ] **Step 1: Create the migration file**

Create `apps/api/alembic/versions/20260413_pipeline_budget_schema.py`:

```python
"""pipeline and budget schemas

Revision ID: 0008_pipeline_budget
Revises: 0007_board_prep_schema
Create Date: 2026-04-13

Adds pipeline.deals, budget.budget_plans, budget.budget_line_items.
"""

from __future__ import annotations

from collections.abc import Sequence

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision: str = "0008_pipeline_budget"
down_revision: str | None = "0007_board_prep_schema"
branch_labels: str | Sequence[str] | None = None
depends_on: str | Sequence[str] | None = None


def upgrade() -> None:
    # ---- pipeline schema ---------------------------------------------------
    op.execute("CREATE SCHEMA IF NOT EXISTS pipeline")

    op.create_table(
        "deals",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("company_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("owner_tenant_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("deal_name", sa.String(256), nullable=False),
        sa.Column("customer_name", sa.String(256), nullable=False),
        sa.Column("stage", sa.String(32), nullable=False),
        sa.Column("amount", sa.Numeric(20, 4), nullable=False),
        sa.Column("currency", sa.String(3), nullable=False, server_default="USD"),
        sa.Column("probability", sa.Numeric(5, 2), nullable=False, server_default="0"),
        sa.Column("expected_close_date", sa.Date(), nullable=False),
        sa.Column("product_line", sa.String(128), nullable=False),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.text("now()"),
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.text("now()"),
        ),
        sa.PrimaryKeyConstraint("id"),
        schema="pipeline",
    )
    op.create_index("ix_pipeline_deals_company", "deals", ["company_id"], schema="pipeline")
    op.create_index("ix_pipeline_deals_tenant", "deals", ["owner_tenant_id"], schema="pipeline")

    # ---- budget schema -----------------------------------------------------
    op.execute("CREATE SCHEMA IF NOT EXISTS budget")

    op.create_table(
        "budget_plans",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("company_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("owner_tenant_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("period_label", sa.String(16), nullable=False),
        sa.Column("period_start", sa.Date(), nullable=False),
        sa.Column("period_end", sa.Date(), nullable=False),
        sa.Column("status", sa.String(16), nullable=False, server_default="draft"),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.text("now()"),
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.text("now()"),
        ),
        sa.PrimaryKeyConstraint("id"),
        schema="budget",
    )
    op.create_index(
        "ix_budget_plans_company", "budget_plans", ["company_id"], schema="budget"
    )

    op.create_table(
        "budget_line_items",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column(
            "budget_plan_id",
            postgresql.UUID(as_uuid=True),
            sa.ForeignKey("budget.budget_plans.id", ondelete="CASCADE"),
            nullable=False,
        ),
        sa.Column("category", sa.String(32), nullable=False),
        sa.Column("subcategory", sa.String(64), nullable=False),
        sa.Column("label", sa.String(128), nullable=False),
        sa.Column("planned_amount", sa.Numeric(20, 4), nullable=False),
        sa.Column("actual_amount", sa.Numeric(20, 4), nullable=True),
        sa.Column("sort_order", sa.Integer(), nullable=False, server_default="0"),
        sa.PrimaryKeyConstraint("id"),
        schema="budget",
    )
    op.create_index(
        "ix_budget_line_items_plan",
        "budget_line_items",
        ["budget_plan_id"],
        schema="budget",
    )


def downgrade() -> None:
    op.drop_table("budget_line_items", schema="budget")
    op.drop_table("budget_plans", schema="budget")
    op.execute("DROP SCHEMA IF EXISTS budget CASCADE")
    op.drop_table("deals", schema="pipeline")
    op.execute("DROP SCHEMA IF EXISTS pipeline CASCADE")
```

- [ ] **Step 2: Create pipeline table model**

Create empty `__init__.py` files for pipeline package:
- `apps/api/src/prescient/pipeline/__init__.py`
- `apps/api/src/prescient/pipeline/infrastructure/__init__.py`
- `apps/api/src/prescient/pipeline/infrastructure/tables/__init__.py`

Create `apps/api/src/prescient/pipeline/infrastructure/tables/deal.py`:

```python
"""`pipeline.deals` — sales pipeline deals."""

from __future__ import annotations

from datetime import date, datetime
from decimal import Decimal
from uuid import UUID

from sqlalchemy import Date, DateTime, Integer, Numeric, String, func
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class DealTable(Base):
    __tablename__ = "deals"
    __table_args__ = {"schema": "pipeline"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    company_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False, index=True)
    owner_tenant_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False, index=True)
    deal_name: Mapped[str] = mapped_column(String(256), nullable=False)
    customer_name: Mapped[str] = mapped_column(String(256), nullable=False)
    stage: Mapped[str] = mapped_column(String(32), nullable=False)
    amount: Mapped[Decimal] = mapped_column(Numeric(20, 4), nullable=False)
    currency: Mapped[str] = mapped_column(String(3), nullable=False, server_default="USD")
    probability: Mapped[Decimal] = mapped_column(Numeric(5, 2), nullable=False, server_default="0")
    expected_close_date: Mapped[date] = mapped_column(Date(), nullable=False)
    product_line: Mapped[str] = mapped_column(String(128), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
```

- [ ] **Step 3: Create budget table models**

Create empty `__init__.py` files for budget package:
- `apps/api/src/prescient/budget/__init__.py`
- `apps/api/src/prescient/budget/infrastructure/__init__.py`
- `apps/api/src/prescient/budget/infrastructure/tables/__init__.py`

Create `apps/api/src/prescient/budget/infrastructure/tables/budget_plan.py`:

```python
"""`budget.budget_plans` — quarterly/annual budget plans."""

from __future__ import annotations

from datetime import date, datetime
from uuid import UUID

from sqlalchemy import Date, DateTime, String, func
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class BudgetPlanTable(Base):
    __tablename__ = "budget_plans"
    __table_args__ = {"schema": "budget"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    company_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False, index=True)
    owner_tenant_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False, index=True)
    period_label: Mapped[str] = mapped_column(String(16), nullable=False)
    period_start: Mapped[date] = mapped_column(Date(), nullable=False)
    period_end: Mapped[date] = mapped_column(Date(), nullable=False)
    status: Mapped[str] = mapped_column(String(16), nullable=False, server_default="draft")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
```

Create `apps/api/src/prescient/budget/infrastructure/tables/budget_line_item.py`:

```python
"""`budget.budget_line_items` — individual line items within a budget plan."""

from __future__ import annotations

from decimal import Decimal
from uuid import UUID

from sqlalchemy import ForeignKey, Integer, Numeric, String
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class BudgetLineItemTable(Base):
    __tablename__ = "budget_line_items"
    __table_args__ = {"schema": "budget"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    budget_plan_id: Mapped[UUID] = mapped_column(
        PgUUID(as_uuid=True),
        ForeignKey("budget.budget_plans.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    category: Mapped[str] = mapped_column(String(32), nullable=False)
    subcategory: Mapped[str] = mapped_column(String(64), nullable=False)
    label: Mapped[str] = mapped_column(String(128), nullable=False)
    planned_amount: Mapped[Decimal] = mapped_column(Numeric(20, 4), nullable=False)
    actual_amount: Mapped[Decimal | None] = mapped_column(Numeric(20, 4), nullable=True)
    sort_order: Mapped[int] = mapped_column(Integer(), nullable=False, server_default="0")
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/alembic/versions/20260413_pipeline_budget_schema.py \
  apps/api/src/prescient/pipeline/ \
  apps/api/src/prescient/budget/
git commit -m "feat(financial): add pipeline and budget database schemas"
```

---

## Task 2: Pipeline & Budget CRUD API Endpoints

**Files:**
- Create: `apps/api/src/prescient/pipeline/api/__init__.py`
- Create: `apps/api/src/prescient/pipeline/api/routes.py`
- Create: `apps/api/src/prescient/budget/api/__init__.py`
- Create: `apps/api/src/prescient/budget/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`

- [ ] **Step 1: Create pipeline routes**

Create `apps/api/src/prescient/pipeline/api/__init__.py` (empty).

Create `apps/api/src/prescient/pipeline/api/routes.py`:

```python
"""Pipeline API routes — CRUD for sales deals."""

from __future__ import annotations

from datetime import UTC, datetime
from decimal import Decimal
from typing import Annotated
from uuid import UUID, uuid4

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import delete, select, update

from prescient.context import RequestContext, get_request_context
from prescient.pipeline.infrastructure.tables.deal import DealTable

router = APIRouter(prefix="/pipeline", tags=["pipeline"])
CtxDep = Annotated[RequestContext, Depends(get_request_context)]


class DealPayload(BaseModel):
    model_config = ConfigDict(frozen=True)
    id: str
    company_id: str
    deal_name: str
    customer_name: str
    stage: str
    amount: float
    currency: str
    probability: float
    expected_close_date: str
    product_line: str


class CreateDealRequest(BaseModel):
    company_id: str
    deal_name: str
    customer_name: str
    stage: str = Field(pattern=r"^(prospecting|qualification|proposal|negotiation|closed_won|closed_lost)$")
    amount: float
    currency: str = "USD"
    probability: float = Field(ge=0, le=100)
    expected_close_date: str
    product_line: str


class UpdateDealRequest(BaseModel):
    deal_name: str | None = None
    customer_name: str | None = None
    stage: str | None = None
    amount: float | None = None
    probability: float | None = None
    expected_close_date: str | None = None
    product_line: str | None = None


def _row_to_payload(row: DealTable) -> DealPayload:
    return DealPayload(
        id=str(row.id),
        company_id=str(row.company_id),
        deal_name=row.deal_name,
        customer_name=row.customer_name,
        stage=row.stage,
        amount=float(row.amount),
        currency=row.currency,
        probability=float(row.probability),
        expected_close_date=row.expected_close_date.isoformat(),
        product_line=row.product_line,
    )


@router.get("/deals", response_model=list[DealPayload])
async def list_deals(ctx: CtxDep, company_id: str | None = None) -> list[DealPayload]:
    stmt = select(DealTable).order_by(DealTable.expected_close_date)
    if company_id:
        stmt = stmt.where(DealTable.company_id == UUID(company_id))
    result = await ctx.session.execute(stmt)
    return [_row_to_payload(r) for r in result.scalars().all()]


@router.post("/deals", response_model=DealPayload)
async def create_deal(body: CreateDealRequest, ctx: CtxDep) -> DealPayload:
    from datetime import date as date_type

    row = DealTable(
        id=uuid4(),
        company_id=UUID(body.company_id),
        owner_tenant_id=ctx.user.fund_id,
        deal_name=body.deal_name,
        customer_name=body.customer_name,
        stage=body.stage,
        amount=Decimal(str(body.amount)),
        currency=body.currency,
        probability=Decimal(str(body.probability)),
        expected_close_date=date_type.fromisoformat(body.expected_close_date),
        product_line=body.product_line,
    )
    ctx.session.add(row)
    await ctx.session.commit()
    return _row_to_payload(row)


@router.delete("/deals/{deal_id}")
async def delete_deal(deal_id: UUID, ctx: CtxDep) -> dict:
    await ctx.session.execute(delete(DealTable).where(DealTable.id == deal_id))
    await ctx.session.commit()
    return {"deleted": str(deal_id)}
```

- [ ] **Step 2: Create budget routes**

Create `apps/api/src/prescient/budget/api/__init__.py` (empty).

Create `apps/api/src/prescient/budget/api/routes.py`:

```python
"""Budget API routes — CRUD for budget plans and line items."""

from __future__ import annotations

from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, ConfigDict
from sqlalchemy import select

from prescient.budget.infrastructure.tables.budget_line_item import BudgetLineItemTable
from prescient.budget.infrastructure.tables.budget_plan import BudgetPlanTable
from prescient.context import RequestContext, get_request_context

router = APIRouter(prefix="/budget", tags=["budget"])
CtxDep = Annotated[RequestContext, Depends(get_request_context)]


class LineItemPayload(BaseModel):
    model_config = ConfigDict(frozen=True)
    id: str
    category: str
    subcategory: str
    label: str
    planned_amount: float
    actual_amount: float | None
    variance: float | None
    variance_pct: float | None
    sort_order: int


class BudgetPlanPayload(BaseModel):
    model_config = ConfigDict(frozen=True)
    id: str
    company_id: str
    period_label: str
    period_start: str
    period_end: str
    status: str
    line_items: list[LineItemPayload]


def _build_line_item(row: BudgetLineItemTable) -> LineItemPayload:
    planned = float(row.planned_amount)
    actual = float(row.actual_amount) if row.actual_amount is not None else None
    variance = (actual - planned) if actual is not None else None
    variance_pct = (variance / planned * 100) if variance is not None and planned != 0 else None
    return LineItemPayload(
        id=str(row.id),
        category=row.category,
        subcategory=row.subcategory,
        label=row.label,
        planned_amount=planned,
        actual_amount=actual,
        variance=round(variance, 4) if variance is not None else None,
        variance_pct=round(variance_pct, 2) if variance_pct is not None else None,
        sort_order=row.sort_order,
    )


@router.get("/plans", response_model=list[BudgetPlanPayload])
async def list_budget_plans(ctx: CtxDep, company_id: str | None = None) -> list[BudgetPlanPayload]:
    stmt = select(BudgetPlanTable).order_by(BudgetPlanTable.period_start.desc())
    if company_id:
        stmt = stmt.where(BudgetPlanTable.company_id == UUID(company_id))
    result = await ctx.session.execute(stmt)
    plans = result.scalars().all()

    payloads = []
    for plan in plans:
        items_stmt = (
            select(BudgetLineItemTable)
            .where(BudgetLineItemTable.budget_plan_id == plan.id)
            .order_by(BudgetLineItemTable.sort_order)
        )
        items_result = await ctx.session.execute(items_stmt)
        items = [_build_line_item(r) for r in items_result.scalars().all()]
        payloads.append(
            BudgetPlanPayload(
                id=str(plan.id),
                company_id=str(plan.company_id),
                period_label=plan.period_label,
                period_start=plan.period_start.isoformat(),
                period_end=plan.period_end.isoformat(),
                status=plan.status,
                line_items=items,
            )
        )
    return payloads
```

- [ ] **Step 3: Register routers in main.py**

Add to the imports section of `apps/api/src/prescient/main.py`:

```python
from prescient.pipeline.api.routes import router as pipeline_router
from prescient.budget.api.routes import router as budget_router
```

Add after the existing `app.include_router(...)` calls:

```python
app.include_router(pipeline_router)
app.include_router(budget_router)
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/pipeline/api/ \
  apps/api/src/prescient/budget/api/ \
  apps/api/src/prescient/main.py
git commit -m "feat(financial): add pipeline and budget CRUD API endpoints"
```

---

## Task 3: Seed Data — Pipeline Deals & Budget

**Files:**
- Create: `seed/peloton/pipeline_deals.py`
- Create: `seed/peloton/budget.py`
- Modify: `seed/loader.py`

- [ ] **Step 1: Create pipeline seed data**

Create `seed/peloton/pipeline_deals.py`:

```python
"""Seed pipeline deals for Peloton demo."""

from __future__ import annotations

from datetime import date
from decimal import Decimal
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.pipeline.infrastructure.tables.deal import DealTable

_NS = UUID("d1e2f3a4-b5c6-7890-abcd-ef1234567890")


def _deal_id(name: str) -> UUID:
    return uuid5(_NS, name)


async def seed_pipeline_deals(session: AsyncSession, company_id: UUID, tenant_id: UUID) -> None:
    deals = [
        {
            "id": _deal_id("equinox-corporate-wellness"),
            "company_id": company_id,
            "owner_tenant_id": tenant_id,
            "deal_name": "Equinox Corporate Wellness Partnership",
            "customer_name": "Equinox Holdings",
            "stage": "prospecting",
            "amount": Decimal("150000"),
            "probability": Decimal("10"),
            "expected_close_date": date(2026, 7, 15),
            "product_line": "Corporate Wellness",
        },
        {
            "id": _deal_id("marriott-hotel-fitness"),
            "company_id": company_id,
            "owner_tenant_id": tenant_id,
            "deal_name": "Marriott Hotel Fitness Program",
            "customer_name": "Marriott International",
            "stage": "prospecting",
            "amount": Decimal("200000"),
            "probability": Decimal("15"),
            "expected_close_date": date(2026, 8, 1),
            "product_line": "Connected Fitness",
        },
        {
            "id": _deal_id("google-campus-wellness"),
            "company_id": company_id,
            "owner_tenant_id": tenant_id,
            "deal_name": "Google Campus Wellness Deployment",
            "customer_name": "Alphabet Inc.",
            "stage": "qualification",
            "amount": Decimal("450000"),
            "probability": Decimal("25"),
            "expected_close_date": date(2026, 6, 30),
            "product_line": "Connected Fitness",
        },
        {
            "id": _deal_id("united-health-subscription"),
            "company_id": company_id,
            "owner_tenant_id": tenant_id,
            "deal_name": "UnitedHealth Group Subscription Bundle",
            "customer_name": "UnitedHealth Group",
            "stage": "qualification",
            "amount": Decimal("380000"),
            "probability": Decimal("30"),
            "expected_close_date": date(2026, 6, 15),
            "product_line": "Subscription",
        },
        {
            "id": _deal_id("amazon-corporate-fleet"),
            "company_id": company_id,
            "owner_tenant_id": tenant_id,
            "deal_name": "Amazon Corporate Bike Fleet",
            "customer_name": "Amazon.com Inc.",
            "stage": "proposal",
            "amount": Decimal("850000"),
            "probability": Decimal("50"),
            "expected_close_date": date(2026, 5, 31),
            "product_line": "Connected Fitness",
        },
        {
            "id": _deal_id("deloitte-wellness-program"),
            "company_id": company_id,
            "owner_tenant_id": tenant_id,
            "deal_name": "Deloitte Employee Wellness Program",
            "customer_name": "Deloitte LLP",
            "stage": "proposal",
            "amount": Decimal("275000"),
            "probability": Decimal("45"),
            "expected_close_date": date(2026, 5, 15),
            "product_line": "Corporate Wellness",
        },
        {
            "id": _deal_id("jpmorgan-enterprise-sub"),
            "company_id": company_id,
            "owner_tenant_id": tenant_id,
            "deal_name": "JPMorgan Enterprise Subscription",
            "customer_name": "JPMorgan Chase & Co.",
            "stage": "negotiation",
            "amount": Decimal("1500000"),
            "probability": Decimal("70"),
            "expected_close_date": date(2026, 5, 1),
            "product_line": "Subscription",
        },
        {
            "id": _deal_id("kaiser-permanente-wellness"),
            "company_id": company_id,
            "owner_tenant_id": tenant_id,
            "deal_name": "Kaiser Permanente Wellness Integration",
            "customer_name": "Kaiser Permanente",
            "stage": "closed_won",
            "amount": Decimal("800000"),
            "probability": Decimal("100"),
            "expected_close_date": date(2026, 4, 10),
            "product_line": "Corporate Wellness",
        },
    ]

    stmt = insert(DealTable).values(deals)
    stmt = stmt.on_conflict_do_update(
        index_elements=["id"],
        set_={
            "deal_name": stmt.excluded.deal_name,
            "stage": stmt.excluded.stage,
            "amount": stmt.excluded.amount,
            "probability": stmt.excluded.probability,
            "expected_close_date": stmt.excluded.expected_close_date,
        },
    )
    await session.execute(stmt)
```

- [ ] **Step 2: Create budget seed data**

Create `seed/peloton/budget.py`:

```python
"""Seed budget plan and line items for Peloton demo."""

from __future__ import annotations

from datetime import date
from decimal import Decimal
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.budget.infrastructure.tables.budget_line_item import BudgetLineItemTable
from prescient.budget.infrastructure.tables.budget_plan import BudgetPlanTable

_NS = UUID("a1b2c3d4-e5f6-7890-abcd-budget123456")


def _plan_id(label: str) -> UUID:
    return uuid5(_NS, f"plan:{label}")


def _item_id(plan_label: str, subcategory: str) -> UUID:
    return uuid5(_NS, f"item:{plan_label}:{subcategory}")


async def seed_budget(session: AsyncSession, company_id: UUID, tenant_id: UUID) -> None:
    plan_label = "2026-Q2"
    plan_id = _plan_id(plan_label)

    plan_stmt = insert(BudgetPlanTable).values(
        id=plan_id,
        company_id=company_id,
        owner_tenant_id=tenant_id,
        period_label=plan_label,
        period_start=date(2026, 4, 1),
        period_end=date(2026, 6, 30),
        status="approved",
    )
    plan_stmt = plan_stmt.on_conflict_do_update(
        index_elements=["id"],
        set_={"status": plan_stmt.excluded.status},
    )
    await session.execute(plan_stmt)

    line_items = [
        # Revenue
        {"subcategory": "subscription_revenue", "category": "revenue", "label": "Subscription Revenue", "planned_amount": Decimal("441000000"), "actual_amount": Decimal("449800000"), "sort_order": 1},
        {"subcategory": "connected_fitness_revenue", "category": "revenue", "label": "Connected Fitness Revenue", "planned_amount": Decimal("240000000"), "actual_amount": Decimal("220800000"), "sort_order": 2},
        {"subcategory": "corporate_other_revenue", "category": "revenue", "label": "Corporate & Other Revenue", "planned_amount": Decimal("15000000"), "actual_amount": Decimal("13100000"), "sort_order": 3},
        # COGS
        {"subcategory": "hardware_cogs", "category": "cogs", "label": "Hardware COGS", "planned_amount": Decimal("180000000"), "actual_amount": Decimal("187200000"), "sort_order": 10},
        {"subcategory": "content_software", "category": "cogs", "label": "Content & Software", "planned_amount": Decimal("95000000"), "actual_amount": Decimal("93100000"), "sort_order": 11},
        {"subcategory": "logistics_fulfillment", "category": "cogs", "label": "Logistics & Fulfillment", "planned_amount": Decimal("35000000"), "actual_amount": Decimal("36400000"), "sort_order": 12},
        # Opex
        {"subcategory": "sales_marketing", "category": "opex", "label": "Sales & Marketing", "planned_amount": Decimal("120000000"), "actual_amount": Decimal("114000000"), "sort_order": 20},
        {"subcategory": "research_development", "category": "opex", "label": "Research & Development", "planned_amount": Decimal("85000000"), "actual_amount": Decimal("84200000"), "sort_order": 21},
        {"subcategory": "general_admin", "category": "opex", "label": "General & Administrative", "planned_amount": Decimal("65000000"), "actual_amount": Decimal("67600000"), "sort_order": 22},
    ]

    for item in line_items:
        stmt = insert(BudgetLineItemTable).values(
            id=_item_id(plan_label, item["subcategory"]),
            budget_plan_id=plan_id,
            **item,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=["id"],
            set_={
                "planned_amount": stmt.excluded.planned_amount,
                "actual_amount": stmt.excluded.actual_amount,
            },
        )
        await session.execute(stmt)
```

- [ ] **Step 3: Wire seed into loader.py**

In `seed/loader.py`, add imports near the top:

```python
from seed.peloton.pipeline_deals import seed_pipeline_deals
from seed.peloton.budget import seed_budget
```

Add after the existing KPI/artifact seeding calls (find the section that seeds per-company data and add these two calls after it):

```python
# Pipeline and budget seed data
await seed_pipeline_deals(session, company_id, tenant_id)
await seed_budget(session, company_id, tenant_id)
```

The `company_id` and `tenant_id` variables should already be available in the loader's company iteration loop.

- [ ] **Step 4: Commit**

```bash
git add seed/peloton/pipeline_deals.py seed/peloton/budget.py seed/loader.py
git commit -m "feat(financial): add pipeline deals and budget seed data"
```

---

## Task 4: Typed Slide Domain & API Models

**Files:**
- Modify: `apps/api/src/prescient/board_prep/domain/entities/board_deck.py`
- Modify: `apps/api/src/prescient/board_prep/api/routes.py`

- [ ] **Step 1: Add slide_type and data to domain entity**

In `apps/api/src/prescient/board_prep/domain/entities/board_deck.py`, update `BoardSection`:

```python
@dataclass
class BoardSection:
    """One slide of a board deck."""

    number: int
    title: str
    content: str  # Markdown with [[n]] citations (used by narrative slides)
    citations: list[dict] = field(default_factory=list)
    slide_type: str = "narrative"  # narrative, pl_summary, cash_flow, etc.
    data: dict | None = None  # Structured payload for data slides

    def to_dict(self) -> dict:
        result = {
            "number": self.number,
            "title": self.title,
            "content": self.content,
            "citations": self.citations,
            "slide_type": self.slide_type,
        }
        if self.data is not None:
            result["data"] = self.data
        return result

    @classmethod
    def from_dict(cls, data: dict) -> BoardSection:
        return cls(
            number=data["number"],
            title=data["title"],
            content=data.get("content", ""),
            citations=data.get("citations", []),
            slide_type=data.get("slide_type", "narrative"),
            data=data.get("data"),
        )
```

- [ ] **Step 2: Update API models**

In `apps/api/src/prescient/board_prep/api/routes.py`, update `SlidePayload`:

```python
class SlidePayload(BaseModel):
    model_config = ConfigDict(frozen=True)
    number: int
    title: str
    content: str = ""
    citations: list[dict] = Field(default_factory=list)
    slide_type: str = "narrative"
    data: dict | None = None
```

Update `_deck_to_response` to include the new fields:

```python
def _deck_to_response(deck) -> DeckResponse:
    return DeckResponse(
        id=str(deck.id),
        company_slug=deck.company_slug,
        quarter=deck.quarter,
        slides=[
            SlidePayload(
                number=s.number,
                title=s.title,
                content=s.content,
                citations=s.citations,
                slide_type=s.slide_type,
                data=s.data,
            )
            for s in deck.sections
        ],
        status=deck.status,
        generated_at=deck.generated_at.isoformat(),
    )
```

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/board_prep/domain/entities/board_deck.py \
  apps/api/src/prescient/board_prep/api/routes.py
git commit -m "feat(financial): add slide_type and data fields to slide models"
```

---

## Task 5: Data Slide Assembler

**Files:**
- Create: `apps/api/src/prescient/board_prep/application/data_slides.py`
- Modify: `apps/api/src/prescient/board_prep/application/assembler.py`

- [ ] **Step 1: Create data slide assembly functions**

Create `apps/api/src/prescient/board_prep/application/data_slides.py`:

```python
"""Data slide assemblers — build structured payloads for typed slides."""

from __future__ import annotations

from collections import defaultdict
from decimal import Decimal
from typing import Any
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.budget.infrastructure.tables.budget_line_item import BudgetLineItemTable
from prescient.budget.infrastructure.tables.budget_plan import BudgetPlanTable
from prescient.documents.infrastructure.tables.kpi_definition import KpiDefinitionTable
from prescient.documents.infrastructure.tables.kpi_value import KpiValueTable
from prescient.pipeline.infrastructure.tables.deal import DealTable


def _dec(v: Decimal | None) -> float | None:
    return float(v) if v is not None else None


async def build_pl_summary(
    session: AsyncSession,
    company_id: UUID,
    periods: list[str],
) -> dict:
    """Build P&L summary data for given periods.

    Returns: { periods: [...], rows: [{ label, values, is_subtotal, indent }] }
    """
    # Fetch all financial KPIs for the given periods
    stmt = (
        select(KpiDefinitionTable.label, KpiValueTable.period_label, KpiValueTable.value)
        .join(KpiDefinitionTable, KpiValueTable.kpi_id == KpiDefinitionTable.id)
        .where(
            KpiValueTable.company_id == company_id,
            KpiValueTable.period_label.in_(periods),
        )
        .order_by(KpiValueTable.period_start)
    )
    result = await session.execute(stmt)
    rows = result.all()

    # Group by KPI label -> {period: value}
    kpi_data: dict[str, dict[str, float]] = defaultdict(dict)
    for label, period, value in rows:
        kpi_data[label][period] = _dec(value)

    # Build P&L rows in standard order
    pl_rows = [
        {"label": "Revenue", "kpi": "Revenue", "is_subtotal": True, "indent": 0},
        {"label": "Connected Fitness Revenue", "kpi": "Connected Fitness Revenue", "is_subtotal": False, "indent": 1},
        {"label": "Subscription Revenue", "kpi": "Subscription Revenue", "is_subtotal": False, "indent": 1},
        {"label": "Gross Profit", "kpi": "Gross Profit", "is_subtotal": True, "indent": 0},
        {"label": "Gross Margin %", "kpi": "Gross Margin %", "is_subtotal": False, "indent": 1},
        {"label": "Operating Income", "kpi": "Operating Income", "is_subtotal": False, "indent": 0},
        {"label": "EBITDA", "kpi": "EBITDA", "is_subtotal": True, "indent": 0},
        {"label": "Net Income", "kpi": "Net Income", "is_subtotal": True, "indent": 0},
        {"label": "Free Cash Flow", "kpi": "Free Cash Flow", "is_subtotal": False, "indent": 0},
    ]

    output_rows = []
    for row in pl_rows:
        values = [kpi_data.get(row["kpi"], {}).get(p) for p in periods]
        if any(v is not None for v in values):
            output_rows.append({
                "label": row["label"],
                "values": values,
                "is_subtotal": row["is_subtotal"],
                "indent": row["indent"],
            })

    return {"periods": periods, "rows": output_rows}


async def build_segment_breakdown(
    session: AsyncSession,
    company_id: UUID,
    period: str,
) -> dict:
    """Build revenue and margin by segment for a period.

    Returns: { segments: [{ name, revenue, margin, pct_of_total }], period }
    """
    segment_kpis = {
        "Connected Fitness": {"revenue": "Connected Fitness Revenue", "margin": "Connected Fitness Gross Margin %"},
        "Subscription": {"revenue": "Subscription Revenue", "margin": "Subscription Gross Margin %"},
    }

    stmt = (
        select(KpiDefinitionTable.label, KpiValueTable.value)
        .join(KpiDefinitionTable, KpiValueTable.kpi_id == KpiDefinitionTable.id)
        .where(
            KpiValueTable.company_id == company_id,
            KpiValueTable.period_label == period,
        )
    )
    result = await session.execute(stmt)
    kpi_map = {label: _dec(value) for label, value in result.all()}

    segments = []
    total_revenue = 0
    for seg_name, kpis in segment_kpis.items():
        rev = kpi_map.get(kpis["revenue"])
        if rev is not None:
            total_revenue += rev
            segments.append({
                "name": seg_name,
                "revenue": rev,
                "margin": kpi_map.get(kpis["margin"]),
                "pct_of_total": 0,  # Computed below
            })

    for seg in segments:
        if total_revenue > 0:
            seg["pct_of_total"] = round(seg["revenue"] / total_revenue * 100, 1)

    return {"segments": segments, "period": period}


async def build_cash_flow(
    session: AsyncSession,
    company_id: UUID,
    periods: list[str],
) -> dict:
    """Build cash flow data for given periods."""
    cash_kpis = ["Operating Cash Flow", "Free Cash Flow"]

    stmt = (
        select(KpiDefinitionTable.label, KpiValueTable.period_label, KpiValueTable.value)
        .join(KpiDefinitionTable, KpiValueTable.kpi_id == KpiDefinitionTable.id)
        .where(
            KpiValueTable.company_id == company_id,
            KpiDefinitionTable.label.in_(cash_kpis),
            KpiValueTable.period_label.in_(periods),
        )
        .order_by(KpiValueTable.period_start)
    )
    result = await session.execute(stmt)

    kpi_data: dict[str, dict[str, float]] = defaultdict(dict)
    for label, period, value in result.all():
        kpi_data[label][period] = _dec(value)

    output_rows = []
    for kpi in cash_kpis:
        values = [kpi_data.get(kpi, {}).get(p) for p in periods]
        if any(v is not None for v in values):
            output_rows.append({
                "label": kpi,
                "values": values,
                "is_subtotal": kpi == "Free Cash Flow",
            })

    return {"periods": periods, "rows": output_rows}


async def build_kpi_trends(
    session: AsyncSession,
    company_id: UUID,
    periods: list[str],
) -> dict:
    """Build KPI trend data for charting.

    Returns: { metrics: [{ label, unit, data_points: [{ period, value }] }] }
    """
    trend_kpis = ["Revenue", "Subscribers", "Gross Margin %", "EBITDA", "Free Cash Flow"]

    stmt = (
        select(KpiDefinitionTable.label, KpiDefinitionTable.unit, KpiValueTable.period_label, KpiValueTable.value)
        .join(KpiDefinitionTable, KpiValueTable.kpi_id == KpiDefinitionTable.id)
        .where(
            KpiValueTable.company_id == company_id,
            KpiDefinitionTable.label.in_(trend_kpis),
            KpiValueTable.period_label.in_(periods),
        )
        .order_by(KpiDefinitionTable.label, KpiValueTable.period_start)
    )
    result = await session.execute(stmt)

    metrics: dict[str, dict] = {}
    for label, unit, period, value in result.all():
        if label not in metrics:
            metrics[label] = {"label": label, "unit": unit or "", "data_points": []}
        metrics[label]["data_points"].append({"period": period, "value": _dec(value)})

    return {"metrics": list(metrics.values())}


async def build_margin_analysis(
    session: AsyncSession,
    company_id: UUID,
    periods: list[str],
) -> dict:
    """Build margin analysis by product line.

    Returns: { product_lines: [{ name, periods: [{ period, gross_margin, op_margin }] }] }
    """
    margin_kpis = {
        "Overall": {"gross": "Gross Margin %", "op": "Operating Margin %"},
        "Connected Fitness": {"gross": "Connected Fitness Gross Margin %", "op": None},
        "Subscription": {"gross": "Subscription Gross Margin %", "op": None},
    }

    stmt = (
        select(KpiDefinitionTable.label, KpiValueTable.period_label, KpiValueTable.value)
        .join(KpiDefinitionTable, KpiValueTable.kpi_id == KpiDefinitionTable.id)
        .where(
            KpiValueTable.company_id == company_id,
            KpiValueTable.period_label.in_(periods),
        )
        .order_by(KpiValueTable.period_start)
    )
    result = await session.execute(stmt)

    kpi_data: dict[str, dict[str, float]] = defaultdict(dict)
    for label, period, value in result.all():
        kpi_data[label][period] = _dec(value)

    product_lines = []
    for name, kpis in margin_kpis.items():
        period_data = []
        for p in periods:
            gm = kpi_data.get(kpis["gross"], {}).get(p) if kpis["gross"] else None
            om = kpi_data.get(kpis["op"], {}).get(p) if kpis.get("op") else None
            period_data.append({"period": p, "gross_margin": gm, "op_margin": om})
        product_lines.append({"name": name, "periods": period_data})

    return {"product_lines": product_lines}


async def build_pipeline(
    session: AsyncSession,
    company_id: UUID,
) -> dict:
    """Build pipeline summary and deal list.

    Returns: { deals: [...], summary: { total_pipeline, weighted, by_stage } }
    """
    stmt = (
        select(DealTable)
        .where(DealTable.company_id == company_id)
        .order_by(DealTable.expected_close_date)
    )
    result = await session.execute(stmt)
    rows = result.scalars().all()

    deals = []
    total = 0.0
    weighted = 0.0
    by_stage: dict[str, dict] = {}

    for r in rows:
        amt = float(r.amount)
        prob = float(r.probability)
        deals.append({
            "name": r.deal_name,
            "customer": r.customer_name,
            "stage": r.stage,
            "amount": amt,
            "probability": prob,
            "expected_close": r.expected_close_date.isoformat(),
        })
        if r.stage not in ("closed_won", "closed_lost"):
            total += amt
            weighted += amt * prob / 100
            if r.stage not in by_stage:
                by_stage[r.stage] = {"count": 0, "amount": 0}
            by_stage[r.stage]["count"] += 1
            by_stage[r.stage]["amount"] += amt

    return {
        "deals": deals,
        "summary": {
            "total_pipeline": round(total, 2),
            "weighted": round(weighted, 2),
            "by_stage": by_stage,
        },
    }


async def build_budget_vs_actual(
    session: AsyncSession,
    company_id: UUID,
    period_label: str,
) -> dict:
    """Build budget vs. actual comparison.

    Returns: { period, rows: [{ label, category, planned, actual, variance, variance_pct }] }
    """
    plan_stmt = select(BudgetPlanTable).where(
        BudgetPlanTable.company_id == company_id,
        BudgetPlanTable.period_label == period_label,
    )
    plan_result = await session.execute(plan_stmt)
    plan = plan_result.scalar_one_or_none()

    if plan is None:
        return {"period": period_label, "rows": []}

    items_stmt = (
        select(BudgetLineItemTable)
        .where(BudgetLineItemTable.budget_plan_id == plan.id)
        .order_by(BudgetLineItemTable.sort_order)
    )
    items_result = await session.execute(items_stmt)

    rows = []
    for item in items_result.scalars().all():
        planned = float(item.planned_amount)
        actual = _dec(item.actual_amount)
        variance = (actual - planned) if actual is not None else None
        variance_pct = (variance / planned * 100) if variance is not None and planned != 0 else None
        rows.append({
            "label": item.label,
            "category": item.category,
            "planned": planned,
            "actual": actual,
            "variance": round(variance, 2) if variance is not None else None,
            "variance_pct": round(variance_pct, 2) if variance_pct is not None else None,
        })

    return {"period": period_label, "rows": rows}
```

- [ ] **Step 2: Update assembler to generate typed slides**

In `apps/api/src/prescient/board_prep/application/assembler.py`, add this import at the top:

```python
from prescient.board_prep.application.data_slides import (
    build_pl_summary,
    build_segment_breakdown,
    build_cash_flow,
    build_kpi_trends,
    build_margin_analysis,
    build_pipeline,
    build_budget_vs_actual,
)
```

Replace the `generate_deck` method with:

```python
async def generate_deck(
    self,
    company_slug: str,
    quarter: str,
    model: str,
) -> BoardDeck:
    """Generate a full board deck with typed slides."""
    # Resolve company
    result = await self._session.execute(
        select(CompanyTable.id, CompanyTable.name).where(CompanyTable.slug == company_slug)
    )
    row = result.one_or_none()
    if row is None:
        raise ValueError(f"Company with slug '{company_slug}' not found")
    company_id, company_name = row

    # Determine trailing periods for trends (8 quarters back)
    try:
        quarter_range = _quarter_to_date_range(quarter)
    except ValueError:
        quarter_range = None

    # Build period labels for P&L / trends (current + 3 prior quarters)
    periods = self._build_period_labels(quarter, count=4)
    trend_periods = self._build_period_labels(quarter, count=8)

    sections: list[BoardSection] = []

    # --- Data slides (no LLM) ---

    # Slide 2: P&L Summary
    pl_data = await build_pl_summary(self._session, company_id, periods)
    sections.append(BoardSection(
        number=2, title="P&L Summary", content="", slide_type="pl_summary", data=pl_data,
    ))

    # Slide 3: Revenue & Margins by Segment
    seg_data = await build_segment_breakdown(self._session, company_id, quarter)
    margin_data = await build_margin_analysis(self._session, company_id, periods)
    sections.append(BoardSection(
        number=3, title="Revenue & Margins by Segment", content="", slide_type="segment_breakdown",
        data={"segments": seg_data, "margins": margin_data},
    ))

    # Slide 4: Cash Flow
    cf_data = await build_cash_flow(self._session, company_id, periods)
    sections.append(BoardSection(
        number=4, title="Cash Flow", content="", slide_type="cash_flow", data=cf_data,
    ))

    # Slide 5: KPI Trends
    kpi_data = await build_kpi_trends(self._session, company_id, trend_periods)
    sections.append(BoardSection(
        number=5, title="KPI Trends", content="", slide_type="kpi_trends", data=kpi_data,
    ))

    # Slide 6: Pipeline & Go-Get
    pipeline_data = await build_pipeline(self._session, company_id)
    sections.append(BoardSection(
        number=6, title="Pipeline & Go-Get", content="", slide_type="pipeline", data=pipeline_data,
    ))

    # Slide 7: Budget vs. Actual
    budget_data = await build_budget_vs_actual(self._session, company_id, quarter)
    sections.append(BoardSection(
        number=7, title="Budget vs. Actual", content="", slide_type="budget_vs_actual", data=budget_data,
    ))

    # --- Narrative slides (LLM) ---

    # Sections 8-10: Competitive, Risk, People
    narrative_templates = [t for t in SECTION_TEMPLATES if t.number in (6, 7, 8)]
    for template in narrative_templates:
        logger.info("Generating section: %s", template.title)
        data_text, citations_map = await self.gather_section_data(template, company_slug, quarter)
        section = await self.generate_section(
            template, data_text, company_name, model, citations_map=citations_map
        )
        # Renumber: template 6->8, 7->9, 8->10
        section_number = template.number + 2
        sections.append(BoardSection(
            number=section_number,
            title=section.title,
            content=section.content,
            citations=section.citations,
            slide_type="narrative",
        ))

    # Sort by number
    sections.sort(key=lambda s: s.number)

    # Slide 1: Executive Summary (LLM using all other sections)
    all_sections_text = "\n\n---\n\n".join(
        f"## {s.title}\n{s.content}" if s.slide_type == "narrative"
        else f"## {s.title}\n[Data slide: {s.slide_type}]"
        for s in sections
    )
    exec_template = next(t for t in SECTION_TEMPLATES if t.number == 1)
    exec_section = await self.generate_section(
        exec_template, all_sections_text, company_name, model,
    )
    sections.insert(0, BoardSection(
        number=1,
        title=exec_section.title,
        content=exec_section.content,
        citations=exec_section.citations,
        slide_type="narrative",
    ))

    # Final renumber
    for i, s in enumerate(sections):
        s.number = i + 1

    return BoardDeck(
        id=uuid4(),
        company_slug=company_slug,
        quarter=quarter,
        sections=sections,
        generated_at=datetime.now(UTC),
        status="draft",
    )

def _build_period_labels(self, current_quarter: str, count: int = 4) -> list[str]:
    """Build a list of period labels going back `count` quarters from current."""
    import re
    m = re.match(r"(\d{4})[- ]Q([1-4])", current_quarter)
    if not m:
        return [current_quarter]
    year = int(m.group(1))
    q = int(m.group(2))

    labels = []
    for _ in range(count):
        labels.append(f"Q{q} FY{year}")
        q -= 1
        if q == 0:
            q = 4
            year -= 1
    labels.reverse()
    return labels
```

Note: The `_build_period_labels` method generates labels in the format used by the KPI seed data (e.g., "Q2 FY2026"). Check the actual seed data format in `seed/peloton/iq_next/kpi_values.yaml` and adjust the format string if needed.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/board_prep/application/data_slides.py \
  apps/api/src/prescient/board_prep/application/assembler.py
git commit -m "feat(financial): add data slide assembler with typed slide generation"
```

---

## Task 6: Frontend Slide Type & Renderer Registry

**Files:**
- Modify: `apps/web/src/stores/deck-store.ts`
- Create: `apps/web/src/components/board-prep/slide-renderer.tsx`
- Modify: `apps/web/src/components/board-prep/slide-canvas.tsx`

- [ ] **Step 1: Install recharts**

```bash
cd apps/web && npm install recharts
```

- [ ] **Step 2: Update Slide type in store**

In `apps/web/src/stores/deck-store.ts`, update the `Slide` interface:

```typescript
export interface Slide {
  number: number;
  title: string;
  content: string;
  citations: Array<{ marker?: number; source_type?: string; excerpt?: string }>;
  slide_type?: string;
  data?: Record<string, unknown>;
}
```

- [ ] **Step 3: Create slide renderer registry**

Create `apps/web/src/components/board-prep/slide-renderer.tsx`:

```tsx
"use client";

import type { Slide } from "@/stores/deck-store";
import { SlideContent } from "./slide-content";
import { PlSummarySlide } from "./slides/pl-summary-slide";
import { CashFlowSlide } from "./slides/cash-flow-slide";
import { SegmentBreakdownSlide } from "./slides/segment-breakdown-slide";
import { KpiTrendsSlide } from "./slides/kpi-trends-slide";
import { MarginAnalysisSlide } from "./slides/margin-analysis-slide";
import { PipelineSlide } from "./slides/pipeline-slide";
import { BudgetVsActualSlide } from "./slides/budget-vs-actual-slide";

interface SlideRendererProps {
  slide: Slide;
  onContentChange: (content: string) => void;
  onTextSelect?: (text: string, rect: DOMRect) => void;
}

export function SlideRenderer({ slide, onContentChange, onTextSelect }: SlideRendererProps) {
  const slideType = slide.slide_type ?? "narrative";

  switch (slideType) {
    case "pl_summary":
      return <PlSummarySlide data={slide.data} />;
    case "cash_flow":
      return <CashFlowSlide data={slide.data} />;
    case "segment_breakdown":
      return <SegmentBreakdownSlide data={slide.data} />;
    case "kpi_trends":
      return <KpiTrendsSlide data={slide.data} />;
    case "margin_analysis":
      return <MarginAnalysisSlide data={slide.data} />;
    case "pipeline":
      return <PipelineSlide data={slide.data} />;
    case "budget_vs_actual":
      return <BudgetVsActualSlide data={slide.data} />;
    default:
      return (
        <SlideContent
          content={slide.content}
          citations={slide.citations}
          onContentChange={onContentChange}
          onTextSelect={onTextSelect}
        />
      );
  }
}
```

- [ ] **Step 4: Update SlideCanvas to use SlideRenderer**

In `apps/web/src/components/board-prep/slide-canvas.tsx`, replace the `SlideContent` import and usage:

Replace:
```tsx
import { SlideContent } from "./slide-content";
```
With:
```tsx
import { SlideRenderer } from "./slide-renderer";
```

Replace the `<SlideContent ... />` usage in the return JSX with:
```tsx
<SlideRenderer
  slide={activeSlide}
  onContentChange={(content) => updateSlideContent(activeSlideIndex, content)}
  onTextSelect={handleTextSelect}
/>
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/stores/deck-store.ts \
  apps/web/src/components/board-prep/slide-renderer.tsx \
  apps/web/src/components/board-prep/slide-canvas.tsx \
  apps/web/package.json apps/web/package-lock.json
git commit -m "feat(financial): add slide renderer registry with recharts"
```

---

## Task 7: Financial Table Component

**Files:**
- Create: `apps/web/src/components/board-prep/slides/financial-table.tsx`

- [ ] **Step 1: Create shared financial table component**

Create directory: `apps/web/src/components/board-prep/slides/`

Create `apps/web/src/components/board-prep/slides/financial-table.tsx`:

```tsx
"use client";

interface TableRow {
  label: string;
  values: (number | null)[];
  is_subtotal?: boolean;
  indent?: number;
}

interface FinancialTableProps {
  periods: string[];
  rows: TableRow[];
  formatValue?: (value: number | null, label: string) => string;
}

function defaultFormat(value: number | null, label: string): string {
  if (value === null || value === undefined) return "—";
  if (label.includes("%") || label.includes("Margin")) {
    return `${value.toFixed(1)}%`;
  }
  const abs = Math.abs(value);
  if (abs >= 1_000_000_000) return `$${(value / 1_000_000_000).toFixed(1)}B`;
  if (abs >= 1_000_000) return `$${(value / 1_000_000).toFixed(1)}M`;
  if (abs >= 1_000) return `$${(value / 1_000).toFixed(1)}K`;
  return `$${value.toFixed(0)}`;
}

export function FinancialTable({ periods, rows, formatValue = defaultFormat }: FinancialTableProps) {
  return (
    <div className="overflow-x-auto p-6">
      <table className="w-full border-collapse text-sm">
        <thead>
          <tr className="border-b-2 border-indigo-200">
            <th className="py-2 pr-4 text-left font-semibold text-neutral-700"></th>
            {periods.map((p) => (
              <th key={p} className="px-3 py-2 text-right font-semibold text-neutral-700">
                {p}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {rows.map((row, i) => (
            <tr
              key={i}
              className={`border-b border-neutral-100 ${
                row.is_subtotal ? "bg-indigo-50 font-semibold" : ""
              }`}
            >
              <td
                className="py-2 pr-4 text-neutral-800"
                style={{ paddingLeft: `${(row.indent ?? 0) * 1.5 + 0.5}rem` }}
              >
                {row.label}
              </td>
              {row.values.map((v, j) => (
                <td
                  key={j}
                  className={`px-3 py-2 text-right tabular-nums ${
                    v !== null && v < 0 ? "text-red-600" : "text-neutral-800"
                  }`}
                >
                  {formatValue(v, row.label)}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/board-prep/slides/financial-table.tsx
git commit -m "feat(financial): add shared financial table component"
```

---

## Task 8: P&L Summary & Cash Flow Slide Components

**Files:**
- Create: `apps/web/src/components/board-prep/slides/pl-summary-slide.tsx`
- Create: `apps/web/src/components/board-prep/slides/cash-flow-slide.tsx`

- [ ] **Step 1: Create P&L summary slide**

Create `apps/web/src/components/board-prep/slides/pl-summary-slide.tsx`:

```tsx
"use client";

import { FinancialTable } from "./financial-table";

interface PlSummarySlideProps {
  data?: Record<string, unknown>;
}

export function PlSummarySlide({ data }: PlSummarySlideProps) {
  if (!data) return <div className="p-6 text-sm text-neutral-400">No P&L data available.</div>;

  const periods = (data.periods as string[]) ?? [];
  const rows = (data.rows as Array<{
    label: string;
    values: (number | null)[];
    is_subtotal?: boolean;
    indent?: number;
  }>) ?? [];

  return <FinancialTable periods={periods} rows={rows} />;
}
```

- [ ] **Step 2: Create cash flow slide**

Create `apps/web/src/components/board-prep/slides/cash-flow-slide.tsx`:

```tsx
"use client";

import { FinancialTable } from "./financial-table";

interface CashFlowSlideProps {
  data?: Record<string, unknown>;
}

export function CashFlowSlide({ data }: CashFlowSlideProps) {
  if (!data) return <div className="p-6 text-sm text-neutral-400">No cash flow data available.</div>;

  const periods = (data.periods as string[]) ?? [];
  const rows = (data.rows as Array<{
    label: string;
    values: (number | null)[];
    is_subtotal?: boolean;
  }>) ?? [];

  return <FinancialTable periods={periods} rows={rows} />;
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/board-prep/slides/pl-summary-slide.tsx \
  apps/web/src/components/board-prep/slides/cash-flow-slide.tsx
git commit -m "feat(financial): add P&L and cash flow slide components"
```

---

## Task 9: KPI Trends & Margin Analysis Slide Components

**Files:**
- Create: `apps/web/src/components/board-prep/slides/kpi-trends-slide.tsx`
- Create: `apps/web/src/components/board-prep/slides/margin-analysis-slide.tsx`

- [ ] **Step 1: Create KPI trends slide**

Create `apps/web/src/components/board-prep/slides/kpi-trends-slide.tsx`:

```tsx
"use client";

import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer,
} from "recharts";

const COLORS = ["#4f46e5", "#0891b2", "#059669", "#d97706", "#dc2626"];

interface KpiTrendsSlideProps {
  data?: Record<string, unknown>;
}

interface Metric {
  label: string;
  unit: string;
  data_points: Array<{ period: string; value: number | null }>;
}

export function KpiTrendsSlide({ data }: KpiTrendsSlideProps) {
  if (!data) return <div className="p-6 text-sm text-neutral-400">No KPI data available.</div>;

  const metrics = (data.metrics as Metric[]) ?? [];
  if (metrics.length === 0) return <div className="p-6 text-sm text-neutral-400">No KPI data available.</div>;

  // Group metrics into separate charts (different units shouldn't share an axis)
  const unitGroups: Record<string, Metric[]> = {};
  for (const m of metrics) {
    const unit = m.unit || "value";
    if (!unitGroups[unit]) unitGroups[unit] = [];
    unitGroups[unit]!.push(m);
  }

  return (
    <div className="space-y-6 p-6">
      {Object.entries(unitGroups).map(([unit, groupMetrics]) => {
        // Build chart data: merge all metrics by period
        const periodMap = new Map<string, Record<string, number | null>>();
        for (const m of groupMetrics) {
          for (const dp of m.data_points) {
            if (!periodMap.has(dp.period)) periodMap.set(dp.period, { period: dp.period } as Record<string, number | null>);
            periodMap.get(dp.period)![m.label] = dp.value;
          }
        }
        const chartData = Array.from(periodMap.values());

        return (
          <div key={unit}>
            <h4 className="mb-2 text-xs font-semibold uppercase tracking-wider text-neutral-500">
              {unit === "value" ? "Metrics" : unit}
            </h4>
            <ResponsiveContainer width="100%" height={250}>
              <LineChart data={chartData}>
                <CartesianGrid strokeDasharray="3 3" stroke="#e5e7eb" />
                <XAxis dataKey="period" tick={{ fontSize: 11 }} />
                <YAxis tick={{ fontSize: 11 }} />
                <Tooltip />
                <Legend />
                {groupMetrics.map((m, i) => (
                  <Line
                    key={m.label}
                    type="monotone"
                    dataKey={m.label}
                    stroke={COLORS[i % COLORS.length]}
                    strokeWidth={2}
                    dot={{ r: 3 }}
                  />
                ))}
              </LineChart>
            </ResponsiveContainer>
          </div>
        );
      })}
    </div>
  );
}
```

- [ ] **Step 2: Create margin analysis slide**

Create `apps/web/src/components/board-prep/slides/margin-analysis-slide.tsx`:

```tsx
"use client";

import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer,
} from "recharts";

const COLORS = ["#4f46e5", "#0891b2", "#059669"];

interface MarginAnalysisSlideProps {
  data?: Record<string, unknown>;
}

interface ProductLine {
  name: string;
  periods: Array<{ period: string; gross_margin: number | null; op_margin: number | null }>;
}

export function MarginAnalysisSlide({ data }: MarginAnalysisSlideProps) {
  if (!data) return <div className="p-6 text-sm text-neutral-400">No margin data available.</div>;

  const productLines = (data.product_lines as ProductLine[]) ?? [];
  if (productLines.length === 0) return <div className="p-6 text-sm text-neutral-400">No margin data available.</div>;

  // Build chart data: periods as x-axis, each product line's gross margin as a line
  const periods = productLines[0]?.periods.map((p) => p.period) ?? [];
  const chartData = periods.map((period) => {
    const point: Record<string, string | number | null> = { period };
    for (const pl of productLines) {
      const match = pl.periods.find((p) => p.period === period);
      point[`${pl.name} GM%`] = match?.gross_margin ?? null;
    }
    return point;
  });

  // Summary table: latest period
  const latestPeriod = periods[periods.length - 1];

  return (
    <div className="space-y-6 p-6">
      <ResponsiveContainer width="100%" height={280}>
        <LineChart data={chartData}>
          <CartesianGrid strokeDasharray="3 3" stroke="#e5e7eb" />
          <XAxis dataKey="period" tick={{ fontSize: 11 }} />
          <YAxis tick={{ fontSize: 11 }} tickFormatter={(v) => `${v}%`} />
          <Tooltip formatter={(value: number) => `${value?.toFixed(1)}%`} />
          <Legend />
          {productLines.map((pl, i) => (
            <Line
              key={pl.name}
              type="monotone"
              dataKey={`${pl.name} GM%`}
              stroke={COLORS[i % COLORS.length]}
              strokeWidth={2}
              dot={{ r: 3 }}
            />
          ))}
        </LineChart>
      </ResponsiveContainer>

      {/* Summary table */}
      <table className="w-full text-sm">
        <thead>
          <tr className="border-b-2 border-indigo-200">
            <th className="py-2 text-left font-semibold text-neutral-700">Segment</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Gross Margin</th>
          </tr>
        </thead>
        <tbody>
          {productLines.map((pl) => {
            const latest = pl.periods.find((p) => p.period === latestPeriod);
            return (
              <tr key={pl.name} className="border-b border-neutral-100">
                <td className="py-2 text-neutral-800">{pl.name}</td>
                <td className="px-3 py-2 text-right tabular-nums text-neutral-800">
                  {latest?.gross_margin != null ? `${latest.gross_margin.toFixed(1)}%` : "—"}
                </td>
              </tr>
            );
          })}
        </tbody>
      </table>
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/board-prep/slides/kpi-trends-slide.tsx \
  apps/web/src/components/board-prep/slides/margin-analysis-slide.tsx
git commit -m "feat(financial): add KPI trends and margin analysis chart slides"
```

---

## Task 10: Segment Breakdown Slide Component

**Files:**
- Create: `apps/web/src/components/board-prep/slides/segment-breakdown-slide.tsx`

- [ ] **Step 1: Create segment breakdown slide**

Create `apps/web/src/components/board-prep/slides/segment-breakdown-slide.tsx`:

```tsx
"use client";

import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  Cell,
} from "recharts";

const COLORS = ["#4f46e5", "#0891b2", "#059669", "#d97706"];

interface SegmentBreakdownSlideProps {
  data?: Record<string, unknown>;
}

interface Segment {
  name: string;
  revenue: number | null;
  margin: number | null;
  pct_of_total: number;
}

function formatRevenue(value: number | null): string {
  if (value === null) return "—";
  if (Math.abs(value) >= 1_000_000) return `$${(value / 1_000_000).toFixed(1)}M`;
  if (Math.abs(value) >= 1_000) return `$${(value / 1_000).toFixed(0)}K`;
  return `$${value.toFixed(0)}`;
}

export function SegmentBreakdownSlide({ data }: SegmentBreakdownSlideProps) {
  if (!data) return <div className="p-6 text-sm text-neutral-400">No segment data available.</div>;

  const segData = (data.segments as { segments: Segment[]; period: string }) ?? { segments: [], period: "" };
  const segments = segData.segments ?? [];
  const period = segData.period ?? "";

  if (segments.length === 0) return <div className="p-6 text-sm text-neutral-400">No segment data available.</div>;

  const chartData = segments.map((s) => ({
    name: s.name,
    revenue: s.revenue,
  }));

  return (
    <div className="space-y-6 p-6">
      <p className="text-xs text-neutral-500">Period: {period}</p>

      <ResponsiveContainer width="100%" height={220}>
        <BarChart data={chartData} layout="vertical">
          <CartesianGrid strokeDasharray="3 3" stroke="#e5e7eb" />
          <XAxis type="number" tick={{ fontSize: 11 }} tickFormatter={(v) => formatRevenue(v)} />
          <YAxis type="category" dataKey="name" tick={{ fontSize: 12 }} width={140} />
          <Tooltip formatter={(value: number) => formatRevenue(value)} />
          <Bar dataKey="revenue" radius={[0, 4, 4, 0]}>
            {chartData.map((_, i) => (
              <Cell key={i} fill={COLORS[i % COLORS.length]} />
            ))}
          </Bar>
        </BarChart>
      </ResponsiveContainer>

      {/* Summary table */}
      <table className="w-full text-sm">
        <thead>
          <tr className="border-b-2 border-indigo-200">
            <th className="py-2 text-left font-semibold text-neutral-700">Segment</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Revenue</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Margin</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">% of Total</th>
          </tr>
        </thead>
        <tbody>
          {segments.map((seg) => (
            <tr key={seg.name} className="border-b border-neutral-100">
              <td className="py-2 text-neutral-800">{seg.name}</td>
              <td className="px-3 py-2 text-right tabular-nums text-neutral-800">
                {formatRevenue(seg.revenue)}
              </td>
              <td className="px-3 py-2 text-right tabular-nums text-neutral-800">
                {seg.margin != null ? `${seg.margin.toFixed(1)}%` : "—"}
              </td>
              <td className="px-3 py-2 text-right tabular-nums text-neutral-800">
                {seg.pct_of_total.toFixed(1)}%
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/board-prep/slides/segment-breakdown-slide.tsx
git commit -m "feat(financial): add segment breakdown slide with bar chart"
```

---

## Task 11: Pipeline & Budget Slide Components

**Files:**
- Create: `apps/web/src/components/board-prep/slides/pipeline-slide.tsx`
- Create: `apps/web/src/components/board-prep/slides/budget-vs-actual-slide.tsx`

- [ ] **Step 1: Create pipeline slide**

Create `apps/web/src/components/board-prep/slides/pipeline-slide.tsx`:

```tsx
"use client";

interface PipelineSlideProps {
  data?: Record<string, unknown>;
}

interface Deal {
  name: string;
  customer: string;
  stage: string;
  amount: number;
  probability: number;
  expected_close: string;
}

interface Summary {
  total_pipeline: number;
  weighted: number;
  by_stage: Record<string, { count: number; amount: number }>;
}

const STAGE_COLORS: Record<string, string> = {
  prospecting: "bg-neutral-100 text-neutral-700",
  qualification: "bg-blue-100 text-blue-700",
  proposal: "bg-indigo-100 text-indigo-700",
  negotiation: "bg-amber-100 text-amber-700",
  closed_won: "bg-green-100 text-green-700",
  closed_lost: "bg-red-100 text-red-700",
};

function formatAmount(v: number): string {
  if (v >= 1_000_000) return `$${(v / 1_000_000).toFixed(1)}M`;
  if (v >= 1_000) return `$${(v / 1_000).toFixed(0)}K`;
  return `$${v.toFixed(0)}`;
}

function formatStage(s: string): string {
  return s.replace(/_/g, " ").replace(/\b\w/g, (c) => c.toUpperCase());
}

export function PipelineSlide({ data }: PipelineSlideProps) {
  if (!data) return <div className="p-6 text-sm text-neutral-400">No pipeline data available.</div>;

  const deals = (data.deals as Deal[]) ?? [];
  const summary = (data.summary as Summary) ?? { total_pipeline: 0, weighted: 0, by_stage: {} };

  return (
    <div className="space-y-6 p-6">
      {/* Summary cards */}
      <div className="grid grid-cols-3 gap-4">
        <div className="rounded-lg border border-neutral-200 bg-neutral-50 p-4 text-center">
          <p className="text-xs font-medium uppercase tracking-wider text-neutral-500">Total Pipeline</p>
          <p className="mt-1 text-2xl font-bold text-neutral-900">{formatAmount(summary.total_pipeline)}</p>
        </div>
        <div className="rounded-lg border border-neutral-200 bg-neutral-50 p-4 text-center">
          <p className="text-xs font-medium uppercase tracking-wider text-neutral-500">Weighted</p>
          <p className="mt-1 text-2xl font-bold text-indigo-600">{formatAmount(summary.weighted)}</p>
        </div>
        <div className="rounded-lg border border-neutral-200 bg-neutral-50 p-4 text-center">
          <p className="text-xs font-medium uppercase tracking-wider text-neutral-500">Active Deals</p>
          <p className="mt-1 text-2xl font-bold text-neutral-900">
            {deals.filter((d) => d.stage !== "closed_won" && d.stage !== "closed_lost").length}
          </p>
        </div>
      </div>

      {/* Deal table */}
      <table className="w-full text-sm">
        <thead>
          <tr className="border-b-2 border-indigo-200">
            <th className="py-2 text-left font-semibold text-neutral-700">Deal</th>
            <th className="px-3 py-2 text-left font-semibold text-neutral-700">Customer</th>
            <th className="px-3 py-2 text-left font-semibold text-neutral-700">Stage</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Amount</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Prob.</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Close Date</th>
          </tr>
        </thead>
        <tbody>
          {deals.map((deal, i) => (
            <tr key={i} className="border-b border-neutral-100">
              <td className="py-2 text-neutral-800">{deal.name}</td>
              <td className="px-3 py-2 text-neutral-600">{deal.customer}</td>
              <td className="px-3 py-2">
                <span className={`inline-block rounded-full px-2 py-0.5 text-xs font-medium ${STAGE_COLORS[deal.stage] ?? ""}`}>
                  {formatStage(deal.stage)}
                </span>
              </td>
              <td className="px-3 py-2 text-right tabular-nums text-neutral-800">{formatAmount(deal.amount)}</td>
              <td className="px-3 py-2 text-right tabular-nums text-neutral-800">{deal.probability}%</td>
              <td className="px-3 py-2 text-right tabular-nums text-neutral-600">{deal.expected_close}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

- [ ] **Step 2: Create budget vs. actual slide**

Create `apps/web/src/components/board-prep/slides/budget-vs-actual-slide.tsx`:

```tsx
"use client";

interface BudgetVsActualSlideProps {
  data?: Record<string, unknown>;
}

interface BudgetRow {
  label: string;
  category: string;
  planned: number;
  actual: number | null;
  variance: number | null;
  variance_pct: number | null;
}

function formatAmount(v: number | null): string {
  if (v === null) return "—";
  const abs = Math.abs(v);
  if (abs >= 1_000_000_000) return `$${(v / 1_000_000_000).toFixed(1)}B`;
  if (abs >= 1_000_000) return `$${(v / 1_000_000).toFixed(1)}M`;
  if (abs >= 1_000) return `$${(v / 1_000).toFixed(0)}K`;
  return `$${v.toFixed(0)}`;
}

function varianceColor(variance: number | null, category: string): string {
  if (variance === null) return "text-neutral-400";
  // For revenue: positive variance is good (green)
  // For costs (cogs, opex): negative variance is good (green = under budget)
  const isRevenue = category === "revenue";
  const isFavorable = isRevenue ? variance > 0 : variance < 0;
  return isFavorable ? "text-green-600" : "text-red-600";
}

const CATEGORY_LABELS: Record<string, string> = {
  revenue: "Revenue",
  cogs: "Cost of Goods Sold",
  opex: "Operating Expenses",
  capex: "Capital Expenditures",
};

export function BudgetVsActualSlide({ data }: BudgetVsActualSlideProps) {
  if (!data) return <div className="p-6 text-sm text-neutral-400">No budget data available.</div>;

  const period = (data.period as string) ?? "";
  const rows = (data.rows as BudgetRow[]) ?? [];

  if (rows.length === 0) return <div className="p-6 text-sm text-neutral-400">No budget data for {period}.</div>;

  // Group rows by category
  const categories = Array.from(new Set(rows.map((r) => r.category)));

  return (
    <div className="p-6">
      <p className="mb-4 text-xs text-neutral-500">Period: {period}</p>
      <table className="w-full text-sm">
        <thead>
          <tr className="border-b-2 border-indigo-200">
            <th className="py-2 text-left font-semibold text-neutral-700">Line Item</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Budget</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Actual</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Variance</th>
            <th className="px-3 py-2 text-right font-semibold text-neutral-700">Var %</th>
          </tr>
        </thead>
        <tbody>
          {categories.map((cat) => {
            const catRows = rows.filter((r) => r.category === cat);
            return (
              <tbody key={cat}>
                <tr className="bg-neutral-50">
                  <td colSpan={5} className="py-2 pl-2 text-xs font-bold uppercase tracking-wider text-neutral-500">
                    {CATEGORY_LABELS[cat] ?? cat}
                  </td>
                </tr>
                {catRows.map((row, i) => (
                  <tr key={i} className="border-b border-neutral-100">
                    <td className="py-2 pl-6 text-neutral-800">{row.label}</td>
                    <td className="px-3 py-2 text-right tabular-nums text-neutral-800">
                      {formatAmount(row.planned)}
                    </td>
                    <td className="px-3 py-2 text-right tabular-nums text-neutral-800">
                      {formatAmount(row.actual)}
                    </td>
                    <td className={`px-3 py-2 text-right tabular-nums ${varianceColor(row.variance, cat)}`}>
                      {row.variance !== null ? formatAmount(row.variance) : "—"}
                    </td>
                    <td className={`px-3 py-2 text-right tabular-nums ${varianceColor(row.variance, cat)}`}>
                      {row.variance_pct !== null ? `${row.variance_pct > 0 ? "+" : ""}${row.variance_pct.toFixed(1)}%` : "—"}
                    </td>
                  </tr>
                ))}
              </tbody>
            );
          })}
        </tbody>
      </table>
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/board-prep/slides/pipeline-slide.tsx \
  apps/web/src/components/board-prep/slides/budget-vs-actual-slide.tsx
git commit -m "feat(financial): add pipeline and budget vs actual slide components"
```

---

## Task 12: Browser Smoke Test

**Files:** None (testing only)

- [ ] **Step 1: Run database migrations and seed data**

```bash
make docker-migrate
make docker-seed
```

- [ ] **Step 2: Start Docker stack**

```bash
make up-all
```

- [ ] **Step 3: Navigate to Board Prep and generate a deck**

Go to `http://localhost:3000/board-prep`, select a quarter, click Generate Deck.

Verify:
1. Slides 2-7 render as data slides (tables, charts) — not narrative text
2. P&L slide shows a financial table with period columns and formatted values
3. Segment breakdown shows a horizontal bar chart + summary table
4. Cash flow shows a table
5. KPI trends shows line charts
6. Pipeline shows summary cards + deal table with stage badges
7. Budget shows table with green/red variance coloring
8. Slides 1, 8, 9, 10 render as narrative text (as before)
9. Filmstrip thumbnails still work for all slide types
10. AI panel context updates when switching between slide types

- [ ] **Step 4: Fix any issues found during testing**

- [ ] **Step 5: Commit fixes**

```bash
git add -A
git commit -m "fix(financial): address smoke test issues"
```
