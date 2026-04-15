# Phase 1: Foundation — Organization, Artifacts, Auth & Scaffold

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the foundational data model, auth, and scaffold for the operator-first Prescient OS — Organization & Access context, Artifact Store, email/password JWT auth, and base API + frontend shell.

**Architecture:** Greenfield bounded contexts built on the existing shared kernel (outbox, UoW, events, types, errors). Existing companies context carries forward with minor evolution. Tenants context is replaced by Organization & Access. Knowledge context is replaced by Artifact Store. New auth context for email/password + JWT. Frontend gets a new shell designed for operator daily workflow.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Alembic, Next.js 15, React 19, Tailwind CSS, shadcn/ui, PostgreSQL 16, OpenSearch, passlib + python-jose for auth.

**Spec:** `docs/superpowers/specs/2026-04-12-prescient-os-redesign.md`

**Existing code survey:** The shared kernel (`apps/api/src/prescient/shared/`) is solid and carries forward: `db_base.py`, `events.py`, `outbox.py`, `uow.py`, `types.py`, `errors.py`, `metadata.py`. The companies context domain model is rich and useful. Docker, alembic, Makefile all carry forward. The intelligence, knowledge, tenants, and action_items contexts will be reshaped or replaced.

---

## File Structure

### New Files (API)

```
apps/api/src/prescient/
├── shared/
│   └── types.py                          # Add new ID types (OrganizationId, etc.)
├── organizations/                         # NEW — replaces tenants context
│   ├── __init__.py
│   ├── domain/
│   │   ├── __init__.py
│   │   ├── entities/
│   │   │   ├── __init__.py
│   │   │   ├── organization.py           # Top-level tenant (PE firm or independent company)
│   │   │   ├── user.py                   # User with role enum
│   │   │   ├── membership.py             # User-to-org role binding
│   │   │   ├── portfolio_link.py         # PE parent ↔ portfolio company relationship
│   │   │   └── visibility_rule.py        # What flows up, what's gated
│   │   └── errors.py
│   ├── application/
│   │   ├── __init__.py
│   │   └── use_cases/
│   │       ├── __init__.py
│   │       ├── create_organization.py
│   │       ├── invite_user.py
│   │       └── manage_visibility.py
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   ├── tables/
│   │   │   ├── __init__.py
│   │   │   ├── organization.py
│   │   │   ├── user.py
│   │   │   ├── membership.py
│   │   │   ├── portfolio_link.py
│   │   │   └── visibility_rule.py
│   │   └── repositories/
│   │       ├── __init__.py
│   │       ├── sql_organization_repository.py
│   │       └── sql_user_repository.py
│   └── api/
│       ├── __init__.py
│       └── routes.py
├── artifacts/                             # NEW — replaces knowledge context
│   ├── __init__.py
│   ├── domain/
│   │   ├── __init__.py
│   │   ├── entities/
│   │   │   ├── __init__.py
│   │   │   ├── artifact.py               # Aggregate root with type enum
│   │   │   ├── artifact_version.py       # Immutable version
│   │   │   ├── artifact_block.py         # Content block within version
│   │   │   ├── artifact_citation.py      # Source citation for block
│   │   │   ├── artifact_reference.py     # NEW — cross-reference between artifacts
│   │   │   └── artifact_subject.py       # Polymorphic subject link
│   │   ├── state/
│   │   │   ├── __init__.py
│   │   │   └── artifact_state_machine.py
│   │   └── errors.py
│   ├── application/
│   │   ├── __init__.py
│   │   └── use_cases/
│   │       ├── __init__.py
│   │       ├── create_artifact.py
│   │       ├── get_artifact.py
│   │       └── add_reference.py
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   ├── tables/
│   │   │   ├── __init__.py
│   │   │   ├── artifact.py
│   │   │   ├── artifact_version.py
│   │   │   ├── artifact_reference.py
│   │   │   └── artifact_subject.py
│   │   └── repositories/
│   │       ├── __init__.py
│   │       └── sql_artifact_repository.py
│   └── api/
│       ├── __init__.py
│       └── routes.py
├── auth/                                  # NEW — email/password + JWT
│   ├── __init__.py
│   ├── domain/
│   │   ├── __init__.py
│   │   └── entities/
│   │       ├── __init__.py
│   │       └── credentials.py
│   ├── application/
│   │   ├── __init__.py
│   │   └── use_cases/
│   │       ├── __init__.py
│   │       ├── register.py
│   │       └── login.py
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   ├── tables/
│   │   │   ├── __init__.py
│   │   │   └── credentials.py
│   │   ├── hashing.py
│   │   └── jwt.py
│   └── api/
│       ├── __init__.py
│       └── routes.py
```

### New Files (Frontend)

```
apps/web/src/
├── app/
│   ├── layout.tsx                        # MODIFY — new shell with auth
│   ├── page.tsx                          # MODIFY — redirect to /brief or login
│   ├── login/
│   │   └── page.tsx                      # NEW — login page
│   ├── onboarding/
│   │   └── page.tsx                      # NEW — placeholder for Phase 2
│   └── brief/
│       └── page.tsx                      # NEW — morning brief placeholder
├── components/
│   ├── nav.tsx                           # MODIFY — operator-first navigation
│   └── auth-provider.tsx                 # NEW — JWT auth context
├── lib/
│   ├── api.ts                            # MODIFY — add auth headers
│   └── auth.ts                           # NEW — token storage, login/register calls
```

### New Files (Tests)

```
tests/
├── organizations/
│   ├── test_organization_domain.py
│   ├── test_membership_domain.py
│   ├── test_visibility_domain.py
│   ├── test_organization_api.py
│   └── test_user_api.py
├── artifacts/
│   ├── test_artifact_domain.py
│   ├── test_artifact_references.py
│   ├── test_artifact_state_machine.py
│   └── test_artifact_api.py
├── auth/
│   ├── test_hashing.py
│   ├── test_jwt.py
│   ├── test_register.py
│   └── test_login.py
```

### Modified Files

```
apps/api/src/prescient/shared/types.py    # Add OrganizationId, VisibilityRuleId
apps/api/src/prescient/shared/metadata.py  # Register new context tables
apps/api/src/prescient/main.py             # New routers, JWT auth middleware
apps/api/src/prescient/config/base.py      # Add JWT secret, token expiry settings
apps/api/pyproject.toml                    # Add passlib, python-jose deps
```

---

## Task 1: Add Auth Dependencies and Config

**Files:**
- Modify: `apps/api/pyproject.toml`
- Modify: `apps/api/src/prescient/config/base.py`
- Modify: `apps/api/src/prescient/shared/types.py`

- [ ] **Step 1: Add auth dependencies to pyproject.toml**

Add `passlib[bcrypt]` and `python-jose[cryptography]` to the dependencies list in `apps/api/pyproject.toml`:

```toml
# Add to [project] dependencies list:
"passlib[bcrypt]>=1.7.4",
"python-jose[cryptography]>=3.3.0",
```

- [ ] **Step 2: Add JWT config to Settings**

In `apps/api/src/prescient/config/base.py`, add:

```python
jwt_secret: str = "dev-secret-change-in-production"
jwt_algorithm: str = "HS256"
jwt_access_token_expire_minutes: int = 480  # 8 hours
```

- [ ] **Step 3: Add new ID types to shared/types.py**

In `apps/api/src/prescient/shared/types.py`, add:

```python
OrganizationId = NewType("OrganizationId", UUID)
MembershipId = NewType("MembershipId", UUID)
PortfolioLinkId = NewType("PortfolioLinkId", UUID)
VisibilityRuleId = NewType("VisibilityRuleId", UUID)
CredentialsId = NewType("CredentialsId", UUID)
ArtifactReferenceId = NewType("ArtifactReferenceId", UUID)
```

- [ ] **Step 4: Install dependencies**

Run: `cd apps/api && uv sync`
Expected: Dependencies install without errors.

- [ ] **Step 5: Commit**

```bash
git add apps/api/pyproject.toml apps/api/src/prescient/config/base.py apps/api/src/prescient/shared/types.py
git commit -m "feat: add auth deps, JWT config, and new ID types for Phase 1"
```

---

## Task 2: Organization Domain Entities

**Files:**
- Create: `apps/api/src/prescient/organizations/__init__.py`
- Create: `apps/api/src/prescient/organizations/domain/__init__.py`
- Create: `apps/api/src/prescient/organizations/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/organizations/domain/entities/organization.py`
- Create: `apps/api/src/prescient/organizations/domain/entities/user.py`
- Create: `apps/api/src/prescient/organizations/domain/entities/membership.py`
- Create: `apps/api/src/prescient/organizations/domain/entities/portfolio_link.py`
- Create: `apps/api/src/prescient/organizations/domain/entities/visibility_rule.py`
- Create: `apps/api/src/prescient/organizations/domain/errors.py`
- Test: `tests/organizations/test_organization_domain.py`

- [ ] **Step 1: Create context package structure**

Create empty `__init__.py` files for the organizations context:

```
apps/api/src/prescient/organizations/__init__.py
apps/api/src/prescient/organizations/domain/__init__.py
apps/api/src/prescient/organizations/domain/entities/__init__.py
```

- [ ] **Step 2: Write failing test for Organization entity**

Create `tests/organizations/test_organization_domain.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.organizations.domain.entities.organization import (
    Organization,
    OrganizationType,
    OrganizationStatus,
)
from prescient.shared.types import OrganizationId


class TestOrganization:
    def test_create_pe_firm(self) -> None:
        org_id = OrganizationId(uuid4())
        now = datetime.now(UTC)
        org = Organization(
            id=org_id,
            slug="meridian-capital",
            name="Meridian Capital Partners",
            org_type=OrganizationType.PE_FIRM,
            status=OrganizationStatus.ACTIVE,
            industry=None,
            description="Lower middle-market PE firm",
            website=None,
            created_at=now,
            updated_at=now,
        )
        assert org.slug == "meridian-capital"
        assert org.org_type == OrganizationType.PE_FIRM
        assert org.is_sponsor is True

    def test_create_portfolio_company(self) -> None:
        org_id = OrganizationId(uuid4())
        now = datetime.now(UTC)
        org = Organization(
            id=org_id,
            slug="olaplex",
            name="Olaplex Holdings",
            org_type=OrganizationType.PORTFOLIO_COMPANY,
            status=OrganizationStatus.ACTIVE,
            industry="Beauty & Personal Care",
            description=None,
            website="https://olaplex.com",
            created_at=now,
            updated_at=now,
        )
        assert org.org_type == OrganizationType.PORTFOLIO_COMPANY
        assert org.is_sponsor is False

    def test_create_independent_company(self) -> None:
        org_id = OrganizationId(uuid4())
        now = datetime.now(UTC)
        org = Organization(
            id=org_id,
            slug="acme-corp",
            name="Acme Corp",
            org_type=OrganizationType.INDEPENDENT,
            status=OrganizationStatus.ACTIVE,
            industry="Manufacturing",
            description=None,
            website=None,
            created_at=now,
            updated_at=now,
        )
        assert org.org_type == OrganizationType.INDEPENDENT
        assert org.is_sponsor is False
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/organizations/test_organization_domain.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement Organization entity**

Create `apps/api/src/prescient/organizations/domain/entities/organization.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field

from prescient.shared.types import OrganizationId


class OrganizationType(StrEnum):
    PE_FIRM = "pe_firm"
    PORTFOLIO_COMPANY = "portfolio_company"
    INDEPENDENT = "independent"
    CREDIT_FUND = "credit_fund"


class OrganizationStatus(StrEnum):
    ACTIVE = "active"
    ONBOARDING = "onboarding"
    SUSPENDED = "suspended"
    CHURNED = "churned"


