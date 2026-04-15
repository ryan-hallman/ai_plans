# Slug-to-UUID Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate company slugs as identifiers — every position that identifies a company uses a UUID, and the user's primary company is resolved at login and carried in the session.

**Architecture:** Three-tier identity model: primary company in JWT/session (resolved at login), target company as explicit UUID param in routes, conversation scope for copilot. A `CompanyIdPath` validator rejects non-UUID values with clear error messages. OpenSearch index names still use slugs internally (infrastructure detail, separate migration later).

**Tech Stack:** FastAPI, SQLAlchemy, Alembic, Next.js App Router, Zustand, React Query, PostgreSQL

**Important context:** OpenSearch indexes are named `documents__{slug}` (e.g., `documents__peloton`). The `ToolContext` must carry both `primary_company_id` (the identifier) and `primary_company_slug` (for OpenSearch index naming only). This is an infrastructure detail — the slug is NOT used as an identifier, only as an index key. Migrating OpenSearch index names is out of scope.

---

### Task 1: Backend Foundation — CompanyIdPath Validator and CurrentUser Expansion

**Files:**
- Create: `apps/api/src/prescient/shared/validators.py`
- Modify: `apps/api/src/prescient/auth/base.py`
- Modify: `apps/api/src/prescient/shared/types.py`
- Test: `apps/api/tests/unit/shared/test_validators.py`

- [ ] **Step 1: Write the failing test for CompanyIdPath**

```python
# apps/api/tests/unit/shared/test_validators.py
"""Tests for the shared UUID validators."""
from __future__ import annotations

from uuid import uuid4

import pytest
from fastapi import HTTPException

from prescient.shared.validators import parse_company_id


def test_parse_valid_uuid():
    uid = uuid4()
    result = parse_company_id(str(uid))
    assert result == uid


def test_parse_rejects_slug():
    with pytest.raises(HTTPException) as exc_info:
        parse_company_id("peloton")
    assert exc_info.value.status_code == 422
    assert "not a valid company UUID" in exc_info.value.detail
    assert "peloton" in exc_info.value.detail


def test_parse_rejects_empty_string():
    with pytest.raises(HTTPException):
        parse_company_id("")


def test_parse_rejects_partial_uuid():
    with pytest.raises(HTTPException):
        parse_company_id("a1b2c3d4")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/unit/shared/test_validators.py -v`
Expected: FAIL with "ModuleNotFoundError" or "ImportError"

- [ ] **Step 3: Create the validator module**

```python
# apps/api/src/prescient/shared/validators.py
"""Shared FastAPI validators for common identifier types."""
from __future__ import annotations

from typing import Annotated
from uuid import UUID

from fastapi import Depends, HTTPException

from prescient.shared.types import CompanyId


def parse_company_id(company_id: str) -> CompanyId:
    """Parse and validate a company_id path parameter.

    Rejects non-UUID strings (e.g., slugs) with a 422 and an explicit
    error message so agents and developers immediately know what went wrong.
    """
    try:
        val = UUID(company_id)
    except ValueError:
        raise HTTPException(
            status_code=422,
            detail=(
                f"'{company_id}' is not a valid company UUID. "
                "Company slugs are no longer accepted as identifiers — "
                "use the company's UUID instead."
            ),
        )
    return CompanyId(val)


CompanyIdPath = Annotated[CompanyId, Depends(parse_company_id)]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/unit/shared/test_validators.py -v`
Expected: PASS (4 tests)

- [ ] **Step 5: Expand CurrentUser with primary_company_id**

Edit `apps/api/src/prescient/auth/base.py`:

```python
"""Shared auth types and dependencies.

This module defines the shape of the authenticated principal that every
route and use case depends on. The concrete middleware that populates
`request.state.current_user` lives in a sibling module (`demo.py` or
`workos.py`) and is wired at app-creation time.
"""

from __future__ import annotations

from dataclasses import dataclass

from fastapi import Request

from prescient.shared.types import CompanyId, FundId, UserId


@dataclass(frozen=True, slots=True)
class CurrentUser:
    user_id: UserId
    fund_id: FundId
    primary_company_id: CompanyId


def get_current_user(request: Request) -> CurrentUser:
    """FastAPI dependency — the authenticated user for the current request."""
    user: CurrentUser | None = getattr(request.state, "current_user", None)
    if user is None:
        raise RuntimeError(
            "request.state.current_user is not set — the auth middleware "
            "did not run before this request reached the route."
        )
    return user
```

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/shared/validators.py \
        apps/api/src/prescient/auth/base.py \
        apps/api/tests/unit/shared/test_validators.py
git commit -m "feat: add CompanyIdPath validator and expand CurrentUser with primary_company_id"
```

---

### Task 2: Login & Auth — JWT, Login Use Case, Demo Middleware

**Files:**
- Modify: `apps/api/src/prescient/auth/infrastructure/jwt.py`
- Modify: `apps/api/src/prescient/auth/application/use_cases/login.py`
- Modify: `apps/api/src/prescient/auth/demo.py`
- Test: `apps/api/tests/unit/auth/test_jwt.py` (if exists, update)

- [ ] **Step 1: Expand JWT to carry primary_company_id**

Edit `apps/api/src/prescient/auth/infrastructure/jwt.py` — add `primary_company_id` to token payload:

```python
from datetime import UTC, datetime, timedelta
from typing import Any

from jose import JWTError, jwt


def create_access_token(
    *,
    user_id: str,
    fund_id: str,
    primary_company_id: str,
    secret: str,
    algorithm: str,
    expires_delta: timedelta,
) -> str:
    expire = datetime.now(UTC) + expires_delta
    payload = {
        "sub": user_id,
        "fund_id": fund_id,
        "primary_company_id": primary_company_id,
        "exp": expire,
    }
    return str(jwt.encode(payload, secret, algorithm=algorithm))


def decode_access_token(token: str, *, secret: str, algorithm: str) -> dict[str, Any]:
    try:
        payload: dict[str, Any] = jwt.decode(token, secret, algorithms=[algorithm])
    except JWTError as exc:
        error_str = str(exc).lower()
        if "expired" in error_str:
            msg = "Token has expired"
            raise ValueError(msg) from exc
        msg = "Token is invalid"
        raise ValueError(msg) from exc
    return payload
```

- [ ] **Step 2: Expand LoginResult and login use case**

Edit `apps/api/src/prescient/auth/application/use_cases/login.py`:

Add `primary_company_id: str` to `LoginResult`:

```python
@dataclass(frozen=True)
class LoginResult:
    access_token: str
    user_id: str
    organization_id: str
    fund_id: str
    role: str
    primary_company_id: str
