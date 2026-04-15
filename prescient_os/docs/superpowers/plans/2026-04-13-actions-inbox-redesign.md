# Actions Inbox Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the flat actions table with an inbox-style UI that shows only the current user's items, prioritized for quick triage.

**Architecture:** Add `assigned_to` column to the backend, extend the list endpoint with `user_id` and `scope` params, then rewrite the frontend as a three-zone inbox (summary bar, action queue, resolved list). The existing table view is preserved behind `?scope=org`.

**Tech Stack:** Python/FastAPI/SQLAlchemy (backend), Next.js/React/Tailwind (frontend), Alembic (migrations)

---

## File Structure

### Backend (apps/api)

| Action | File | Responsibility |
|--------|------|----------------|
| Modify | `src/prescient/action_items/infrastructure/tables/action_item.py` | Add `assigned_to` column |
| Modify | `src/prescient/action_items/domain/entities/action_item.py` | Add `assigned_to` field |
| Modify | `src/prescient/action_items/application/use_cases/list_action_items.py` | Add `user_id` filter to port/command |
| Modify | `src/prescient/action_items/api/routes.py` | Add `user_id`/`scope` query params, update adapter, update response model |
| Modify | `src/prescient/action_items/application/use_cases/create_action_item.py` | Accept `assigned_to` in command |
| Create | `alembic/versions/20260413_add_assigned_to.py` | Migration for assigned_to column |
| Modify | `tests/unit/action_items/test_create_action_item.py` | Test assigned_to flows through |
| Create | `tests/unit/action_items/test_list_action_items.py` | Test user-scoped filtering |

### Frontend (apps/web)

| Action | File | Responsibility |
|--------|------|----------------|
| Create | `src/components/actions/summary-bar.tsx` | Stats row: needs action / overdue / resolved counts |
| Create | `src/components/actions/inbox-card.tsx` | Single action card with inline approve/reject/defer |
| Create | `src/components/actions/resolved-list.tsx` | Collapsed list of recently resolved items |
| Modify | `src/app/(main)/actions/page.tsx` | Rewrite as inbox layout, keep table behind `?scope=org` |

---

### Task 1: Add `assigned_to` column to ActionItemTable

**Files:**
- Modify: `apps/api/src/prescient/action_items/infrastructure/tables/action_item.py:59-64`
- Create: `apps/api/alembic/versions/20260413_add_assigned_to.py`

- [ ] **Step 1: Add column to ORM model**

In `apps/api/src/prescient/action_items/infrastructure/tables/action_item.py`, add after the `created_by` column (after line 55):

```python
    assigned_to: Mapped[str | None] = mapped_column(
        String(64),
        ForeignKey("tenants.users.id", ondelete="SET NULL"),
        nullable=True,
        index=True,
    )
```

- [ ] **Step 2: Create Alembic migration**

Create `apps/api/alembic/versions/20260413_add_assigned_to.py`:

```python
"""Add assigned_to column to action_items table.

Revision ID: 20260413_add_assigned_to
Revises: 20260413_research_engine
Create Date: 2026-04-13
"""

from alembic import op
import sqlalchemy as sa

revision = "20260413_add_assigned_to"
down_revision = "20260413_research_engine"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.add_column(
        "action_items",
        sa.Column(
            "assigned_to",
            sa.String(64),
            sa.ForeignKey("tenants.users.id", ondelete="SET NULL"),
            nullable=True,
        ),
        schema="action_items",
    )
    op.create_index(
        "ix_action_items_action_items_assigned_to",
        "action_items",
        ["assigned_to"],
        schema="action_items",
    )


def downgrade() -> None:
    op.drop_index(
        "ix_action_items_action_items_assigned_to",
        table_name="action_items",
        schema="action_items",
    )
    op.drop_column("action_items", "assigned_to", schema="action_items")
```

- [ ] **Step 3: Verify migration file is syntactically valid**

Run: `cd apps/api && python -c "import importlib.util; spec = importlib.util.spec_from_file_location('m', 'alembic/versions/20260413_add_assigned_to.py'); mod = importlib.util.module_from_spec(spec)"`

Expected: No import errors.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/action_items/infrastructure/tables/action_item.py apps/api/alembic/versions/20260413_add_assigned_to.py
git commit -m "feat(actions): add assigned_to column to action_items table"
```

---

### Task 2: Update domain entity and create use case

**Files:**
- Modify: `apps/api/src/prescient/action_items/domain/entities/action_item.py:38-41`
- Modify: `apps/api/src/prescient/action_items/application/use_cases/create_action_item.py:35-49,76-100`
- Modify: `apps/api/src/prescient/action_items/application/use_cases/list_action_items.py` (full rewrite of port/command)

- [ ] **Step 1: Add `assigned_to` to domain entity**

In `apps/api/src/prescient/action_items/domain/entities/action_item.py`, add after `created_by` (line 38) and before `created_at` (line 39):

```python
    assigned_to: UserId | None = None
```

- [ ] **Step 2: Add `assigned_to` to CreateActionItemCommand**

In `apps/api/src/prescient/action_items/application/use_cases/create_action_item.py`, add to the `CreateActionItemCommand` dataclass after `due_date` (line 47):

```python
    assigned_to: str | None = None