class Organization(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: OrganizationId
    slug: str = Field(min_length=1, max_length=64)
    name: str = Field(min_length=1, max_length=256)
    org_type: OrganizationType
    status: OrganizationStatus
    industry: str | None = None
    description: str | None = None
    website: str | None = None
    created_at: datetime
    updated_at: datetime

    @property
    def is_sponsor(self) -> bool:
        return self.org_type in {OrganizationType.PE_FIRM, OrganizationType.CREDIT_FUND}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/organizations/test_organization_domain.py::TestOrganization -v`
Expected: 3 PASSED

- [ ] **Step 6: Write failing test for User entity**

Append to `tests/organizations/test_organization_domain.py`:

```python
from prescient.organizations.domain.entities.user import (
    User,
    UserRole,
)
from prescient.shared.types import UserId


class TestUser:
    def test_create_operator(self) -> None:
        now = datetime.now(UTC)
        user = User(
            id=UserId("ryan"),
            email="ryan@olaplex.com",
            display_name="Ryan Hallman",
            role=UserRole.OPERATOR,
            primary_org_id=OrganizationId(uuid4()),
            is_active=True,
            created_at=now,
        )
        assert user.role == UserRole.OPERATOR
        assert user.is_active is True

    def test_create_pe_analyst(self) -> None:
        now = datetime.now(UTC)
        user = User(
            id=UserId("analyst1"),
            email="analyst@meridian.com",
            display_name="Jane Smith",
            role=UserRole.PE_ANALYST,
            primary_org_id=OrganizationId(uuid4()),
            is_active=True,
            created_at=now,
        )
        assert user.role == UserRole.PE_ANALYST

    def test_all_roles_exist(self) -> None:
        expected = {"operator", "functional_leader", "operating_partner", "pe_analyst", "board_member", "admin"}
        assert {r.value for r in UserRole} == expected
```

- [ ] **Step 7: Implement User entity**

Create `apps/api/src/prescient/organizations/domain/entities/user.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field

from prescient.shared.types import OrganizationId, UserId


class UserRole(StrEnum):
    OPERATOR = "operator"
    FUNCTIONAL_LEADER = "functional_leader"
    OPERATING_PARTNER = "operating_partner"
    PE_ANALYST = "pe_analyst"
    BOARD_MEMBER = "board_member"
    ADMIN = "admin"


class User(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: UserId
    email: str = Field(min_length=1, max_length=256)
    display_name: str = Field(min_length=1, max_length=128)
    role: UserRole
    primary_org_id: OrganizationId
    is_active: bool = True
    created_at: datetime
```

- [ ] **Step 8: Run tests to verify User passes**

Run: `cd apps/api && python -m pytest tests/organizations/test_organization_domain.py -v`
Expected: 6 PASSED

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/organizations/ tests/organizations/
git commit -m "feat: add Organization and User domain entities with role model"
```

---

## Task 3: Membership, Portfolio Link, and Visibility Rule Entities

**Files:**
- Create: `apps/api/src/prescient/organizations/domain/entities/membership.py`
- Create: `apps/api/src/prescient/organizations/domain/entities/portfolio_link.py`
- Create: `apps/api/src/prescient/organizations/domain/entities/visibility_rule.py`
- Create: `apps/api/src/prescient/organizations/domain/errors.py`
- Test: `tests/organizations/test_membership_domain.py`
- Test: `tests/organizations/test_visibility_domain.py`

- [ ] **Step 1: Write failing test for Membership**

Create `tests/organizations/test_membership_domain.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.organizations.domain.entities.membership import (
    Membership,
    MembershipScope,
)
from prescient.organizations.domain.entities.user import UserRole
from prescient.shared.types import MembershipId, OrganizationId, UserId


class TestMembership:
    def test_create_membership(self) -> None:
        now = datetime.now(UTC)
        m = Membership(
            id=MembershipId(uuid4()),
            user_id=UserId("ryan"),
            organization_id=OrganizationId(uuid4()),
            role=UserRole.OPERATOR,
            scope=MembershipScope.FULL,
            granted_at=now,
            revoked_at=None,
        )
        assert m.is_active is True

    def test_revoked_membership_is_inactive(self) -> None:
        now = datetime.now(UTC)
        m = Membership(
            id=MembershipId(uuid4()),
            user_id=UserId("ryan"),
            organization_id=OrganizationId(uuid4()),
            role=UserRole.OPERATOR,
            scope=MembershipScope.FULL,
            granted_at=now,
            revoked_at=now,
        )
        assert m.is_active is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/organizations/test_membership_domain.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement Membership entity**

Create `apps/api/src/prescient/organizations/domain/entities/membership.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict

from prescient.organizations.domain.entities.user import UserRole
from prescient.shared.types import MembershipId, OrganizationId, UserId


class MembershipScope(StrEnum):
    FULL = "full"
    READ_ONLY = "read_only"
    KPI_ONLY = "kpi_only"


class Membership(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: MembershipId
    user_id: UserId
    organization_id: OrganizationId
    role: UserRole
    scope: MembershipScope
    granted_at: datetime
    revoked_at: datetime | None = None

    @property
    def is_active(self) -> bool:
        return self.revoked_at is None
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/organizations/test_membership_domain.py -v`
Expected: 2 PASSED

- [ ] **Step 5: Write failing test for PortfolioLink**

Append to `tests/organizations/test_membership_domain.py`:

```python
from prescient.organizations.domain.entities.portfolio_link import (
    PortfolioLink,
    PortfolioLinkStatus,
    BillingModel,
)
from prescient.shared.types import PortfolioLinkId


class TestPortfolioLink:
    def test_create_link(self) -> None:
        now = datetime.now(UTC)
        link = PortfolioLink(
            id=PortfolioLinkId(uuid4()),
            sponsor_org_id=OrganizationId(uuid4()),
            company_org_id=OrganizationId(uuid4()),
            status=PortfolioLinkStatus.ACTIVE,
            billing_model=BillingModel.SPONSOR_FUNDED,
            linked_at=now,
            unlinked_at=None,
        )
        assert link.is_active is True
        assert link.billing_model == BillingModel.SPONSOR_FUNDED

    def test_all_billing_models(self) -> None:
        expected = {"sponsor_funded", "company_funded", "hybrid"}
        assert {b.value for b in BillingModel} == expected
```

- [ ] **Step 6: Implement PortfolioLink entity**

Create `apps/api/src/prescient/organizations/domain/entities/portfolio_link.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict

from prescient.shared.types import OrganizationId, PortfolioLinkId


class PortfolioLinkStatus(StrEnum):
    ACTIVE = "active"
    PENDING = "pending"
    UNLINKED = "unlinked"


class BillingModel(StrEnum):
    SPONSOR_FUNDED = "sponsor_funded"
    COMPANY_FUNDED = "company_funded"
    HYBRID = "hybrid"


class PortfolioLink(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: PortfolioLinkId
    sponsor_org_id: OrganizationId
    company_org_id: OrganizationId
    status: PortfolioLinkStatus
    billing_model: BillingModel
    linked_at: datetime
    unlinked_at: datetime | None = None

    @property
    def is_active(self) -> bool:
        return self.status == PortfolioLinkStatus.ACTIVE and self.unlinked_at is None
```

- [ ] **Step 7: Run test to verify PortfolioLink passes**

Run: `cd apps/api && python -m pytest tests/organizations/test_membership_domain.py -v`
Expected: 4 PASSED

- [ ] **Step 8: Write failing test for VisibilityRule**

Create `tests/organizations/test_visibility_domain.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.organizations.domain.entities.visibility_rule import (
    VisibilityRule,
    DataCategory,
    FlowDirection,
    GateType,
)
from prescient.shared.types import OrganizationId, PortfolioLinkId, VisibilityRuleId


class TestVisibilityRule:
    def test_auto_flow_kpis(self) -> None:
        now = datetime.now(UTC)
        rule = VisibilityRule(
            id=VisibilityRuleId(uuid4()),
            portfolio_link_id=PortfolioLinkId(uuid4()),
            data_category=DataCategory.KPI_METRICS,
            direction=FlowDirection.COMPANY_TO_SPONSOR,
            gate_type=GateType.AUTOMATIC,
            created_at=now,
        )
        assert rule.requires_approval is False

    def test_gated_narrative(self) -> None:
        now = datetime.now(UTC)
        rule = VisibilityRule(
            id=VisibilityRuleId(uuid4()),
            portfolio_link_id=PortfolioLinkId(uuid4()),
            data_category=DataCategory.NARRATIVE,
            direction=FlowDirection.COMPANY_TO_SPONSOR,
            gate_type=GateType.OPERATOR_APPROVED,
            created_at=now,
        )
        assert rule.requires_approval is True

    def test_all_data_categories(self) -> None:
        expected = {"kpi_metrics", "narrative", "decisions", "action_items", "intelligence_signals", "briefings"}
        assert {c.value for c in DataCategory} == expected

    def test_default_rules_for_tier_b(self) -> None:
        link_id = PortfolioLinkId(uuid4())
        rules = VisibilityRule.default_tier_b(link_id)
        auto_categories = {r.data_category for r in rules if r.gate_type == GateType.AUTOMATIC}
        gated_categories = {r.data_category for r in rules if r.gate_type == GateType.OPERATOR_APPROVED}
        assert DataCategory.KPI_METRICS in auto_categories
        assert DataCategory.NARRATIVE in gated_categories
        assert DataCategory.DECISIONS in gated_categories
```

- [ ] **Step 9: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/organizations/test_visibility_domain.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 10: Implement VisibilityRule entity**

Create `apps/api/src/prescient/organizations/domain/entities/visibility_rule.py`:

```python
from __future__ import annotations

from datetime import datetime, UTC
from enum import StrEnum
from uuid import uuid4

from pydantic import BaseModel, ConfigDict

from prescient.shared.types import PortfolioLinkId, VisibilityRuleId


class DataCategory(StrEnum):
    KPI_METRICS = "kpi_metrics"
    NARRATIVE = "narrative"
    DECISIONS = "decisions"
    ACTION_ITEMS = "action_items"
    INTELLIGENCE_SIGNALS = "intelligence_signals"
    BRIEFINGS = "briefings"


class FlowDirection(StrEnum):
    COMPANY_TO_SPONSOR = "company_to_sponsor"
    SPONSOR_TO_COMPANY = "sponsor_to_company"
    BIDIRECTIONAL = "bidirectional"


class GateType(StrEnum):
    AUTOMATIC = "automatic"
    OPERATOR_APPROVED = "operator_approved"
    BLOCKED = "blocked"


class VisibilityRule(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: VisibilityRuleId
    portfolio_link_id: PortfolioLinkId
    data_category: DataCategory
    direction: FlowDirection
    gate_type: GateType
    created_at: datetime

    @property
    def requires_approval(self) -> bool:
        return self.gate_type == GateType.OPERATOR_APPROVED

    @staticmethod
    def default_tier_b(link_id: PortfolioLinkId) -> list[VisibilityRule]:
        """Tier B defaults: KPIs auto-flow, everything else operator-gated."""
        now = datetime.now(UTC)
        auto = {DataCategory.KPI_METRICS}
        gated = {
            DataCategory.NARRATIVE,
            DataCategory.DECISIONS,
            DataCategory.ACTION_ITEMS,
            DataCategory.INTELLIGENCE_SIGNALS,
            DataCategory.BRIEFINGS,
        }
        rules: list[VisibilityRule] = []
        for cat in auto:
            rules.append(
                VisibilityRule(
                    id=VisibilityRuleId(uuid4()),
                    portfolio_link_id=link_id,
                    data_category=cat,
                    direction=FlowDirection.COMPANY_TO_SPONSOR,
                    gate_type=GateType.AUTOMATIC,
                    created_at=now,
                )
            )
        for cat in gated:
            rules.append(
                VisibilityRule(
                    id=VisibilityRuleId(uuid4()),
                    portfolio_link_id=link_id,
                    data_category=cat,
                    direction=FlowDirection.COMPANY_TO_SPONSOR,
                    gate_type=GateType.OPERATOR_APPROVED,
                    created_at=now,
                )
            )
        return rules
```

- [ ] **Step 11: Run all organization domain tests**

Run: `cd apps/api && python -m pytest tests/organizations/ -v`
Expected: 10 PASSED

- [ ] **Step 12: Create domain errors**

Create `apps/api/src/prescient/organizations/domain/errors.py`:

```python
from prescient.shared.errors import DomainError, NotFoundError, ValidationError


class OrganizationNotFoundError(NotFoundError):
    def __init__(self, identifier: str) -> None:
        super().__init__(f"Organization not found: {identifier}")


class UserNotFoundError(NotFoundError):
    def __init__(self, identifier: str) -> None:
        super().__init__(f"User not found: {identifier}")


class DuplicateSlugError(ValidationError):
    def __init__(self, slug: str) -> None:
        super().__init__(f"Organization slug already exists: {slug}")


class InvalidPortfolioLinkError(ValidationError):
    def __init__(self, reason: str) -> None:
        super().__init__(f"Invalid portfolio link: {reason}")
```

- [ ] **Step 13: Commit**

```bash
git add apps/api/src/prescient/organizations/ tests/organizations/
git commit -m "feat: add Membership, PortfolioLink, and VisibilityRule domain entities"
```

---

## Task 4: Artifact Store Domain Entities

**Files:**
- Create: `apps/api/src/prescient/artifacts/__init__.py`
- Create: `apps/api/src/prescient/artifacts/domain/__init__.py`
- Create: `apps/api/src/prescient/artifacts/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/artifacts/domain/entities/artifact.py`
- Create: `apps/api/src/prescient/artifacts/domain/entities/artifact_version.py`
- Create: `apps/api/src/prescient/artifacts/domain/entities/artifact_block.py`
- Create: `apps/api/src/prescient/artifacts/domain/entities/artifact_citation.py`
- Create: `apps/api/src/prescient/artifacts/domain/entities/artifact_reference.py`
- Create: `apps/api/src/prescient/artifacts/domain/entities/artifact_subject.py`
- Create: `apps/api/src/prescient/artifacts/domain/state/__init__.py`
- Create: `apps/api/src/prescient/artifacts/domain/state/artifact_state_machine.py`
- Create: `apps/api/src/prescient/artifacts/domain/errors.py`
- Test: `tests/artifacts/test_artifact_domain.py`
- Test: `tests/artifacts/test_artifact_references.py`
- Test: `tests/artifacts/test_artifact_state_machine.py`

- [ ] **Step 1: Create context package structure**

Create empty `__init__.py` files:

```
apps/api/src/prescient/artifacts/__init__.py
apps/api/src/prescient/artifacts/domain/__init__.py
apps/api/src/prescient/artifacts/domain/entities/__init__.py
apps/api/src/prescient/artifacts/domain/state/__init__.py
```

- [ ] **Step 2: Write failing test for ArtifactType enum and Artifact entity**

Create `tests/artifacts/test_artifact_domain.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.artifacts.domain.entities.artifact import (
    Artifact,
    ArtifactType,
)
from prescient.artifacts.domain.entities.artifact_version import (
    ArtifactVersion,
    ArtifactState,
    ConfidenceLabel,
)
from prescient.shared.types import ArtifactId, ArtifactVersionId, OrganizationId, UserId


class TestArtifactTypes:
    def test_all_mvp_types_exist(self) -> None:
        expected = {
            "company_profile",
            "intelligence_signal",
            "decision_record",
            "action_item",
            "kpi_snapshot",
            "briefing",
            "board_narrative",
            "triage_record",
        }
        assert {t.value for t in ArtifactType} == expected


class TestArtifact:
    def test_create_company_profile_artifact(self) -> None:
        now = datetime.now(UTC)
        artifact = Artifact(
            id=ArtifactId(uuid4()),
            organization_id=OrganizationId(uuid4()),
            artifact_type=ArtifactType.COMPANY_PROFILE,
            title="Olaplex Holdings — Company Profile",
            visibility="shared",
            active_version_id=None,
            versions=(),
            created_at=now,
            updated_at=now,
        )
        assert artifact.artifact_type == ArtifactType.COMPANY_PROFILE
        assert artifact.active_version() is None
        assert artifact.next_version_number() == 1

    def test_artifact_with_versions(self) -> None:
        now = datetime.now(UTC)
        artifact_id = ArtifactId(uuid4())
        v1_id = ArtifactVersionId(uuid4())
        v1 = ArtifactVersion(
            id=v1_id,
            artifact_id=artifact_id,
            version_number=1,
            state=ArtifactState.ACTIVE,
            confidence_label=ConfidenceLabel.HIGH,
            blocks=(),
            citations=(),
            subjects=(),
            summary="Initial profile",
            created_by=UserId("ryan"),
            created_at=now,
            approved_by=UserId("ryan"),
            approved_at=now,
        )
        artifact = Artifact(
            id=artifact_id,
            organization_id=OrganizationId(uuid4()),
            artifact_type=ArtifactType.COMPANY_PROFILE,
            title="Olaplex Holdings",
            visibility="shared",
            active_version_id=v1_id,
            versions=(v1,),
            created_at=now,
            updated_at=now,
        )
        assert artifact.active_version() == v1
        assert artifact.next_version_number() == 2
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/artifacts/test_artifact_domain.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement ArtifactBlock and ArtifactCitation**

Create `apps/api/src/prescient/artifacts/domain/entities/artifact_block.py`:

```python
from __future__ import annotations

from pydantic import BaseModel, ConfigDict, Field


class ArtifactBlock(BaseModel):
    model_config = ConfigDict(frozen=True)

    block_id: str = Field(min_length=1, max_length=64)
    heading: str | None = None
    body: str
    order: int = Field(ge=0)
```

Create `apps/api/src/prescient/artifacts/domain/entities/artifact_citation.py`:

```python
from __future__ import annotations

from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field


class ArtifactCitation(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: UUID
    block_id: str = Field(min_length=1, max_length=64)
    document_id: UUID | None = None
    source_type: str = Field(min_length=1, max_length=32)
    excerpt: str
    locator: str | None = None
```

- [ ] **Step 5: Implement ArtifactSubject**

Create `apps/api/src/prescient/artifacts/domain/entities/artifact_subject.py`:

```python
from __future__ import annotations

from enum import StrEnum
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field

from prescient.shared.types import ArtifactVersionId


class ArtifactSubjectType(StrEnum):
    COMPANY = "company"
    PRODUCT = "product"
    MARKET = "market"
    PERSON = "person"
    TOPIC = "topic"


class ArtifactSubject(BaseModel):
    model_config = ConfigDict(frozen=True)

    artifact_version_id: ArtifactVersionId
    subject_type: ArtifactSubjectType
    subject_id: UUID
    role: str = Field(min_length=1, max_length=32, default="subject")
    is_primary: bool = False
```

- [ ] **Step 6: Implement ArtifactVersion**

Create `apps/api/src/prescient/artifacts/domain/entities/artifact_version.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field

from prescient.artifacts.domain.entities.artifact_block import ArtifactBlock
from prescient.artifacts.domain.entities.artifact_citation import ArtifactCitation
from prescient.artifacts.domain.entities.artifact_subject import ArtifactSubject
from prescient.shared.types import ArtifactVersionId, UserId


class ArtifactState(StrEnum):
    DRAFT = "draft"
    ACTIVE = "active"
    APPROVED = "approved"
    ARCHIVED = "archived"


class ConfidenceLabel(StrEnum):
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"
    UNCERTAIN = "uncertain"


class ArtifactVersion(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: ArtifactVersionId
    artifact_id: UUID
    version_number: int = Field(ge=1)
    state: ArtifactState
    confidence_label: ConfidenceLabel
    blocks: tuple[ArtifactBlock, ...] = ()
    citations: tuple[ArtifactCitation, ...] = ()
    subjects: tuple[ArtifactSubject, ...] = ()
    summary: str | None = None
    created_by: UserId
    created_at: datetime
    approved_by: UserId | None = None
    approved_at: datetime | None = None

    def primary_subject(self) -> ArtifactSubject | None:
        for s in self.subjects:
            if s.is_primary:
                return s
        return None
```

- [ ] **Step 7: Implement Artifact aggregate root**

Create `apps/api/src/prescient/artifacts/domain/entities/artifact.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field

from prescient.artifacts.domain.entities.artifact_version import ArtifactVersion
from prescient.shared.types import ArtifactId, ArtifactVersionId, OrganizationId


class ArtifactType(StrEnum):
    COMPANY_PROFILE = "company_profile"
    INTELLIGENCE_SIGNAL = "intelligence_signal"
    DECISION_RECORD = "decision_record"
    ACTION_ITEM = "action_item"
    KPI_SNAPSHOT = "kpi_snapshot"
    BRIEFING = "briefing"
    BOARD_NARRATIVE = "board_narrative"
    TRIAGE_RECORD = "triage_record"


class Artifact(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: ArtifactId
    organization_id: OrganizationId
    artifact_type: ArtifactType
    title: str = Field(min_length=1, max_length=256)
    visibility: str = "shared"
    active_version_id: ArtifactVersionId | None = None
    versions: tuple[ArtifactVersion, ...] = ()
    created_at: datetime
    updated_at: datetime

    def active_version(self) -> ArtifactVersion | None:
        if self.active_version_id is None:
            return None
        for v in self.versions:
            if v.id == self.active_version_id:
                return v
        return None

    def draft_versions(self) -> tuple[ArtifactVersion, ...]:
        from prescient.artifacts.domain.entities.artifact_version import ArtifactState

        return tuple(v for v in self.versions if v.state == ArtifactState.DRAFT)

    def next_version_number(self) -> int:
        if not self.versions:
            return 1
        return max(v.version_number for v in self.versions) + 1
```

- [ ] **Step 8: Run tests to verify Artifact passes**

Run: `cd apps/api && python -m pytest tests/artifacts/test_artifact_domain.py -v`
Expected: 3 PASSED

- [ ] **Step 9: Write failing test for ArtifactReference**

Create `tests/artifacts/test_artifact_references.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.artifacts.domain.entities.artifact_reference import (
    ArtifactReference,
    ReferenceType,
)
from prescient.shared.types import ArtifactId, ArtifactReferenceId


class TestArtifactReference:
    def test_create_informs_reference(self) -> None:
        now = datetime.now(UTC)
        ref = ArtifactReference(
            id=ArtifactReferenceId(uuid4()),
            source_artifact_id=ArtifactId(uuid4()),
            target_artifact_id=ArtifactId(uuid4()),
            reference_type=ReferenceType.INFORMS,
            description="Market signal informed this decision",
            created_at=now,
        )
        assert ref.reference_type == ReferenceType.INFORMS

    def test_all_reference_types(self) -> None:
        expected = {"informs", "spawned", "summarizes", "supersedes", "relates_to"}
        assert {t.value for t in ReferenceType} == expected

    def test_source_and_target_must_differ(self) -> None:
        now = datetime.now(UTC)
        same_id = ArtifactId(uuid4())
        with pytest.raises(ValueError, match="cannot reference itself"):
            ArtifactReference(
                id=ArtifactReferenceId(uuid4()),
                source_artifact_id=same_id,
                target_artifact_id=same_id,
                reference_type=ReferenceType.RELATES_TO,
                description=None,
                created_at=now,
            )
```

- [ ] **Step 10: Implement ArtifactReference**

Create `apps/api/src/prescient/artifacts/domain/entities/artifact_reference.py`:

```python
from __future__ import annotations

from datetime import datetime
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, model_validator

from prescient.shared.types import ArtifactId, ArtifactReferenceId


class ReferenceType(StrEnum):
    INFORMS = "informs"
    SPAWNED = "spawned"
    SUMMARIZES = "summarizes"
    SUPERSEDES = "supersedes"
    RELATES_TO = "relates_to"


class ArtifactReference(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: ArtifactReferenceId
    source_artifact_id: ArtifactId
    target_artifact_id: ArtifactId
    reference_type: ReferenceType
    description: str | None = None
    created_at: datetime

    @model_validator(mode="after")
    def _check_not_self_referencing(self) -> ArtifactReference:
        if self.source_artifact_id == self.target_artifact_id:
            msg = "An artifact cannot reference itself"
            raise ValueError(msg)
        return self
```

- [ ] **Step 11: Run reference tests**

Run: `cd apps/api && python -m pytest tests/artifacts/test_artifact_references.py -v`
Expected: 3 PASSED

- [ ] **Step 12: Write failing test for state machine**

Create `tests/artifacts/test_artifact_state_machine.py`:

```python
from datetime import datetime, UTC
from uuid import uuid4

import pytest

from prescient.artifacts.domain.entities.artifact_version import ArtifactState, ArtifactVersion, ConfidenceLabel
from prescient.artifacts.domain.state.artifact_state_machine import transition_version
from prescient.shared.errors import InvalidStateError
from prescient.shared.types import ArtifactVersionId, UserId


def _make_version(state: ArtifactState) -> ArtifactVersion:
    now = datetime.now(UTC)
    return ArtifactVersion(
        id=ArtifactVersionId(uuid4()),
        artifact_id=uuid4(),
        version_number=1,
        state=state,
        confidence_label=ConfidenceLabel.MEDIUM,
        blocks=(),
        citations=(),
        subjects=(),
        summary=None,
        created_by=UserId("ryan"),
        created_at=now,
    )


class TestArtifactStateMachine:
    def test_draft_to_active(self) -> None:
        v = _make_version(ArtifactState.DRAFT)
        result = transition_version(v, ArtifactState.ACTIVE, approved_by=UserId("ryan"))
        assert result.state == ArtifactState.ACTIVE
        assert result.approved_by == UserId("ryan")
        assert result.approved_at is not None

    def test_active_to_archived(self) -> None:
        v = _make_version(ArtifactState.ACTIVE)
        result = transition_version(v, ArtifactState.ARCHIVED)
        assert result.state == ArtifactState.ARCHIVED

    def test_draft_to_archived_rejected(self) -> None:
        v = _make_version(ArtifactState.DRAFT)
        with pytest.raises(InvalidStateError):
            transition_version(v, ArtifactState.ARCHIVED)

    def test_archived_to_active_rejected(self) -> None:
        v = _make_version(ArtifactState.ARCHIVED)
        with pytest.raises(InvalidStateError):
            transition_version(v, ArtifactState.ACTIVE)
```

- [ ] **Step 13: Implement state machine**

Create `apps/api/src/prescient/artifacts/domain/state/artifact_state_machine.py`:

```python
from __future__ import annotations

from datetime import datetime, UTC

from prescient.artifacts.domain.entities.artifact_version import ArtifactState, ArtifactVersion
from prescient.shared.errors import InvalidStateError
from prescient.shared.types import UserId

_LEGAL_TRANSITIONS: dict[ArtifactState, set[ArtifactState]] = {
    ArtifactState.DRAFT: {ArtifactState.ACTIVE},
    ArtifactState.ACTIVE: {ArtifactState.ARCHIVED},
    ArtifactState.APPROVED: {ArtifactState.ACTIVE, ArtifactState.ARCHIVED},
    ArtifactState.ARCHIVED: set(),
}


def transition_version(
    version: ArtifactVersion,
    to_state: ArtifactState,
    *,
    approved_by: UserId | None = None,
) -> ArtifactVersion:
    allowed = _LEGAL_TRANSITIONS.get(version.state, set())
    if to_state not in allowed:
        msg = f"Cannot transition artifact version from {version.state} to {to_state}"
        raise InvalidStateError(msg)

    updates: dict = {"state": to_state}
    if to_state == ArtifactState.ACTIVE and approved_by is not None:
        updates["approved_by"] = approved_by
        updates["approved_at"] = datetime.now(UTC)

    return version.model_copy(update=updates)
```

- [ ] **Step 14: Run state machine tests**

Run: `cd apps/api && python -m pytest tests/artifacts/test_artifact_state_machine.py -v`
Expected: 4 PASSED

- [ ] **Step 15: Create artifact domain errors**

Create `apps/api/src/prescient/artifacts/domain/errors.py`:

```python
from prescient.shared.errors import NotFoundError, ValidationError


class ArtifactNotFoundError(NotFoundError):
    def __init__(self, identifier: str) -> None:
        super().__init__(f"Artifact not found: {identifier}")


class ArtifactVersionNotFoundError(NotFoundError):
    def __init__(self, identifier: str) -> None:
        super().__init__(f"Artifact version not found: {identifier}")
```

- [ ] **Step 16: Run all artifact tests**

Run: `cd apps/api && python -m pytest tests/artifacts/ -v`
Expected: 10 PASSED

- [ ] **Step 17: Commit**

```bash
git add apps/api/src/prescient/artifacts/ tests/artifacts/
git commit -m "feat: add Artifact Store domain — types, versions, references, state machine"
```

---

## Task 5: Auth Domain and Infrastructure

**Files:**
- Create: `apps/api/src/prescient/auth/__init__.py`
- Create: `apps/api/src/prescient/auth/domain/__init__.py`
- Create: `apps/api/src/prescient/auth/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/auth/domain/entities/credentials.py`
- Create: `apps/api/src/prescient/auth/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/auth/infrastructure/hashing.py`
- Create: `apps/api/src/prescient/auth/infrastructure/jwt.py`
- Test: `tests/auth/test_hashing.py`
- Test: `tests/auth/test_jwt.py`

- [ ] **Step 1: Create auth package structure**

Create empty `__init__.py` files:

```
apps/api/src/prescient/auth/__init__.py
apps/api/src/prescient/auth/domain/__init__.py
apps/api/src/prescient/auth/domain/entities/__init__.py
apps/api/src/prescient/auth/infrastructure/__init__.py
```

- [ ] **Step 2: Write failing test for password hashing**

Create `tests/auth/test_hashing.py`:

```python
import pytest

from prescient.auth.infrastructure.hashing import hash_password, verify_password


class TestHashing:
    def test_hash_and_verify(self) -> None:
        password = "secure-password-123"
        hashed = hash_password(password)
        assert hashed != password
        assert verify_password(password, hashed) is True

    def test_wrong_password_fails(self) -> None:
        hashed = hash_password("correct-password")
        assert verify_password("wrong-password", hashed) is False

    def test_different_hashes_for_same_password(self) -> None:
        password = "same-password"
        hash1 = hash_password(password)
        hash2 = hash_password(password)
        assert hash1 != hash2  # bcrypt salts differ
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/auth/test_hashing.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement password hashing**

Create `apps/api/src/prescient/auth/infrastructure/hashing.py`:

```python
from passlib.context import CryptContext

_ctx = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(plain: str) -> str:
    return _ctx.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    return _ctx.verify(plain, hashed)
```

- [ ] **Step 5: Run hashing tests**

Run: `cd apps/api && python -m pytest tests/auth/test_hashing.py -v`
Expected: 3 PASSED

- [ ] **Step 6: Write failing test for JWT**

Create `tests/auth/test_jwt.py`:

```python
from datetime import timedelta

import pytest

from prescient.auth.infrastructure.jwt import create_access_token, decode_access_token


class TestJWT:
    def test_create_and_decode(self) -> None:
        token = create_access_token(
            user_id="ryan",
            secret="test-secret",
            algorithm="HS256",
            expires_delta=timedelta(hours=8),
        )
        payload = decode_access_token(token, secret="test-secret", algorithm="HS256")
        assert payload["sub"] == "ryan"

    def test_expired_token_raises(self) -> None:
        token = create_access_token(
            user_id="ryan",
            secret="test-secret",
            algorithm="HS256",
            expires_delta=timedelta(seconds=-1),
        )
        with pytest.raises(ValueError, match="expired"):
            decode_access_token(token, secret="test-secret", algorithm="HS256")

    def test_invalid_token_raises(self) -> None:
        with pytest.raises(ValueError, match="invalid"):
            decode_access_token("garbage-token", secret="test-secret", algorithm="HS256")
```

- [ ] **Step 7: Implement JWT**

Create `apps/api/src/prescient/auth/infrastructure/jwt.py`:

```python
from datetime import datetime, timedelta, UTC

from jose import JWTError, jwt


def create_access_token(
    *,
    user_id: str,
    secret: str,
    algorithm: str,
    expires_delta: timedelta,
) -> str:
    expire = datetime.now(UTC) + expires_delta
    payload = {"sub": user_id, "exp": expire}
    return jwt.encode(payload, secret, algorithm=algorithm)


def decode_access_token(
    token: str,
    *,
    secret: str,
    algorithm: str,
) -> dict:
    try:
        payload = jwt.decode(token, secret, algorithms=[algorithm])
    except JWTError as exc:
        error_str = str(exc).lower()
        if "expired" in error_str:
            msg = "Token has expired"
            raise ValueError(msg) from exc
        msg = "Token is invalid"
        raise ValueError(msg) from exc
    return payload
```

- [ ] **Step 8: Run JWT tests**

Run: `cd apps/api && python -m pytest tests/auth/test_jwt.py -v`
Expected: 3 PASSED

- [ ] **Step 9: Implement Credentials domain entity**

Create `apps/api/src/prescient/auth/domain/entities/credentials.py`:

```python
from __future__ import annotations

from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field

from prescient.shared.types import CredentialsId, UserId


class Credentials(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: CredentialsId
    user_id: UserId
    email: str = Field(min_length=1, max_length=256)
    password_hash: str
    created_at: datetime
    last_login_at: datetime | None = None
```

- [ ] **Step 10: Commit**

```bash
git add apps/api/src/prescient/auth/ tests/auth/
git commit -m "feat: add auth context — password hashing, JWT, credentials entity"
```

---

## Task 6: Organization Infrastructure (SQLAlchemy Tables)

**Files:**
- Create: `apps/api/src/prescient/organizations/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/organizations/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/organizations/infrastructure/tables/organization.py`
- Create: `apps/api/src/prescient/organizations/infrastructure/tables/user.py`
- Create: `apps/api/src/prescient/organizations/infrastructure/tables/membership.py`
- Create: `apps/api/src/prescient/organizations/infrastructure/tables/portfolio_link.py`
- Create: `apps/api/src/prescient/organizations/infrastructure/tables/visibility_rule.py`

- [ ] **Step 1: Create infrastructure package structure**

Create empty `__init__.py` files:

```
apps/api/src/prescient/organizations/infrastructure/__init__.py
apps/api/src/prescient/organizations/infrastructure/tables/__init__.py
apps/api/src/prescient/organizations/infrastructure/repositories/__init__.py
```

- [ ] **Step 2: Implement organization table**

Create `apps/api/src/prescient/organizations/infrastructure/tables/organization.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class OrganizationRow(Base):
    __tablename__ = "organizations"
    __table_args__ = {"schema": "organizations"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    slug: Mapped[str] = mapped_column(String(64), unique=True, nullable=False, index=True)
    name: Mapped[str] = mapped_column(String(256), nullable=False)
    org_type: Mapped[str] = mapped_column(String(32), nullable=False)
    status: Mapped[str] = mapped_column(String(32), nullable=False, default="active")
    industry: Mapped[str | None] = mapped_column(String(128), nullable=True)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    website: Mapped[str | None] = mapped_column(String(512), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 3: Implement user table**

Create `apps/api/src/prescient/organizations/infrastructure/tables/user.py`:

```python
from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, Boolean
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class UserRow(Base):
    __tablename__ = "users"
    __table_args__ = {"schema": "organizations"}

    id: Mapped[str] = mapped_column(String(64), primary_key=True)
    email: Mapped[str] = mapped_column(String(256), unique=True, nullable=False, index=True)
    display_name: Mapped[str] = mapped_column(String(128), nullable=False)
    role: Mapped[str] = mapped_column(String(32), nullable=False)
    primary_org_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("organizations.organizations.id"), nullable=False
    )
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 4: Implement membership table**

Create `apps/api/src/prescient/organizations/infrastructure/tables/membership.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class MembershipRow(Base):
    __tablename__ = "memberships"
    __table_args__ = {"schema": "organizations"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    user_id: Mapped[str] = mapped_column(
        String(64), ForeignKey("organizations.users.id"), nullable=False, index=True
    )
    organization_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("organizations.organizations.id"), nullable=False, index=True
    )
    role: Mapped[str] = mapped_column(String(32), nullable=False)
    scope: Mapped[str] = mapped_column(String(16), nullable=False, default="full")
    granted_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    revoked_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

- [ ] **Step 5: Implement portfolio_link table**

Create `apps/api/src/prescient/organizations/infrastructure/tables/portfolio_link.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class PortfolioLinkRow(Base):
    __tablename__ = "portfolio_links"
    __table_args__ = {"schema": "organizations"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    sponsor_org_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("organizations.organizations.id"), nullable=False, index=True
    )
    company_org_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("organizations.organizations.id"), nullable=False, index=True
    )
    status: Mapped[str] = mapped_column(String(16), nullable=False, default="active")
    billing_model: Mapped[str] = mapped_column(String(24), nullable=False)
    linked_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    unlinked_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

- [ ] **Step 6: Implement visibility_rule table**

Create `apps/api/src/prescient/organizations/infrastructure/tables/visibility_rule.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class VisibilityRuleRow(Base):
    __tablename__ = "visibility_rules"
    __table_args__ = {"schema": "organizations"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    portfolio_link_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("organizations.portfolio_links.id"), nullable=False, index=True
    )
    data_category: Mapped[str] = mapped_column(String(32), nullable=False)
    direction: Mapped[str] = mapped_column(String(32), nullable=False)
    gate_type: Mapped[str] = mapped_column(String(24), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 7: Update tables __init__.py to register all tables**

Update `apps/api/src/prescient/organizations/infrastructure/tables/__init__.py`:

```python
from prescient.organizations.infrastructure.tables.organization import OrganizationRow
from prescient.organizations.infrastructure.tables.user import UserRow
from prescient.organizations.infrastructure.tables.membership import MembershipRow
from prescient.organizations.infrastructure.tables.portfolio_link import PortfolioLinkRow
from prescient.organizations.infrastructure.tables.visibility_rule import VisibilityRuleRow

__all__ = [
    "OrganizationRow",
    "UserRow",
    "MembershipRow",
    "PortfolioLinkRow",
    "VisibilityRuleRow",
]
```

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/organizations/infrastructure/
git commit -m "feat: add Organization context SQLAlchemy table definitions"
```

---

## Task 7: Artifact Store and Auth Infrastructure (SQLAlchemy Tables)

**Files:**
- Create: `apps/api/src/prescient/artifacts/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/artifacts/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/artifacts/infrastructure/tables/artifact.py`
- Create: `apps/api/src/prescient/artifacts/infrastructure/tables/artifact_version.py`
- Create: `apps/api/src/prescient/artifacts/infrastructure/tables/artifact_reference.py`
- Create: `apps/api/src/prescient/artifacts/infrastructure/tables/artifact_subject.py`
- Create: `apps/api/src/prescient/auth/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/auth/infrastructure/tables/credentials.py`

- [ ] **Step 1: Create infrastructure package structure**

Create empty `__init__.py` files:

```
apps/api/src/prescient/artifacts/infrastructure/__init__.py
apps/api/src/prescient/artifacts/infrastructure/tables/__init__.py
apps/api/src/prescient/artifacts/infrastructure/repositories/__init__.py
apps/api/src/prescient/auth/infrastructure/tables/__init__.py
```

- [ ] **Step 2: Implement artifact table**

Create `apps/api/src/prescient/artifacts/infrastructure/tables/artifact.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, ForeignKey, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class ArtifactRow(Base):
    __tablename__ = "artifacts"
    __table_args__ = {"schema": "artifacts"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    artifact_type: Mapped[str] = mapped_column(String(32), nullable=False, index=True)
    title: Mapped[str] = mapped_column(String(256), nullable=False)
    visibility: Mapped[str] = mapped_column(String(16), nullable=False, default="shared")
    active_version_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 3: Implement artifact_version table**

Create `apps/api/src/prescient/artifacts/infrastructure/tables/artifact_version.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, ForeignKey, Integer, String, Text
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class ArtifactVersionRow(Base):
    __tablename__ = "artifact_versions"
    __table_args__ = {"schema": "artifacts"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    artifact_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("artifacts.artifacts.id"), nullable=False, index=True
    )
    version_number: Mapped[int] = mapped_column(Integer, nullable=False)
    state: Mapped[str] = mapped_column(String(16), nullable=False, default="draft")
    confidence_label: Mapped[str] = mapped_column(String(16), nullable=False, default="medium")
    blocks: Mapped[dict] = mapped_column(JSONB, nullable=False, default=list)
    citations: Mapped[dict] = mapped_column(JSONB, nullable=False, default=list)
    summary: Mapped[str | None] = mapped_column(Text, nullable=True)
    created_by: Mapped[str] = mapped_column(String(64), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    approved_by: Mapped[str | None] = mapped_column(String(64), nullable=True)
    approved_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

- [ ] **Step 4: Implement artifact_reference table**

Create `apps/api/src/prescient/artifacts/infrastructure/tables/artifact_reference.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, ForeignKey, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class ArtifactReferenceRow(Base):
    __tablename__ = "artifact_references"
    __table_args__ = {"schema": "artifacts"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    source_artifact_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("artifacts.artifacts.id"), nullable=False, index=True
    )
    target_artifact_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("artifacts.artifacts.id"), nullable=False, index=True
    )
    reference_type: Mapped[str] = mapped_column(String(16), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
```

- [ ] **Step 5: Implement artifact_subject table**

Create `apps/api/src/prescient/artifacts/infrastructure/tables/artifact_subject.py`:

```python
from __future__ import annotations

from sqlalchemy import Boolean, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class ArtifactSubjectRow(Base):
    __tablename__ = "artifact_subjects"
    __table_args__ = {"schema": "artifacts"}

    artifact_version_id: Mapped[str] = mapped_column(
        String(36), ForeignKey("artifacts.artifact_versions.id"), primary_key=True
    )
    subject_type: Mapped[str] = mapped_column(String(16), primary_key=True)
    subject_id: Mapped[str] = mapped_column(String(36), primary_key=True)
    role: Mapped[str] = mapped_column(String(32), nullable=False, default="subject")
    is_primary: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False)
```

- [ ] **Step 6: Update artifact tables __init__.py**

Update `apps/api/src/prescient/artifacts/infrastructure/tables/__init__.py`:

```python
from prescient.artifacts.infrastructure.tables.artifact import ArtifactRow
from prescient.artifacts.infrastructure.tables.artifact_version import ArtifactVersionRow
from prescient.artifacts.infrastructure.tables.artifact_reference import ArtifactReferenceRow
from prescient.artifacts.infrastructure.tables.artifact_subject import ArtifactSubjectRow

__all__ = [
    "ArtifactRow",
    "ArtifactVersionRow",
    "ArtifactReferenceRow",
    "ArtifactSubjectRow",
]
```

- [ ] **Step 7: Implement auth credentials table**

Create `apps/api/src/prescient/auth/infrastructure/tables/credentials.py`:

```python
from __future__ import annotations

from datetime import datetime
from uuid import uuid4

from sqlalchemy import DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class CredentialsRow(Base):
    __tablename__ = "credentials"
    __table_args__ = {"schema": "auth"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    user_id: Mapped[str] = mapped_column(String(64), unique=True, nullable=False, index=True)
    email: Mapped[str] = mapped_column(String(256), unique=True, nullable=False, index=True)
    password_hash: Mapped[str] = mapped_column(String(256), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    last_login_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

- [ ] **Step 8: Update auth tables __init__.py**

Update `apps/api/src/prescient/auth/infrastructure/tables/__init__.py`:

```python
from prescient.auth.infrastructure.tables.credentials import CredentialsRow

__all__ = ["CredentialsRow"]
```

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/artifacts/infrastructure/ apps/api/src/prescient/auth/infrastructure/tables/
git commit -m "feat: add Artifact Store and Auth SQLAlchemy table definitions"
```

---

## Task 8: Alembic Migration for New Schemas

**Files:**
- Create: `apps/api/alembic/versions/20260412_0005_phase1_foundation.py`
- Modify: `apps/api/src/prescient/shared/metadata.py`

- [ ] **Step 1: Update metadata.py to register new tables**

In `apps/api/src/prescient/shared/metadata.py`, add imports for the new table modules so Alembic discovers them:

```python
# Add these imports alongside existing ones:
import prescient.organizations.infrastructure.tables  # noqa: F401
import prescient.artifacts.infrastructure.tables  # noqa: F401
import prescient.auth.infrastructure.tables  # noqa: F401
```

- [ ] **Step 2: Auto-generate migration**

Run: `cd apps/api && python -m alembic revision --autogenerate -m "phase1_foundation_organizations_artifacts_auth"`

Verify the generated migration creates schemas (`organizations`, `artifacts`, `auth`) and all expected tables.

- [ ] **Step 3: Review and fix the migration if needed**

Open the generated migration file and verify it:
- Creates `organizations` schema with tables: organizations, users, memberships, portfolio_links, visibility_rules
- Creates `artifacts` schema with tables: artifacts, artifact_versions, artifact_references, artifact_subjects
- Creates `auth` schema with table: credentials
- Has proper `op.create_schema()` calls in `upgrade()` and `op.drop_schema()` in `downgrade()`

- [ ] **Step 4: Run the migration**

Run: `cd apps/api && python -m alembic upgrade head`
Expected: Migration applies without errors.

- [ ] **Step 5: Verify tables exist**

Run: `cd apps/api && python -c "import asyncio; from prescient.db import engine; from sqlalchemy import text; asyncio.run((lambda: None)())" && PGPASSWORD=prescient psql -h localhost -U prescient -d prescient -c "\dt organizations.*" -c "\dt artifacts.*" -c "\dt auth.*"`

Expected: All tables listed under their respective schemas.

- [ ] **Step 6: Commit**

```bash
git add apps/api/alembic/ apps/api/src/prescient/shared/metadata.py
git commit -m "feat: add Alembic migration for Phase 1 schemas — organizations, artifacts, auth"
```

---

## Task 9: Auth API — Register and Login Endpoints

**Files:**
- Create: `apps/api/src/prescient/auth/application/__init__.py`
- Create: `apps/api/src/prescient/auth/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/auth/application/use_cases/register.py`
- Create: `apps/api/src/prescient/auth/application/use_cases/login.py`
- Create: `apps/api/src/prescient/auth/api/__init__.py`
- Create: `apps/api/src/prescient/auth/api/routes.py`
- Test: `tests/auth/test_register.py`
- Test: `tests/auth/test_login.py`

- [ ] **Step 1: Create application package structure**

Create empty `__init__.py` files:

```
apps/api/src/prescient/auth/application/__init__.py
apps/api/src/prescient/auth/application/use_cases/__init__.py
apps/api/src/prescient/auth/api/__init__.py
```

- [ ] **Step 2: Write failing test for register use case**

Create `tests/auth/test_register.py`:

```python
from datetime import datetime, UTC
from unittest.mock import AsyncMock
from uuid import uuid4

import pytest

from prescient.auth.application.use_cases.register import RegisterUser, RegisterRequest, RegisterResult
from prescient.auth.infrastructure.hashing import verify_password
from prescient.organizations.domain.entities.user import UserRole


class TestRegisterUser:
    @pytest.fixture
    def mock_session(self) -> AsyncMock:
        session = AsyncMock()
        session.execute = AsyncMock(return_value=AsyncMock(scalar_one_or_none=lambda: None))
        return session

    @pytest.mark.asyncio
    async def test_register_creates_user_and_credentials(self, mock_session: AsyncMock) -> None:
        use_case = RegisterUser(session=mock_session)
        request = RegisterRequest(
            email="ryan@olaplex.com",
            password="secure-password-123",
            display_name="Ryan Hallman",
            role=UserRole.OPERATOR,
            organization_slug="olaplex",
        )
        result = await use_case.execute(request)
        assert result.user_id is not None
        assert result.email == "ryan@olaplex.com"
        assert mock_session.add.called
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/auth/test_register.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement register use case**

Create `apps/api/src/prescient/auth/application/use_cases/register.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import uuid4

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.auth.infrastructure.hashing import hash_password
from prescient.auth.infrastructure.tables.credentials import CredentialsRow
from prescient.organizations.domain.entities.user import UserRole
from prescient.organizations.infrastructure.tables.organization import OrganizationRow
from prescient.organizations.infrastructure.tables.user import UserRow
from prescient.shared.errors import NotFoundError, ValidationError


@dataclass(frozen=True)
class RegisterRequest:
    email: str
    password: str
    display_name: str
    role: UserRole
    organization_slug: str


@dataclass(frozen=True)
class RegisterResult:
    user_id: str
    email: str
    organization_id: str


class RegisterUser:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, request: RegisterRequest) -> RegisterResult:
        # Check email not already taken
        existing = await self._session.execute(
            select(CredentialsRow).where(CredentialsRow.email == request.email)
        )
        if existing.scalar_one_or_none() is not None:
            msg = f"Email already registered: {request.email}"
            raise ValidationError(msg)

        # Find organization
        org_result = await self._session.execute(
            select(OrganizationRow).where(OrganizationRow.slug == request.organization_slug)
        )
        org = org_result.scalar_one_or_none()
        if org is None:
            msg = f"Organization not found: {request.organization_slug}"
            raise NotFoundError(msg)

        now = datetime.now(UTC)
        user_id = request.email.split("@")[0] + "-" + uuid4().hex[:6]

        user_row = UserRow(
            id=user_id,
            email=request.email,
            display_name=request.display_name,
            role=request.role.value,
            primary_org_id=org.id,
            is_active=True,
            created_at=now,
        )
        self._session.add(user_row)

        cred_row = CredentialsRow(
            id=str(uuid4()),
            user_id=user_id,
            email=request.email,
            password_hash=hash_password(request.password),
            created_at=now,
        )
        self._session.add(cred_row)

        return RegisterResult(
            user_id=user_id,
            email=request.email,
            organization_id=org.id,
        )
```

- [ ] **Step 5: Write failing test for login use case**

Create `tests/auth/test_login.py`:

```python
from datetime import datetime, timedelta, UTC
from unittest.mock import AsyncMock, MagicMock
from uuid import uuid4

import pytest

from prescient.auth.application.use_cases.login import LoginUser, LoginRequest, LoginResult
from prescient.auth.infrastructure.hashing import hash_password
from prescient.auth.infrastructure.jwt import decode_access_token
from prescient.shared.errors import AuthorizationError


class TestLoginUser:
    def _make_cred_row(self, email: str, password: str) -> MagicMock:
        row = MagicMock()
        row.user_id = "ryan"
        row.email = email
        row.password_hash = hash_password(password)
        return row

    @pytest.fixture
    def mock_session(self) -> AsyncMock:
        return AsyncMock()

    @pytest.mark.asyncio
    async def test_login_success(self, mock_session: AsyncMock) -> None:
        cred_row = self._make_cred_row("ryan@olaplex.com", "correct-password")
        result_mock = AsyncMock()
        result_mock.scalar_one_or_none = lambda: cred_row
        mock_session.execute = AsyncMock(return_value=result_mock)

        use_case = LoginUser(
            session=mock_session,
            jwt_secret="test-secret",
            jwt_algorithm="HS256",
            token_expire_minutes=480,
        )
        result = await use_case.execute(LoginRequest(email="ryan@olaplex.com", password="correct-password"))
        assert result.access_token is not None
        payload = decode_access_token(result.access_token, secret="test-secret", algorithm="HS256")
        assert payload["sub"] == "ryan"

    @pytest.mark.asyncio
    async def test_login_wrong_password(self, mock_session: AsyncMock) -> None:
        cred_row = self._make_cred_row("ryan@olaplex.com", "correct-password")
        result_mock = AsyncMock()
        result_mock.scalar_one_or_none = lambda: cred_row
        mock_session.execute = AsyncMock(return_value=result_mock)

        use_case = LoginUser(
            session=mock_session,
            jwt_secret="test-secret",
            jwt_algorithm="HS256",
            token_expire_minutes=480,
        )
        with pytest.raises(AuthorizationError):
            await use_case.execute(LoginRequest(email="ryan@olaplex.com", password="wrong-password"))

    @pytest.mark.asyncio
    async def test_login_unknown_email(self, mock_session: AsyncMock) -> None:
        result_mock = AsyncMock()
        result_mock.scalar_one_or_none = lambda: None
        mock_session.execute = AsyncMock(return_value=result_mock)

        use_case = LoginUser(
            session=mock_session,
            jwt_secret="test-secret",
            jwt_algorithm="HS256",
            token_expire_minutes=480,
        )
        with pytest.raises(AuthorizationError):
            await use_case.execute(LoginRequest(email="unknown@foo.com", password="anything"))
```

- [ ] **Step 6: Implement login use case**

Create `apps/api/src/prescient/auth/application/use_cases/login.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import timedelta

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.auth.infrastructure.hashing import verify_password
from prescient.auth.infrastructure.jwt import create_access_token
from prescient.auth.infrastructure.tables.credentials import CredentialsRow
from prescient.shared.errors import AuthorizationError


@dataclass(frozen=True)
class LoginRequest:
    email: str
    password: str


@dataclass(frozen=True)
class LoginResult:
    access_token: str
    user_id: str


class LoginUser:
    def __init__(
        self,
        session: AsyncSession,
        jwt_secret: str,
        jwt_algorithm: str,
        token_expire_minutes: int,
    ) -> None:
        self._session = session
        self._jwt_secret = jwt_secret
        self._jwt_algorithm = jwt_algorithm
        self._token_expire_minutes = token_expire_minutes

    async def execute(self, request: LoginRequest) -> LoginResult:
        result = await self._session.execute(
            select(CredentialsRow).where(CredentialsRow.email == request.email)
        )
        cred = result.scalar_one_or_none()
        if cred is None:
            msg = "Invalid email or password"
            raise AuthorizationError(msg)

        if not verify_password(request.password, cred.password_hash):
            msg = "Invalid email or password"
            raise AuthorizationError(msg)

        token = create_access_token(
            user_id=cred.user_id,
            secret=self._jwt_secret,
            algorithm=self._jwt_algorithm,
            expires_delta=timedelta(minutes=self._token_expire_minutes),
        )
        return LoginResult(access_token=token, user_id=cred.user_id)
```

- [ ] **Step 7: Run auth use case tests**

Run: `cd apps/api && python -m pytest tests/auth/ -v`
Expected: 9 PASSED (3 hashing + 3 JWT + 3 login)

- [ ] **Step 8: Implement auth API routes**

Create `apps/api/src/prescient/auth/api/routes.py`:

```python
from __future__ import annotations

from pydantic import BaseModel, Field
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.auth.application.use_cases.login import LoginUser, LoginRequest
from prescient.auth.application.use_cases.register import RegisterUser, RegisterRequest
from prescient.config import get_settings
from prescient.db import get_session
from prescient.organizations.domain.entities.user import UserRole

router = APIRouter(prefix="/auth", tags=["auth"])


class RegisterBody(BaseModel):
    email: str = Field(min_length=1, max_length=256)
    password: str = Field(min_length=8, max_length=128)
    display_name: str = Field(min_length=1, max_length=128)
    role: UserRole
    organization_slug: str = Field(min_length=1, max_length=64)


class LoginBody(BaseModel):
    email: str = Field(min_length=1, max_length=256)
    password: str = Field(min_length=1, max_length=128)


class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    user_id: str


class RegisterResponse(BaseModel):
    user_id: str
    email: str
    organization_id: str


@router.post("/register", response_model=RegisterResponse, status_code=201)
async def register(body: RegisterBody, session: AsyncSession = Depends(get_session)) -> RegisterResponse:
    use_case = RegisterUser(session=session)
    result = await use_case.execute(
        RegisterRequest(
            email=body.email,
            password=body.password,
            display_name=body.display_name,
            role=body.role,
            organization_slug=body.organization_slug,
        )
    )
    await session.commit()
    return RegisterResponse(user_id=result.user_id, email=result.email, organization_id=result.organization_id)


@router.post("/login", response_model=TokenResponse)
async def login(body: LoginBody, session: AsyncSession = Depends(get_session)) -> TokenResponse:
    settings = get_settings()
    use_case = LoginUser(
        session=session,
        jwt_secret=settings.jwt_secret,
        jwt_algorithm=settings.jwt_algorithm,
        token_expire_minutes=settings.jwt_access_token_expire_minutes,
    )
    result = await use_case.execute(LoginRequest(email=body.email, password=body.password))
    return TokenResponse(access_token=result.access_token, user_id=result.user_id)
```

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/auth/ tests/auth/
git commit -m "feat: add auth register/login use cases and API routes"
```

---

## Task 10: Wire New Routers into FastAPI App

**Files:**
- Modify: `apps/api/src/prescient/main.py`

- [ ] **Step 1: Read current main.py**

Read `apps/api/src/prescient/main.py` to understand the current router setup.

- [ ] **Step 2: Add auth router to the app**

In `apps/api/src/prescient/main.py`, add the auth router import and include it:

```python
from prescient.auth.api.routes import router as auth_router
```

And in the `create_app` function, add:

```python
app.include_router(auth_router)
```

- [ ] **Step 3: Verify the app starts**

Run: `cd apps/api && timeout 5 python -c "from prescient.main import create_app; app = create_app(); print('App created with routes:', [r.path for r in app.routes])" || true`

Expected: Routes list includes `/auth/register` and `/auth/login`.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/main.py
git commit -m "feat: wire auth router into FastAPI app"
```

---

## Task 11: Frontend Auth — Login Page and Auth Provider

**Files:**
- Create: `apps/web/src/lib/auth.ts`
- Create: `apps/web/src/components/auth-provider.tsx`
- Create: `apps/web/src/app/login/page.tsx`
- Modify: `apps/web/src/app/layout.tsx`
- Modify: `apps/web/src/lib/api.ts`

- [ ] **Step 1: Create auth utility**

Create `apps/web/src/lib/auth.ts`:

```typescript
const TOKEN_KEY = "prescient_token";
const USER_KEY = "prescient_user_id";

const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

export function getToken(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem(TOKEN_KEY);
}

export function getUserId(): string | null {
  if (typeof window === "undefined") return null;
  return localStorage.getItem(USER_KEY);
}

export function setAuth(token: string, userId: string): void {
  localStorage.setItem(TOKEN_KEY, token);
  localStorage.setItem(USER_KEY, userId);
}

export function clearAuth(): void {
  localStorage.removeItem(TOKEN_KEY);
  localStorage.removeItem(USER_KEY);
}

export async function login(
  email: string,
  password: string
): Promise<{ access_token: string; user_id: string }> {
  const res = await fetch(`${API_BASE}/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email, password }),
  });
  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(err.detail ?? "Login failed");
  }
  return res.json();
}

export async function register(data: {
  email: string;
  password: string;
  display_name: string;
  role: string;
  organization_slug: string;
}): Promise<{ user_id: string; email: string; organization_id: string }> {
  const res = await fetch(`${API_BASE}/auth/register`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  });
  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(err.detail ?? "Registration failed");
  }
  return res.json();
}
```

- [ ] **Step 2: Create auth provider component**

Create `apps/web/src/components/auth-provider.tsx`:

```tsx
"use client";

import {
  createContext,
  useContext,
  useEffect,
  useState,
  type ReactNode,
} from "react";
import { getToken, getUserId, clearAuth } from "@/lib/auth";
import { useRouter } from "next/navigation";

type AuthContextValue = {
  token: string | null;
  userId: string | null;
  isAuthenticated: boolean;
  logout: () => void;
};

const AuthContext = createContext<AuthContextValue>({
  token: null,
  userId: null,
  isAuthenticated: false,
  logout: () => {},
});

export function useAuth() {
  return useContext(AuthContext);
}

export function AuthProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(null);
  const [userId, setUserId] = useState<string | null>(null);
  const router = useRouter();

  useEffect(() => {
    setToken(getToken());
    setUserId(getUserId());
  }, []);

  const logout = () => {
    clearAuth();
    setToken(null);
    setUserId(null);
    router.push("/login");
  };

  return (
    <AuthContext.Provider
      value={{
        token,
        userId,
        isAuthenticated: token !== null,
        logout,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}
```

- [ ] **Step 3: Create login page**

Create `apps/web/src/app/login/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { login, setAuth } from "@/lib/auth";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function LoginPage() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);
  const router = useRouter();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);
    setLoading(true);
    try {
      const result = await login(email, password);
      setAuth(result.access_token, result.user_id);
      router.push("/brief");
    } catch (err) {
      setError(err instanceof Error ? err.message : "Login failed");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex min-h-screen items-center justify-center">
      <Card className="w-full max-w-md">
        <CardHeader>
          <CardTitle className="text-center text-2xl">Prescient OS</CardTitle>
          <p className="text-center text-sm text-neutral-500">
            Sign in to your account
          </p>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit} className="flex flex-col gap-4">
            <Input
              type="email"
              placeholder="Email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              required
            />
            <Input
              type="password"
              placeholder="Password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              required
            />
            {error && (
              <p className="text-sm text-red-600">{error}</p>
            )}
            <Button type="submit" disabled={loading}>
              {loading ? "Signing in..." : "Sign in"}
            </Button>
          </form>
        </CardContent>
      </Card>
    </div>
  );
}
```

- [ ] **Step 4: Create morning brief placeholder page**

Create `apps/web/src/app/brief/page.tsx`:

```tsx
export default function BriefPage() {
  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-semibold">Morning Brief</h1>
      <p className="text-neutral-500">
        Your personalized daily briefing will appear here. This is where the day starts.
      </p>
    </div>
  );
}
```

- [ ] **Step 5: Update layout.tsx to include AuthProvider**

In `apps/web/src/app/layout.tsx`, wrap children with `AuthProvider`:

```tsx
import { AuthProvider } from "@/components/auth-provider";
```

And in the body:

```tsx
<Providers>
  <AuthProvider>
    <Nav />
    <main className="mx-auto max-w-6xl px-6 py-8">
      {children}
    </main>
  </AuthProvider>
