# Phase 5d: Sponsor Portal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the sponsor-facing portal — role-based routing after login, portfolio dashboard, company detail with KPI/shared artifacts/Q&A tabs, and pre-call briefing. Michael Torres (Summit Partners) sees a different experience than Sarah Chen (Peloton operator).

**Architecture:** New `/(sponsor)/` Next.js route group with its own layout. Login page detects role and redirects. Sponsor API endpoints (`/sponsor/portfolio`, `/sponsor/companies/{slug}/kpis`, `/sponsor/companies/{slug}/briefing`) serve visibility-scoped data. Uses AccessContext from Phase 5a for enforcement.

**Tech Stack:** Next.js 15, React 19, TypeScript, Tailwind CSS, shadcn/ui, FastAPI

**Depends on:** Phase 5a (AccessContext), Phase 5b (triage API for Q&A)

---

## File Structure

### New files:

```
apps/api/src/prescient/sponsor/__init__.py
apps/api/src/prescient/sponsor/api/__init__.py
apps/api/src/prescient/sponsor/api/routes.py             — sponsor-specific API endpoints
apps/web/src/app/(sponsor)/layout.tsx                     — sponsor layout with nav
apps/web/src/app/(sponsor)/page.tsx                       — portfolio dashboard
apps/web/src/app/(sponsor)/companies/[slug]/page.tsx      — company detail with tabs
apps/web/src/app/(sponsor)/companies/[slug]/briefing/page.tsx — pre-call briefing
apps/web/src/components/sponsor-nav.tsx                   — sponsor navigation
apps/web/src/components/sponsor-portfolio-card.tsx        — company card with KPI sparklines
apps/web/src/components/sponsor-kpi-dashboard.tsx         — read-only KPI charts
apps/web/src/components/sponsor-shared-artifacts.tsx      — shared artifacts list
apps/web/src/components/sponsor-qa.tsx                    — Q&A tab with question submission
```

### Modified files:

```
apps/api/src/prescient/main.py                           — include sponsor router
apps/web/src/app/(auth)/login/page.tsx                   — redirect by role after login
apps/web/src/lib/api.ts                                  — add sponsor API methods
apps/web/src/lib/auth.ts                                 — store role, add getRole()
```

---

### Task 1: Sponsor API Endpoints

**Files:**
- Create: `apps/api/src/prescient/sponsor/__init__.py`
- Create: `apps/api/src/prescient/sponsor/api/__init__.py`
- Create: `apps/api/src/prescient/sponsor/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`

- [ ] **Step 1: Create sponsor API package**

```python
# apps/api/src/prescient/sponsor/__init__.py
```

```python
# apps/api/src/prescient/sponsor/api/__init__.py
```

- [ ] **Step 2: Implement sponsor routes**