```

- [ ] **Step 3: Pass `assigned_to` through in CreateActionItem.execute**

In the same file, in the `execute` method, add `assigned_to=command.assigned_to,` to the `row_factory` call, after the `approved_at=None,` line (line 97):

The `row_factory(...)` call should now end with:
```python
            approved_by=None,
            approved_at=None,
            assigned_to=command.assigned_to,
        )
```

- [ ] **Step 4: Extend ListActionItems use case with user filtering**

Replace the full contents of `apps/api/src/prescient/action_items/application/use_cases/list_action_items.py` with:

```python
"""ListActionItems use case.

Lists action items for a tenant, with optional status and user filters.
The query port is Protocol-based so infrastructure stays out.
"""

from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol
from uuid import UUID

from prescient.action_items.domain.entities.action_item import ActionItem
from prescient.action_items.domain.enums import ActionItemStatus

# ---------------------------------------------------------------------------
# Port
# ---------------------------------------------------------------------------


class ListActionItemsQueryPort(Protocol):
    """Query port for listing action items."""

    async def list_for_tenant(
        self,
        owner_tenant_id: UUID,
        status: ActionItemStatus | None,
        user_id: str | None = None,
    ) -> list[ActionItem]: ...


# ---------------------------------------------------------------------------
# Command / Result
# ---------------------------------------------------------------------------


@dataclass(frozen=True)
class ListActionItemsCommand:
    owner_tenant_id: UUID
    status: ActionItemStatus | None = None
    user_id: str | None = None


# ---------------------------------------------------------------------------
# Use case
# ---------------------------------------------------------------------------


class ListActionItems:
    """Return action items for a tenant, optionally filtered by status and user."""

    def __init__(self, query: ListActionItemsQueryPort) -> None:
        self._query = query

    async def execute(self, command: ListActionItemsCommand) -> list[ActionItem]:
        return await self._query.list_for_tenant(
            owner_tenant_id=command.owner_tenant_id,
            status=command.status,
            user_id=command.user_id,
        )
```

- [ ] **Step 5: Run type check**

Run: `cd apps/api && python -m mypy src/prescient/action_items/application/use_cases/list_action_items.py src/prescient/action_items/application/use_cases/create_action_item.py src/prescient/action_items/domain/entities/action_item.py --ignore-missing-imports`

Expected: No errors.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/action_items/domain/entities/action_item.py apps/api/src/prescient/action_items/application/use_cases/create_action_item.py apps/api/src/prescient/action_items/application/use_cases/list_action_items.py
git commit -m "feat(actions): add assigned_to to domain model and user-scoped list query"
```

---

### Task 3: Update backend routes and adapter

**Files:**
- Modify: `apps/api/src/prescient/action_items/api/routes.py:77-94,134-159,166-202,263-312`

- [ ] **Step 1: Update `_ListQueryAdapter.list_for_tenant` to accept `user_id`**

In `apps/api/src/prescient/action_items/api/routes.py`, replace the `_ListQueryAdapter` class (lines 77-94) with:

```python
class _ListQueryAdapter:
    """Adapts AsyncSession to ListActionItemsQueryPort."""

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def list_for_tenant(
        self,
        owner_tenant_id: UUID,
        status: ActionItemStatus | None,
        user_id: str | None = None,
    ) -> list[ActionItem]:
        stmt = select(ActionItemTable).where(ActionItemTable.owner_tenant_id == owner_tenant_id)
        if status is not None:
            stmt = stmt.where(ActionItemTable.status == status.value)
        if user_id is not None:
            from sqlalchemy import or_

            stmt = stmt.where(
                or_(
                    ActionItemTable.created_by == user_id,
                    ActionItemTable.assigned_to == user_id,
                )
            )
        stmt = stmt.order_by(ActionItemTable.created_at.desc())
        result = await self._session.execute(stmt)
        rows = list(result.scalars().all())
        return [_row_to_action_item(r) for r in rows]
```

- [ ] **Step 2: Update `_row_to_action_item` to include `assigned_to`**

In the same file, update the `_row_to_action_item` function (lines 134-159). Add `assigned_to` after `created_by`:

```python
def _row_to_action_item(row: ActionItemTable) -> ActionItem:
    """Map an ActionItemTable ORM row to the ActionItem domain entity."""
    return ActionItem(
        id=ActionItemId(row.id),
        owner_tenant_id=FundId(row.owner_tenant_id),
        conversation_id=row.conversation_id,
        owner_name=row.owner_name,
        owner_role=row.owner_role,
        title=row.title,
        description=row.description,
        rationale=row.rationale,
        priority=ActionItemPriority(row.priority),
        priority_score=row.priority_score,
        priority_rationale=row.priority_rationale,
        linked_focus_priorities=tuple(row.linked_focus_priorities or []),
        status=ActionItemStatus(row.status),
        due_date=row.due_date,
        deferred_until=row.deferred_until,
        deferred_reason=row.deferred_reason,
        citations=tuple(row.citations or []),
        subjects=(),
        created_by=UserId(row.created_by),
        assigned_to=UserId(row.assigned_to) if row.assigned_to else None,
        created_at=row.created_at,
        approved_by=UserId(row.approved_by) if row.approved_by else None,
        approved_at=row.approved_at,
    )
```

- [ ] **Step 3: Add `assigned_to` to response model and create body**

In the same file, add `assigned_to` to `CreateActionItemBody` after `due_date` (line 179):

```python
    assigned_to: str | None = None
```

