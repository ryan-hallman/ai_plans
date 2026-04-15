# Knowledge Engine Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish the data model, S3 storage, RLS policies, and domain layer for the Knowledge Engine — the foundation that Plans 2-6 build on.

**Architecture:** Evolve the existing `knowledge_mgmt` bounded context. Add new enums (IngestionStatus, TrustTier, expanded SourceType/Visibility), denormalize access-critical fields onto chunks/source_versions/entity_mentions, replace LocalFileStorage with S3 (boto3 async via aioboto3), and add RLS policies to all KM tables keyed on `organization_id` (references `organizations.organizations.id`, not the legacy `tenants.funds`).

**Tech Stack:** Python 3.12, SQLAlchemy 2.0 (async), Alembic, Pydantic v2, aioboto3, Postgres 16 (RLS), LocalStack (S3 for dev)

**Key context from slug-to-UUID migration:**
- The codebase recently migrated from slug-based to UUID-based identifiers
- `ToolContext` still carries `primary_company_slug` solely for OpenSearch index naming — we will eliminate this dependency for KM by using UUID-based index names
- `CurrentUser` carries `fund_id` (tenant) and `primary_company_id` (UUID) — the KE introduces `organization_id` from `AccessContext.org_id` as the isolation boundary
- `AccessContext` already resolves `org_id` from `organizations.memberships` — we use this for KM RLS
- This is greenfield (not in production) — we truncate existing KM seed data and rebuild clean rather than doing complex data migration

---

## File Structure

### New files
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/s3_storage.py` — S3FileStorage implementation
- `apps/api/alembic/versions/20260414_ke_foundation.py` — Schema migration: truncate + rebuild KM tables, RLS
- `apps/api/tests/unit/knowledge_mgmt/test_s3_storage.py` — S3 storage unit tests
- `apps/api/tests/unit/knowledge_mgmt/test_domain_entities.py` — Domain entity/enum tests
- `infrastructure/localstack/init-s3.sh` — Create S3 buckets on LocalStack startup

### Modified files
- `apps/api/src/prescient/knowledge_mgmt/domain/entities/source.py` — New enums (IngestionStatus, TrustTier), expanded SourceType/Visibility, new fields on Source/SourceVersion
- `apps/api/src/prescient/knowledge_mgmt/domain/entities/chunk.py` — Add span offsets, page_number, denormalized access fields
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/source.py` — Rebuilt with organization_id, storage_key, trust_tier, etc.
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/chunk.py` — Rebuilt with denormalized access fields, span offsets
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/entity.py` — organization_id replaces tenant_id
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/source_repository.py` — Handle new fields, organization_id
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/chunk_repository.py` — Handle denormalized fields
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/entity_repository.py` — tenant_id → organization_id
- `apps/api/src/prescient/knowledge_mgmt/application/use_cases/create_source.py` — Use organization_id, storage_key, trust_tier, create initial source_version
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/file_storage.py` — Update Protocol signature, remove LocalFileStorage
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py` — INDEX_TEMPLATE uses UUID, tenant_slug → organization_id
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingest_celery.py` — S3FileStorage, organization_id, app.current_org_id GUC
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingestion.py` — tenant_id → organization_id
- `apps/api/src/prescient/knowledge_mgmt/api/routes.py` — organization_id from AccessContext, new response fields
- `apps/api/src/prescient/config/base.py` — Add S3 settings
- `apps/api/src/prescient/db.py` — Also set app.current_org_id GUC when AccessContext is available
- `apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py` — Use organization_id for index naming instead of slug

---

## Tasks

### Task 1: Update domain enums and entities

**Files:**
- Modify: `apps/api/src/prescient/knowledge_mgmt/domain/entities/source.py`
- Modify: `apps/api/src/prescient/knowledge_mgmt/domain/entities/chunk.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_domain_entities.py`

- [ ] **Step 1: Write tests for new enums and entity fields**

```python
# apps/api/tests/unit/knowledge_mgmt/test_domain_entities.py
"""Unit tests for Knowledge Engine domain entities and enums."""
from __future__ import annotations

import pytest

from prescient.knowledge_mgmt.domain.entities.source import (
    IngestionStatus,
    Source,
    SourceType,
    SourceVersion,
    TrustTier,
    Visibility,
)
from prescient.knowledge_mgmt.domain.entities.chunk import Chunk


class TestSourceTypeEnum:
    def test_uploaded_document(self):
        assert SourceType.UPLOADED_DOCUMENT == "uploaded_document"

    def test_research_output(self):
        assert SourceType.RESEARCH_OUTPUT == "research_output"

    def test_filing(self):
        assert SourceType.FILING == "filing"

    def test_artifact(self):
        assert SourceType.ARTIFACT == "artifact"

    def test_transcript(self):
        assert SourceType.TRANSCRIPT == "transcript"

    def test_news(self):
        assert SourceType.NEWS == "news"

    def test_authored(self):
        assert SourceType.AUTHORED == "authored"

    def test_legacy_file_upload_removed(self):
        """Old FILE_UPLOAD value no longer exists."""
        with pytest.raises(ValueError):
            SourceType("file_upload")


class TestIngestionStatusEnum:
    def test_all_stages_present(self):
        expected = {
            "pending", "scanning", "extracting", "chunking",
            "embedding", "enriching", "ready", "failed", "quarantined",
        }
        actual = {s.value for s in IngestionStatus}
        assert actual == expected


class TestTrustTierEnum:
    def test_all_tiers_present(self):
        expected = {
            "approved_internal", "internal_raw", "system_generated",
            "external_verified", "external_raw",
        }
        actual = {t.value for t in TrustTier}
        assert actual == expected


class TestVisibilityEnum:
    def test_org_wide_added(self):
        assert Visibility.ORG_WIDE == "org_wide"

    def test_company_wide(self):
        assert Visibility.COMPANY_WIDE == "company_wide"

    def test_department_only(self):
        assert Visibility.DEPARTMENT_ONLY == "department_only"


class TestSourceEntity:
    def test_source_has_organization_id(self):
        s = Source(
            id="src-1",
            organization_id="org-1",
            company_id="co-1",
            title="Test",
            source_type=SourceType.UPLOADED_DOCUMENT,
            created_by="user-1",
            updated_by="user-1",
        )
        assert s.organization_id == "org-1"

    def test_source_defaults(self):
        s = Source(
            id="src-1",
            organization_id="org-1",
            company_id="co-1",
            title="Test",
            source_type=SourceType.UPLOADED_DOCUMENT,
            created_by="user-1",
            updated_by="user-1",
        )
        assert s.ingestion_status == IngestionStatus.PENDING
        assert s.trust_tier == TrustTier.INTERNAL_RAW
        assert s.visibility == Visibility.COMPANY_WIDE
        assert s.storage_key is None
        assert s.processing_profile is None
        assert s.current_version_id is None

    def test_source_storage_key_replaces_storage_path(self):
        s = Source(
            id="src-1",
            organization_id="org-1",
            company_id="co-1",
            title="Test",
            source_type=SourceType.UPLOADED_DOCUMENT,
            storage_key="dirty/org-1/src-1/file.pdf",
            created_by="user-1",
            updated_by="user-1",
        )
        assert s.storage_key == "dirty/org-1/src-1/file.pdf"