```python
# apps/api/src/prescient/sponsor/api/routes.py
"""Sponsor-facing API routes.

GET  /sponsor/portfolio                    — portfolio dashboard data
GET  /sponsor/companies/{slug}/kpis        — KPIs visible to sponsor
GET  /sponsor/companies/{slug}/briefing    — pre-call briefing

All routes require sponsor role (operating_partner, pe_analyst, board_member).
Data is scoped by AccessContext visibility rules.
"""

from __future__ import annotations

from datetime import datetime
from typing import Annotated, Any

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.auth.access_context import AccessContext, get_access_context
from prescient.companies.infrastructure.tables import CompanyTable
from prescient.db import get_session
from prescient.kpis.infrastructure.tables.kpi_value import KpiValueRow
from prescient.kpis.infrastructure.tables.kpi_definition import KpiDefinitionRow
from prescient.organizations.domain.entities.visibility_rule import DataCategory
from prescient.triage.infrastructure.tables.shared_item import SharedItemTable

router = APIRouter(prefix="/sponsor", tags=["sponsor"])

SessionDep = Annotated[AsyncSession, Depends(get_session)]
AccessDep = Annotated[AccessContext, Depends(get_access_context)]


def _require_sponsor(access: AccessContext) -> None:
    if not access.is_sponsor:
        raise HTTPException(status_code=403, detail="Sponsor access required")


# ---------------------------------------------------------------------------
# Response models
# ---------------------------------------------------------------------------


class PortfolioCompanyCard(BaseModel):
    org_id: str
    slug: str
    name: str
    sector: str | None
    kpi_sparklines: list[dict[str, Any]]
    pending_questions: int
    shared_artifact_count: int


class PortfolioResponse(BaseModel):
    sponsor_org_name: str
    companies: list[PortfolioCompanyCard]


class KpiDataPoint(BaseModel):
    period: str
    value: float


class SponsorKpi(BaseModel):
    name: str
    unit: str | None
    data: list[KpiDataPoint]


class SponsorKpisResponse(BaseModel):
    company_slug: str
    kpis: list[SponsorKpi]


class BriefingSection(BaseModel):
    title: str
    content: str


class PreCallBriefingResponse(BaseModel):
    company_slug: str
    company_name: str
    generated_at: str
    sections: list[BriefingSection]


# ---------------------------------------------------------------------------
# Routes
# ---------------------------------------------------------------------------


@router.get("/portfolio", response_model=PortfolioResponse)
async def get_portfolio(
    session: SessionDep,
    access: AccessDep,
) -> PortfolioResponse:
    _require_sponsor(access)

    companies: list[PortfolioCompanyCard] = []
    for company_org_id in access.company_org_ids:
        # Get company info
        company_stmt = select(CompanyTable).where(CompanyTable.id == company_org_id)
        company_result = await session.execute(company_stmt)
        company_row = company_result.scalar_one_or_none()
        if company_row is None:
            continue

        # Get KPI sparklines if KPIs are visible
        sparklines: list[dict[str, Any]] = []
        if access.can_see_category(DataCategory.KPI_METRICS):
            kpi_stmt = (
                select(KpiDefinitionRow)
                .where(KpiDefinitionRow.organization_id == company_org_id)
                .limit(3)
            )
            kpi_result = await session.execute(kpi_stmt)
            kpi_defs = list(kpi_result.scalars().all())
            for kpi_def in kpi_defs:
                values_stmt = (
                    select(KpiValueRow)
                    .where(KpiValueRow.kpi_definition_id == kpi_def.id)
                    .order_by(KpiValueRow.period.desc())
                    .limit(6)
                )
                values_result = await session.execute(values_stmt)
                values = list(values_result.scalars().all())
                sparklines.append({
                    "name": kpi_def.name,
                    "values": [float(v.value) for v in reversed(values)],
                })

        # Count pending questions
        from prescient.triage.infrastructure.tables.triage_question import TriageQuestionTable
        q_stmt = select(TriageQuestionTable).where(
            TriageQuestionTable.asker_org_id == access.org_id,
            TriageQuestionTable.target_company_org_id == company_org_id,
            TriageQuestionTable.status.in_(["submitted", "classifying", "partially_answered", "escalated"]),
        )
        q_result = await session.execute(q_stmt)
        pending_count = len(list(q_result.scalars().all()))

        # Count shared artifacts
        shared_stmt = select(SharedItemTable).where(
            SharedItemTable.sponsor_org_id == access.org_id,
        )
        shared_result = await session.execute(shared_stmt)
        shared_count = len(list(shared_result.scalars().all()))

        companies.append(PortfolioCompanyCard(
            org_id=company_org_id,
            slug=company_row.slug,
            name=company_row.name,
            sector=getattr(company_row, "sector", None),
            kpi_sparklines=sparklines,
            pending_questions=pending_count,
            shared_artifact_count=shared_count,
        ))

    return PortfolioResponse(
        sponsor_org_name="Summit Partners",  # resolve from org table in production
        companies=companies,
    )


@router.get("/companies/{slug}/kpis", response_model=SponsorKpisResponse)
async def get_sponsor_kpis(
    slug: str,
    session: SessionDep,
    access: AccessDep,
) -> SponsorKpisResponse:
    _require_sponsor(access)

    if not access.can_see_category(DataCategory.KPI_METRICS):
        raise HTTPException(status_code=403, detail="KPI data not visible to your access level")

    # Look up company by slug
    company_stmt = select(CompanyTable).where(CompanyTable.slug == slug)
    company_result = await session.execute(company_stmt)
    company_row = company_result.scalar_one_or_none()
    if company_row is None:
        raise HTTPException(status_code=404, detail="Company not found")

    # Verify sponsor has access to this company
    if company_row.id not in access.company_org_ids:
        raise HTTPException(status_code=403, detail="No access to this company")

    # Fetch all KPI definitions and values
    kpi_stmt = select(KpiDefinitionRow).where(
        KpiDefinitionRow.organization_id == company_row.id
    )
    kpi_result = await session.execute(kpi_stmt)
    kpi_defs = list(kpi_result.scalars().all())

    kpis: list[SponsorKpi] = []
    for kpi_def in kpi_defs:
        values_stmt = (
            select(KpiValueRow)
            .where(KpiValueRow.kpi_definition_id == kpi_def.id)
            .order_by(KpiValueRow.period.asc())
        )
        values_result = await session.execute(values_stmt)
        values = list(values_result.scalars().all())
        kpis.append(SponsorKpi(
            name=kpi_def.name,
            unit=getattr(kpi_def, "unit", None),
            data=[
                KpiDataPoint(period=v.period, value=float(v.value))
                for v in values
            ],
        ))

    return SponsorKpisResponse(company_slug=slug, kpis=kpis)


@router.get("/companies/{slug}/briefing", response_model=PreCallBriefingResponse)
async def get_pre_call_briefing(
    slug: str,
    session: SessionDep,
    access: AccessDep,
) -> PreCallBriefingResponse:
    _require_sponsor(access)

    company_stmt = select(CompanyTable).where(CompanyTable.slug == slug)
    company_result = await session.execute(company_stmt)
    company_row = company_result.scalar_one_or_none()
    if company_row is None:
        raise HTTPException(status_code=404, detail="Company not found")

    if company_row.id not in access.company_org_ids:
        raise HTTPException(status_code=403, detail="No access to this company")

    # Build briefing sections from visible data
    sections: list[BriefingSection] = []

    # KPI summary
    if access.can_see_category(DataCategory.KPI_METRICS):
        kpi_stmt = select(KpiDefinitionRow).where(
            KpiDefinitionRow.organization_id == company_row.id
        ).limit(5)
        kpi_result = await session.execute(kpi_stmt)
        kpi_defs = list(kpi_result.scalars().all())
        if kpi_defs:
            kpi_lines = []
            for kpi_def in kpi_defs:
                values_stmt = (
                    select(KpiValueRow)
                    .where(KpiValueRow.kpi_definition_id == kpi_def.id)
                    .order_by(KpiValueRow.period.desc())
                    .limit(1)
                )
                val_result = await session.execute(values_stmt)
                latest = val_result.scalar_one_or_none()
                if latest:
                    kpi_lines.append(f"- **{kpi_def.name}**: {latest.value}")
            if kpi_lines:
                sections.append(BriefingSection(
                    title="Key Metrics",
                    content="\n".join(kpi_lines),
                ))

    # Shared artifacts summary
    shared_stmt = (
        select(SharedItemTable)
        .where(SharedItemTable.sponsor_org_id == access.org_id)
        .order_by(SharedItemTable.shared_at.desc())
        .limit(5)
    )
    shared_result = await session.execute(shared_stmt)
    shared_rows = list(shared_result.scalars().all())
    if shared_rows:
        sections.append(BriefingSection(
            title="Recently Shared",
            content=f"{len(shared_rows)} artifact(s) shared since last interaction.",
        ))

    # Answered questions
    from prescient.triage.infrastructure.tables.triage_question import TriageQuestionTable
    answered_stmt = (
        select(TriageQuestionTable)
        .where(
            TriageQuestionTable.asker_org_id == access.org_id,
            TriageQuestionTable.target_company_org_id == company_row.id,
            TriageQuestionTable.status == "answered",
        )
        .order_by(TriageQuestionTable.answered_at.desc())
        .limit(3)
    )
    answered_result = await session.execute(answered_stmt)
    answered_rows = list(answered_result.scalars().all())
    if answered_rows:
        qa_lines = [f"- **Q:** {q.question_text[:100]}..." for q in answered_rows]
        sections.append(BriefingSection(
            title="Recent Q&A",
            content="\n".join(qa_lines),
        ))

    if not sections:
        sections.append(BriefingSection(
            title="Summary",
            content="No updates available for this period.",
        ))

    return PreCallBriefingResponse(
        company_slug=slug,
        company_name=company_row.name,
        generated_at=datetime.now().isoformat(),
        sections=sections,
    )
```