```

In `LoginUser.execute()`, after resolving `fund_id`, resolve the primary company ID. The company and fund share the same slug via the org:

```python
    async def execute(self, request: LoginRequest) -> LoginResult:
        result = await self._session.execute(
            select(CredentialsRow).where(CredentialsRow.email == request.email)
        )
        cred = result.scalar_one_or_none()
        if cred is None:
            raise AuthorizationError("Invalid email or password")

        if not verify_password(request.password, cred.password_hash):
            raise AuthorizationError("Invalid email or password")

        # Look up the user's primary organization via membership.
        membership = (
            await self._session.execute(
                select(MembershipRow)
                .where(MembershipRow.user_id == cred.user_id, MembershipRow.revoked_at.is_(None))
                .limit(1)
            )
        ).scalar_one_or_none()

        organization_id = ""
        fund_id = ""
        primary_company_id = ""
        role = "operator"
        if membership is not None:
            organization_id = membership.organization_id
            role = membership.role

            # Look up the matching fund UUID and company UUID.
            # The fund slug matches the organization slug.
            org = (
                await self._session.execute(
                    select(OrganizationRow.slug).where(
                        OrganizationRow.id == membership.organization_id
                    )
                )
            ).scalar_one_or_none()
            if org is not None:
                fund = (
                    await self._session.execute(select(FundTable.id).where(FundTable.slug == org))
                ).scalar_one_or_none()
                if fund is not None:
                    fund_id = str(fund)

                # Resolve primary company UUID from the same slug
                from prescient.companies.queries import get_company_id_by_slug

                company_id = await get_company_id_by_slug(self._session, org)
                if company_id is not None:
                    primary_company_id = str(company_id)

        token = create_access_token(
            user_id=cred.user_id,
            fund_id=fund_id,
            primary_company_id=primary_company_id,
            secret=self._jwt_secret,
            algorithm=self._jwt_algorithm,
            expires_delta=timedelta(minutes=self._token_expire_minutes),
        )
        return LoginResult(
            access_token=token,
            user_id=cred.user_id,
            organization_id=organization_id,
            fund_id=fund_id,
            role=role,
            primary_company_id=primary_company_id,
        )
```

- [ ] **Step 3: Update DemoAuthMiddleware to resolve primary_company_id**

Edit `apps/api/src/prescient/auth/demo.py`:

Add a company resolution helper and update the middleware to populate `primary_company_id`:

```python
"""Demo auth middleware — resolves user from JWT or falls back to config.

When a valid JWT is present, extracts the `sub` claim to identify the user.
This enables multi-user demo flows (operator Sarah + sponsor Michael).
When no token is present (e.g., health checks), falls back to the configured
demo user.

On MVP day, delete this file and wire `WorkOSAuthMiddleware` in
`create_app` instead. Routes, use cases, and the RLS GUC plumbing in
`db.get_session` stay unchanged.
"""

from __future__ import annotations

import logging
from collections.abc import Awaitable, Callable
from uuid import UUID

from fastapi import Request, Response
from jose import JWTError, jwt
from sqlalchemy import select

from prescient.auth.base import CurrentUser
from prescient.companies.infrastructure.tables.company import CompanyTable
from prescient.config.demo import DemoSettings
from prescient.db import SessionLocal
from prescient.shared.types import CompanyId, FundId, UserId
from prescient.tenants.infrastructure.tables import FundTable

logger = logging.getLogger(__name__)


async def _resolve_fund_id(slug: str) -> FundId:
    async with SessionLocal() as session:
        result = await session.execute(select(FundTable.id).where(FundTable.slug == slug))
        fund_id = result.scalar_one_or_none()
    if fund_id is None:
        raise RuntimeError(
            f"Demo fund slug '{slug}' not found in tenants.funds — "
            "run the seed loader before starting the API."
        )
    return FundId(fund_id)


async def _resolve_primary_company_id(slug: str) -> CompanyId:
    async with SessionLocal() as session:
        result = await session.execute(
            select(CompanyTable.id).where(CompanyTable.slug == slug)
        )
        company_id = result.scalar_one_or_none()
    if company_id is None:
        raise RuntimeError(
            f"Demo company slug '{slug}' not found in companies — "
            "run the seed loader before starting the API."
        )
    return CompanyId(company_id)


class DemoAuthMiddleware:
    """Resolve user from JWT token or fall back to configured demo user.

    The fund UUID and primary company UUID are resolved from `funds.slug`
    and `companies.slug` on the first request and cached on `app.state`.
    """

    def __init__(self, settings: DemoSettings) -> None:
        self._settings = settings
        self._jwt_secret = settings.jwt_secret
        self._jwt_algorithm = settings.jwt_algorithm

    async def __call__(
        self,
        request: Request,
        call_next: Callable[[Request], Awaitable[Response]],
    ) -> Response:
        app_state = request.app.state
        fund_id: FundId | None = getattr(app_state, "demo_fund_id", None)
        if fund_id is None:
            fund_id = await _resolve_fund_id(self._settings.demo_fund_slug)
            app_state.demo_fund_id = fund_id

        primary_company_id: CompanyId | None = getattr(
            app_state, "demo_primary_company_id", None
        )
        if primary_company_id is None:
            primary_company_id = await _resolve_primary_company_id(
                self._settings.demo_fund_slug
            )
            app_state.demo_primary_company_id = primary_company_id

        # Try to extract user_id and primary_company_id from JWT token
        user_id = self._settings.demo_user_id
        token_company_id = primary_company_id
        auth_header = request.headers.get("authorization", "")
        if auth_header.startswith("Bearer "):
            token = auth_header[7:]
            try:
                payload = jwt.decode(token, self._jwt_secret, algorithms=[self._jwt_algorithm])
                user_id = payload.get("sub", user_id)
                if "primary_company_id" in payload:
                    token_company_id = CompanyId(UUID(payload["primary_company_id"]))
            except JWTError:
                pass  # fall back to default demo user

        request.state.current_user = CurrentUser(
            user_id=UserId(user_id),
            fund_id=fund_id,
            primary_company_id=token_company_id,
        )
        return await call_next(request)
```

- [ ] **Step 4: Run existing tests to check nothing is broken**

Run: `cd apps/api && python -m pytest tests/unit/auth/ -v`
Expected: PASS (update any test that constructs `CurrentUser` without `primary_company_id`)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/auth/infrastructure/jwt.py \
        apps/api/src/prescient/auth/application/use_cases/login.py \
        apps/api/src/prescient/auth/demo.py
git commit -m "feat(auth): add primary_company_id to JWT, login result, and demo middleware"
```

---

### Task 3: Frontend Auth — Store and Expose primaryCompanyId

**Files:**
- Modify: `apps/web/src/lib/auth.ts`
- Modify: `apps/web/src/components/auth-provider.tsx`
- Modify: `apps/web/src/app/(auth)/login/page.tsx`

- [ ] **Step 1: Add primaryCompanyId to auth storage**

Edit `apps/web/src/lib/auth.ts`:

```typescript
const TOKEN_KEY = "prescient_token";
const USER_KEY = "prescient_user_id";
const ORG_KEY = "prescient_org_id";
const ROLE_KEY = "prescient_role";
const PRIMARY_COMPANY_KEY = "prescient_primary_company_id";
const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:8000";

export function getToken(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem(TOKEN_KEY);
}

export function getUserId(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem(USER_KEY);
}

export function getOrgId(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem(ORG_KEY);
}

export function getRole(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem(ROLE_KEY);
}

export function getPrimaryCompanyId(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem(PRIMARY_COMPANY_KEY);
}

const SPONSOR_ROLES = new Set(["operating_partner", "pe_analyst", "board_member"]);

export function isSponsor(): boolean {
  const role = getRole();
  return role !== null && SPONSOR_ROLES.has(role);
}

export function setAuth(
  token: string,
  userId: string,
  orgId?: string,
  role?: string,
  primaryCompanyId?: string,
): void {
  localStorage.setItem(TOKEN_KEY, token);
  localStorage.setItem(USER_KEY, userId);
  if (orgId) localStorage.setItem(ORG_KEY, orgId);
  if (role) localStorage.setItem(ROLE_KEY, role);
  if (primaryCompanyId) localStorage.setItem(PRIMARY_COMPANY_KEY, primaryCompanyId);
}

export function clearAuth(): void {
  localStorage.removeItem(TOKEN_KEY);
  localStorage.removeItem(USER_KEY);
  localStorage.removeItem(ORG_KEY);
  localStorage.removeItem(ROLE_KEY);
  localStorage.removeItem(PRIMARY_COMPANY_KEY);
}

export async function login(
  email: string,
  password: string,
): Promise<{
  access_token: string;
  user_id: string;
  organization_id: string;
  fund_id: string;
  role: string;
  primary_company_id: string;
}> {
  const res = await fetch(`${API_BASE}/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email, password }),
  });
  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error((err as { detail?: string }).detail ?? "Login failed");
  }
  return res.json() as Promise<{
    access_token: string;
    user_id: string;
    organization_id: string;
    fund_id: string;
    role: string;
    primary_company_id: string;
  }>;
}
```

- [ ] **Step 2: Add primaryCompanyId to AuthProvider context**

Edit `apps/web/src/components/auth-provider.tsx`:

```typescript
"use client";

import { createContext, useContext, useEffect, useState, type ReactNode } from "react";
import { useRouter } from "next/navigation";

import { clearAuth, getPrimaryCompanyId, getToken, getUserId } from "@/lib/auth";

type AuthContextValue = {
  token: string | null;
  userId: string | null;
  primaryCompanyId: string | null;
  isAuthenticated: boolean;
  logout: () => void;
};

const AuthContext = createContext<AuthContextValue>({
  token: null,
  userId: null,
  primaryCompanyId: null,
  isAuthenticated: false,
  logout: () => {},
});

export function useAuth() {
  return useContext(AuthContext);
}

export function AuthProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(null);
  const [userId, setUserId] = useState<string | null>(null);
  const [primaryCompanyId, setPrimaryCompanyId] = useState<string | null>(null);
  const router = useRouter();

  useEffect(() => {
    setToken(getToken());
    setUserId(getUserId());
    setPrimaryCompanyId(getPrimaryCompanyId());
  }, []);

  const logout = () => {
    clearAuth();
    setToken(null);
    setUserId(null);
    setPrimaryCompanyId(null);
    router.push("/login");
  };

  return (
    <AuthContext.Provider
      value={{ token, userId, primaryCompanyId, isAuthenticated: token !== null, logout }}
    >
      {children}
    </AuthContext.Provider>
  );
}
```

- [ ] **Step 3: Update login page to store primaryCompanyId**

Edit `apps/web/src/app/(auth)/login/page.tsx` — update the `setAuth` call:

Change:
```typescript
setAuth(result.access_token, result.user_id, result.organization_id, result.role);
```
To:
```typescript
setAuth(
  result.access_token,
  result.user_id,
  result.organization_id,
  result.role,
  result.primary_company_id,
);
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/lib/auth.ts \
        apps/web/src/components/auth-provider.tsx \
        "apps/web/src/app/(auth)/login/page.tsx"
git commit -m "feat(web): add primaryCompanyId to auth storage and provider"
```

---

### Task 4: Companies API Routes — Slug to UUID Path Params

**Files:**
- Modify: `apps/api/src/prescient/companies/api/routes.py`

- [ ] **Step 1: Replace slug path params with CompanyIdPath**

Edit `apps/api/src/prescient/companies/api/routes.py`:

```python
"""Companies HTTP routes — read-only for the demo."""

from __future__ import annotations

from typing import Annotated

from fastapi import APIRouter, Depends

from prescient.companies.domain.entities import Company, CompanyRelationship
from prescient.companies.infrastructure.repositories import (
    SqlCompanyRelationshipRepository,
    SqlCompanyRepository,
)
from prescient.context import RequestContext, get_request_context
from prescient.documents.public_queries import (
    DocumentSummaryView,
    KpiSeriesGroup,
    fetch_all_kpi_series,
    fetch_recent_documents,
)
from prescient.knowledge.public_queries import (
    ArtifactSummaryView,
    list_artifacts_for_company,
)
from prescient.shared.errors import NotFoundError
from prescient.shared.types import CompanyId
from prescient.shared.validators import CompanyIdPath

router = APIRouter(prefix="/companies", tags=["companies"])

CtxDep = Annotated[RequestContext, Depends(get_request_context)]


@router.get("", response_model=list[Company])
async def list_companies(ctx: CtxDep) -> list[Company]:
    return await SqlCompanyRepository(ctx.session).list_all()


@router.get("/{company_id}", response_model=Company)
async def get_company(company_id: CompanyIdPath, ctx: CtxDep) -> Company:
    company = await SqlCompanyRepository(ctx.session).get(company_id)
    if company is None:
        raise NotFoundError(f"company '{company_id}' not found")
    return company


@router.get("/{company_id}/relationships", response_model=list[CompanyRelationship])
async def list_relationships(company_id: CompanyIdPath, ctx: CtxDep) -> list[CompanyRelationship]:
    repo = SqlCompanyRepository(ctx.session)
    company = await repo.get(company_id)
    if company is None:
        raise NotFoundError(f"company '{company_id}' not found")
    rel_repo = SqlCompanyRelationshipRepository(ctx.session)
    return await rel_repo.list_for_source(company_id)


@router.get("/{company_id}/kpis", response_model=list[KpiSeriesGroup])
async def list_company_kpis(company_id: CompanyIdPath, ctx: CtxDep) -> list[KpiSeriesGroup]:
    company = await SqlCompanyRepository(ctx.session).get(company_id)
    if company is None:
        raise NotFoundError(f"company '{company_id}' not found")
    return await fetch_all_kpi_series(ctx.session, company_id=company.id)


@router.get("/{company_id}/artifacts", response_model=list[ArtifactSummaryView])
async def list_company_artifacts(
    company_id: CompanyIdPath, ctx: CtxDep
) -> list[ArtifactSummaryView]:
    company = await SqlCompanyRepository(ctx.session).get(company_id)
    if company is None:
        raise NotFoundError(f"company '{company_id}' not found")
    return await list_artifacts_for_company(
        ctx.session,
        company_id=company.id,
        owner_tenant_id=ctx.user.fund_id,
    )