class TestSourceVersionEntity:
    def test_source_version_has_denormalized_fields(self):
        sv = SourceVersion(
            id="sv-1",
            source_id="src-1",
            organization_id="org-1",
            company_id="co-1",
            version_number=1,
            storage_key="clean/org-1/src-1/v1/file.pdf",
            visibility=Visibility.COMPANY_WIDE,
            min_access_role="OPERATOR",
            trust_tier=TrustTier.INTERNAL_RAW,
            created_by="user-1",
        )
        assert sv.organization_id == "org-1"
        assert sv.company_id == "co-1"
        assert sv.trust_tier == TrustTier.INTERNAL_RAW


class TestChunkEntity:
    def test_chunk_has_span_fields(self):
        c = Chunk(
            id="ch-1",
            source_id="src-1",
            source_version_id="sv-1",
            organization_id="org-1",
            chunk_index=0,
            content="Test content",
            page_number=3,
            section_heading="Introduction",
            span_start=0,
            span_end=12,
        )
        assert c.span_start == 0
        assert c.span_end == 12
        assert c.page_number == 3
        assert c.section_heading == "Introduction"

    def test_chunk_has_denormalized_access_fields(self):
        c = Chunk(
            id="ch-1",
            source_id="src-1",
            source_version_id="sv-1",
            organization_id="org-1",
            company_id="co-1",
            chunk_index=0,
            content="Test content",
            visibility=Visibility.COMPANY_WIDE,
            min_access_role="OPERATOR",
            trust_tier=TrustTier.APPROVED_INTERNAL,
            span_start=0,
            span_end=12,
        )
        assert c.organization_id == "org-1"
        assert c.visibility == Visibility.COMPANY_WIDE
        assert c.trust_tier == TrustTier.APPROVED_INTERNAL
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_domain_entities.py -v`
Expected: FAIL — new enums and fields don't exist yet.

- [ ] **Step 3: Update source.py domain entities**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/domain/entities/source.py`:

```python
from __future__ import annotations

import enum
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class SourceType(str, enum.Enum):
    UPLOADED_DOCUMENT = "uploaded_document"
    RESEARCH_OUTPUT = "research_output"
    FILING = "filing"
    ARTIFACT = "artifact"
    TRANSCRIPT = "transcript"
    NEWS = "news"
    AUTHORED = "authored"


class IngestionStatus(str, enum.Enum):
    PENDING = "pending"
    SCANNING = "scanning"
    EXTRACTING = "extracting"
    CHUNKING = "chunking"
    EMBEDDING = "embedding"
    ENRICHING = "enriching"
    READY = "ready"
    FAILED = "failed"
    QUARANTINED = "quarantined"


class Visibility(str, enum.Enum):
    ORG_WIDE = "org_wide"
    COMPANY_WIDE = "company_wide"
    DEPARTMENT_ONLY = "department_only"


class TrustTier(str, enum.Enum):
    APPROVED_INTERNAL = "approved_internal"
    INTERNAL_RAW = "internal_raw"
    SYSTEM_GENERATED = "system_generated"
    EXTERNAL_VERIFIED = "external_verified"
    EXTERNAL_RAW = "external_raw"


class Source(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    organization_id: str
    company_id: str | None = None
    title: str
    source_type: SourceType
    original_filename: str | None = None
    mime_type: str | None = None
    storage_key: str | None = None
    current_version_id: str | None = None
    ingestion_status: IngestionStatus = IngestionStatus.PENDING
    processing_profile: str | None = None
    visibility: Visibility = Visibility.COMPANY_WIDE
    min_access_role: str = "OPERATOR"
    department_id: str | None = None
    trust_tier: TrustTier = TrustTier.INTERNAL_RAW
    created_by: str
    updated_by: str
    meta: dict = Field(default_factory=dict)
    created_at: datetime | None = None
    updated_at: datetime | None = None


class SourceVersion(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    source_id: str
    organization_id: str
    company_id: str | None = None
    version_number: int
    storage_key: str
    visibility: Visibility = Visibility.COMPANY_WIDE
    min_access_role: str = "OPERATOR"
    department_id: str | None = None
    trust_tier: TrustTier = TrustTier.INTERNAL_RAW
    change_summary: str | None = None
    created_by: str
    created_at: datetime | None = None
```

- [ ] **Step 4: Update chunk.py domain entity**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/domain/entities/chunk.py`:

```python
from __future__ import annotations

from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field

from prescient.knowledge_mgmt.domain.entities.source import TrustTier, Visibility


class Chunk(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    source_id: str
    source_version_id: str | None = None
    organization_id: str | None = None
    company_id: str | None = None
    chunk_index: int
    content: str
    summary: str | None = None
    token_count: int | None = None
    page_number: int | None = None
    section_heading: str | None = None
    span_start: int | None = None
    span_end: int | None = None
    visibility: Visibility | None = None
    min_access_role: str | None = None
    department_id: str | None = None
    trust_tier: TrustTier | None = None
    meta: dict = Field(default_factory=dict)
    created_at: datetime | None = None
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_domain_entities.py -v`
Expected: All PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/domain/entities/source.py \
       apps/api/src/prescient/knowledge_mgmt/domain/entities/chunk.py \
       apps/api/tests/unit/knowledge_mgmt/test_domain_entities.py
git commit -m "feat(ke): update domain entities with KE enums and denormalized fields"
```

---

### Task 2: Update ORM table definitions

**Files:**
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/source.py`
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/chunk.py`
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/entity.py`

- [ ] **Step 1: Update SourceRow and SourceVersionRow**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/source.py`:

```python
"""`km.sources` and `km.source_versions` tables."""

from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Integer, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class SourceRow(Base):
    __tablename__ = "sources"
    __table_args__ = {"schema": "km"}  # noqa: RUF012

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    company_id: Mapped[str | None] = mapped_column(String(36), nullable=True, index=True)
    title: Mapped[str] = mapped_column(Text(), nullable=False)
    source_type: Mapped[str] = mapped_column(String(32), nullable=False)
    original_filename: Mapped[str | None] = mapped_column(Text(), nullable=True)
    mime_type: Mapped[str | None] = mapped_column(String(128), nullable=True)
    storage_key: Mapped[str | None] = mapped_column(Text(), nullable=True)
    current_version_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    ingestion_status: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="pending"
    )
    processing_profile: Mapped[str | None] = mapped_column(String(64), nullable=True)
    department_id: Mapped[str | None] = mapped_column(
        String(36),
        ForeignKey("km.departments.id", ondelete="SET NULL"),
        nullable=True,
    )
    visibility: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="company_wide"
    )
    min_access_role: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="OPERATOR"
    )
    trust_tier: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="internal_raw"
    )
    created_by: Mapped[str] = mapped_column(String(64), nullable=False)
    updated_by: Mapped[str] = mapped_column(String(64), nullable=False)
    meta: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="{}")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        server_default=func.now(),
        onupdate=func.now(),
    )


class SourceVersionRow(Base):
    __tablename__ = "source_versions"
    __table_args__ = {"schema": "km"}  # noqa: RUF012

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    company_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    source_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("km.sources.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    version_number: Mapped[int] = mapped_column(Integer(), nullable=False)
    storage_key: Mapped[str] = mapped_column(Text(), nullable=False)
    visibility: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="company_wide"
    )
    min_access_role: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="OPERATOR"
    )
    department_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    trust_tier: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="internal_raw"
    )
    change_summary: Mapped[str | None] = mapped_column(Text(), nullable=True)
    created_by: Mapped[str] = mapped_column(String(64), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
```

- [ ] **Step 2: Update ChunkRow**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/chunk.py`:

```python
"""`km.chunks` table."""

from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Integer, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class ChunkRow(Base):
    __tablename__ = "chunks"
    __table_args__ = {"schema": "km"}  # noqa: RUF012

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    company_id: Mapped[str | None] = mapped_column(String(36), nullable=True, index=True)
    source_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("km.sources.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    source_version_id: Mapped[str | None] = mapped_column(
        String(36),
        ForeignKey("km.source_versions.id", ondelete="CASCADE"),
        nullable=True,
    )
    chunk_index: Mapped[int] = mapped_column(Integer(), nullable=False)
    content: Mapped[str] = mapped_column(Text(), nullable=False)
    summary: Mapped[str | None] = mapped_column(Text(), nullable=True)
    token_count: Mapped[int | None] = mapped_column(Integer(), nullable=True)
    page_number: Mapped[int | None] = mapped_column(Integer(), nullable=True)
    section_heading: Mapped[str | None] = mapped_column(Text(), nullable=True)
    span_start: Mapped[int | None] = mapped_column(Integer(), nullable=True)
    span_end: Mapped[int | None] = mapped_column(Integer(), nullable=True)
    visibility: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="company_wide"
    )
    min_access_role: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="OPERATOR"
    )
    department_id: Mapped[str | None] = mapped_column(String(36), nullable=True)
    trust_tier: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="internal_raw"
    )
    meta: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="{}")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