- [ ] **Step 3: Register sponsor router in main.py**

Add to imports in `apps/api/src/prescient/main.py`:

```python
from prescient.sponsor.api.routes import router as sponsor_router
```

And in `create_app()`:

```python
    app.include_router(sponsor_router)
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/sponsor/ apps/api/src/prescient/main.py
git commit -m "feat(sponsor): add sponsor API — portfolio, KPIs, pre-call briefing"
```

---

### Task 2: Auth Role Storage + Redirect

**Files:**
- Modify: `apps/web/src/lib/auth.ts`
- Modify: `apps/web/src/app/(auth)/login/page.tsx`

- [ ] **Step 1: Add role to auth storage**

Open `apps/web/src/lib/auth.ts` and add role storage:

```typescript
const ROLE_KEY = "prescient_role";

export function getRole(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem(ROLE_KEY);
}

export function isSponsor(): boolean {
  const role = getRole();
  return role === "operating_partner" || role === "pe_analyst" || role === "board_member";
}
```

Update `setAuth` to accept and store role:

```typescript
export function setAuth(token: string, userId: string, orgId?: string, role?: string): void {
  localStorage.setItem(TOKEN_KEY, token);
  localStorage.setItem(USER_KEY, userId);
  if (orgId) localStorage.setItem(ORG_KEY, orgId);
  if (role) localStorage.setItem(ROLE_KEY, role);
}
```

Update `clearAuth` to clear role:

```typescript
export function clearAuth(): void {
  localStorage.removeItem(TOKEN_KEY);
  localStorage.removeItem(USER_KEY);
  localStorage.removeItem(ORG_KEY);
  localStorage.removeItem(ROLE_KEY);
}
```