@router.get("/{company_id}/documents", response_model=list[DocumentSummaryView])
async def list_company_documents(
    company_id: CompanyIdPath, ctx: CtxDep, limit: int = 20
) -> list[DocumentSummaryView]:
    company = await SqlCompanyRepository(ctx.session).get(company_id)
    if company is None:
        raise NotFoundError(f"company '{company_id}' not found")
    return await fetch_recent_documents(
        ctx.session,
        company_id=company.id,
        limit=limit,
    )
```

- [ ] **Step 2: Run backend tests**

Run: `cd apps/api && python -m pytest tests/ -k "compan" -v`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/companies/api/routes.py
git commit -m "refactor(companies): replace slug path params with UUID CompanyIdPath"
```

---

### Task 5: Intelligence & Sponsor Routes — Slug to UUID

**Files:**
- Modify: `apps/api/src/prescient/intelligence/api/routes.py`
- Modify: `apps/api/src/prescient/sponsor/api/routes.py`

- [ ] **Step 1: Update strategic-focus route**

In `apps/api/src/prescient/intelligence/api/routes.py`, change the strategic-focus route:

Change the route from `"/strategic-focus/{company_slug}"` to `"/strategic-focus/{company_id}"`.

Replace the function signature and body — instead of filtering by `CompanyTable.slug == company_slug`, filter by `CompanyTable.id == company_id`:

```python
@router.get("/strategic-focus/{company_id}", response_model=StrategicFocusResponse)
async def get_strategic_focus(company_id: CompanyIdPath, ctx: CtxDep) -> StrategicFocusResponse:
    """Return the active StrategicFocus artifact for a company, by UUID."""
    row = (
        await ctx.session.execute(
            select(
                ArtifactTable,
                ArtifactVersionTable,
                CompanyTable,
            )
            .join(
                ArtifactVersionTable,
                ArtifactVersionTable.id == ArtifactTable.active_version_id,
            )
            .join(
                ArtifactSubjectTable,
                and_(
                    ArtifactSubjectTable.artifact_version_id == ArtifactVersionTable.id,
                    ArtifactSubjectTable.is_primary.is_(True),
                ),
            )
            .join(
                CompanyTable,
                CompanyTable.id == ArtifactSubjectTable.subject_id,
            )
            .where(
                and_(
                    ArtifactTable.owner_tenant_id == ctx.user.fund_id,
                    ArtifactTable.artifact_type == "strategic_focus",
                    ArtifactVersionTable.state == "active",
                    CompanyTable.id == company_id,
                )
            )
            .limit(1)
        )
    ).first()
    if row is None:
        raise NotFoundError(f"no active StrategicFocus for company '{company_id}'")
    artifact, version, company = row
    return StrategicFocusResponse(
        artifact_id=artifact.id,
        version_id=version.id,
        version_number=version.version_number,
        title=artifact.title,
        summary=version.summary,
        blocks=list(version.blocks or []),
        approved_at=version.approved_at,
        company_slug=company.slug,
        company_name=company.name,
    )
```

Add the import at the top:
```python
from prescient.shared.validators import CompanyIdPath
```

- [ ] **Step 2: Update sponsor routes**

In `apps/api/src/prescient/sponsor/api/routes.py`:

Add import:
```python
from prescient.shared.validators import CompanyIdPath
```

Change `get_company_kpis`:
- Route: `"/companies/{company_id}/kpis"` (was `/{slug}/kpis`)
- Parameter: `company_id: CompanyIdPath` (was `slug: str`)
- Replace `get_company_by_slug(session, slug)` with direct UUID lookup:

```python
@router.get("/companies/{company_id}/kpis", response_model=SponsorKpisResponse)
async def get_company_kpis(
    company_id: CompanyIdPath,
    session: SessionDep,
    access: AccessDep,
) -> SponsorKpisResponse:
    """All KPI values for a portfolio company visible to this sponsor."""
    if not access.is_sponsor:
        raise AuthorizationError("only sponsors may access sponsor KPI data")
    if not access.can_see_category(DataCategory.KPI_METRICS):
        raise AuthorizationError("KPI data is not visible to your account")

    from prescient.companies.infrastructure.repositories import SqlCompanyRepository

    company = await SqlCompanyRepository(session).get(company_id)
    if company is None:
        raise NotFoundError(f"company '{company_id}' not found")

    # Verify the sponsor has access — check if company's org is in the sponsor's portfolio
    accessible_slugs = set()
    for oid in access.company_org_ids:
        org_slug = await get_org_slug_by_id(session, oid)
        if org_slug:
            accessible_slugs.add(org_slug)
    if company.slug not in accessible_slugs:
        raise AuthorizationError("this company is not in your portfolio")

    company_uuid = str(company.id)

    rows = await get_kpi_values_for_org(session, company_uuid)

    return SponsorKpisResponse(
        company_slug=company.slug,
        company_name=company.name,
        values=[
            SponsorKpiValue(
                value_id=r.id,
                kpi_id=r.kpi_id,
                period_label=r.period_label,
                period_start=r.period_start,
                period_end=r.period_end,
                value=r.value,
                unit=r.unit,
                commentary=r.commentary,
                reported_at=r.reported_at,
            )
            for r in rows
        ],
    )
```

Apply the same pattern to `get_company_briefing`: change route to `"/companies/{company_id}/briefing"`, parameter to `company_id: CompanyIdPath`, replace `get_company_by_slug` with `SqlCompanyRepository(session).get(company_id)`, and verify access via `company.slug`.

Remove the `get_company_by_slug` import from sponsor routes since it's no longer needed there.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/intelligence/api/routes.py \
        apps/api/src/prescient/sponsor/api/routes.py
git commit -m "refactor(intelligence,sponsor): replace slug path params with UUID"
```

---

### Task 6: Board Prep — company_slug to company_id (Including DB Migration)

**Files:**
- Modify: `apps/api/src/prescient/board_prep/domain/entities/board_deck.py`
- Modify: `apps/api/src/prescient/board_prep/infrastructure/tables/board_deck.py`
- Modify: `apps/api/src/prescient/board_prep/infrastructure/repository.py`
- Modify: `apps/api/src/prescient/board_prep/api/routes.py`
- Modify: `apps/api/src/prescient/board_prep/application/assembler.py`
- Create: `apps/api/alembic/versions/20260414_board_deck_company_id.py`

- [ ] **Step 1: Update board_deck domain entity**

In `apps/api/src/prescient/board_prep/domain/entities/board_deck.py`, change `company_slug: str` to `company_id: UUID`:

```python
    id: UUID
    company_id: UUID  # was: company_slug: str
    quarter: str
```

- [ ] **Step 2: Create Alembic migration for board_deck table**

Create `apps/api/alembic/versions/20260414_board_deck_company_id.py`:

```python
"""board_prep: rename company_slug to company_id (UUID)

Revision ID: 20260414_deck_cid
Revises: 20260414_notif_str
Create Date: 2026-04-14
"""

from __future__ import annotations

from collections.abc import Sequence

import sqlalchemy as sa
from alembic import op