```

- [ ] **Step 3: Update entity tables**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/entity.py`:

```python
"""`km.entities` and `km.entity_mentions` tables."""

from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class KMEntityRow(Base):
    __tablename__ = "entities"
    __table_args__ = {"schema": "km"}  # noqa: RUF012

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    name: Mapped[str] = mapped_column(Text(), nullable=False)
    entity_type: Mapped[str] = mapped_column(String(64), nullable=False)
    meta: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="{}")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )


class EntityMentionRow(Base):
    __tablename__ = "entity_mentions"
    __table_args__ = {"schema": "km"}  # noqa: RUF012

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    organization_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    entity_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("km.entities.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    chunk_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("km.chunks.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    context_snippet: Mapped[str | None] = mapped_column(Text(), nullable=True)
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/
git commit -m "feat(ke): update ORM tables with KE columns and denormalized fields"
```

---

### Task 3: Alembic migration — truncate + rebuild KM tables with RLS

**Files:**
- Create: `apps/api/alembic/versions/20260414_ke_foundation.py`

Since this is greenfield with no production data, the cleanest approach is to drop and recreate the KM tables with the correct schema rather than doing incremental ALTER TABLE operations. The migration truncates all KM data (seed/demo only) and rebuilds.

- [ ] **Step 1: Check current alembic heads and merge if needed**

Run: `cd apps/api && uv run alembic heads`

If multiple heads exist, create a merge migration first:
```bash
uv run alembic merge heads -m "merge heads before KE foundation"
```

- [ ] **Step 2: Write the migration**

Create `apps/api/alembic/versions/20260414_ke_foundation.py`:

```python
"""Knowledge Engine foundation: rebuild KM tables with new schema + RLS.

Revision ID: 20260414_ke_foundation
Revises: <SET_AFTER_MERGE>
Create Date: 2026-04-14

Greenfield rebuild of the km schema for the Knowledge Engine:
- Replaces tenant_id with organization_id (references organizations.organizations)
- Replaces storage_path with storage_key (S3 keys)
- Replaces status with ingestion_status (granular pipeline stages)
- Adds trust_tier, processing_profile, current_version_id to sources
- Denormalizes organization_id, company_id, visibility, min_access_role,
  department_id, trust_tier onto source_versions, chunks, entity_mentions
- Adds page_number, section_heading, span_start, span_end to chunks
- Enables RLS on all km tables keyed on organization_id
- Adds app.current_org_id GUC for RLS (separate from app.current_fund_id)

Since this is greenfield (no production data), we truncate existing KM
data and rebuild the tables cleanly rather than doing incremental ALTERs.
"""

from __future__ import annotations

from collections.abc import Sequence

import sqlalchemy as sa
from alembic import op

revision: str = "20260414_ke_foundation"
down_revision: str | None = "<SET_AFTER_MERGE>"  # Set to merged head
branch_labels: str | Sequence[str] | None = None
depends_on: str | Sequence[str] | None = None

# Tables in dependency order (children first for DROP, parents first for CREATE)
KM_TABLES_DROP_ORDER = [
    "entity_mentions",
    "entities",
    "chunks",
    "source_versions",
    "sources",
    "department_memberships",
    "departments",
]

KM_TABLES_FOR_RLS = [
    "sources",
    "source_versions",
    "chunks",
    "entities",
    "entity_mentions",
    "departments",
    "department_memberships",
]


def upgrade() -> None:
    # ── Truncate all KM data (greenfield — no production data) ──
    for table in KM_TABLES_DROP_ORDER:
        op.execute(f"TRUNCATE TABLE km.{table} CASCADE")

    # ── km.departments: rename tenant_id → organization_id ──────
    op.alter_column("departments", "tenant_id", new_column_name="organization_id", schema="km")

    # ── km.department_memberships: add organization_id ──────────
    op.add_column(
        "department_memberships",
        sa.Column("organization_id", sa.String(36), nullable=False, server_default=""),
        schema="km",
    )

    # ── km.sources: rebuild columns ─────────────────────────────
    op.alter_column("sources", "tenant_id", new_column_name="organization_id", schema="km")
    op.alter_column("sources", "storage_path", new_column_name="storage_key", schema="km")
    op.alter_column("sources", "status", new_column_name="ingestion_status", schema="km")
    op.alter_column("sources", "company_id", nullable=True, schema="km")
    op.add_column("sources", sa.Column("trust_tier", sa.String(32), nullable=False, server_default="internal_raw"), schema="km")
    op.add_column("sources", sa.Column("processing_profile", sa.String(64), nullable=True), schema="km")
    op.add_column("sources", sa.Column("current_version_id", sa.String(36), nullable=True), schema="km")

    # ── km.source_versions: add denormalized columns ────────────
    op.alter_column("source_versions", "storage_path", new_column_name="storage_key", schema="km")
    op.add_column("source_versions", sa.Column("organization_id", sa.String(36), nullable=False, server_default=""), schema="km")
    op.add_column("source_versions", sa.Column("company_id", sa.String(36), nullable=True), schema="km")
    op.add_column("source_versions", sa.Column("visibility", sa.String(32), nullable=False, server_default="company_wide"), schema="km")
    op.add_column("source_versions", sa.Column("min_access_role", sa.String(32), nullable=False, server_default="OPERATOR"), schema="km")
    op.add_column("source_versions", sa.Column("department_id", sa.String(36), nullable=True), schema="km")
    op.add_column("source_versions", sa.Column("trust_tier", sa.String(32), nullable=False, server_default="internal_raw"), schema="km")
    op.create_index("ix_km_source_versions_organization_id", "source_versions", ["organization_id"], schema="km")

    # ── km.chunks: add denormalized + span columns ──────────────
    op.add_column("chunks", sa.Column("organization_id", sa.String(36), nullable=False, server_default=""), schema="km")
    op.add_column("chunks", sa.Column("company_id", sa.String(36), nullable=True), schema="km")
    op.add_column("chunks", sa.Column("page_number", sa.Integer(), nullable=True), schema="km")
    op.add_column("chunks", sa.Column("section_heading", sa.Text(), nullable=True), schema="km")
    op.add_column("chunks", sa.Column("span_start", sa.Integer(), nullable=True), schema="km")
    op.add_column("chunks", sa.Column("span_end", sa.Integer(), nullable=True), schema="km")
    op.add_column("chunks", sa.Column("visibility", sa.String(32), nullable=False, server_default="company_wide"), schema="km")
    op.add_column("chunks", sa.Column("min_access_role", sa.String(32), nullable=False, server_default="OPERATOR"), schema="km")
    op.add_column("chunks", sa.Column("department_id", sa.String(36), nullable=True), schema="km")
    op.add_column("chunks", sa.Column("trust_tier", sa.String(32), nullable=False, server_default="internal_raw"), schema="km")
    op.create_index("ix_km_chunks_organization_id", "chunks", ["organization_id"], schema="km")
    op.create_index("ix_km_chunks_company_id", "chunks", ["company_id"], schema="km")

    # ── km.entities: rename tenant_id → organization_id ─────────
    op.alter_column("entities", "tenant_id", new_column_name="organization_id", schema="km")

    # ── km.entity_mentions: add organization_id ─────────────────
    op.add_column("entity_mentions", sa.Column("organization_id", sa.String(36), nullable=False, server_default=""), schema="km")
    op.create_index("ix_km_entity_mentions_organization_id", "entity_mentions", ["organization_id"], schema="km")

    # ── RLS on all km tables ────────────────────────────────────
    for table in KM_TABLES_FOR_RLS:
        qualified = f"km.{table}"
        policy_name = f"{table}_org_isolation"
        op.execute(f"ALTER TABLE {qualified} ENABLE ROW LEVEL SECURITY")
        op.execute(f"ALTER TABLE {qualified} FORCE ROW LEVEL SECURITY")
        op.execute(
            f"CREATE POLICY {policy_name} ON {qualified} "
            f"USING (organization_id = current_setting('app.current_org_id', true)::text)"
        )


def downgrade() -> None:
    # Remove RLS
    for table in reversed(KM_TABLES_FOR_RLS):
        qualified = f"km.{table}"
        policy_name = f"{table}_org_isolation"
        op.execute(f"DROP POLICY IF EXISTS {policy_name} ON {qualified}")
        op.execute(f"ALTER TABLE {qualified} DISABLE ROW LEVEL SECURITY")
        op.execute(f"ALTER TABLE {qualified} NO FORCE ROW LEVEL SECURITY")

    # Reverse column renames
    op.alter_column("sources", "organization_id", new_column_name="tenant_id", schema="km")
    op.alter_column("sources", "storage_key", new_column_name="storage_path", schema="km")
    op.alter_column("sources", "ingestion_status", new_column_name="status", schema="km")
    op.alter_column("sources", "company_id", nullable=False, schema="km")
    op.alter_column("entities", "organization_id", new_column_name="tenant_id", schema="km")
    op.alter_column("source_versions", "storage_key", new_column_name="storage_path", schema="km")
    op.alter_column("departments", "organization_id", new_column_name="tenant_id", schema="km")

    # Drop added columns
    for col in ["trust_tier", "processing_profile", "current_version_id"]:
        op.drop_column("sources", col, schema="km")
    for col in ["organization_id", "company_id", "visibility", "min_access_role", "department_id", "trust_tier"]:
        op.drop_column("source_versions", col, schema="km")
    for col in ["organization_id", "company_id", "page_number", "section_heading", "span_start", "span_end", "visibility", "min_access_role", "department_id", "trust_tier"]:
        op.drop_column("chunks", col, schema="km")
    op.drop_column("entity_mentions", "organization_id", schema="km")
    op.drop_column("department_memberships", "organization_id", schema="km")
```