The `login()` function needs to return role. Update the login API response to include it. Open `apps/api/src/prescient/auth/api/routes.py` and add `role` to the login response. Then update the frontend `login()` return type:

```typescript
export async function login(
  email: string,
  password: string,
): Promise<{ access_token: string; user_id: string; organization_id: string; fund_id: string; role: string }> {
  // ... existing implementation ...
}
```

- [ ] **Step 2: Update login page to redirect by role**

Open `apps/web/src/app/(auth)/login/page.tsx` and update the login handler to store role and redirect accordingly:

After the successful login call, change the redirect logic:

```typescript
setAuth(result.access_token, result.user_id, result.organization_id, result.role);

// Redirect based on role
if (result.role === "operating_partner" || result.role === "pe_analyst" || result.role === "board_member") {
  router.push("/");  // sponsor portal root
} else {
  router.push("/brief");  // operator brief
}
```

Note: The `/(sponsor)/` route group will catch sponsor users at `/` while `/(main)/` catches operator users at `/brief`.

- [ ] **Step 3: Update login API to return role**

Open `apps/api/src/prescient/auth/api/routes.py` and add `role` to the token response. Find the login endpoint and ensure it returns the user's role from their membership record.

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/lib/auth.ts apps/web/src/app/\(auth\)/login/page.tsx apps/api/src/prescient/auth/api/routes.py
git commit -m "feat(auth): store role on login, redirect sponsors to portal"
```

---

### Task 3: Sponsor Layout and Navigation

**Files:**
- Create: `apps/web/src/app/(sponsor)/layout.tsx`
- Create: `apps/web/src/components/sponsor-nav.tsx`

- [ ] **Step 1: Implement sponsor navigation**

```typescript
// apps/web/src/components/sponsor-nav.tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { clearAuth } from "@/lib/auth";
import { useRouter } from "next/navigation";
import { cn } from "@/lib/utils";

export function SponsorNav() {
  const pathname = usePathname();
  const router = useRouter();

  const handleLogout = () => {
    clearAuth();
    router.push("/login");
  };

  return (
    <header className="h-14 border-b border-neutral-200 bg-white px-6 flex items-center justify-between">
      <div className="flex items-center gap-6">
        <Link href="/" className="text-base font-semibold text-neutral-800">
          Prescient OS
        </Link>
        <nav className="flex gap-4">
          <Link
            href="/"
            className={cn(
              "text-sm transition-colors",
              pathname === "/" ? "text-blue-600 font-medium" : "text-neutral-500 hover:text-neutral-800"
            )}
          >
            Portfolio
          </Link>
        </nav>
      </div>
      <button
        onClick={handleLogout}
        className="text-sm text-neutral-500 hover:text-neutral-800"
      >
        Sign out
      </button>
    </header>
  );
}
```

- [ ] **Step 2: Implement sponsor layout**

```typescript
// apps/web/src/app/(sponsor)/layout.tsx
"use client";

import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { getToken, isSponsor } from "@/lib/auth";
import { SponsorNav } from "@/components/sponsor-nav";