</Providers>
```

- [ ] **Step 6: Update nav.tsx for operator-first navigation**

Update `apps/web/src/components/nav.tsx` to include operator-first navigation items:

The nav should include:
- Prescient OS logo/title
- Morning Brief link
- Intelligence link (placeholder)
- Actions link (placeholder)
- Board Prep link (placeholder)
- User menu with logout

- [ ] **Step 7: Update api.ts to include auth headers**

In `apps/web/src/lib/api.ts`, modify the `getJSON` helper to accept an optional token parameter and include it as a Bearer token in the Authorization header for client-side requests.

- [ ] **Step 8: Verify frontend compiles**

Run: `cd apps/web && npx tsc --noEmit`
Expected: No type errors.

- [ ] **Step 9: Commit**

```bash
git add apps/web/src/
git commit -m "feat: add login page, auth provider, morning brief placeholder, operator-first nav"
```

---

## Task 12: Update Root Page to Redirect

**Files:**
- Modify: `apps/web/src/app/page.tsx`

- [ ] **Step 1: Update root page**

Replace `apps/web/src/app/page.tsx` to redirect to `/brief`:

```tsx
import { redirect } from "next/navigation";

export default function RootPage() {
  redirect("/brief");
}
```

- [ ] **Step 2: Verify frontend compiles**

Run: `cd apps/web && npx tsc --noEmit`
Expected: No type errors.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/page.tsx
git commit -m "feat: redirect root page to morning brief"
```