- [ ] **Step 3: Run the migration**

Run: `cd apps/api && uv run alembic upgrade head`
Verify: `uv run alembic current` shows the new revision.

- [ ] **Step 4: Verify RLS is enabled**

```bash
docker exec -i prescient-postgres psql -U prescient -d prescient -c "
SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'km';
"
```

Expected: All KM tables show `rowsecurity = t`.

- [ ] **Step 5: Commit**

```bash
git add apps/api/alembic/versions/
git commit -m "feat(ke): add KE foundation migration — schema evolution + RLS"
```

---

### Task 4: S3 file storage service

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/s3_storage.py`
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/file_storage.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_s3_storage.py`
- Modify: `apps/api/src/prescient/config/base.py`

- [ ] **Step 1: Add S3 settings to config**

In `apps/api/src/prescient/config/base.py`, add after the `file_storage_path` line:

```python
    s3_endpoint_url: str = Field(default="http://localhost:4566")
    s3_dirty_bucket: str = Field(default="prescient-dirty")
    s3_clean_bucket: str = Field(default="prescient-clean")
    aws_region: str = Field(default="us-east-1")
```

- [ ] **Step 2: Write S3 storage tests**

```python
# apps/api/tests/unit/knowledge_mgmt/test_s3_storage.py
"""Unit tests for S3FileStorage using a fake S3 client."""
from __future__ import annotations

import pytest

from prescient.knowledge_mgmt.infrastructure.services.s3_storage import S3FileStorage


class FakeS3Body:
    def __init__(self, data: bytes):
        self._data = data

    async def read(self) -> bytes:
        return self._data


class FakeS3Client:
    """In-memory S3 stub for unit tests."""

    def __init__(self):
        self.buckets: dict[str, dict[str, bytes]] = {}

    async def put_object(self, *, Bucket: str, Key: str, Body: bytes, **kwargs) -> dict:
        self.buckets.setdefault(Bucket, {})[Key] = Body
        return {}

    async def get_object(self, *, Bucket: str, Key: str) -> dict:
        data = self.buckets.get(Bucket, {}).get(Key)
        if data is None:
            raise Exception("NoSuchKey")
        return {"Body": FakeS3Body(data)}

    async def delete_object(self, *, Bucket: str, Key: str) -> dict:
        self.buckets.get(Bucket, {}).pop(Key, None)
        return {}

    async def copy_object(self, *, Bucket: str, Key: str, CopySource: dict, **kwargs) -> dict:
        src_bucket = CopySource["Bucket"]
        src_key = CopySource["Key"]
        data = self.buckets.get(src_bucket, {}).get(src_key)
        if data is None:
            raise Exception("NoSuchKey")
        self.buckets.setdefault(Bucket, {})[Key] = data
        return {}


@pytest.fixture
def fake_client():
    return FakeS3Client()


@pytest.fixture
def storage(fake_client):
    return S3FileStorage(
        s3_client=fake_client,
        dirty_bucket="test-dirty",
        clean_bucket="test-clean",
    )


class TestS3FileStorageStore:
    async def test_store_puts_to_dirty_bucket(self, storage, fake_client):
        key = await storage.store(
            organization_id="org-1",
            source_id="src-1",
            filename="report.pdf",
            content=b"PDF bytes",
        )
        assert key == "org-1/src-1/report.pdf"
        assert fake_client.buckets["test-dirty"]["org-1/src-1/report.pdf"] == b"PDF bytes"

    async def test_store_strips_directory_from_filename(self, storage, fake_client):
        key = await storage.store(
            organization_id="org-1",
            source_id="src-1",
            filename="../../etc/passwd",
            content=b"nope",
        )
        assert key == "org-1/src-1/passwd"

    async def test_store_returns_key_without_bucket(self, storage):
        key = await storage.store(
            organization_id="org-1",
            source_id="src-1",
            filename="data.xlsx",
            content=b"excel",
        )
        assert "test-dirty" not in key


class TestS3FileStorageRetrieve:
    async def test_retrieve_from_clean_bucket(self, storage, fake_client):
        fake_client.buckets["test-clean"] = {"org-1/src-1/file.pdf": b"content"}
        data = await storage.retrieve("org-1/src-1/file.pdf")
        assert data == b"content"

    async def test_retrieve_missing_returns_none(self, storage):
        data = await storage.retrieve("nonexistent/key")
        assert data is None


class TestS3FileStoragePromoteToClean:
    async def test_promote_copies_to_clean_and_deletes_dirty(self, storage, fake_client):
        fake_client.buckets["test-dirty"] = {"org-1/src-1/file.pdf": b"scanned"}
        await storage.promote_to_clean("org-1/src-1/file.pdf")
        assert fake_client.buckets["test-clean"]["org-1/src-1/file.pdf"] == b"scanned"
        assert "org-1/src-1/file.pdf" not in fake_client.buckets.get("test-dirty", {})


class TestS3FileStorageDelete:
    async def test_delete_removes_from_clean_bucket(self, storage, fake_client):
        fake_client.buckets["test-clean"] = {"org-1/src-1/file.pdf": b"content"}
        await storage.delete("org-1/src-1/file.pdf")
        assert "org-1/src-1/file.pdf" not in fake_client.buckets.get("test-clean", {})
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_s3_storage.py -v`
Expected: FAIL — `S3FileStorage` doesn't exist yet.