revision: str = "20260414_deck_cid"
down_revision: str = "20260414_notif_str"
branch_labels: str | Sequence[str] | None = None
depends_on: str | Sequence[str] | None = None


def upgrade() -> None:
    # 1. Add new company_id column (nullable initially)
    op.add_column(
        "board_decks",
        sa.Column("company_id", sa.dialects.postgresql.UUID(as_uuid=True), nullable=True),
        schema="board_prep",
    )
    # 2. Populate company_id from companies table via slug join
    op.execute(
        """
        UPDATE board_prep.board_decks d
        SET company_id = c.id
        FROM companies.companies c
        WHERE c.slug = d.company_slug
        """
    )
    # 3. Make company_id NOT NULL
    op.alter_column(
        "board_decks",
        "company_id",
        nullable=False,
        schema="board_prep",
    )
    # 4. Create index on company_id
    op.create_index(
        "ix_board_decks_company_id",
        "board_decks",
        ["company_id"],
        schema="board_prep",
    )
    # 5. Drop old company_slug column and its index
    op.drop_index(
        "ix_board_decks_company_slug",
        table_name="board_decks",
        schema="board_prep",
    )
    op.drop_column("board_decks", "company_slug", schema="board_prep")


def downgrade() -> None:
    op.add_column(
        "board_decks",
        sa.Column("company_slug", sa.String(128), nullable=True),
        schema="board_prep",
    )
    op.execute(
        """
        UPDATE board_prep.board_decks d
        SET company_slug = c.slug
        FROM companies.companies c
        WHERE c.id = d.company_id
        """
    )
    op.alter_column(
        "board_decks",
        "company_slug",
        nullable=False,
        schema="board_prep",
    )
    op.create_index(
        "ix_board_decks_company_slug",
        "board_decks",
        ["company_slug"],
        schema="board_prep",
    )
    op.drop_index(
        "ix_board_decks_company_id",
        table_name="board_decks",
        schema="board_prep",
    )
    op.drop_column("board_decks", "company_id", schema="board_prep")
```

- [ ] **Step 3: Update board_deck table model**

In `apps/api/src/prescient/board_prep/infrastructure/tables/board_deck.py`, change:
```python
    company_slug: Mapped[str] = mapped_column(String(128), nullable=False, index=True)
```
to:
```python
    company_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), nullable=False, index=True)
```

- [ ] **Step 4: Update repository**

In `apps/api/src/prescient/board_prep/infrastructure/repository.py`, change all `company_slug` references to `company_id` — in `save()`, `_to_domain()`, and `list_decks()`.

- [ ] **Step 5: Update routes and assembler**

In `apps/api/src/prescient/board_prep/api/routes.py`:
- Change `GenerateDeckRequest.company_slug` to `company_id: UUID`
- Change `DeckResponse.company_slug` to `company_id: str`
- Update `_deck_to_response` and all route handlers to use `company_id`
- Replace `get_company_id_by_slug` calls in the assembler with direct UUID usage

In `apps/api/src/prescient/board_prep/application/assembler.py`:
- The `assemble()` method currently receives `company_slug` and calls `get_company_id_by_slug()`. Change it to receive `company_id: UUID` directly. Remove slug-based lookups. Use `get_company_name_by_slug` → replace with a direct company name query by ID.

Add to `apps/api/src/prescient/companies/queries.py`:
```python
async def get_company_name_by_id(session: AsyncSession, company_id: UUID) -> str | None:
    """Return the company name for a UUID."""
    result = await session.execute(
        select(CompanyTable.name).where(CompanyTable.id == company_id)
    )
    return result.scalar_one_or_none()
```

- [ ] **Step 6: Run migration**

Run: `make docker-migrate`
Expected: Migration applies successfully

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/board_prep/ \
        apps/api/src/prescient/companies/queries.py \
        apps/api/alembic/versions/20260414_board_deck_company_id.py
git commit -m "refactor(board-prep): migrate company_slug to company_id UUID"
```

---

### Task 7: Copilot Backend — ToolContext, Planner, System Prompt

**Files:**
- Modify: `apps/api/src/prescient/intelligence/infrastructure/tools/base.py`
- Modify: `apps/api/src/prescient/intelligence/api/copilot.py`
- Modify: `apps/api/src/prescient/intelligence/api/copilot_stream.py`
- Modify: `apps/api/src/prescient/intelligence/application/planner.py`
- Modify: `apps/api/src/prescient/intelligence/application/planner_stream.py`
- Modify: `apps/api/src/prescient/intelligence/prompts/__init__.py`
- Modify: `apps/api/src/prescient/intelligence/prompts/system.md`

- [ ] **Step 1: Update ToolContext**

In `apps/api/src/prescient/intelligence/infrastructure/tools/base.py`:

```python
@dataclass(frozen=True, slots=True)
class ToolContext:
    """Per-call execution context.

    `primary_company_id` is the identity field. `primary_company_slug` is
    kept solely for OpenSearch index naming (indexes are named
    ``documents__{slug}``). It is NOT an identifier — do not use it for
    lookups, authorization, or display. Migrating OpenSearch index names
    to UUIDs is a separate effort.
    """

    tenant_id: FundId
    user_id: UserId
    scope: ConversationScope
    primary_company_id: CompanyId
    primary_company_slug: str  # OpenSearch index key only — NOT an identifier
    session: AsyncSession
    opensearch: AsyncOpenSearch

    def validate_target(self, target_company_id: CompanyId) -> None:
        """Raise if the target is outside the conversation's visible set."""
        if not self.scope.includes_company(target_company_id):
            raise ValueError(
                f"target_company_id {target_company_id} is not in "
                f"conversation scope (primary={self.scope.primary_company_id})"
            )
```

- [ ] **Step 2: Update copilot request models and route**

In `apps/api/src/prescient/intelligence/api/copilot.py`:

Change `AskRequest`:
```python
class AskRequest(BaseModel):
    model_config = ConfigDict(frozen=True)

    question: str = Field(min_length=1, max_length=4096)
    conversation_id: UUID | None = None
    company_id: UUID | None = None
    focus_company_id: UUID | None = None
    page_context: str | None = Field(default=None, max_length=64)
    selected_item_summary: str | None = Field(default=None, max_length=512)
```

Change `_resolve_company` to accept a `CompanyId` instead of a slug:
```python
async def _resolve_company(
    company_repo: CompanyRepository,
    engagement_repo: FundCompanyEngagementRepository,
    company_id: CompanyId,
    fund_id: FundId,
) -> Company:
    """Look up a company by ID and verify it belongs to the fund's portfolio."""
    company = await company_repo.get(company_id)
    if company is None:
        raise NotFoundError(f"company '{company_id}' not found")

    engagement = await engagement_repo.active_for(fund_id, company_id)
    if engagement is None:
        raise NotFoundError(f"company '{company_id}' not found for this fund")

    return company
```