---

## Task 13: Organization API Routes

**Files:**
- Create: `apps/api/src/prescient/organizations/application/__init__.py`
- Create: `apps/api/src/prescient/organizations/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/organizations/application/use_cases/create_organization.py`
- Create: `apps/api/src/prescient/organizations/api/__init__.py`
- Create: `apps/api/src/prescient/organizations/api/routes.py`
- Test: `tests/organizations/test_organization_api.py`

- [ ] **Step 1: Create application package structure**

Create empty `__init__.py` files:

```
apps/api/src/prescient/organizations/application/__init__.py
apps/api/src/prescient/organizations/application/use_cases/__init__.py
apps/api/src/prescient/organizations/api/__init__.py
```

- [ ] **Step 2: Write failing test for create organization use case**

Create `tests/organizations/test_organization_api.py`:

```python
from datetime import datetime, UTC
from unittest.mock import AsyncMock
from uuid import uuid4

import pytest

from prescient.organizations.application.use_cases.create_organization import (
    CreateOrganization,
    CreateOrganizationRequest,
    CreateOrganizationResult,
)
from prescient.organizations.domain.entities.organization import OrganizationType


class TestCreateOrganization:
    @pytest.fixture
    def mock_session(self) -> AsyncMock:
        session = AsyncMock()
        session.execute = AsyncMock(return_value=AsyncMock(scalar_one_or_none=lambda: None))
        return session

    @pytest.mark.asyncio
    async def test_create_independent_company(self, mock_session: AsyncMock) -> None:
        use_case = CreateOrganization(session=mock_session)
        result = await use_case.execute(
            CreateOrganizationRequest(
                name="Acme Corp",
                slug="acme-corp",
                org_type=OrganizationType.INDEPENDENT,
                industry="Manufacturing",
                description=None,
                website=None,
            )
        )
        assert result.slug == "acme-corp"
        assert result.organization_id is not None
        assert mock_session.add.called
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/organizations/test_organization_api.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement create organization use case**

Create `apps/api/src/prescient/organizations/application/use_cases/create_organization.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import uuid4

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.organizations.domain.entities.organization import OrganizationType
from prescient.organizations.domain.errors import DuplicateSlugError
from prescient.organizations.infrastructure.tables.organization import OrganizationRow