- [ ] **Step 4: Implement S3FileStorage**

```python
# apps/api/src/prescient/knowledge_mgmt/infrastructure/services/s3_storage.py
"""S3-backed file storage with dirty/clean bucket pattern."""

from __future__ import annotations

import logging
from pathlib import PurePosixPath

logger = logging.getLogger(__name__)


class S3FileStorage:
    """Stores files in S3 with a dirty → scan → clean bucket flow.

    - store(): uploads to the dirty bucket
    - retrieve(): reads from the clean bucket
    - promote_to_clean(): copies dirty → clean, deletes dirty (called after scan passes)
    - delete(): removes from the clean bucket
    """

    def __init__(
        self,
        s3_client,
        dirty_bucket: str,
        clean_bucket: str,
    ) -> None:
        self._s3 = s3_client
        self._dirty_bucket = dirty_bucket
        self._clean_bucket = clean_bucket

    async def store(
        self,
        organization_id: str,
        source_id: str,
        filename: str,
        content: bytes,
    ) -> str:
        """Upload to the dirty bucket. Returns the S3 key (no bucket prefix)."""
        safe_filename = PurePosixPath(filename).name
        key = f"{organization_id}/{source_id}/{safe_filename}"
        await self._s3.put_object(
            Bucket=self._dirty_bucket,
            Key=key,
            Body=content,
        )
        return key

    async def retrieve(self, key: str) -> bytes | None:
        """Retrieve from the clean bucket. Returns None if not found."""
        try:
            resp = await self._s3.get_object(Bucket=self._clean_bucket, Key=key)
            return await resp["Body"].read()
        except Exception:
            return None

    async def promote_to_clean(self, key: str) -> None:
        """Copy from dirty to clean bucket, then delete the dirty copy."""
        await self._s3.copy_object(
            Bucket=self._clean_bucket,
            Key=key,
            CopySource={"Bucket": self._dirty_bucket, "Key": key},
        )
        await self._s3.delete_object(Bucket=self._dirty_bucket, Key=key)

    async def delete(self, key: str) -> None:
        """Delete from the clean bucket."""
        await self._s3.delete_object(Bucket=self._clean_bucket, Key=key)
```

- [ ] **Step 5: Update file_storage.py Protocol**

Replace the contents of `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/file_storage.py`:

```python
"""File storage protocol for the Knowledge Engine.

Concrete implementation: S3FileStorage in s3_storage.py.
"""
from __future__ import annotations

from typing import Protocol


class FileStorage(Protocol):
    async def store(
        self, organization_id: str, source_id: str, filename: str, content: bytes
    ) -> str: ...
    async def retrieve(self, key: str) -> bytes | None: ...
    async def promote_to_clean(self, key: str) -> None: ...
    async def delete(self, key: str) -> None: ...
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_s3_storage.py -v`
Expected: All PASS.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/services/s3_storage.py \
       apps/api/src/prescient/knowledge_mgmt/infrastructure/services/file_storage.py \
       apps/api/src/prescient/config/base.py \
       apps/api/tests/unit/knowledge_mgmt/test_s3_storage.py
git commit -m "feat(ke): add S3FileStorage with dirty/clean bucket pattern"
```

---

### Task 5: Update repositories for new fields

**Files:**
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/source_repository.py`
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/chunk_repository.py`
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/entity_repository.py`

- [ ] **Step 1: Update SourceRepository**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/source_repository.py`:

```python
from __future__ import annotations

import uuid

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.knowledge_mgmt.domain.entities.source import Source, SourceVersion
from prescient.knowledge_mgmt.infrastructure.tables.source import SourceRow, SourceVersionRow


class SourceRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, source: Source) -> Source:
        row = SourceRow(
            id=source.id or str(uuid.uuid4()),
            organization_id=source.organization_id,
            company_id=source.company_id,
            title=source.title,
            source_type=source.source_type.value,
            original_filename=source.original_filename,
            mime_type=source.mime_type,
            storage_key=source.storage_key,
            current_version_id=source.current_version_id,
            ingestion_status=source.ingestion_status.value,
            processing_profile=source.processing_profile,
            department_id=source.department_id,
            visibility=source.visibility.value,
            min_access_role=source.min_access_role,
            trust_tier=source.trust_tier.value,
            created_by=source.created_by,
            updated_by=source.updated_by,
            meta=source.meta,
        )
        self._session.add(row)
        await self._session.flush()
        return self._row_to_entity(row)

    async def get_by_id(self, source_id: str) -> Source | None:
        stmt = select(SourceRow).where(SourceRow.id == source_id)
        result = await self._session.execute(stmt)
        row = result.scalar_one_or_none()
        return self._row_to_entity(row) if row else None

    async def list_by_company(
        self,
        company_id: str,
        *,
        department_id: str | None = None,
        status: str | None = None,
        search: str | None = None,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Source]:
        stmt = (
            select(SourceRow)
            .where(SourceRow.company_id == company_id)
            .order_by(SourceRow.updated_at.desc())
            .limit(limit)
            .offset(offset)
        )
        if department_id:
            stmt = stmt.where(SourceRow.department_id == department_id)
        if status:
            stmt = stmt.where(SourceRow.ingestion_status == status)
        if search:
            safe_search = search.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_")
            stmt = stmt.where(SourceRow.title.ilike(f"%{safe_search}%", escape="\\"))
        result = await self._session.execute(stmt)
        return [self._row_to_entity(row) for row in result.scalars().all()]

    async def update_status(
        self, source_id: str, status: str, meta: dict | None = None
    ) -> None:
        stmt = select(SourceRow).where(SourceRow.id == source_id)
        result = await self._session.execute(stmt)
        row = result.scalar_one_or_none()
        if row:
            row.ingestion_status = status
            if meta:
                row.meta = {**row.meta, **meta}
            await self._session.flush()

    async def create_version(self, version: SourceVersion) -> SourceVersion:
        row = SourceVersionRow(
            id=version.id or str(uuid.uuid4()),
            organization_id=version.organization_id,
            company_id=version.company_id,
            source_id=version.source_id,
            version_number=version.version_number,
            storage_key=version.storage_key,
            visibility=version.visibility.value,
            min_access_role=version.min_access_role,
            department_id=version.department_id,
            trust_tier=version.trust_tier.value,
            change_summary=version.change_summary,
            created_by=version.created_by,
        )
        self._session.add(row)
        await self._session.flush()
        # Update source.current_version_id
        source_stmt = select(SourceRow).where(SourceRow.id == version.source_id)
        source_result = await self._session.execute(source_stmt)
        source_row = source_result.scalar_one_or_none()
        if source_row:
            source_row.current_version_id = row.id
            await self._session.flush()
        return version

    def _row_to_entity(self, row: SourceRow) -> Source:
        return Source(
            id=row.id,
            organization_id=row.organization_id,
            company_id=row.company_id,
            title=row.title,
            source_type=row.source_type,
            original_filename=row.original_filename,
            mime_type=row.mime_type,
            storage_key=row.storage_key,
            current_version_id=row.current_version_id,
            ingestion_status=row.ingestion_status,
            processing_profile=row.processing_profile,
            department_id=row.department_id,
            visibility=row.visibility,
            min_access_role=row.min_access_role,
            trust_tier=row.trust_tier,
            created_by=row.created_by,
            updated_by=row.updated_by,
            meta=row.meta,
            created_at=row.created_at,
            updated_at=row.updated_at,
        )