Add `assigned_to` to `ActionItemResponse` after `created_by` (line 198):

```python
    assigned_to: str | None
```

- [ ] **Step 4: Update `list_action_items` route with `user_id` and `scope` params**

Replace the `list_action_items` route (lines 263-312) with:

```python
@router.get("/items", response_model=list[ActionItemResponse])
async def list_action_items(
    organization_id: str,
    session: SessionDep,
    status: ActionItemStatus | None = None,
    user_id: str | None = None,
    scope: str = "mine",
) -> list[ActionItemResponse]:
    """List action items for an org.

    Accepts ``scope=mine`` (default) to return only items created by or
    assigned to ``user_id``, or ``scope=org`` for all items in the tenant.
    """
    # Resolve organization_id string -> org slug -> fund UUID
    org_slug = (
        await session.execute(
            select(OrganizationRow.slug).where(OrganizationRow.id == organization_id)
        )
    ).scalar_one_or_none()
    if org_slug is None:
        return []
    owner_tenant_id: UUID | None = (
        await session.execute(select(FundTable.id).where(FundTable.slug == org_slug))
    ).scalar_one_or_none()
    if owner_tenant_id is None:
        return []

    effective_user_id = user_id if scope == "mine" else None

    use_case = ListActionItems(query=_ListQueryAdapter(session))
    items = await use_case.execute(
        ListActionItemsCommand(
            owner_tenant_id=owner_tenant_id,
            status=status,
            user_id=effective_user_id,
        )
    )
    return [
        ActionItemResponse(
            item_id=item.id,
            owner_tenant_id=item.owner_tenant_id,
            owner_name=item.owner_name,
            owner_role=item.owner_role,
            title=item.title,
            description=item.description,
            rationale=item.rationale,
            priority=item.priority.value,
            priority_score=item.priority_score,
            status=item.status.value,
            due_date=item.due_date,
            created_by=item.created_by,
            assigned_to=item.assigned_to,
            created_at=item.created_at,
            approved_by=item.approved_by,
            approved_at=item.approved_at,
        )
        for item in items
    ]
```

- [ ] **Step 5: Update `create_action_item` route to pass `assigned_to`**

In the `create_action_item` route, add `assigned_to=body.assigned_to,` to the `CreateActionItemCommand(...)` call, after the `due_date=body.due_date,` line.

- [ ] **Step 6: Run type check**

Run: `cd apps/api && python -m mypy src/prescient/action_items/api/routes.py --ignore-missing-imports`

Expected: No errors.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/action_items/api/routes.py
git commit -m "feat(actions): add user_id/scope params to list endpoint, assigned_to to create"
```

---

### Task 4: Backend unit tests

**Files:**
- Modify: `apps/api/tests/unit/action_items/test_create_action_item.py`
- Create: `apps/api/tests/unit/action_items/test_list_action_items.py`

- [ ] **Step 1: Write test for assigned_to in create flow**

Add to `apps/api/tests/unit/action_items/test_create_action_item.py`, at the end of the `TestCreateActionItem` class:

```python
    async def test_row_assigned_to_matches_command(self) -> None:
        session = _FakeSession()
        cmd = _make_command(assigned_to="user-sarah-001")
        use_case = CreateActionItem(session=session, row_factory=_FakeRow)
        await use_case.execute(cmd)
        assert session.added[0].assigned_to == "user-sarah-001"

    async def test_row_assigned_to_defaults_to_none(self) -> None:
        session = _FakeSession()
        cmd = _make_command()
        use_case = CreateActionItem(session=session, row_factory=_FakeRow)
        await use_case.execute(cmd)
        assert session.added[0].assigned_to is None
```

- [ ] **Step 2: Run create tests to verify they pass**

Run: `cd apps/api && python -m pytest tests/unit/action_items/test_create_action_item.py -v`

Expected: All tests pass, including the two new ones.

- [ ] **Step 3: Write tests for user-scoped list filtering**

Create `apps/api/tests/unit/action_items/test_list_action_items.py`:

```python
"""Unit tests for ListActionItems use case."""

from __future__ import annotations

from datetime import date, datetime
from decimal import Decimal
from uuid import UUID, uuid4

import pytest

from prescient.action_items.application.use_cases.list_action_items import (
    ListActionItems,
    ListActionItemsCommand,
)
from prescient.action_items.domain.entities.action_item import ActionItem
from prescient.action_items.domain.enums import ActionItemPriority, ActionItemStatus
from prescient.shared.types import FundId, UserId

TENANT_ID = uuid4()
USER_A = "user-alice-001"
USER_B = "user-bob-002"
NOW = datetime(2026, 4, 13, 10, 0, 0)


def _make_item(
    *,
    created_by: str = USER_A,
    assigned_to: str | None = None,
    status: ActionItemStatus = ActionItemStatus.PENDING_REVIEW,
    title: str = "Test item",
) -> ActionItem:
    return ActionItem(
        id=uuid4(),
        owner_tenant_id=FundId(TENANT_ID),
        owner_name="Test Owner",
        title=title,
        description="Test description",
        priority=ActionItemPriority.MEDIUM,
        status=status,
        created_by=UserId(created_by),
        assigned_to=UserId(assigned_to) if assigned_to else None,
        created_at=NOW,
    )