@dataclass(frozen=True)
class CreateOrganizationRequest:
    name: str
    slug: str
    org_type: OrganizationType
    industry: str | None = None
    description: str | None = None
    website: str | None = None


@dataclass(frozen=True)
class CreateOrganizationResult:
    organization_id: str
    slug: str
    name: str


class CreateOrganization:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, request: CreateOrganizationRequest) -> CreateOrganizationResult:
        existing = await self._session.execute(
            select(OrganizationRow).where(OrganizationRow.slug == request.slug)
        )
        if existing.scalar_one_or_none() is not None:
            raise DuplicateSlugError(request.slug)

        now = datetime.now(UTC)
        org_id = str(uuid4())
        row = OrganizationRow(
            id=org_id,
            slug=request.slug,
            name=request.name,
            org_type=request.org_type.value,
            status="active",
            industry=request.industry,
            description=request.description,
            website=request.website,
            created_at=now,
            updated_at=now,
        )
        self._session.add(row)

        return CreateOrganizationResult(
            organization_id=org_id,
            slug=request.slug,
            name=request.name,
        )
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd apps/api && python -m pytest tests/organizations/test_organization_api.py -v`
Expected: 1 PASSED

- [ ] **Step 6: Implement organization API routes**

Create `apps/api/src/prescient/organizations/api/routes.py`:

```python
from __future__ import annotations