Update the `ask` route body:
```python
    effective_id = body.focus_company_id or body.company_id

    if effective_id:
        company = await _resolve_company(
            company_repo, engagement_repo, CompanyId(effective_id), ctx.user.fund_id
        )
        scope = await _build_scope(relationship_repo, CompanyId(company.id))
        company_name = company.name
        company_slug = company.slug
    else:
        scope = _build_home_scope(CompanyId(ctx.user.fund_id))
        company_name = "your portfolio"
        company_slug = ""

    # ...

    tool_context = ToolContext(
        tenant_id=ctx.user.fund_id,
        user_id=ctx.user.user_id,
        scope=scope,
        primary_company_id=CompanyId(company.id) if effective_id else ctx.user.primary_company_id,
        primary_company_slug=company_slug,
        session=ctx.session,
        opensearch=opensearch,
    )
```

- [ ] **Step 3: Update copilot_stream.py the same way**

In `apps/api/src/prescient/intelligence/api/copilot_stream.py`:

Change `StreamRequest`:
```python
class StreamRequest(BaseModel):
    model_config = ConfigDict(frozen=True)

    question: str = Field(min_length=1, max_length=4096)
    conversation_id: UUID | None = None
    company_id: UUID | None = None
    focus_company_id: UUID | None = None
    page_context: str | None = Field(default=None, max_length=64)
    selected_item_summary: str | None = Field(default=None, max_length=512)
```

Update route body to use `body.focus_company_id or body.company_id` instead of slug variants. Same ToolContext construction pattern as the `ask` route above.

- [ ] **Step 4: Update system prompt template**

In `apps/api/src/prescient/intelligence/prompts/system.md`, change:
```
This conversation is anchored to **{primary_company_name}** ({primary_company_slug}).
```
to:
```
This conversation is anchored to **{primary_company_name}**.
```

Update `apps/api/src/prescient/intelligence/prompts/__init__.py` — remove `primary_company_slug` parameter from `load_system_prompt`:

```python
def load_system_prompt(
    *,
    primary_company_name: str,
    tool_descriptions: str,
    strategy_context: str = "",
    page_context: str = "",
) -> str:
    """Load and interpolate the system prompt template."""
    template = (_PROMPTS_DIR / "system.md").read_text()
    return template.format(
        primary_company_name=primary_company_name,
        tool_descriptions=tool_descriptions,
        strategy_context=strategy_context,
        page_context=page_context,
    )
```

- [ ] **Step 5: Update planner.py and planner_stream.py**

In both `apps/api/src/prescient/intelligence/application/planner.py` and `planner_stream.py`:

Remove the `primary_company_slug` parameter from `run_planner()` / `run_planner_stream()`. Update the `load_system_prompt` call to remove the slug argument.

Change:
```python
async def run_planner(
    *,
    llm: LLMProviderPort,
    tools: ToolRegistryPort,
    question: str,
    conversation_messages: list[dict[str, Any]],
    primary_company_name: str,
    primary_company_slug: str,
    model: str,
    ...
```
to:
```python
async def run_planner(
    *,
    llm: LLMProviderPort,
    tools: ToolRegistryPort,
    question: str,
    conversation_messages: list[dict[str, Any]],
    primary_company_name: str,
    model: str,
    ...
```

Update the `load_system_prompt` call to remove `primary_company_slug=primary_company_slug`.

- [ ] **Step 6: Update the callers in copilot.py and copilot_stream.py**

Remove `primary_company_slug=company_slug` from the `run_planner()` and `_event_generator()` calls.

In `copilot_stream.py`, update `_event_generator` signature to remove `primary_company_slug` parameter.

- [ ] **Step 7: Update registry.py debug log**

In `apps/api/src/prescient/intelligence/infrastructure/tools/registry.py` line 62, change:
```python
logger.debug("executing tool %s for company=%s", name, context.primary_company_slug)
```
to:
```python
logger.debug("executing tool %s for company=%s", name, context.primary_company_id)
```

- [ ] **Step 8: Run tests**

Run: `cd apps/api && python -m pytest tests/unit/intelligence/ -v`
Expected: Tests need updating (see Task 10). Fix any immediate import errors.

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/intelligence/
git commit -m "refactor(copilot): replace slug-based identity with UUID in ToolContext, planner, and stream"
```

---

### Task 8: Next.js Company Routes — Rename [slug] to [id]

**Files:**
- Rename: `apps/web/src/app/(main)/companies/[slug]/` → `apps/web/src/app/(main)/companies/[id]/`
- Rename: `apps/web/src/app/(sponsor)/portfolio/companies/[slug]/` → `apps/web/src/app/(sponsor)/portfolio/companies/[id]/`
- Rename: `apps/web/src/app/api/companies/[slug]/` → `apps/web/src/app/api/companies/[id]/`
- Modify: All page.tsx and route.ts files within renamed dirs

- [ ] **Step 1: Rename directories**

Use `[companyId]` (not `[id]`) to avoid collision with nested `[id]` segments for documents and artifacts.

```bash
# Main app routes
mv "apps/web/src/app/(main)/companies/[slug]" "apps/web/src/app/(main)/companies/[companyId]"

# Sponsor routes
mv "apps/web/src/app/(sponsor)/portfolio/companies/[slug]" "apps/web/src/app/(sponsor)/portfolio/companies/[companyId]"

# API proxy routes
mv "apps/web/src/app/api/companies/[slug]" "apps/web/src/app/api/companies/[companyId]"
```

- [ ] **Step 2: Update all page files — change params.slug to params.id**

In every `page.tsx` under the renamed dirs, change:
```typescript
params: Promise<{ slug: string }>
```
to:
```typescript
params: Promise<{ id: string }>
```

And change:
```typescript
const { slug } = await params;
```
to:
```typescript
const { id } = await params;
```

**Files to update** (find-and-replace `slug` → `id` in param destructuring and usage):

1. `apps/web/src/app/(main)/companies/[id]/page.tsx` — Replace all `slug` variable references with `id`. This includes:
   - `api.company(id)` (was `api.company(slug)`)
   - `api.strategicFocus(id)` (was `api.strategicFocus(slug)`)
   - `api.companyRelationships(id)` (was `api.companyRelationships(slug)`)
   - All href templates: `/companies/${id}/...` (was `/companies/${slug}/...`)
   - Component props: `<KpiStrip slug={id} />`, `<ArtifactsList slug={id} />`, `<RecentDocuments slug={id} />`, `<CompetitorSidebar companySlug={id} />`
   - Change competitor mapping to use `c.id` instead of `c.slug`

2. `apps/web/src/app/(main)/companies/[id]/strategic-focus/page.tsx`
3. `apps/web/src/app/(main)/companies/[id]/documents/[id]/page.tsx` — **Note:** This has a nested `[id]` for document ID. The company param must use a different name to avoid collision. Rename the company param to `companyId` or restructure the route. Check if this is actually `[docId]` or similar.
4. `apps/web/src/app/(main)/companies/[id]/artifacts/[id]/page.tsx` — Same nested `[id]` issue. Check actual param names.
5. `apps/web/src/app/(sponsor)/portfolio/companies/[id]/page.tsx`
6. `apps/web/src/app/(sponsor)/portfolio/companies/[id]/briefing/page.tsx`
7. `apps/web/src/app/(sponsor)/portfolio/companies/[id]/artifacts/[id]/page.tsx`

**IMPORTANT:** For pages with nested `[id]` params (documents/[id], artifacts/[id]), the company segment should use `[companyId]` instead of `[id]` to avoid param name collision. Rename:
- `apps/web/src/app/(main)/companies/[id]` → `apps/web/src/app/(main)/companies/[companyId]`
- `apps/web/src/app/(sponsor)/portfolio/companies/[id]` → `apps/web/src/app/(sponsor)/portfolio/companies/[companyId]`

And update all param destructuring to `const { companyId } = await params;`

This avoids the ambiguity of two `[id]` segments in the same route.

- [ ] **Step 3: Update all API proxy route files**

In each `route.ts` under `apps/web/src/app/api/companies/[companyId]/`:

Change:
```typescript
{ params }: { params: Promise<{ slug: string }> }
// const { slug } = await params;
// fetch(`${API_BASE}/companies/${encodeURIComponent(slug)}/...`)
```
to:
```typescript
{ params }: { params: Promise<{ companyId: string }> }
// const { companyId } = await params;
// fetch(`${API_BASE}/companies/${encodeURIComponent(companyId)}/...`)
```

Files:
- `apps/web/src/app/api/companies/[companyId]/artifacts/route.ts`
- `apps/web/src/app/api/companies/[companyId]/documents/route.ts`
- `apps/web/src/app/api/companies/[companyId]/kpis/route.ts`

- [ ] **Step 4: Commit**

```bash
git add "apps/web/src/app/(main)/companies/" \
        "apps/web/src/app/(sponsor)/portfolio/companies/" \
        "apps/web/src/app/api/companies/"
