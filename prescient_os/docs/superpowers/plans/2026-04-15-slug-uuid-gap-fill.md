# Slug-to-UUID Gap Fill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the 6 remaining slug-as-identifier gaps found in the Phase 1 audit, so that all routing, access control, and company resolution use UUIDs.

**Architecture:** The codebase is already ~95% UUID-migrated. These gaps are isolated fixes: add `subject_company_id` to a backend response model, replace 2 frontend slug-matching heuristics with `primaryCompanyId` from auth, convert sponsor access checks to UUID comparison, add `company_id` to sponsor response models, and mark dead slug-keyed config.

**Tech Stack:** Python/FastAPI (backend), TypeScript/Next.js (frontend), pytest, Pydantic

**Spec:** `docs/superpowers/specs/2026-04-15-branch-reconciliation-design.md` (Phase 2)

---

### Task 1: Add `subject_company_id` to HeadspaceDigestItem and fix frontend routing

The headspace page routes to `/companies/${item.subject_slug}` but the `[companyId]` route expects a UUID. The backend needs to return the company UUID alongside the slug, and the frontend needs to use it.

**Files:**
- Modify: `apps/api/src/prescient/intelligence/api/routes.py:272-330`
- Modify: `apps/web/src/lib/api.ts:67-76`
- Modify: `apps/web/src/app/(main)/headspace/page.tsx:60`
- Test: `apps/api/tests/unit/intelligence/test_headspace_digest.py` (create if not exists)

- [ ] **Step 1: Write failing test for subject_company_id in HeadspaceDigestItem**

Check if a headspace test file exists:
```bash
ls apps/api/tests/unit/intelligence/test_headspace*.py 2>/dev/null
```

If it doesn't exist, create `apps/api/tests/unit/intelligence/test_headspace_digest.py`:

```python
"""Tests for HeadspaceDigestItem response model."""

from uuid import uuid4

from prescient.intelligence.api.routes import HeadspaceDigestItem


def test_headspace_digest_item_includes_subject_company_id():
    """HeadspaceDigestItem must include subject_company_id (UUID)."""
    item = HeadspaceDigestItem(
        id=uuid4(),
        subject_company_id=uuid4(),
        subject_slug="peloton",
        subject_name="Peloton",
        finding_type="risk",
        title="Revenue decline",
        summary="Q3 revenue down 12%",
        severity="high",
        discovered_at="2026-04-15T00:00:00Z",
    )
    assert item.subject_company_id is not None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_headspace_digest.py -v`

Expected: FAIL — `subject_company_id` is not a field on `HeadspaceDigestItem`.

- [ ] **Step 3: Add subject_company_id to HeadspaceDigestItem response model**

In `apps/api/src/prescient/intelligence/api/routes.py`, add the field to the model (around line 272):

```python
class HeadspaceDigestItem(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: UUID
    subject_company_id: UUID
    subject_slug: str
    subject_name: str
    finding_type: str
    title: str
    summary: str | None
    severity: str
    discovered_at: datetime
```

And update the response construction (around line 321) to include it:

```python
HeadspaceDigestItem(
    id=finding.id,
    subject_company_id=company.id,
    subject_slug=company.slug,
    subject_name=company.name,
    finding_type=finding.finding_type,
    title=finding.title,
    summary=finding.summary,
    severity=finding.severity,
    discovered_at=finding.discovered_at,
)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/test_headspace_digest.py -v`

Expected: PASS

- [ ] **Step 5: Update frontend HeadspaceDigestItem type**

In `apps/web/src/lib/api.ts`, add the field to the type (around line 67):

```typescript
export type HeadspaceDigestItem = {
  id: string;
  subject_company_id: string;
  subject_slug: string;
  subject_name: string;
  finding_type: string;
  title: string;
  summary: string | null;
  severity: string;
  discovered_at: string;
};
```

- [ ] **Step 6: Update headspace page to route by UUID**

In `apps/web/src/app/(main)/headspace/page.tsx`, change line 60 from:

```tsx
href={`/companies/${item.subject_slug}`}
```

to:

```tsx
href={`/companies/${item.subject_company_id}`}
```

- [ ] **Step 7: Run backend tests**

Run: `cd apps/api && uv run pytest tests/unit/intelligence/ -v`

Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/intelligence/api/routes.py \
       apps/api/tests/unit/intelligence/test_headspace_digest.py \
       apps/web/src/lib/api.ts \
       apps/web/src/app/(main)/headspace/page.tsx