class _FakeQuery:
    """In-memory query adapter for testing."""

    def __init__(self, items: list[ActionItem]) -> None:
        self._items = items

    async def list_for_tenant(
        self,
        owner_tenant_id: UUID,
        status: ActionItemStatus | None,
        user_id: str | None = None,
    ) -> list[ActionItem]:
        result = [i for i in self._items if i.owner_tenant_id == owner_tenant_id]
        if status is not None:
            result = [i for i in result if i.status == status]
        if user_id is not None:
            result = [
                i
                for i in result
                if i.created_by == user_id or i.assigned_to == user_id
            ]
        return result


class TestListActionItems:
    @pytest.fixture()
    def items(self) -> list[ActionItem]:
        return [
            _make_item(created_by=USER_A, title="Alice created"),
            _make_item(created_by=USER_B, assigned_to=USER_A, title="Assigned to Alice"),
            _make_item(created_by=USER_B, title="Bob only"),
        ]

    async def test_no_user_filter_returns_all(self, items: list[ActionItem]) -> None:
        query = _FakeQuery(items)
        use_case = ListActionItems(query=query)
        result = await use_case.execute(
            ListActionItemsCommand(owner_tenant_id=TENANT_ID)
        )
        assert len(result) == 3

    async def test_user_filter_returns_created_and_assigned(
        self, items: list[ActionItem]
    ) -> None:
        query = _FakeQuery(items)
        use_case = ListActionItems(query=query)
        result = await use_case.execute(
            ListActionItemsCommand(owner_tenant_id=TENANT_ID, user_id=USER_A)
        )
        assert len(result) == 2
        titles = {i.title for i in result}
        assert titles == {"Alice created", "Assigned to Alice"}

    async def test_user_filter_with_status(self, items: list[ActionItem]) -> None:
        query = _FakeQuery(items)
        use_case = ListActionItems(query=query)
        result = await use_case.execute(
            ListActionItemsCommand(
                owner_tenant_id=TENANT_ID,
                user_id=USER_A,
                status=ActionItemStatus.PENDING_REVIEW,
            )
        )
        assert len(result) == 2

    async def test_user_with_no_items_returns_empty(
        self, items: list[ActionItem]
    ) -> None:
        query = _FakeQuery(items)
        use_case = ListActionItems(query=query)
        result = await use_case.execute(
            ListActionItemsCommand(owner_tenant_id=TENANT_ID, user_id="user-nobody-999")
        )
        assert len(result) == 0
```

- [ ] **Step 4: Run list tests**

Run: `cd apps/api && python -m pytest tests/unit/action_items/test_list_action_items.py -v`

Expected: All 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/tests/unit/action_items/test_create_action_item.py apps/api/tests/unit/action_items/test_list_action_items.py
git commit -m "test(actions): add unit tests for assigned_to and user-scoped listing"
```

---

### Task 5: Create SummaryBar component

**Files:**
- Create: `apps/web/src/components/actions/summary-bar.tsx`

- [ ] **Step 1: Create the component**

Create `apps/web/src/components/actions/summary-bar.tsx`:

```tsx
"use client";

interface SummaryBarProps {
  needsAction: number;
  overdue: number;
  resolvedThisWeek: number;
  onViewAll: () => void;
}

export function SummaryBar({
  needsAction,
  overdue,
  resolvedThisWeek,
  onViewAll,
}: SummaryBarProps) {
  return (
    <div className="flex items-center justify-between border-b border-neutral-200 bg-white px-6 py-3">
      <div className="flex items-center gap-6 text-sm">
        <span>
          <span className="font-semibold text-neutral-900">{needsAction}</span>{" "}
          <span className="text-neutral-500">needs action</span>
        </span>
        {overdue > 0 && (
          <span>
            <span className="font-semibold text-red-600">{overdue}</span>{" "}
            <span className="text-red-500">overdue</span>
          </span>
        )}
        <span>
          <span className="font-semibold text-neutral-900">{resolvedThisWeek}</span>{" "}
          <span className="text-neutral-500">resolved this week</span>
        </span>
      </div>
      <button
        onClick={onViewAll}
        className="text-sm text-neutral-400 transition-colors hover:text-neutral-600"
      >
        View all org actions
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/actions/summary-bar.tsx
git commit -m "feat(actions): add SummaryBar component"
```

---

### Task 6: Create InboxCard component

**Files:**
- Create: `apps/web/src/components/actions/inbox-card.tsx`

- [ ] **Step 1: Create the component**

Create `apps/web/src/components/actions/inbox-card.tsx`:

```tsx
"use client";

import { useState } from "react";
import { Button } from "@/components/ui/button";

type Priority = "high" | "medium" | "low";

interface InboxCardProps {
  itemId: string;
  title: string;
  ownerName?: string;
  priority: Priority;
  dueDate?: string;
  onApprove: (itemId: string) => Promise<void>;
  onReject: (itemId: string) => Promise<void>;
  onDefer: (itemId: string) => Promise<void>;
  onLogDecision: (itemId: string) => void;
}

const BORDER_COLOR: Record<Priority, string> = {
  high: "border-l-red-500",
  medium: "border-l-amber-400",
  low: "border-l-green-500",
};

function isOverdue(dueDate: string): boolean {
  return new Date(dueDate) < new Date();
}

function formatDueDate(dueDate: string): string {
  return new Date(dueDate).toLocaleDateString("en-US", {
    month: "short",
    day: "numeric",
  });
}

export function InboxCard({
  itemId,
  title,
  ownerName,
  priority,
  dueDate,
  onApprove,
  onReject,
  onDefer,
  onLogDecision,
}: InboxCardProps) {
  const [acting, setActing] = useState<string | null>(null);
  const [dismissed, setDismissed] = useState(false);

  const overdue = dueDate ? isOverdue(dueDate) : false;

  async function handleAction(
    action: string,
    handler: (id: string) => Promise<void>,
  ) {
    setActing(action);
    try {
      await handler(itemId);
      setDismissed(true);
    } catch {
      // Reset on failure so user can retry
      setActing(null);
    }
  }

  return (
    <div
      className={`grid grid-cols-[1fr_auto] items-center gap-4 border-l-4 bg-white px-5 py-4 transition-all duration-300 ${BORDER_COLOR[priority]} ${
        overdue ? "bg-red-50/50" : ""
      } ${dismissed ? "pointer-events-none h-0 overflow-hidden opacity-0 py-0 px-0" : ""}`}
    >
      {/* Left: title + subtitle */}
      <div className="min-w-0">
        <p className="truncate font-medium text-neutral-900">{title}</p>
        <p className="mt-0.5 text-sm text-neutral-500">
          {ownerName && <span>{ownerName}</span>}
          {ownerName && dueDate && <span> · </span>}
          {dueDate && (
            <span className={overdue ? "font-medium text-red-600" : ""}>
              {overdue ? "Overdue · " : "Due "}
              {formatDueDate(dueDate)}
            </span>
          )}
        </p>
      </div>

      {/* Right: action buttons */}
      <div className="flex flex-col items-end gap-1">
        <div className="flex items-center gap-2">
          <Button
            size="sm"
            variant="outline"
            className="border-green-300 text-green-700 hover:bg-green-50"
            disabled={acting !== null}
            onClick={() => void handleAction("approve", onApprove)}
          >
            {acting === "approve" ? "..." : "Approve"}
          </Button>
          <Button
            size="sm"
            variant="outline"
            className="border-red-300 text-red-700 hover:bg-red-50"
            disabled={acting !== null}
            onClick={() => void handleAction("reject", onReject)}
          >
            {acting === "reject" ? "..." : "Reject"}
          </Button>
          <Button
            size="sm"
            variant="outline"
            disabled={acting !== null}
            onClick={() => void handleAction("defer", onDefer)}
          >
            {acting === "defer" ? "..." : "Defer"}
          </Button>
        </div>
        <button
          onClick={() => onLogDecision(itemId)}
          className="text-xs text-neutral-400 transition-colors hover:text-neutral-600"
        >
          Log decision
        </button>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/actions/inbox-card.tsx
git commit -m "feat(actions): add InboxCard component with dismiss animation"
```

---

### Task 7: Create ResolvedList component

**Files:**
- Create: `apps/web/src/components/actions/resolved-list.tsx`

- [ ] **Step 1: Create the component**

Create `apps/web/src/components/actions/resolved-list.tsx`:

```tsx
"use client";

import { useState } from "react";

interface ResolvedItem {
  itemId: string;
  title: string;
  status: string;
  approvedAt?: string;
}

const STATUS_LABELS: Record<string, string> = {
  completed: "Approved",
  rejected: "Rejected",
  deferred: "Deferred",
  sent_to_owner: "Sent to Owner",
};

interface ResolvedListProps {
  items: ResolvedItem[];
}

export function ResolvedList({ items }: ResolvedListProps) {
  const [expanded, setExpanded] = useState(false);

  if (items.length === 0) return null;

  return (
    <div className="border-t border-neutral-200 bg-neutral-50">
      <button
        onClick={() => setExpanded((e) => !e)}
        className="flex w-full items-center justify-between px-6 py-3 text-sm text-neutral-500 transition-colors hover:text-neutral-700"
      >
        <span>Resolved ({items.length})</span>
        <span className="text-xs">{expanded ? "Hide" : "Show"}</span>
      </button>
      {expanded && (
        <ul className="divide-y divide-neutral-200 px-6 pb-4">
          {items.map((item) => (
            <li
              key={item.itemId}
              className="flex items-center justify-between py-2 text-sm"
            >
              <span className="text-neutral-700">{item.title}</span>
              <div className="flex items-center gap-3">
                <span className="text-xs text-neutral-400">
                  {STATUS_LABELS[item.status] ?? item.status}
                </span>
                {item.approvedAt && (
                  <span className="text-xs text-neutral-400">
                    {new Date(item.approvedAt).toLocaleDateString("en-US", {
                      month: "short",
                      day: "numeric",
                    })}
                  </span>
                )}
              </div>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/actions/resolved-list.tsx
git commit -m "feat(actions): add ResolvedList component"
```

---

### Task 8: Rewrite actions page as inbox

**Files:**
- Modify: `apps/web/src/app/(main)/actions/page.tsx` (full rewrite)

- [ ] **Step 1: Rewrite the page**

Replace the full contents of `apps/web/src/app/(main)/actions/page.tsx` with:

```tsx
"use client";

import { useEffect, useState, useCallback } from "react";
import { useSearchParams } from "next/navigation";

import { getOrgId, getUserId } from "@/lib/auth";
import { apiFetch } from "@/lib/api-client";
import { SummaryBar } from "@/components/actions/summary-bar";
import { InboxCard } from "@/components/actions/inbox-card";
import { ResolvedList } from "@/components/actions/resolved-list";

/* -------------------------------------------------------------------------- */
/*  Shared types                                                              */
/* -------------------------------------------------------------------------- */

type Priority = "high" | "medium" | "low";
type ActionStatus =
  | "pending_review"
  | "sent_to_owner"
  | "completed"
  | "rejected"
  | "deferred";

interface ActionItem {
  item_id: string;
  title: string;
  owner_name?: string;
  owner_role?: string;
  priority: Priority;
  status: ActionStatus;
  due_date?: string;
  assigned_to?: string;
  created_by?: string;
  approved_at?: string;
}

/* -------------------------------------------------------------------------- */
/*  Helpers                                                                   */
/* -------------------------------------------------------------------------- */

const PENDING_STATUSES = new Set<ActionStatus>(["pending_review"]);
const RESOLVED_STATUSES = new Set<ActionStatus>([
  "completed",
  "rejected",
  "deferred",
  "sent_to_owner",
]);

const PRIORITY_ORDER: Record<string, number> = { high: 0, medium: 1, low: 2 };

function sortQueue(items: ActionItem[]): ActionItem[] {
  return [...items].sort((a, b) => {
    // Overdue first
    const now = Date.now();
    const aOverdue = a.due_date ? new Date(a.due_date).getTime() < now : false;
    const bOverdue = b.due_date ? new Date(b.due_date).getTime() < now : false;
    if (aOverdue !== bOverdue) return aOverdue ? -1 : 1;

    // Then by priority
    const aPri = PRIORITY_ORDER[a.priority] ?? 9;
    const bPri = PRIORITY_ORDER[b.priority] ?? 9;
    if (aPri !== bPri) return aPri - bPri;

    // Then by due date ascending
    const aDue = a.due_date ? new Date(a.due_date).getTime() : Infinity;
    const bDue = b.due_date ? new Date(b.due_date).getTime() : Infinity;
    return aDue - bDue;
  });
}

function getMondayOfWeek(): Date {
  const now = new Date();
  const day = now.getDay();
  const diff = day === 0 ? 6 : day - 1; // Monday = 0
  const monday = new Date(now);
  monday.setDate(now.getDate() - diff);
  monday.setHours(0, 0, 0, 0);
  return monday;
}

function isResolvedThisWeek(item: ActionItem): boolean {
  if (!item.approved_at) return false;
  return new Date(item.approved_at) >= getMondayOfWeek();
}

function isResolvedRecently(item: ActionItem): boolean {
  if (!item.approved_at) return false;
  const sevenDaysAgo = new Date();
  sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);
  sevenDaysAgo.setHours(0, 0, 0, 0);
  return new Date(item.approved_at) >= sevenDaysAgo;
}

/* -------------------------------------------------------------------------- */
/*  Legacy table (org-wide view)                                              */
/* -------------------------------------------------------------------------- */

import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

const PRIORITY_STYLES: Record<Priority, string> = {
  high: "bg-red-50 text-red-700 border border-red-200",
  medium: "bg-amber-50 text-amber-700 border border-amber-200",
  low: "bg-green-50 text-green-700 border border-green-200",
};

const STATUS_STYLES: Record<ActionStatus, string> = {
  pending_review: "bg-indigo-50 text-indigo-700 border border-indigo-200",
  sent_to_owner: "bg-blue-50 text-blue-700 border border-blue-200",
  completed: "bg-green-50 text-green-700 border border-green-200",
  rejected: "bg-red-50 text-red-700 border border-red-200",
  deferred: "bg-neutral-100 text-neutral-600 border border-neutral-200",
};

const STATUS_LABELS: Record<ActionStatus, string> = {
  pending_review: "Pending Review",
  sent_to_owner: "Sent to Owner",
  completed: "Completed",
  rejected: "Rejected",
  deferred: "Deferred",
};

function Pill({ label, className }: { label: string; className: string }) {
  return (
    <span
      className={`inline-flex items-center rounded-md px-2 py-0.5 text-xs font-semibold capitalize ${className}`}
    >
      {label}
    </span>
  );
}

function OrgTableView({ items }: { items: ActionItem[] }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle className="text-base">All Org Actions ({items.length})</CardTitle>
      </CardHeader>
      <CardContent className="p-0">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead className="pl-6">Title</TableHead>
              <TableHead>Owner</TableHead>
              <TableHead>Priority</TableHead>
              <TableHead>Status</TableHead>
              <TableHead>Due</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {items.map((item) => (
              <TableRow key={item.item_id}>
                <TableCell className="pl-6 font-medium">{item.title}</TableCell>
                <TableCell className="text-neutral-600">
                  {item.owner_name ?? "\u2014"}
                </TableCell>
                <TableCell>
                  <Pill
                    label={item.priority}
                    className={PRIORITY_STYLES[item.priority] ?? PRIORITY_STYLES.medium}
                  />
                </TableCell>
                <TableCell>
                  <Pill
                    label={STATUS_LABELS[item.status] ?? item.status}
                    className={STATUS_STYLES[item.status] ?? STATUS_STYLES.deferred}
                  />
                </TableCell>
                <TableCell>
                  {item.due_date ? (
                    <span
                      className={
                        new Date(item.due_date) < new Date()
                          ? "font-medium text-red-600"
                          : "text-neutral-500"
                      }
                    >
                      {new Date(item.due_date).toLocaleDateString("en-US", {
                        month: "short",
                        day: "numeric",
                      })}
                    </span>
                  ) : (
                    <span className="text-neutral-400">{"\u2014"}</span>
                  )}
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </CardContent>
    </Card>
  );
}

/* -------------------------------------------------------------------------- */
/*  Page component                                                            */
/* -------------------------------------------------------------------------- */

export default function ActionsPage() {
  const searchParams = useSearchParams();
  const isOrgView = searchParams.get("scope") === "org";

  const [items, setItems] = useState<ActionItem[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchItems = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const orgId = getOrgId() ?? "";
      const userId = getUserId() ?? "";
      const params = new URLSearchParams({ organization_id: orgId });
      if (isOrgView) {
        params.set("scope", "org");
      } else {
        params.set("scope", "mine");
        params.set("user_id", userId);
      }
      const data = await apiFetch<ActionItem[] | { items?: ActionItem[] }>(
        `/actions/items?${params.toString()}`,
      );
      const list = Array.isArray(data) ? data : (data.items ?? []);
      setItems(list);
    } catch (err) {
      if (err instanceof Error && err.message !== "Unauthorized") {
        setError(err.message);
      }
    } finally {
      setLoading(false);
    }
  }, [isOrgView]);

  useEffect(() => {
    void fetchItems();
  }, [fetchItems]);

  async function handleAction(itemId: string, action: string) {
    await apiFetch<unknown>(`/actions/items/${itemId}`, {
      method: "PATCH",
      body: JSON.stringify({ action }),
    });
    setItems((prev) =>
      prev.map((item) =>
        item.item_id === itemId
          ? {
              ...item,
              status: (
                action === "approve"
                  ? "sent_to_owner"
                  : action === "reject"
                    ? "rejected"
                    : "deferred"
              ) as ActionStatus,
              approved_at: new Date().toISOString(),
            }
          : item,
      ),
    );
  }

  // Derive queue and resolved lists
  const queue = sortQueue(items.filter((i) => PENDING_STATUSES.has(i.status)));
  const resolved = items.filter(
    (i) => RESOLVED_STATUSES.has(i.status) && isResolvedRecently(i),
  );
  const overdue = queue.filter(
    (i) => i.due_date && new Date(i.due_date) < new Date(),
  ).length;
  const resolvedThisWeek = resolved.filter(isResolvedThisWeek).length;

  function navigateToOrg() {
    window.location.href = "/actions?scope=org";
  }

  if (loading) {
    return (
      <div className="space-y-6">
        <div>
          <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">
            Actions
          </h1>
        </div>
        <div className="space-y-2">
          {[1, 2, 3].map((i) => (
            <div
              key={i}
              className="h-16 animate-pulse rounded-lg border border-neutral-200 bg-neutral-50"
            />
          ))}
        </div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="space-y-6">
        <div>
          <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">
            Actions
          </h1>
        </div>
        <div className="rounded-lg border border-red-200 bg-red-50 px-4 py-3 text-sm text-red-700">
          {error}
        </div>
      </div>
    );
  }

  /* Org-wide table view */
  if (isOrgView) {
    return (
      <div className="space-y-6">
        <div className="flex items-center justify-between">
          <div>
            <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">
              All Org Actions
            </h1>
            <p className="mt-1 text-sm text-neutral-500">
              Showing all action items across the organization.
            </p>
          </div>
          <a
            href="/actions"
            className="text-sm text-neutral-400 transition-colors hover:text-neutral-600"
          >
            Back to inbox
          </a>
        </div>
        {items.length === 0 ? (
          <div className="py-12 text-center text-sm text-neutral-400">
            No action items found.
          </div>
        ) : (
          <OrgTableView items={items} />
        )}
      </div>
    );
  }

  /* Inbox view */
  return (
    <div className="space-y-0">
      <div className="px-6 pt-6 pb-4">
        <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">
          Actions
        </h1>
        <p className="mt-1 text-sm text-neutral-500">
          Review and act on items that need your attention.
        </p>
      </div>

      <SummaryBar
        needsAction={queue.length}
        overdue={overdue}
        resolvedThisWeek={resolvedThisWeek}
        onViewAll={navigateToOrg}
      />

      {/* Queue */}
      {queue.length === 0 ? (
        <div className="flex flex-col items-center justify-center py-20">
          <p className="text-sm font-medium text-neutral-900">
            You're all caught up
          </p>
          <p className="mt-1 text-sm text-neutral-400">
            No action items need your attention right now.
          </p>
        </div>
      ) : (
        <div className="divide-y divide-neutral-100">
          {queue.map((item) => (
            <InboxCard
              key={item.item_id}
              itemId={item.item_id}
              title={item.title}
              ownerName={item.owner_name}
              priority={item.priority}
              dueDate={item.due_date}
              onApprove={(id) => handleAction(id, "approve")}
              onReject={(id) => handleAction(id, "reject")}
              onDefer={(id) => handleAction(id, "defer")}
              onLogDecision={() => {
                /* TODO: wire to existing LogDecisionDialog */
              }}
            />
          ))}
        </div>
      )}

      {/* Resolved */}
      <ResolvedList
        items={resolved.map((i) => ({
          itemId: i.item_id,
          title: i.title,
          status: i.status,
          approvedAt: i.approved_at,
        }))}
      />
    </div>
  );
}
```