git commit -m "refactor(web): rename company routes from [slug] to [companyId] with UUID params"
```

---

### Task 9: Frontend Components — URL Builders, API Client, Queries

**Files:**
- Modify: `apps/web/src/lib/api.ts`
- Modify: `apps/web/src/lib/queries.ts`
- Modify: `apps/web/src/components/nav-actions.tsx`
- Modify: `apps/web/src/components/portfolio-dropdown.tsx`
- Modify: `apps/web/src/components/competitor-sidebar.tsx`
- Modify: `apps/web/src/components/artifacts-list.tsx`
- Modify: `apps/web/src/components/sponsor-portfolio-card.tsx`

- [ ] **Step 1: Update api.ts — change slug-based functions to use id**

In `apps/web/src/lib/api.ts`:

```typescript
export const api = {
  attentionQueue: () => getJSON<AttentionQueueResponse>("/intelligence/attention-queue"),
  strategicFocus: (companyId: string) =>
    getJSON<StrategicFocus>(`/intelligence/strategic-focus/${companyId}`),
  headspaceDigest: () =>
    getJSON<{ items: HeadspaceDigestItem[] }>("/intelligence/headspace/digest"),
  companies: () => getJSON<Company[]>("/companies"),
  company: (companyId: string) => getJSON<Company>(`/companies/${companyId}`),
  companyRelationships: (companyId: string) =>
    getJSON<CompanyRelationship[]>(`/companies/${companyId}/relationships`),
  // ... rest unchanged
};
```

- [ ] **Step 2: Update queries.ts — change slug params to companyId**

In `apps/web/src/lib/queries.ts`:

```typescript
export const queryKeys = {
  companyKpis: (companyId: string) => ["companies", companyId, "kpis"] as const,
  companyArtifacts: (companyId: string) => ["companies", companyId, "artifacts"] as const,
  companyDocuments: (companyId: string) => ["companies", companyId, "documents"] as const,
};

export function useCompanyKpis(companyId: string) {
  return useQuery({
    queryKey: queryKeys.companyKpis(companyId),
    queryFn: () => fetchJSON<KpiSeriesGroup[]>(`/api/companies/${companyId}/kpis`),
  });
}

export function useCompanyArtifacts(companyId: string) {
  return useQuery({
    queryKey: queryKeys.companyArtifacts(companyId),
    queryFn: () => fetchJSON<ArtifactSummary[]>(`/api/companies/${companyId}/artifacts`),
  });
}

export function useCompanyDocuments(companyId: string) {
  return useQuery({
    queryKey: queryKeys.companyDocuments(companyId),
    queryFn: () => fetchJSON<DocumentSummary[]>(`/api/companies/${companyId}/documents`),
  });
}
```

- [ ] **Step 3: Update nav-actions.tsx**

Change interface:
```typescript
interface NavActionsProps {
  companies?: { id: string; name: string }[];
}
```

Change link:
```typescript
{companies.map((c) => (
  <DropdownMenuItem key={c.id} asChild>
    <Link href={`/companies/${c.id}`} className="gap-2">
      <Building2 className="h-3.5 w-3.5" />
      {c.name}
    </Link>
  </DropdownMenuItem>
))}
```

- [ ] **Step 4: Update portfolio-dropdown.tsx**

Change interface:
```typescript
interface PortfolioDropdownProps {
  companies: { id: string; name: string }[];
}
```

Change link:
```typescript
{companies.map((c) => (
  <DropdownMenuItem key={c.id} asChild>
    <Link href={`/companies/${c.id}`}>{c.name}</Link>
  </DropdownMenuItem>
))}
```

- [ ] **Step 5: Update competitor-sidebar.tsx**

Change interface:
```typescript
interface CompetitorInfo {
  id: string;
  name: string;
  relationship_type: string;
}