from pydantic import BaseModel, Field
from fastapi import APIRouter, Depends
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.db import get_session
from prescient.organizations.application.use_cases.create_organization import (
    CreateOrganization,
    CreateOrganizationRequest,
)
from prescient.organizations.domain.entities.organization import OrganizationType
from prescient.organizations.infrastructure.tables.organization import OrganizationRow

router = APIRouter(prefix="/organizations", tags=["organizations"])


class CreateOrganizationBody(BaseModel):
    name: str = Field(min_length=1, max_length=256)
    slug: str = Field(min_length=1, max_length=64)
    org_type: OrganizationType
    industry: str | None = None
    description: str | None = None
    website: str | None = None


class OrganizationResponse(BaseModel):
    id: str
    slug: str
    name: str
    org_type: str
    status: str
    industry: str | None
    description: str | None
    website: str | None


@router.post("/", response_model=OrganizationResponse, status_code=201)
async def create_organization(
    body: CreateOrganizationBody,
    session: AsyncSession = Depends(get_session),
) -> OrganizationResponse:
    use_case = CreateOrganization(session=session)
    result = await use_case.execute(
        CreateOrganizationRequest(
            name=body.name,
            slug=body.slug,
            org_type=body.org_type,
            industry=body.industry,
            description=body.description,
            website=body.website,
        )
    )
    await session.commit()
    return OrganizationResponse(
        id=result.organization_id,
        slug=result.slug,
        name=result.name,
        org_type=body.org_type.value,
        status="active",
        industry=body.industry,
        description=body.description,
        website=body.website,
    )