```

- [ ] **Step 2: Update ChunkRepository**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/chunk_repository.py`:

```python
from __future__ import annotations

from sqlalchemy import delete, select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.knowledge_mgmt.domain.entities.chunk import Chunk
from prescient.knowledge_mgmt.infrastructure.tables.chunk import ChunkRow


class ChunkRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def bulk_create(self, chunks: list[Chunk]) -> None:
        for chunk in chunks:
            row = ChunkRow(
                id=chunk.id,
                organization_id=chunk.organization_id or "",
                company_id=chunk.company_id,
                source_id=chunk.source_id,
                source_version_id=chunk.source_version_id,
                chunk_index=chunk.chunk_index,
                content=chunk.content,
                summary=chunk.summary,
                token_count=chunk.token_count,
                page_number=chunk.page_number,
                section_heading=chunk.section_heading,
                span_start=chunk.span_start,
                span_end=chunk.span_end,
                visibility=chunk.visibility.value if chunk.visibility else "company_wide",
                min_access_role=chunk.min_access_role or "OPERATOR",
                department_id=chunk.department_id,
                trust_tier=chunk.trust_tier.value if chunk.trust_tier else "internal_raw",
                meta=chunk.meta,
            )
            self._session.add(row)
        await self._session.flush()

    async def list_by_source(self, source_id: str) -> list[Chunk]:
        stmt = (
            select(ChunkRow)
            .where(ChunkRow.source_id == source_id)
            .order_by(ChunkRow.chunk_index)
        )
        result = await self._session.execute(stmt)
        return [
            Chunk(
                id=row.id,
                source_id=row.source_id,
                source_version_id=row.source_version_id,
                organization_id=row.organization_id,
                company_id=row.company_id,
                chunk_index=row.chunk_index,
                content=row.content,
                summary=row.summary,
                token_count=row.token_count,
                page_number=row.page_number,
                section_heading=row.section_heading,
                span_start=row.span_start,
                span_end=row.span_end,
                visibility=row.visibility,
                min_access_role=row.min_access_role,
                department_id=row.department_id,
                trust_tier=row.trust_tier,
                meta=row.meta,
                created_at=row.created_at,
            )
            for row in result.scalars().all()
        ]

    async def delete_by_source(self, source_id: str) -> None:
        stmt = delete(ChunkRow).where(ChunkRow.source_id == source_id)
        await self._session.execute(stmt)
```

- [ ] **Step 3: Update EntityRepository — tenant_id → organization_id**

In `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/entity_repository.py`, replace all references to `tenant_id` with `organization_id`:
- `KMEntityRow.tenant_id` → `KMEntityRow.organization_id` in the dedup query
- `entity.tenant_id` → `entity.organization_id` when creating rows
- Add `organization_id` to `EntityMentionRow` creation

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/
git commit -m "feat(ke): update repositories for new KE schema fields"
```

---

### Task 6: Update CreateSource use case

**Files:**
- Modify: `apps/api/src/prescient/knowledge_mgmt/application/use_cases/create_source.py`

- [ ] **Step 1: Update CreateSource**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/application/use_cases/create_source.py`:

```python
"""Use case: create a knowledge-management source (file upload or authored)."""

from __future__ import annotations

import uuid
from dataclasses import dataclass

from prescient.knowledge_mgmt.domain.entities.source import (
    IngestionStatus,
    Source,
    SourceType,
    SourceVersion,
    TrustTier,
    Visibility,
)
from prescient.knowledge_mgmt.infrastructure.repositories.source_repository import (
    SourceRepository,
)
from prescient.knowledge_mgmt.infrastructure.services.file_storage import FileStorage


@dataclass
class CreateSourceRequest:
    organization_id: str
    company_id: str | None
    title: str
    source_type: SourceType
    created_by: str
    visibility: Visibility = Visibility.COMPANY_WIDE
    min_access_role: str = "OPERATOR"
    trust_tier: TrustTier = TrustTier.INTERNAL_RAW
    department_id: str | None = None
    # For file uploads
    file_content: bytes | None = None
    original_filename: str | None = None
    mime_type: str | None = None
    # For authored entries
    authored_content: str | None = None


class CreateSource:
    def __init__(
        self,
        repo: SourceRepository,
        file_storage: FileStorage,
    ) -> None:
        self._repo = repo
        self._file_storage = file_storage

    async def execute(self, req: CreateSourceRequest) -> Source:
        source_id = str(uuid.uuid4())
        version_id = str(uuid.uuid4())
        storage_key: str | None = None
        meta: dict = {}

        if req.source_type == SourceType.UPLOADED_DOCUMENT:
            if req.file_content is None or req.original_filename is None:
                raise ValueError("file_content and original_filename are required for file uploads")
            storage_key = await self._file_storage.store(
                organization_id=req.organization_id,
                source_id=source_id,
                filename=req.original_filename,
                content=req.file_content,
            )
        elif req.source_type == SourceType.AUTHORED:
            if req.authored_content is None:
                raise ValueError("authored_content is required for authored sources")
            meta["content"] = req.authored_content

        source = Source(
            id=source_id,
            organization_id=req.organization_id,
            company_id=req.company_id,
            title=req.title,
            source_type=req.source_type,
            original_filename=req.original_filename,
            mime_type=req.mime_type,
            storage_key=storage_key,
            department_id=req.department_id,
            visibility=req.visibility,
            min_access_role=req.min_access_role,
            trust_tier=req.trust_tier,
            ingestion_status=IngestionStatus.PENDING,
            created_by=req.created_by,
            updated_by=req.created_by,
            meta=meta,
        )
        created = await self._repo.create(source)

        # Create initial version for file uploads
        if storage_key:
            version = SourceVersion(
                id=version_id,
                source_id=source_id,
                organization_id=req.organization_id,
                company_id=req.company_id,
                version_number=1,
                storage_key=storage_key,
                visibility=req.visibility,
                min_access_role=req.min_access_role,
                department_id=req.department_id,
                trust_tier=req.trust_tier,
                created_by=req.created_by,
            )
            await self._repo.create_version(version)

        return created
```