interface CompetitorSidebarProps {
  competitors: CompetitorInfo[];
  companyId: string;
}
```

Change link:
```typescript
<Link href={`/companies/${c.id}`} ...>
```

Change key: `key={`${c.id}-${c.relationship_type}`}`

- [ ] **Step 6: Update artifacts-list.tsx**

Change prop:
```typescript
export function ArtifactsList({ companyId }: { companyId: string }) {
  const { data, isLoading, error } = useCompanyArtifacts(companyId);
```

Change link:
```typescript
href={`/companies/${companyId}/artifacts/${artifact.id}`}
```

- [ ] **Step 7: Update sponsor-portfolio-card.tsx**

Change interface:
```typescript
interface PortfolioCompanyCardProps {
  id: string;  // was: slug
  name: string;
  // ... rest same
}
```

Change link:
```typescript
<Link href={`/portfolio/companies/${id}`} className="block">
```

- [ ] **Step 8: Update parent components that pass these props**

Search for any parent that passes `slug` to these components and update to pass `id` instead. Key places:
- The company overview page (already updated in Task 8)
- Any layout or page that renders `NavActions`, `PortfolioDropdown`

- [ ] **Step 9: Commit**

```bash
git add apps/web/src/lib/api.ts \
        apps/web/src/lib/queries.ts \
        apps/web/src/components/nav-actions.tsx \
        apps/web/src/components/portfolio-dropdown.tsx \
        apps/web/src/components/competitor-sidebar.tsx \
        apps/web/src/components/artifacts-list.tsx \
        apps/web/src/components/sponsor-portfolio-card.tsx
git commit -m "refactor(web): replace slug with UUID in all company URL builders and API client"
```

---

### Task 10: Frontend Copilot — chat-stream, copilot-store, use-copilot-context

**Files:**
- Modify: `apps/web/src/lib/chat-stream.ts`
- Modify: `apps/web/src/stores/copilot-store.ts`
- Modify: `apps/web/src/hooks/use-copilot-context.ts`

- [ ] **Step 1: Update chat-stream.ts**

```typescript
export interface StreamChatOptions {
  question: string;
  conversationId?: string;
  authToken?: string;
  companyId?: string;
  focusCompanyId?: string;
  pageContext?: string;
  selectedItemSummary?: string;
}

export async function* streamChat(options: StreamChatOptions): AsyncGenerator<ChatEvent> {
  const headers: Record<string, string> = { "Content-Type": "application/json" };
  if (options.authToken) headers["Authorization"] = `Bearer ${options.authToken}`;

  const body: Record<string, unknown> = {
    question: options.question,
  };
  if (options.conversationId) body.conversation_id = options.conversationId;
  if (options.companyId) body.company_id = options.companyId;
  if (options.focusCompanyId) body.focus_company_id = options.focusCompanyId;
  if (options.pageContext) body.page_context = options.pageContext;
  if (options.selectedItemSummary) body.selected_item_summary = options.selectedItemSummary;

  // ... rest unchanged
}
```

- [ ] **Step 2: Update copilot-store.ts**

Change `PageContext`:
```typescript
export interface PageContext {
  pageType: PageType;
  companyId?: string;  // was: companySlug
  selectedItemId?: string;
  selectedItemSummary?: string;
}
```

- [ ] **Step 3: Update use-copilot-context.ts**

```typescript
export function useCopilotContext(ctx: PageContext): void {
  const setPageContext = useCopilotStore((s) => s.setPageContext);

  useEffect(() => {
    setPageContext(ctx);
    return () => setPageContext(null);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [ctx.pageType, ctx.companyId, ctx.selectedItemId, ctx.selectedItemSummary, setPageContext]);
}
```

- [ ] **Step 4: Update all callers of useCopilotContext**

Search for all components that call `useCopilotContext` and change `companySlug:` to `companyId:` in the context object they pass.

Run: `grep -rn "companySlug" apps/web/src/` to find any remaining references.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/lib/chat-stream.ts \
        apps/web/src/stores/copilot-store.ts \
        apps/web/src/hooks/use-copilot-context.ts
git commit -m "refactor(copilot): replace companySlug with companyId in frontend copilot"
```

---

### Task 11: Update All Backend Tests

**Files:**
- Modify: `apps/api/tests/unit/intelligence/test_planner.py`
- Modify: `apps/api/tests/unit/intelligence/test_system_prompt.py`
- Modify: `apps/api/tests/unit/knowledge_mgmt/test_search_knowledge_base_tool.py`
- Modify: `apps/api/tests/integration/intelligence/test_tools.py`
- Modify: Any other test that constructs `CurrentUser` or `ToolContext`

- [ ] **Step 1: Update test_planner.py**

In all `run_planner()` calls, remove the `primary_company_slug="testco"` argument.

Find all occurrences of `primary_company_slug` in this file and remove them.

- [ ] **Step 2: Update test_system_prompt.py**

Remove `primary_company_slug="peloton"` from all `load_system_prompt()` calls.

Update the assertion that checks the prompt content — it should look for `**Peloton**` without the slug in parens.

- [ ] **Step 3: Update test_search_knowledge_base_tool.py**

Change ToolContext construction:
```python
ToolContext(
    tenant_id=...,
    user_id=...,
    scope=scope,
    primary_company_id=scope.primary_company_id,
    primary_company_slug="acme-corp",
    session=AsyncMock(),
    opensearch=AsyncMock(),
)
```

- [ ] **Step 4: Update test_tools.py (integration)**

Change ToolContext construction to include `primary_company_id=CompanyId(company.id)` alongside `primary_company_slug=company.slug`.

- [ ] **Step 5: Update any test that constructs CurrentUser**

Search: `grep -rn "CurrentUser(" apps/api/tests/`

Add `primary_company_id=CompanyId(uuid4())` to all CurrentUser constructors.

- [ ] **Step 6: Run full test suite**

Run: `cd apps/api && python -m pytest tests/ -v`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git add apps/api/tests/
git commit -m "test: update all tests for slug-to-UUID migration"
```

---

### Task 12: Briefing Route Cleanup

**Files:**
- Modify: `apps/api/src/prescient/briefing/api/routes.py`

- [ ] **Step 1: Update briefing route**

The briefing route (`GET /briefing/today`) uses `get_company_id_by_slug` internally — it receives `organization_id` as a query param, resolves it to a slug, then resolves the slug to a company UUID. This indirect chain can be simplified.

The route already receives `organization_id` as a string. The resolution chain is:
1. `get_org_slug_by_id(session, organization_id)` → slug
2. `get_company_id_by_slug(session, slug)` → company UUID

This can stay as-is since the briefing route doesn't use company slugs as identifiers in its API contract — it uses `organization_id` as the query param. The internal slug resolution is an implementation detail. No changes needed to the route signature.

However, if we want to be thorough, we can add a `get_company_id_by_org_id` query that does the join directly. This is optional and can be deferred.

- [ ] **Step 2: Verify no slug params in public API**

Run: `grep -n "slug" apps/api/src/prescient/briefing/api/routes.py`

Confirm that `slug` only appears in internal resolution logic, not in route signatures or response models.

- [ ] **Step 3: Commit (if changes made)**

```bash
git add apps/api/src/prescient/briefing/api/routes.py
git commit -m "refactor(briefing): clean up slug-based resolution"
```

---

### Task 13: Full Smoke Test

- [ ] **Step 1: Run backend tests**

```bash
cd apps/api && python -m pytest tests/ -v
```

Expected: All PASS

- [ ] **Step 2: Run frontend lint/typecheck**

```bash
cd apps/web && npx tsc --noEmit
cd apps/web && npx next lint
```

Expected: No errors

- [ ] **Step 3: Search for remaining slug references that should be UUIDs**

```bash
# Backend — any route handler still using slug as an identifier
grep -rn "/{slug}" apps/api/src/prescient/*/api/routes.py
# Should return 0 results

# Frontend — any URL builder still using .slug for routing
grep -rn "companies/\${.*\.slug}" apps/web/src/
# Should return 0 results

# Frontend — any companySlug references
grep -rn "companySlug" apps/web/src/
# Should return 0 results (except possibly display-only contexts)
```

- [ ] **Step 4: Run full stack in Docker and verify**

```bash
make docker-migrate
make docker-seed-full
# Start the dev servers and verify:
# - Login works and returns primary_company_id
# - Company pages load at /companies/<uuid>
# - Copilot stream works with company_id params
# - Board prep generates with company_id
# - Notification bell still works
```

- [ ] **Step 5: Final commit (if any fixups needed)**

```bash
git add -A
git commit -m "fix: address smoke test findings from slug-to-UUID migration"
```