@router.get("/", response_model=list[OrganizationResponse])
async def list_organizations(
    session: AsyncSession = Depends(get_session),
) -> list[OrganizationResponse]:
    result = await session.execute(select(OrganizationRow))
    rows = result.scalars().all()
    return [
        OrganizationResponse(
            id=r.id,
            slug=r.slug,
            name=r.name,
            org_type=r.org_type,
            status=r.status,
            industry=r.industry,
            description=r.description,
            website=r.website,
        )
        for r in rows
    ]
```

- [ ] **Step 7: Wire organization router into main.py**

In `apps/api/src/prescient/main.py`, add:

```python
from prescient.organizations.api.routes import router as organizations_router
```

And include: `app.include_router(organizations_router)`

- [ ] **Step 8: Run all tests**

Run: `cd apps/api && python -m pytest tests/organizations/ tests/auth/ tests/artifacts/ -v`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```bash
git add apps/api/src/prescient/organizations/ apps/api/src/prescient/main.py tests/organizations/
git commit -m "feat: add Organization API — create and list endpoints"
```

---

## Task 14: Artifact Store API Routes

**Files:**
- Create: `apps/api/src/prescient/artifacts/application/use_cases/create_artifact.py`
- Create: `apps/api/src/prescient/artifacts/application/use_cases/get_artifact.py`
- Create: `apps/api/src/prescient/artifacts/application/use_cases/add_reference.py`
- Create: `apps/api/src/prescient/artifacts/api/routes.py`
- Test: `tests/artifacts/test_artifact_api.py`

- [ ] **Step 1: Create application package structure**

Create empty `__init__.py` files:

```
apps/api/src/prescient/artifacts/application/__init__.py
apps/api/src/prescient/artifacts/application/use_cases/__init__.py
apps/api/src/prescient/artifacts/api/__init__.py
```

- [ ] **Step 2: Write failing test for create artifact use case**

Create `tests/artifacts/test_artifact_api.py`:

```python
from datetime import datetime, UTC
from unittest.mock import AsyncMock
from uuid import uuid4