- [ ] **Step 2: Run TypeScript type check**

Run: `cd apps/web && npx tsc --noEmit`

Expected: No new errors.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/\(main\)/actions/page.tsx
git commit -m "feat(actions): rewrite actions page as inbox with org table fallback"
```

---

### Task 9: Wire LogDecisionDialog into inbox cards

**Files:**
- Modify: `apps/web/src/app/(main)/actions/page.tsx`

- [ ] **Step 1: Add LogDecisionDialog state to page**

In `apps/web/src/app/(main)/actions/page.tsx`, add an import for the existing `LogDecisionDialog` at the top of the file. Since the `LogDecisionDialog` is currently defined inline in the old page and not extracted, we need to create a minimal version. Add this import and state after the existing imports:

Add a state variable in the `ActionsPage` component, after the existing `useState` calls:

```tsx
  const [decisionItemId, setDecisionItemId] = useState<string | null>(null);
```

- [ ] **Step 2: Replace the onLogDecision TODO**

Replace the `onLogDecision` callback in the InboxCard rendering:

```tsx
              onLogDecision={(id) => setDecisionItemId(id)}
```

- [ ] **Step 3: Add a simple decision dialog at the bottom of the inbox view**

Add these imports at the top:

```tsx
import { Input } from "@/components/ui/input";
import {
  Dialog,
  DialogContent,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
```

Then, just before the closing `</div>` of the inbox view return, add:

```tsx
      {/* Decision dialog */}
      <Dialog
        open={decisionItemId !== null}
        onOpenChange={(open) => {
          if (!open) setDecisionItemId(null);
        }}
      >
        <DialogContent>
          <DialogHeader>
            <DialogTitle>Log a Decision</DialogTitle>
          </DialogHeader>
          <DecisionForm
            actionItemId={decisionItemId ?? ""}
            onDone={() => setDecisionItemId(null)}
          />
        </DialogContent>
      </Dialog>
```

Add the `DecisionForm` component above the `ActionsPage` export:

```tsx
function DecisionForm({
  actionItemId,
  onDone,
}: {
  actionItemId: string;
  onDone: () => void;
}) {
  const [title, setTitle] = useState("");
  const [rationale, setRationale] = useState("");
  const [madeBy, setMadeBy] = useState("");
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!title.trim()) return;
    setSubmitting(true);
    setError(null);
    try {
      const orgId = getOrgId() ?? "";
      await apiFetch<unknown>("/actions/decisions", {
        method: "POST",
        body: JSON.stringify({
          action_item_id: actionItemId,
          title,
          rationale,
          made_by: madeBy,
          organization_id: orgId,
        }),
      });
      onDone();
    } catch (err) {
      if (err instanceof Error && err.message !== "Unauthorized") {
        setError(err.message);
      }
    } finally {
      setSubmitting(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div className="flex flex-col gap-1.5">
        <label htmlFor="dec-title" className="text-sm font-medium text-neutral-700">
          Decision Title <span className="text-red-500">*</span>
        </label>
        <Input
          id="dec-title"
          placeholder="e.g. Approved Q2 budget increase"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          required
        />
      </div>
      <div className="flex flex-col gap-1.5">
        <label htmlFor="dec-rationale" className="text-sm font-medium text-neutral-700">
          Rationale
        </label>
        <Input
          id="dec-rationale"
          placeholder="Why was this decision made?"
          value={rationale}
          onChange={(e) => setRationale(e.target.value)}
        />
      </div>
      <div className="flex flex-col gap-1.5">
        <label htmlFor="dec-made-by" className="text-sm font-medium text-neutral-700">
          Made By
        </label>
        <Input
          id="dec-made-by"
          placeholder="e.g. Sarah Chen"
          value={madeBy}
          onChange={(e) => setMadeBy(e.target.value)}
        />
      </div>
      {error && <p className="text-sm text-red-600">{error}</p>}
      <DialogFooter>
        <Button type="submit" disabled={submitting || !title.trim()}>
          {submitting ? "Saving..." : "Save Decision"}
        </Button>
      </DialogFooter>
    </form>
  );
}
```

- [ ] **Step 4: Run TypeScript type check**

Run: `cd apps/web && npx tsc --noEmit`

Expected: No new errors.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/app/\(main\)/actions/page.tsx
git commit -m "feat(actions): wire LogDecision dialog into inbox cards"
```

---

### Task 10: Manual browser testing

**Files:** None (testing only)

- [ ] **Step 1: Start the dev servers**

Run: `docker compose up` (or however the dev environment starts per project conventions)

- [ ] **Step 2: Navigate to /actions**

Verify:
- Summary bar shows at top with counts
- Inbox cards display for pending items (if any exist for current user)
- "You're all caught up" shows if no pending items
- Resolved section is collapsed at bottom

- [ ] **Step 3: Test card actions**

For a pending_review item:
- Click Approve — card should fade out, summary count should decrease
- Click Reject on another — same fade behavior
- Click Defer — same

- [ ] **Step 4: Test "View all org actions" link**

Click the link — should navigate to `/actions?scope=org` and show the table view with all org items. "Back to inbox" link should return to inbox view.

- [ ] **Step 5: Test "Log Decision" on a card**

Click "Log decision" text — dialog should open. Fill in title, submit. Dialog closes.

- [ ] **Step 6: Test empty state**

If all items are resolved, verify "You're all caught up" message displays.