export default function SponsorLayout({ children }: { children: React.ReactNode }) {
  const router = useRouter();

  useEffect(() => {
    const token = getToken();
    if (!token) {
      router.push("/login");
      return;
    }
    if (!isSponsor()) {
      router.push("/brief");
      return;
    }
  }, [router]);

  return (
    <div className="min-h-screen bg-neutral-50">
      <SponsorNav />
      <main>{children}</main>
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/\(sponsor\)/layout.tsx apps/web/src/components/sponsor-nav.tsx
git commit -m "feat(web): add sponsor layout with role-gated routing"
```

---

### Task 4: Portfolio Dashboard

**Files:**
- Create: `apps/web/src/app/(sponsor)/page.tsx`
- Create: `apps/web/src/components/sponsor-portfolio-card.tsx`

- [ ] **Step 1: Implement portfolio card**

```typescript
// apps/web/src/components/sponsor-portfolio-card.tsx
"use client";

import Link from "next/link";

interface SparklineData {
  name: string;
  values: number[];
}

interface PortfolioCardProps {
  slug: string;
  name: string;
  sector: string | null;
  sparklines: SparklineData[];
  pendingQuestions: number;
  sharedArtifacts: number;
}

function MiniSparkline({ values }: { values: number[] }) {
  if (values.length < 2) return null;
  const min = Math.min(...values);
  const max = Math.max(...values);
  const range = max - min || 1;
  const h = 24;
  const w = 60;
  const step = w / (values.length - 1);

  const points = values.map((v, i) => `${i * step},${h - ((v - min) / range) * h}`).join(" ");

  return (
    <svg width={w} height={h} className="inline-block">
      <polyline fill="none" stroke="currentColor" strokeWidth="1.5" points={points} />
    </svg>
  );
}

export function SponsorPortfolioCard({
  slug,
  name,
  sector,
  sparklines,
  pendingQuestions,
  sharedArtifacts,
}: PortfolioCardProps) {
  return (
    <Link
      href={`/companies/${slug}`}
      className="block bg-white rounded-lg border border-neutral-200 p-5 hover:border-blue-300 hover:shadow-sm transition-all"
    >
      <div className="mb-3">
        <h3 className="text-base font-semibold text-neutral-800">{name}</h3>
        {sector && <p className="text-xs text-neutral-400 mt-0.5">{sector}</p>}
      </div>

      {/* KPI sparklines */}
      {sparklines.length > 0 && (
        <div className="space-y-2 mb-4">
          {sparklines.map((s) => (
            <div key={s.name} className="flex items-center justify-between">
              <span className="text-xs text-neutral-500">{s.name}</span>
              <span className="text-blue-500">
                <MiniSparkline values={s.values} />
              </span>
            </div>
          ))}
        </div>
      )}

      {/* Stats */}
      <div className="flex gap-4 text-xs text-neutral-400 pt-3 border-t border-neutral-100">
        <span>{pendingQuestions} pending Q&A</span>
        <span>{sharedArtifacts} shared artifacts</span>
      </div>
    </Link>
  );
}
```

- [ ] **Step 2: Implement portfolio page**

```typescript
// apps/web/src/app/(sponsor)/page.tsx
"use client";

import { useEffect, useState } from "react";
import { getToken } from "@/lib/auth";
import { SponsorPortfolioCard } from "@/components/sponsor-portfolio-card";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

interface PortfolioCompany {
  org_id: string;
  slug: string;
  name: string;
  sector: string | null;
  kpi_sparklines: { name: string; values: number[] }[];
  pending_questions: number;
  shared_artifact_count: number;
}

interface PortfolioData {
  sponsor_org_name: string;
  companies: PortfolioCompany[];
}

export default function SponsorPortfolioPage() {
  const [data, setData] = useState<PortfolioData | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchPortfolio = async () => {
      const token = getToken();
      try {
        const res = await fetch(`${API}/sponsor/portfolio`, {
          headers: token ? { Authorization: `Bearer ${token}` } : {},
        });
        if (res.ok) {
          setData(await res.json());
        }
      } finally {
        setLoading(false);
      }
    };
    fetchPortfolio();
  }, []);

  if (loading) {
    return (
      <div className="p-8 text-center text-neutral-400">Loading portfolio...</div>
    );
  }

  if (!data) {
    return (
      <div className="p-8 text-center text-neutral-400">Failed to load portfolio data.</div>
    );
  }

  return (
    <div className="max-w-5xl mx-auto p-8">
      <div className="mb-6">
        <h1 className="text-2xl font-bold text-neutral-800">{data.sponsor_org_name}</h1>
        <p className="text-sm text-neutral-400 mt-1">Portfolio Overview</p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {data.companies.map((company) => (
          <SponsorPortfolioCard
            key={company.org_id}
            slug={company.slug}
            name={company.name}
            sector={company.sector}
            sparklines={company.kpi_sparklines}
            pendingQuestions={company.pending_questions}
            sharedArtifacts={company.shared_artifact_count}
          />
        ))}
      </div>

      {data.companies.length === 0 && (
        <div className="text-center text-neutral-400 py-12">
          No portfolio companies linked yet.
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/\(sponsor\)/page.tsx apps/web/src/components/sponsor-portfolio-card.tsx
git commit -m "feat(web): add sponsor portfolio dashboard with KPI sparklines"
```

---

### Task 5: Company Detail with Tabs (KPIs, Shared, Q&A)

**Files:**
- Create: `apps/web/src/app/(sponsor)/companies/[slug]/page.tsx`
- Create: `apps/web/src/components/sponsor-kpi-dashboard.tsx`
- Create: `apps/web/src/components/sponsor-shared-artifacts.tsx`
- Create: `apps/web/src/components/sponsor-qa.tsx`

- [ ] **Step 1: Implement KPI dashboard component**

```typescript
// apps/web/src/components/sponsor-kpi-dashboard.tsx
"use client";

import { useEffect, useState } from "react";
import { getToken } from "@/lib/auth";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

interface KpiDataPoint {
  period: string;
  value: number;
}

interface SponsorKpi {
  name: string;
  unit: string | null;
  data: KpiDataPoint[];
}

export function SponsorKpiDashboard({ slug }: { slug: string }) {
  const [kpis, setKpis] = useState<SponsorKpi[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchKpis = async () => {
      const token = getToken();
      try {
        const res = await fetch(`${API}/sponsor/companies/${slug}/kpis`, {
          headers: token ? { Authorization: `Bearer ${token}` } : {},
        });
        if (res.ok) {
          const data = await res.json();
          setKpis(data.kpis);
        }
      } finally {
        setLoading(false);
      }
    };
    fetchKpis();
  }, [slug]);

  if (loading) return <div className="p-4 text-neutral-400 text-sm">Loading KPIs...</div>;
  if (kpis.length === 0) return <div className="p-4 text-neutral-400 text-sm">No KPI data available.</div>;

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 gap-4 p-4">
      {kpis.map((kpi) => {
        const latest = kpi.data[kpi.data.length - 1];
        const prev = kpi.data.length > 1 ? kpi.data[kpi.data.length - 2] : null;
        const change = prev ? ((latest.value - prev.value) / prev.value) * 100 : null;

        return (
          <div key={kpi.name} className="bg-white rounded border border-neutral-200 p-4">
            <p className="text-xs text-neutral-500 mb-1">{kpi.name}</p>
            <div className="flex items-baseline gap-2">
              <span className="text-2xl font-bold text-neutral-800">
                {latest.value.toLocaleString()}
              </span>
              {kpi.unit && <span className="text-xs text-neutral-400">{kpi.unit}</span>}
              {change !== null && (
                <span
                  className={`text-xs font-medium ${
                    change >= 0 ? "text-green-600" : "text-red-600"
                  }`}
                >
                  {change >= 0 ? "+" : ""}
                  {change.toFixed(1)}%
                </span>
              )}
            </div>
            <div className="mt-3 flex gap-1">
              {kpi.data.slice(-8).map((d, i) => {
                const max = Math.max(...kpi.data.map((p) => p.value));
                const min = Math.min(...kpi.data.map((p) => p.value));
                const range = max - min || 1;
                const h = ((d.value - min) / range) * 32 + 4;
                return (
                  <div
                    key={i}
                    className="flex-1 bg-blue-200 rounded-sm"
                    style={{ height: `${h}px` }}
                    title={`${d.period}: ${d.value.toLocaleString()}`}
                  />
                );
              })}
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

- [ ] **Step 2: Implement shared artifacts component**

```typescript
// apps/web/src/components/sponsor-shared-artifacts.tsx
"use client";

import { useEffect, useState } from "react";
import { getToken } from "@/lib/auth";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

interface SharedArtifact {
  id: string;
  artifact_id: string;
  sponsor_org_id: string;
  shared_by_user_id: string;
  shared_at: string;
  triage_question_id: string | null;
}

export function SponsorSharedArtifacts({ slug }: { slug: string }) {
  const [items, setItems] = useState<SharedArtifact[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchShared = async () => {
      const token = getToken();
      try {
        const res = await fetch(`${API}/sharing/items`, {
          headers: token ? { Authorization: `Bearer ${token}` } : {},
        });
        if (res.ok) {
          setItems(await res.json());
        }
      } finally {
        setLoading(false);
      }
    };
    fetchShared();
  }, [slug]);

  if (loading) return <div className="p-4 text-neutral-400 text-sm">Loading...</div>;
  if (items.length === 0) {
    return <div className="p-4 text-neutral-400 text-sm">No shared artifacts yet.</div>;
  }

  return (
    <div className="p-4 space-y-2">
      {items.map((item) => (
        <div
          key={item.id}
          className="bg-white rounded border border-neutral-200 p-3 hover:border-blue-300 transition-colors cursor-pointer"
        >
          <div className="flex items-center justify-between">
            <div>
              <p className="text-sm font-medium text-neutral-800">
                Artifact {item.artifact_id.slice(0, 8)}...
              </p>
              <p className="text-xs text-neutral-400 mt-0.5">
                Shared {new Date(item.shared_at).toLocaleDateString()} by {item.shared_by_user_id}
              </p>
            </div>
            {item.triage_question_id && (
              <span className="text-[10px] px-1.5 py-0.5 bg-purple-100 text-purple-700 rounded">
                Q&A response
              </span>
            )}
          </div>
        </div>
      ))}
    </div>
  );
}
```

- [ ] **Step 3: Implement Q&A component**

```typescript
// apps/web/src/components/sponsor-qa.tsx
"use client";

import { useEffect, useState } from "react";
import { getToken, getOrgId } from "@/lib/auth";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

interface Question {
  id: string;
  question_text: string;
  status: string;
  partial_answer: string | null;
  final_answer: string | null;
  created_at: string;
  answered_at: string | null;
}

export function SponsorQA({ slug, companyOrgId }: { slug: string; companyOrgId: string }) {
  const [questions, setQuestions] = useState<Question[]>([]);
  const [newQuestion, setNewQuestion] = useState("");
  const [submitting, setSubmitting] = useState(false);
  const [loading, setLoading] = useState(true);

  const fetchQuestions = async () => {
    const token = getToken();
    try {
      const res = await fetch(
        `${API}/triage/questions?target_company_org_id=${companyOrgId}`,
        {
          headers: token ? { Authorization: `Bearer ${token}` } : {},
        }
      );
      if (res.ok) {
        setQuestions(await res.json());
      }
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchQuestions();
  }, [companyOrgId]);

  const handleSubmit = async () => {
    if (!newQuestion.trim()) return;
    setSubmitting(true);
    try {
      const token = getToken();
      const res = await fetch(`${API}/triage/questions`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          ...(token ? { Authorization: `Bearer ${token}` } : {}),
        },
        body: JSON.stringify({
          target_company_org_id: companyOrgId,
          question_text: newQuestion,
        }),
      });
      if (res.ok) {
        setNewQuestion("");
        await fetchQuestions();
      }
    } finally {
      setSubmitting(false);
    }
  };

  const statusColor: Record<string, string> = {
    submitted: "bg-gray-100 text-gray-600",
    classifying: "bg-yellow-100 text-yellow-700",
    partially_answered: "bg-blue-100 text-blue-700",
    escalated: "bg-amber-100 text-amber-700",
    answered: "bg-green-100 text-green-700",
    closed: "bg-neutral-100 text-neutral-500",
  };

  return (
    <div className="p-4">
      {/* Ask a question */}
      <div className="mb-6">
        <label className="text-sm font-medium text-neutral-700 block mb-1">
          Ask a question
        </label>
        <div className="flex gap-2">
          <input
            type="text"
            value={newQuestion}
            onChange={(e) => setNewQuestion(e.target.value)}
            onKeyDown={(e) => e.key === "Enter" && handleSubmit()}
            placeholder="What would you like to know?"
            className="flex-1 rounded border border-neutral-300 px-3 py-2 text-sm focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
          />
          <button
            onClick={handleSubmit}
            disabled={submitting || !newQuestion.trim()}
            className="px-4 py-2 text-sm rounded font-medium bg-blue-600 text-white hover:bg-blue-700 disabled:opacity-50"
          >
            {submitting ? "..." : "Ask"}
          </button>
        </div>
      </div>

      {/* Question history */}
      {loading ? (
        <div className="text-neutral-400 text-sm">Loading...</div>
      ) : questions.length === 0 ? (
        <div className="text-neutral-400 text-sm text-center py-8">
          No questions yet. Ask something above!
        </div>
      ) : (
        <div className="space-y-3">
          {questions.map((q) => (
            <div key={q.id} className="bg-white rounded border border-neutral-200 p-4">
              <div className="flex items-start justify-between mb-2">
                <p className="text-sm font-medium text-neutral-800">{q.question_text}</p>
                <span
                  className={`text-[10px] px-1.5 py-0.5 rounded ml-2 whitespace-nowrap ${
                    statusColor[q.status] ?? "bg-gray-100"
                  }`}
                >
                  {q.status.replace("_", " ")}
                </span>
              </div>

              {/* Partial answer */}
              {q.partial_answer && (
                <div className="mt-2 bg-blue-50 rounded p-2">
                  <p className="text-xs font-medium text-blue-600 mb-0.5">Available now:</p>
                  <p className="text-sm text-blue-900">{q.partial_answer}</p>
                  {q.status === "partially_answered" && (
                    <p className="text-[10px] text-blue-400 mt-1">
                      We&apos;ve shared what&apos;s available. The team has been flagged for the rest.
                    </p>
                  )}
                </div>
              )}

              {/* Final answer */}
              {q.final_answer && (
                <div className="mt-2 bg-green-50 rounded p-2">
                  <p className="text-xs font-medium text-green-600 mb-0.5">Full answer:</p>
                  <p className="text-sm text-green-900">{q.final_answer}</p>
                </div>
              )}

              <p className="text-[10px] text-neutral-400 mt-2">
                Asked {new Date(q.created_at).toLocaleDateString()}
                {q.answered_at && ` · Answered ${new Date(q.answered_at).toLocaleDateString()}`}
              </p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 4: Implement company detail page with tabs**

```typescript
// apps/web/src/app/(sponsor)/companies/[slug]/page.tsx
"use client";

import { useState } from "react";
import { useParams } from "next/navigation";
import { cn } from "@/lib/utils";
import { SponsorKpiDashboard } from "@/components/sponsor-kpi-dashboard";
import { SponsorSharedArtifacts } from "@/components/sponsor-shared-artifacts";
import { SponsorQA } from "@/components/sponsor-qa";
import Link from "next/link";

type Tab = "kpis" | "shared" | "qa";

export default function SponsorCompanyDetailPage() {
  const params = useParams();
  const slug = params.slug as string;
  const [activeTab, setActiveTab] = useState<Tab>("kpis");

  // For now, hardcode the company org ID mapping
  // In production, this would come from the portfolio API
  const companyOrgId = slug === "peloton" ? "org-peloton-001" : slug;

  const tabs: { key: Tab; label: string }[] = [
    { key: "kpis", label: "KPI Dashboard" },
    { key: "shared", label: "Shared Artifacts" },
    { key: "qa", label: "Q&A" },
  ];

  return (
    <div className="max-w-5xl mx-auto p-8">
      <div className="flex items-center justify-between mb-6">
        <div>
          <Link href="/" className="text-xs text-blue-600 hover:text-blue-800 mb-1 block">
            &larr; Portfolio
          </Link>
          <h1 className="text-2xl font-bold text-neutral-800 capitalize">{slug}</h1>
        </div>
        <Link
          href={`/companies/${slug}/briefing`}
          className="px-3 py-1.5 text-sm rounded font-medium bg-blue-600 text-white hover:bg-blue-700"
        >
          Pre-Call Briefing
        </Link>
      </div>

      {/* Tabs */}
      <div className="flex gap-1 border-b border-neutral-200 mb-4">
        {tabs.map((tab) => (
          <button
            key={tab.key}
            onClick={() => setActiveTab(tab.key)}
            className={cn(
              "px-4 py-2 text-sm font-medium -mb-px border-b-2 transition-colors",
              activeTab === tab.key
                ? "border-blue-600 text-blue-600"
                : "border-transparent text-neutral-500 hover:text-neutral-700"
            )}
          >
            {tab.label}
          </button>
        ))}
      </div>

      {/* Tab content */}
      {activeTab === "kpis" && <SponsorKpiDashboard slug={slug} />}
      {activeTab === "shared" && <SponsorSharedArtifacts slug={slug} />}
      {activeTab === "qa" && <SponsorQA slug={slug} companyOrgId={companyOrgId} />}
    </div>
  );
}
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/app/\(sponsor\)/companies/ apps/web/src/components/sponsor-kpi-dashboard.tsx apps/web/src/components/sponsor-shared-artifacts.tsx apps/web/src/components/sponsor-qa.tsx
git commit -m "feat(web): add sponsor company detail — KPI dashboard, shared artifacts, Q&A tabs"
```

---

### Task 6: Pre-Call Briefing Page

**Files:**
- Create: `apps/web/src/app/(sponsor)/companies/[slug]/briefing/page.tsx`

- [ ] **Step 1: Implement pre-call briefing**

```typescript
// apps/web/src/app/(sponsor)/companies/[slug]/briefing/page.tsx
"use client";

import { useEffect, useState } from "react";
import { useParams } from "next/navigation";
import Link from "next/link";
import { getToken } from "@/lib/auth";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

interface BriefingSection {
  title: string;
  content: string;
}

interface BriefingData {
  company_slug: string;
  company_name: string;
  generated_at: string;
  sections: BriefingSection[];
}

export default function PreCallBriefingPage() {
  const params = useParams();
  const slug = params.slug as string;
  const [data, setData] = useState<BriefingData | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchBriefing = async () => {
      const token = getToken();
      try {
        const res = await fetch(`${API}/sponsor/companies/${slug}/briefing`, {
          headers: token ? { Authorization: `Bearer ${token}` } : {},
        });
        if (res.ok) {
          setData(await res.json());
        }
      } finally {
        setLoading(false);
      }
    };
    fetchBriefing();
  }, [slug]);

  if (loading) {
    return <div className="max-w-3xl mx-auto p-8 text-neutral-400">Generating briefing...</div>;
  }

  if (!data) {
    return <div className="max-w-3xl mx-auto p-8 text-neutral-400">Failed to load briefing.</div>;
  }

  return (
    <div className="max-w-3xl mx-auto p-8">
      <Link
        href={`/companies/${slug}`}
        className="text-xs text-blue-600 hover:text-blue-800 mb-4 block"
      >
        &larr; Back to {data.company_name}
      </Link>

      <div className="mb-6">
        <h1 className="text-2xl font-bold text-neutral-800">
          Pre-Call Briefing: {data.company_name}
        </h1>
        <p className="text-xs text-neutral-400 mt-1">
          Generated {new Date(data.generated_at).toLocaleString()}
        </p>
      </div>

      <div className="space-y-6">
        {data.sections.map((section, i) => (
          <div key={i} className="bg-white rounded-lg border border-neutral-200 p-5">
            <h2 className="text-base font-semibold text-neutral-800 mb-2">
              {section.title}
            </h2>
            <div className="text-sm text-neutral-600 whitespace-pre-wrap">
              {section.content}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/app/\(sponsor\)/companies/\[slug\]/briefing/
git commit -m "feat(web): add sponsor pre-call briefing page"
```

---

### Task 7: Verify Full Integration

**Files:** None (verification only)

- [ ] **Step 1: TypeScript check**

Run: `cd apps/web && npx tsc --noEmit`
Expected: No type errors.

- [ ] **Step 2: Lint check**

Run: `cd apps/web && npx eslint src/app/\(sponsor\)/ src/components/sponsor-*.tsx`
Expected: No lint errors.

- [ ] **Step 3: Fix any issues and commit**

```bash
git add -A
git commit -m "fix: resolve lint/type issues from Phase 5d"
```

---