git commit -m "fix(headspace): route by subject_company_id UUID instead of slug"
```

---

### Task 2: Replace slug-matching heuristic in board-prep page with primaryCompanyId

The board-prep page resolves the user's company by checking `orgId.includes(c.slug)` — a fragile substring match. The `primaryCompanyId` is already available from auth context.

**Files:**
- Modify: `apps/web/src/app/(main)/board-prep/page.tsx:106-113`

- [ ] **Step 1: Update board-prep page to use primaryCompanyId from auth**

In `apps/web/src/app/(main)/board-prep/page.tsx`, the current code around lines 106-113 is:

```tsx
const res = await fetch(`${API}/companies`, {
  headers: { Authorization: `Bearer ${token}` },
});
if (res.ok) {
  const companies: { id: string; slug: string }[] = await res.json();
  const match = companies.find((c) => orgId.includes(c.slug));
  setCompanyId(match?.id ?? companies[0]?.id ?? null);
}
```

Replace with:

```tsx
const primaryId = getPrimaryCompanyId();
if (primaryId) {
  setCompanyId(primaryId);
} else {
  const res = await fetch(`${API}/companies`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (res.ok) {
    const companies: { id: string }[] = await res.json();
    setCompanyId(companies[0]?.id ?? null);
  }
}
```

Add the import at the top of the file:

```tsx
import { getPrimaryCompanyId } from "@/lib/auth";
```

- [ ] **Step 2: Verify the page builds**

Run: `cd apps/web && npx next build 2>&1 | grep -E "error|Error|✓|✗" | head -10`

If that's too slow, use the TypeScript compiler:
Run: `cd apps/web && npx tsc --noEmit 2>&1 | grep "board-prep" | head -10`

Expected: No type errors in board-prep page.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(main)/board-prep/page.tsx
git commit -m "fix(board-prep): use primaryCompanyId from auth instead of slug heuristic"
```

---

### Task 3: Replace slug-matching heuristic in NavBrand with primaryCompanyId

Same pattern as Task 2 — NavBrand uses `orgId.includes(c.slug)` to resolve the display name.

**Files:**
- Modify: `apps/web/src/components/nav-brand.tsx`

- [ ] **Step 1: Update NavBrand to use primaryCompanyId**

The current `apps/web/src/components/nav-brand.tsx` (22 lines) is:

```tsx
"use client";

import { useEffect, useState } from "react";
import { getOrgId } from "@/lib/auth";

export default function NavBrand({
  companies,
  fallback = "Prescient",
}: {
  companies: { id: string; slug: string; name: string }[];
  fallback?: string;
}) {
  const [name, setName] = useState(fallback);

  useEffect(() => {
    const orgId = getOrgId() ?? "";
    const match = companies.find((c) => orgId.includes(c.slug));
    setName(match?.name ?? fallback);
  }, [companies, fallback]);

  return <span className="text-lg font-semibold tracking-tight">{name}</span>;
}
```

Replace with:

```tsx
"use client";

import { useEffect, useState } from "react";
import { getPrimaryCompanyId } from "@/lib/auth";

export default function NavBrand({
  companies,
  fallback = "Prescient",
}: {
  companies: { id: string; name: string }[];
  fallback?: string;
}) {
  const [name, setName] = useState(fallback);

  useEffect(() => {
    const companyId = getPrimaryCompanyId();
    const match = companies.find((c) => c.id === companyId);
    setName(match?.name ?? fallback);
  }, [companies, fallback]);

  return <span className="text-lg font-semibold tracking-tight">{name}</span>;
}
```

Key changes:
- Import `getPrimaryCompanyId` instead of `getOrgId`
- Match by `c.id === companyId` (UUID equality) instead of `orgId.includes(c.slug)`
- Remove `slug` from the companies type (callers already pass objects with `id`)

- [ ] **Step 2: Check for callers that pass the companies prop**

Run: `grep -rn "NavBrand" apps/web/src/ --include="*.tsx" --include="*.ts"`

Verify that callers pass objects with `id` and `name` fields. If any caller relies on `slug` being in the type, update it.

- [ ] **Step 3: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit 2>&1 | grep -E "nav-brand|NavBrand" | head -10`

Expected: No type errors.

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/components/nav-brand.tsx
git commit -m "fix(nav): use primaryCompanyId for company name instead of slug heuristic"
```

---

### Task 4: Convert sponsor access control from slug comparison to UUID comparison

The sponsor KPI and briefing endpoints verify access by building a set of slugs from org IDs, then checking if the company's slug is in that set. This should compare UUIDs directly.

**Files:**
- Modify: `apps/api/src/prescient/sponsor/api/routes.py:211-290`
- Test: `apps/api/tests/unit/sponsor/test_sponsor_access.py` (create if not exists)

- [ ] **Step 1: Write failing test for UUID-based sponsor access check**

Create `apps/api/tests/unit/sponsor/test_sponsor_access.py`:

```python
"""Tests for sponsor access control using UUID comparison."""

import pytest
from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4


def test_sponsor_access_uses_uuid_not_slug():
    """Verify the sponsor access pattern uses UUID comparison.

    This is a design-level test: we import the route module and inspect
    that the access check helper (if extracted) or the route function
    does not reference 'slug' for authorization.
    """
    import inspect
    from prescient.sponsor.api import routes

    # Read the source of get_company_kpis
    source = inspect.getsource(routes.get_company_kpis)
    # The access check should NOT build slug sets
    assert "accessible_slugs" not in source, (
        "get_company_kpis still uses slug-based access control"
    )
    assert "company_row.slug" not in source or "slug" not in source.split("not in")[0] if "not in" in source else True, (
        "get_company_kpis checks slug membership for access"
    )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/sponsor/test_sponsor_access.py -v`

Expected: FAIL — `accessible_slugs` is still in the source.

- [ ] **Step 3: Refactor get_company_kpis to use UUID access check**

In `apps/api/src/prescient/sponsor/api/routes.py`, the current access check in `get_company_kpis` (around lines 222-234) builds accessible slugs:

```python
accessible_slugs: set[str] = set()
for org_id_str in access.company_org_ids:
    org_row = await session.get(OrganizationRow, org_id_str)
    if org_row:
        accessible_slugs.add(org_row.slug)

company_row = await SqlCompanyRepository(session).get(company_id)
if not company_row:
    raise NotFoundError(f"company '{company_id}' not found")

if company_row.slug not in accessible_slugs:
    raise AuthorizationError("sponsor cannot access this company")
```

Replace with UUID-based check using the existing `get_company_id_by_slug` pattern. Since `access.company_org_ids` contains org string IDs and companies are linked to orgs by matching slug, we need to build a set of company UUIDs:

```python
from prescient.companies.queries import get_company_id_by_slug

accessible_company_ids: set[UUID] = set()
for org_id_str in access.company_org_ids:
    org_row = await session.get(OrganizationRow, org_id_str)
    if org_row:
        cid = await get_company_id_by_slug(session, org_row.slug)
        if cid:
            accessible_company_ids.add(cid)

company_row = await SqlCompanyRepository(session).get(company_id)
if not company_row:
    raise NotFoundError(f"company '{company_id}' not found")

if company_id not in accessible_company_ids:
    raise AuthorizationError("sponsor cannot access this company")
```

Add the `UUID` import at the top if not present:
```python
from uuid import UUID
```

- [ ] **Step 4: Apply the same refactor to get_company_briefing**

In the same file, `get_company_briefing` (around lines 270-288) has the identical slug-based access check. Apply the same UUID-based pattern:

```python
accessible_company_ids: set[UUID] = set()
for org_id_str in access.company_org_ids:
    org_row = await session.get(OrganizationRow, org_id_str)
    if org_row:
        cid = await get_company_id_by_slug(session, org_row.slug)
        if cid:
            accessible_company_ids.add(cid)

company_row = await SqlCompanyRepository(session).get(company_id)
if not company_row:
    raise NotFoundError(f"company '{company_id}' not found")

if company_id not in accessible_company_ids:
    raise AuthorizationError("sponsor cannot access this company")
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd apps/api && uv run pytest tests/unit/sponsor/test_sponsor_access.py -v`

Expected: PASS

- [ ] **Step 6: Run full sponsor tests**

Run: `cd apps/api && uv run pytest tests/unit/sponsor/ -v`

Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/sponsor/api/routes.py \
       apps/api/tests/unit/sponsor/test_sponsor_access.py
git commit -m "fix(sponsor): use UUID comparison for access control instead of slug matching"
```

---

### Task 5: Add company_id to sponsor response models

The `SponsorKpisResponse` and `PreCallBriefingResponse` models return `company_slug` as the company identifier but lack `company_id`. Add `company_id` so consumers can use UUID-based references.

**Files:**
- Modify: `apps/api/src/prescient/sponsor/api/routes.py` (response models + construction)

- [ ] **Step 1: Write failing test for company_id in sponsor responses**

Add to `apps/api/tests/unit/sponsor/test_sponsor_access.py`:

```python
from prescient.sponsor.api.routes import SponsorKpisResponse, PreCallBriefingResponse


def test_sponsor_kpis_response_has_company_id():
    """SponsorKpisResponse must include company_id field."""
    assert "company_id" in SponsorKpisResponse.model_fields


def test_precall_briefing_response_has_company_id():
    """PreCallBriefingResponse must include company_id field."""
    assert "company_id" in PreCallBriefingResponse.model_fields
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/sponsor/test_sponsor_access.py -v`

Expected: FAIL — `company_id` is not in the model fields.

- [ ] **Step 3: Add company_id to SponsorKpisResponse**

In `apps/api/src/prescient/sponsor/api/routes.py`, find the `SponsorKpisResponse` model (around line 82):

```python
class SponsorKpisResponse(BaseModel):
    company_slug: str
```

Add `company_id`:

```python
class SponsorKpisResponse(BaseModel):
    company_id: UUID
    company_slug: str
```

And update the response construction (around line 241) to include it:

```python
return SponsorKpisResponse(
    company_id=company_row.id,
    company_slug=company_row.slug,
    ...
)
```

- [ ] **Step 4: Add company_id to PreCallBriefingResponse**

Find the `PreCallBriefingResponse` model (around line 93):

```python
class PreCallBriefingResponse(BaseModel):
    company_slug: str
```

Add `company_id`:

```python
class PreCallBriefingResponse(BaseModel):
    company_id: UUID
    company_slug: str
```

And update the response construction (around line 348) to include it:

```python
return PreCallBriefingResponse(
    company_id=company_row.id,
    company_slug=company_row.slug,
    ...
)
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/sponsor/test_sponsor_access.py -v`

Expected: PASS

- [ ] **Step 6: Update frontend types to include company_id**

In `apps/web/src/app/(sponsor)/portfolio/companies/[companyId]/briefing/page.tsx`, find the response type (around line 16):

```typescript
company_slug: string;
```

Add above it:

```typescript
company_id: string;
company_slug: string;
```

In `apps/web/src/components/sponsor-kpi-dashboard.tsx`, find the response type (around line 22):

```typescript
company_slug: string;
```

Add above it:

```typescript
company_id: string;
company_slug: string;
```

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/sponsor/api/routes.py \
       apps/api/tests/unit/sponsor/test_sponsor_access.py \
       apps/web/src/app/(sponsor)/portfolio/companies/[companyId]/briefing/page.tsx \
       apps/web/src/components/sponsor-kpi-dashboard.tsx
git commit -m "feat(sponsor): add company_id UUID to KPI and briefing response models"
```

---

### Task 6: Mark dead COMPANY_KPI_CONFIG as deprecated

The `_kpi_ids_for` function and `COMPANY_KPI_CONFIG` dict in the board-prep assembler are never called (dead code). Rather than migrating the slug keys now, mark them deprecated. When board-prep KPI filtering is wired up later, the config should be UUID-keyed or moved to the database.

**Files:**
- Modify: `apps/api/src/prescient/board_prep/application/assembler.py:53-82`

- [ ] **Step 1: Add deprecation warning to the dead code**

In `apps/api/src/prescient/board_prep/application/assembler.py`, update the config dict and function (around lines 53-82):

```python
# DEPRECATED: This config is keyed by company slug and _kpi_ids_for() is never
# called. When board-prep KPI filtering is wired up, migrate to UUID-keyed
# config or move to the database. See slug-to-UUID gap fill plan.
COMPANY_KPI_CONFIG: dict[str, dict[str, list[str]]] = {
    "peloton": {
        "financial": [
            "revenue",
            "gross_margin",
            "ebitda",
            "operating_cash_flow",
            "net_income",
            "free_cash_flow",
        ],
        "subscriber": [
            "subscribers",
            "connected_fitness_subscribers",
            "churn_rate",
            "monthly_churn",
            "arpu",
            "average_net_monthly_connected_fitness_churn",
        ],
    },
    "_default": {
        "financial": [],
        "subscriber": [],
    },
}


def _kpi_ids_for(company_slug: str, category: str) -> list[str]:
    """Return KPI IDs for a company and category, falling back to _default.

    .. deprecated::
        Never called. Slug-keyed config needs UUID migration when activated.
    """
    config = COMPANY_KPI_CONFIG.get(company_slug, COMPANY_KPI_CONFIG["_default"])
    return config.get(category, [])
```

- [ ] **Step 2: Verify no callers exist**

Run: `grep -rn "_kpi_ids_for\|COMPANY_KPI_CONFIG" apps/api/src/ --include="*.py" | grep -v "__pycache__"`

Expected: Only hits in `assembler.py` itself — confirms dead code.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/board_prep/application/assembler.py
git commit -m "docs(board-prep): mark dead COMPANY_KPI_CONFIG as deprecated, needs UUID migration"
```