import pytest

from prescient.artifacts.application.use_cases.create_artifact import (
    CreateArtifact,
    CreateArtifactRequest,
)
from prescient.artifacts.domain.entities.artifact import ArtifactType


class TestCreateArtifact:
    @pytest.fixture
    def mock_session(self) -> AsyncMock:
        return AsyncMock()

    @pytest.mark.asyncio
    async def test_create_company_profile(self, mock_session: AsyncMock) -> None:
        use_case = CreateArtifact(session=mock_session)
        result = await use_case.execute(
            CreateArtifactRequest(
                organization_id=str(uuid4()),
                artifact_type=ArtifactType.COMPANY_PROFILE,
                title="Olaplex Holdings — Company Profile",
                created_by="ryan",
                summary="Initial company research profile",
                blocks=[
                    {"block_id": "overview", "heading": "Overview", "body": "Olaplex is a beauty company.", "order": 0},
                ],
            )
        )
        assert result.artifact_id is not None
        assert result.version_id is not None
        assert mock_session.add.called
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/api && python -m pytest tests/artifacts/test_artifact_api.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement create artifact use case**

Create `apps/api/src/prescient/artifacts/application/use_cases/create_artifact.py`:

```python
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, UTC
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.domain.entities.artifact import ArtifactType
from prescient.artifacts.infrastructure.tables.artifact import ArtifactRow
from prescient.artifacts.infrastructure.tables.artifact_version import ArtifactVersionRow


@dataclass(frozen=True)
class CreateArtifactRequest:
    organization_id: str
    artifact_type: ArtifactType
    title: str
    created_by: str
    summary: str | None = None
    blocks: list[dict] = field(default_factory=list)
    visibility: str = "shared"


@dataclass(frozen=True)
class CreateArtifactResult:
    artifact_id: str
    version_id: str


class CreateArtifact:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, request: CreateArtifactRequest) -> CreateArtifactResult:
        now = datetime.now(UTC)
        artifact_id = str(uuid4())
        version_id = str(uuid4())

        artifact_row = ArtifactRow(
            id=artifact_id,
            organization_id=request.organization_id,
            artifact_type=request.artifact_type.value,
            title=request.title,
            visibility=request.visibility,
            active_version_id=version_id,
            created_at=now,
            updated_at=now,
        )
        self._session.add(artifact_row)

        version_row = ArtifactVersionRow(
            id=version_id,
            artifact_id=artifact_id,
            version_number=1,
            state="draft",
            confidence_label="medium",
            blocks=request.blocks,
            citations=[],
            summary=request.summary,
            created_by=request.created_by,
            created_at=now,
        )
        self._session.add(version_row)

        return CreateArtifactResult(artifact_id=artifact_id, version_id=version_id)
```

- [ ] **Step 5: Implement get artifact use case**

Create `apps/api/src/prescient/artifacts/application/use_cases/get_artifact.py`:

```python
from __future__ import annotations

from dataclasses import dataclass

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.domain.errors import ArtifactNotFoundError
from prescient.artifacts.infrastructure.tables.artifact import ArtifactRow
from prescient.artifacts.infrastructure.tables.artifact_version import ArtifactVersionRow


@dataclass(frozen=True)
class ArtifactDetail:
    artifact_id: str
    organization_id: str
    artifact_type: str
    title: str
    visibility: str
    active_version_id: str | None
    versions: list[dict]


class GetArtifact:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, artifact_id: str) -> ArtifactDetail:
        result = await self._session.execute(
            select(ArtifactRow).where(ArtifactRow.id == artifact_id)
        )
        row = result.scalar_one_or_none()
        if row is None:
            raise ArtifactNotFoundError(artifact_id)

        versions_result = await self._session.execute(
            select(ArtifactVersionRow)
            .where(ArtifactVersionRow.artifact_id == artifact_id)
            .order_by(ArtifactVersionRow.version_number)
        )
        version_rows = versions_result.scalars().all()

        return ArtifactDetail(
            artifact_id=row.id,
            organization_id=row.organization_id,
            artifact_type=row.artifact_type,
            title=row.title,
            visibility=row.visibility,
            active_version_id=row.active_version_id,
            versions=[
                {
                    "id": v.id,
                    "version_number": v.version_number,
                    "state": v.state,
                    "confidence_label": v.confidence_label,
                    "summary": v.summary,
                    "blocks": v.blocks,
                    "created_by": v.created_by,
                    "created_at": v.created_at.isoformat() if v.created_at else None,
                }
                for v in version_rows
            ],
        )
```

- [ ] **Step 6: Implement add reference use case**

Create `apps/api/src/prescient/artifacts/application/use_cases/add_reference.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.domain.entities.artifact_reference import ReferenceType
from prescient.artifacts.infrastructure.tables.artifact_reference import ArtifactReferenceRow


@dataclass(frozen=True)
class AddReferenceRequest:
    source_artifact_id: str
    target_artifact_id: str
    reference_type: ReferenceType
    description: str | None = None


@dataclass(frozen=True)
class AddReferenceResult:
    reference_id: str


class AddReference:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(self, request: AddReferenceRequest) -> AddReferenceResult:
        if request.source_artifact_id == request.target_artifact_id:
            msg = "An artifact cannot reference itself"
            raise ValueError(msg)

        now = datetime.now(UTC)
        ref_id = str(uuid4())
        row = ArtifactReferenceRow(
            id=ref_id,
            source_artifact_id=request.source_artifact_id,
            target_artifact_id=request.target_artifact_id,
            reference_type=request.reference_type.value,
            description=request.description,
            created_at=now,
        )
        self._session.add(row)

        return AddReferenceResult(reference_id=ref_id)
```

- [ ] **Step 7: Implement artifact API routes**

Create `apps/api/src/prescient/artifacts/api/routes.py`:

```python
from __future__ import annotations

from pydantic import BaseModel, Field
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.artifacts.application.use_cases.create_artifact import (
    CreateArtifact,
    CreateArtifactRequest,
)
from prescient.artifacts.application.use_cases.get_artifact import GetArtifact
from prescient.artifacts.application.use_cases.add_reference import (
    AddReference,
    AddReferenceRequest,
)
from prescient.artifacts.domain.entities.artifact import ArtifactType
from prescient.artifacts.domain.entities.artifact_reference import ReferenceType
from prescient.db import get_session

router = APIRouter(prefix="/artifacts", tags=["artifacts"])


class CreateArtifactBody(BaseModel):
    organization_id: str
    artifact_type: ArtifactType
    title: str = Field(min_length=1, max_length=256)
    created_by: str
    summary: str | None = None
    blocks: list[dict] = []
    visibility: str = "shared"


class ArtifactResponse(BaseModel):
    artifact_id: str
    version_id: str


class ArtifactDetailResponse(BaseModel):
    artifact_id: str
    organization_id: str
    artifact_type: str
    title: str
    visibility: str
    active_version_id: str | None
    versions: list[dict]


class AddReferenceBody(BaseModel):
    source_artifact_id: str
    target_artifact_id: str
    reference_type: ReferenceType
    description: str | None = None


class ReferenceResponse(BaseModel):
    reference_id: str


@router.post("/", response_model=ArtifactResponse, status_code=201)
async def create_artifact(
    body: CreateArtifactBody,
    session: AsyncSession = Depends(get_session),
) -> ArtifactResponse:
    use_case = CreateArtifact(session=session)
    result = await use_case.execute(
        CreateArtifactRequest(
            organization_id=body.organization_id,
            artifact_type=body.artifact_type,
            title=body.title,
            created_by=body.created_by,
            summary=body.summary,
            blocks=body.blocks,
            visibility=body.visibility,
        )
    )
    await session.commit()
    return ArtifactResponse(artifact_id=result.artifact_id, version_id=result.version_id)


@router.get("/{artifact_id}", response_model=ArtifactDetailResponse)
async def get_artifact(
    artifact_id: str,
    session: AsyncSession = Depends(get_session),
) -> ArtifactDetailResponse:
    use_case = GetArtifact(session=session)
    detail = await use_case.execute(artifact_id)
    return ArtifactDetailResponse(
        artifact_id=detail.artifact_id,
        organization_id=detail.organization_id,
        artifact_type=detail.artifact_type,
        title=detail.title,
        visibility=detail.visibility,
        active_version_id=detail.active_version_id,
        versions=detail.versions,
    )


@router.post("/references", response_model=ReferenceResponse, status_code=201)
async def add_reference(
    body: AddReferenceBody,
    session: AsyncSession = Depends(get_session),
) -> ReferenceResponse:
    use_case = AddReference(session=session)
    result = await use_case.execute(
        AddReferenceRequest(
            source_artifact_id=body.source_artifact_id,
            target_artifact_id=body.target_artifact_id,
            reference_type=body.reference_type,
            description=body.description,
        )
    )
    await session.commit()
    return ReferenceResponse(reference_id=result.reference_id)
```

- [ ] **Step 8: Wire artifact router into main.py**

In `apps/api/src/prescient/main.py`, add:

```python
from prescient.artifacts.api.routes import router as artifacts_router
```

And include: `app.include_router(artifacts_router)`

- [ ] **Step 9: Run all tests**

Run: `cd apps/api && python -m pytest tests/ -v`
Expected: All tests pass.

- [ ] **Step 10: Commit**

```bash
git add apps/api/src/prescient/artifacts/ apps/api/src/prescient/main.py tests/artifacts/
git commit -m "feat: add Artifact Store API — create, get, and add reference endpoints"
```

---

## Task 15: Lint, Typecheck, and Final Verification

**Files:** None new — verification only.

- [ ] **Step 1: Run linter**

Run: `cd apps/api && python -m ruff check src/ tests/`
Expected: No errors (or only pre-existing ones).

- [ ] **Step 2: Run type checker**

Run: `cd apps/api && python -m mypy src/prescient/organizations/ src/prescient/artifacts/ src/prescient/auth/`
Expected: No errors.

- [ ] **Step 3: Run frontend type check**

Run: `cd apps/web && npx tsc --noEmit`
Expected: No errors.

- [ ] **Step 4: Run full test suite**

Run: `cd apps/api && python -m pytest tests/ -v`
Expected: All tests pass.

- [ ] **Step 5: Fix any issues found**

Address lint, type, or test failures.

- [ ] **Step 6: Final commit if fixes were needed**

```bash
git add -A
git commit -m "fix: address lint and typecheck issues from Phase 1 foundation"
```

---

## Summary

Phase 1 delivers:
- **Organization & Access context** — Organization, User (with 6 roles), Membership, PortfolioLink (with billing models), VisibilityRule (Tier B defaults, Tier C configurable)
- **Artifact Store context** — 8 artifact types, versioned content with blocks/citations, cross-references between artifacts, state machine (draft → active → archived)
- **Auth context** — email/password registration, bcrypt hashing, JWT access tokens, login/register API endpoints
- **API scaffold** — FastAPI with auth, organizations, and artifacts routers
- **Frontend scaffold** — Login page, auth provider, morning brief placeholder, operator-first navigation
- **Database** — Alembic migration for 3 new schemas (organizations, artifacts, auth)

Next: **Phase 2 — Intelligence Engine, Onboarding Wizard, Ad-hoc Company Research**