- [ ] **Step 2: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/application/use_cases/create_source.py
git commit -m "feat(ke): update CreateSource for organization_id, S3 storage_key, trust_tier"
```

---

### Task 7: Wire S3 into Celery task and update ingestion pipeline

**Files:**
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingest_celery.py`
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingestion.py`

- [ ] **Step 1: Update Celery task**

Replace the full contents of `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingest_celery.py`:

```python
"""Celery task wrapper for the KM ingestion pipeline."""

from __future__ import annotations

import asyncio
import logging

from celery import shared_task

logger = logging.getLogger(__name__)


@shared_task(  # type: ignore[misc]
    name="prescient.knowledge_mgmt.infrastructure.tasks.ingest_celery.run_ingestion",
    soft_time_limit=300,
    time_limit=360,
    max_retries=2,
    default_retry_delay=30,
)
def run_ingestion(source_id: str, organization_id: str) -> dict:
    """Run the KM ingestion pipeline as a Celery task."""
    from prescient.config import get_settings
    from prescient.db import SessionLocal
    from prescient.knowledge_mgmt.infrastructure.repositories.chunk_repository import (
        ChunkRepository,
    )
    from prescient.knowledge_mgmt.infrastructure.repositories.entity_repository import (
        EntityRepository,
    )
    from prescient.knowledge_mgmt.infrastructure.repositories.source_repository import (
        SourceRepository,
    )
    from prescient.knowledge_mgmt.infrastructure.services.chunker import SemanticChunker
    from prescient.knowledge_mgmt.infrastructure.services.embedding import get_embed_fn
    from prescient.knowledge_mgmt.infrastructure.services.enrichment import EnrichmentService
    from prescient.knowledge_mgmt.infrastructure.services.s3_storage import S3FileStorage
    from prescient.knowledge_mgmt.infrastructure.services.search import KMSearchService
    from prescient.knowledge_mgmt.infrastructure.services.text_extractor import TextExtractor
    from prescient.knowledge_mgmt.infrastructure.tasks.ingestion import run_ingestion_pipeline

    async def _run() -> None:
        settings = get_settings()

        import aioboto3

        s3_session = aioboto3.Session()
        async with s3_session.client(
            "s3",
            endpoint_url=settings.s3_endpoint_url,
            region_name=settings.aws_region,
        ) as s3_client:
            async with SessionLocal() as session:
                from sqlalchemy import func as sa_func
                from sqlalchemy import select as sa_select

                # Set org-level RLS GUC for KM tables
                await session.execute(
                    sa_select(
                        sa_func.set_config("app.current_org_id", organization_id, True)
                    )
                )

                source_repo = SourceRepository(session)
                chunk_repo = ChunkRepository(session)
                entity_repo = EntityRepository(session)
                file_storage = S3FileStorage(
                    s3_client=s3_client,
                    dirty_bucket=settings.s3_dirty_bucket,
                    clean_bucket=settings.s3_clean_bucket,
                )

                if not settings.anthropic_api_key:
                    logger.warning(
                        "No Anthropic API key — skipping enrichment for %s",
                        source_id,
                    )
                    await source_repo.update_status(
                        source_id, "ready", meta={"skipped_enrichment": True}
                    )
                    await session.commit()
                    return

                import anthropic

                anthropic_client = anthropic.AsyncAnthropic(
                    api_key=settings.anthropic_api_key
                )
                enrichment = EnrichmentService(
                    anthropic_client=anthropic_client, model=settings.anthropic_model
                )

                from opensearchpy import AsyncOpenSearch

                os_client = AsyncOpenSearch(
                    hosts=[settings.opensearch_url],
                    use_ssl=False,
                    verify_certs=False,
                )
                search_service = KMSearchService(
                    opensearch=os_client, embed_fn=get_embed_fn()
                )

                await run_ingestion_pipeline(
                    source_id=source_id,
                    organization_id=organization_id,
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

    asyncio.run(_run())
    return {"source_id": source_id, "status": "completed"}
```

- [ ] **Step 2: Update ingestion.py — tenant_id → organization_id**

In `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingestion.py`:
- Rename the `tenant_id` parameter to `organization_id` in `run_ingestion_pipeline`
- Update all internal references from `tenant_id` to `organization_id`
- Update the `search_service.index_chunks()` call: change `tenant_slug=tenant_id` to `organization_id=organization_id`

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/
git commit -m "feat(ke): wire S3, organization_id, app.current_org_id into ingestion tasks"
```

---

### Task 8: Update search service — UUID-based index naming

**Files:**
- Modify: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py`
- Modify: `apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py`

- [ ] **Step 1: Update KMSearchService index template and parameter names**

In `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py`:

Change the index template from slug-based to UUID-based:
```python
# Before:
INDEX_TEMPLATE = "km_chunks__{slug}"

# After:
INDEX_TEMPLATE = "km_chunks__{organization_id}"
```

Rename all `tenant_slug` parameters to `organization_id` in:
- `search()` method signature
- `index_chunks()` method signature
- `delete_by_source()` method signature
- All internal calls to `INDEX_TEMPLATE.format()`

Example:
```python
# Before:
def search(self, query, user, tenant_slug, ...):
    index = INDEX_TEMPLATE.format(slug=tenant_slug)

# After:
def search(self, query, user, organization_id, ...):
    index = INDEX_TEMPLATE.format(organization_id=organization_id)
```

- [ ] **Step 2: Update search_knowledge_base tool**

In `apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py`:

Replace the slug-based index lookup with UUID-based:
```python
# Before:
tenant_slug=context.primary_company_slug

# After — use organization_id from AccessContext or derive from company:
organization_id=str(context.tenant_id)  # For now; will be replaced with org_id from AccessContext in Plan 3
```

Note: This is a temporary bridge. Plan 3 (Unified Retrieval) will refactor ToolContext to carry `organization_id` natively. For now, use the tenant_id (fund_id) as a stand-in since the KM migration truncated all data anyway — new data will be indexed with the correct organization_id.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py \
       apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py
git commit -m "feat(ke): switch OpenSearch index naming from slug to UUID"
```

---

### Task 9: Update API routes and set app.current_org_id GUC

**Files:**
- Modify: `apps/api/src/prescient/knowledge_mgmt/api/routes.py`
- Modify: `apps/api/src/prescient/db.py`

- [ ] **Step 1: Update db.py to also set app.current_org_id**

In `apps/api/src/prescient/db.py`, after the existing `app.current_fund_id` GUC, add the org_id GUC. Since `AccessContext` is a separate dependency and not always available in `get_session`, we'll set the org GUC from the KM routes directly. No change to `db.py` for now — each KM route will set it explicitly via session.

- [ ] **Step 2: Update KM API routes**

In `apps/api/src/prescient/knowledge_mgmt/api/routes.py`:
- Import `AccessContext` and `get_access_context` from `prescient.auth.access_context`
- Add `AccessContext` as a dependency to KM routes
- Use `access_ctx.org_id` as `organization_id` instead of `str(ctx.user.fund_id)`
- Set `app.current_org_id` GUC at the start of each route:
  ```python
  await ctx.session.execute(
      sa_select(func.set_config("app.current_org_id", access_ctx.org_id, True))
  )
  ```
- Update `_trigger_ingestion_celery(source_id, organization_id)` to pass org_id
- Update response model: rename `status` → `ingestion_status`, add `trust_tier`, `processing_profile`, `organization_id`
- Replace `SourceType.FILE_UPLOAD` → `SourceType.UPLOADED_DOCUMENT`
- Replace `SourceStatus` → `IngestionStatus` in imports

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/api/routes.py \
       apps/api/src/prescient/db.py
git commit -m "feat(ke): wire AccessContext.org_id into KM routes, set app.current_org_id GUC"
```

---

### Task 10: Add aioboto3 dependency and LocalStack S3 bucket init

**Files:**
- Modify: `apps/api/pyproject.toml`
- Create: `infrastructure/localstack/init-s3.sh`
- Modify: `infrastructure/docker-compose.yml`

- [ ] **Step 1: Add aioboto3 to dependencies**

In `apps/api/pyproject.toml`, add to the `[project.dependencies]` list:
```
"aioboto3>=13.0.0",
```

- [ ] **Step 2: Install**

Run: `cd apps/api && uv sync`

- [ ] **Step 3: Create S3 bucket init script**

Create `infrastructure/localstack/init-s3.sh`:

```bash
#!/bin/bash
echo "Creating S3 buckets..."
awslocal s3 mb s3://prescient-dirty
awslocal s3 mb s3://prescient-clean
echo "S3 buckets created."
```

Make it executable: `chmod +x infrastructure/localstack/init-s3.sh`

- [ ] **Step 4: Mount init script in docker-compose**

In `infrastructure/docker-compose.yml`, add to the localstack service volumes:

```yaml
      - ./localstack/init-s3.sh:/etc/localstack/init/ready.d/init-s3.sh
```

- [ ] **Step 5: Test bucket creation**

Run: `docker compose -f infrastructure/docker-compose.yml restart localstack`
Then: `docker exec prescient-localstack awslocal s3 ls`
Expected: Both `prescient-dirty` and `prescient-clean` buckets listed.

- [ ] **Step 6: Commit**

```bash
git add apps/api/pyproject.toml apps/api/uv.lock \
       infrastructure/localstack/ infrastructure/docker-compose.yml
git commit -m "feat(ke): add aioboto3 dep and LocalStack S3 bucket init"
```

---

### Task 11: Grep and fix all remaining legacy references

**Files:**
- Various files across `knowledge_mgmt` and `intelligence` contexts

- [ ] **Step 1: Find all remaining references**

```bash
cd apps/api
grep -rn "tenant_id" src/prescient/knowledge_mgmt/ --include="*.py" | grep -v __pycache__
grep -rn "storage_path" src/prescient/knowledge_mgmt/ --include="*.py" | grep -v __pycache__
grep -rn "SourceStatus" src/prescient/knowledge_mgmt/ --include="*.py" | grep -v __pycache__
grep -rn "FILE_UPLOAD" src/prescient/knowledge_mgmt/ --include="*.py" | grep -v __pycache__
grep -rn "\.status" src/prescient/knowledge_mgmt/ --include="*.py" | grep -v __pycache__ | grep -v ingestion_status
grep -rn "tenant_slug" src/prescient/knowledge_mgmt/ --include="*.py" | grep -v __pycache__
```

- [ ] **Step 2: Update each file**

For each hit:
- `tenant_id` → `organization_id` (in function params, variable names, SQL GUC settings)
- `storage_path` → `storage_key` (in domain entities, ORM rows, API responses)
- `SourceStatus` → `IngestionStatus` (in imports and usage)
- `SourceType.FILE_UPLOAD` → `SourceType.UPLOADED_DOCUMENT`
- `source.status` → `source.ingestion_status`
- `tenant_slug` → `organization_id`

- [ ] **Step 3: Check intelligence tools**

```bash
grep -rn "tenant_id\|storage_path\|SourceStatus\|FILE_UPLOAD\|tenant_slug" \
  src/prescient/intelligence/ --include="*.py" | grep -v __pycache__ | grep -i "knowledge\|km\|search_knowledge"
```

Update any references.

- [ ] **Step 4: Check KM enrichment service for tenant_id**

```bash
grep -rn "tenant_id" src/prescient/knowledge_mgmt/infrastructure/services/ --include="*.py" | grep -v __pycache__
```

The enrichment service's `enrich_chunks` takes `tenant_id` — rename to `organization_id`.

- [ ] **Step 5: Run all KM tests**

Run: `cd apps/api && python -m pytest tests/unit/knowledge_mgmt/ -v`

- [ ] **Step 6: Run full test suite**

Run: `cd apps/api && python -m pytest tests/unit/ -v --timeout=60`

Fix any breakage from the renames.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "refactor(ke): complete rename tenant_id→organization_id, storage_path→storage_key, SourceStatus→IngestionStatus"
```

---

## Verification

After all tasks are complete:

1. **Migration applies cleanly:**
   ```bash
   cd apps/api && uv run alembic upgrade head
   ```

2. **RLS is active on all KM tables:**
   ```bash
   docker exec -i prescient-postgres psql -U prescient -d prescient -c "
   SELECT schemaname, tablename, rowsecurity FROM pg_tables WHERE schemaname = 'km';
   "
   ```

3. **S3 buckets exist in LocalStack:**
   ```bash
   docker exec prescient-localstack awslocal s3 ls
   ```

4. **All unit tests pass:**
   ```bash
   cd apps/api && python -m pytest tests/unit/ -v
   ```

5. **Domain entities use new enums consistently:**
   ```bash
   cd apps/api && python -c "
   from prescient.knowledge_mgmt.domain.entities.source import SourceType, IngestionStatus, TrustTier, Visibility
   print('SourceType:', [e.value for e in SourceType])
   print('IngestionStatus:', [e.value for e in IngestionStatus])
   print('TrustTier:', [e.value for e in TrustTier])
   print('Visibility:', [e.value for e in Visibility])
   "
   ```

6. **OpenSearch index template uses UUID (no slugs):**
   ```bash
   cd apps/api && python -c "
   from prescient.knowledge_mgmt.infrastructure.services.search import INDEX_TEMPLATE
   print('Index template:', INDEX_TEMPLATE)
   assert 'slug' not in INDEX_TEMPLATE
   assert 'organization_id' in INDEX_TEMPLATE
   print('OK — UUID-based index naming')
   "
   ```

---

## Notes for subsequent plans

- **Plan 2 (Staged Pipeline):** Will split `run_ingestion_pipeline` into chained Celery tasks. Builds on the S3 storage and organization_id wiring from this plan.
- **Plan 3 (Unified Retrieval):** Will refactor `ToolContext` to carry `organization_id` natively (replacing the temporary bridge in Task 8 Step 2). Will also remove `primary_company_slug` from `ToolContext` once all indices use UUID naming.
- **Plan 6 (Sponsor Projection):** Will use `AccessContext.org_id` and `AccessContext.company_org_ids` to build the sponsor-share projection. The `app.current_org_id` GUC from this plan is the foundation for that isolation.
