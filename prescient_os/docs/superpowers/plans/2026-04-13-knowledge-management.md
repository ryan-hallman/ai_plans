# Knowledge Management System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a knowledge management system that lets departments upload/author institutional knowledge and query it through the existing copilot chat with grounded, cited answers.

**Architecture:** New `knowledge_mgmt` bounded context following existing DDD patterns (domain → application → infrastructure → api). Celery pipeline for async ingestion. Hybrid BM25 + k-NN search via OpenSearch. Integrated into the copilot via a new tool in the existing registry.

**Tech Stack:** FastAPI, SQLAlchemy 2.0 (async), Alembic, Celery + Redis, OpenSearch (BM25 + k-NN), pgvector, pdfplumber, python-docx, Next.js 15 / React 19

**Spec:** `docs/superpowers/specs/2026-04-13-knowledge-management-design.md`

---

## File Structure

### API (Python — `apps/api/src/prescient/knowledge_mgmt/`)

```
knowledge_mgmt/
├── __init__.py
├── domain/
│   ├── __init__.py
│   ├── entities/
│   │   ├── __init__.py
│   │   ├── source.py              # Source, SourceVersion domain entities
│   │   ├── chunk.py               # Chunk domain entity
│   │   ├── entity.py              # KMEntity, EntityMention domain entities
│   │   └── department.py          # Department, DepartmentMembership domain entities
│   └── errors.py                  # Domain-specific exceptions
├── application/
│   ├── __init__.py
│   └── use_cases/
│       ├── __init__.py
│       ├── create_source.py       # Upload file or create authored entry
│       ├── get_source.py          # Fetch source with access check
│       ├── list_sources.py        # List/filter sources for a company
│       ├── update_source.py       # Edit metadata, visibility, access
│       ├── create_version.py      # Upload new version of existing source
│       ├── create_department.py   # Create department
│       ├── list_departments.py    # List departments for company
│       └── manage_membership.py   # Add/remove department members
├── infrastructure/
│   ├── __init__.py
│   ├── tables/
│   │   ├── __init__.py
│   │   ├── source.py              # SourceRow, SourceVersionRow
│   │   ├── chunk.py               # ChunkRow
│   │   ├── entity.py              # KMEntityRow, EntityMentionRow
│   │   └── department.py          # DepartmentRow, DepartmentMembershipRow
│   ├── repositories/
│   │   ├── __init__.py
│   │   ├── source_repository.py   # CRUD for sources + versions
│   │   ├── chunk_repository.py    # CRUD for chunks
│   │   ├── entity_repository.py   # CRUD for entities + mentions
│   │   └── department_repository.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── file_storage.py        # Store/retrieve files (local → S3-ready)
│   │   ├── text_extractor.py      # PDF, Word, Markdown → text
│   │   ├── chunker.py             # Semantic chunking
│   │   ├── enrichment.py          # Embeddings, summaries, entity extraction
│   │   ├── search.py              # Hybrid BM25 + k-NN + RRF
│   │   └── access_policy.py       # KMAccessPolicy — permission checks
│   └── tasks/
│       ├── __init__.py
│       └── ingestion.py           # Celery task chain
└── api/
    ├── __init__.py
    └── routes.py                  # FastAPI router — sources, departments, upload
```

### Copilot tool (added to existing intelligence context)

```
apps/api/src/prescient/intelligence/infrastructure/tools/
└── search_knowledge_base.py       # New tool for copilot
```

### Celery worker

```
apps/api/src/prescient/
├── celery_app.py                  # Celery app factory
```

### Migrations

```
apps/api/alembic/versions/
└── 20260413_XXXX_km_schema.py     # km schema + all tables
```

### Web (Next.js — `apps/web/src/`)

```
app/(main)/knowledge/
├── page.tsx                       # Source list landing page
├── upload/page.tsx                # Upload flow
├── [sourceId]/page.tsx            # Source detail view
└── departments/page.tsx           # Department management (admin)

components/
├── omni-search.tsx                # Cmd+K search bar
├── km-source-list.tsx             # Source list table with filters
├── km-upload-form.tsx             # Upload + metadata confirmation
├── km-source-detail.tsx           # Source detail with versions
└── km-department-manager.tsx      # Department CRUD

lib/
└── api.ts                         # Add KM API client methods (extend existing)
```

---

## Task 1: Database Schema and Migration

**Files:**
- Create: `apps/api/alembic/versions/20260413_km_schema.py`

- [ ] **Step 1: Read existing migration files for pattern reference**

Read the latest migration file to confirm revision chain:
```bash
ls -la apps/api/alembic/versions/ | tail -5
```
Note the latest `revision` value — this becomes `down_revision` for the new migration.

- [ ] **Step 2: Write the migration**

Create `apps/api/alembic/versions/20260413_km_schema.py`:

```python
"""Knowledge Management schema and tables."""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = "km_001_schema"
down_revision = "<LATEST_REVISION>"  # Fill from Step 1
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("CREATE SCHEMA IF NOT EXISTS km")

    # Departments
    op.create_table(
        "departments",
        sa.Column("id", sa.String(36), nullable=False),
        sa.Column("tenant_id", sa.String(36), nullable=False),
        sa.Column("company_id", sa.String(36), nullable=False),
        sa.Column("name", sa.String(128), nullable=False),
        sa.Column("slug", sa.String(128), nullable=False),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("company_id", "slug", name="uq_km_dept_company_slug"),
        schema="km",
    )
    op.create_index(
        "ix_km_departments_company_id",
        "departments",
        ["company_id"],
        schema="km",
    )

    # Department memberships
    op.create_table(
        "department_memberships",
        sa.Column("id", sa.String(36), nullable=False),
        sa.Column("user_id", sa.String(36), nullable=False),
        sa.Column(
            "department_id",
            sa.String(36),
            sa.ForeignKey("km.departments.id", ondelete="CASCADE"),
            nullable=False,
        ),
        sa.Column("is_lead", sa.Boolean(), nullable=False, server_default=sa.false()),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("user_id", "department_id", name="uq_km_membership"),
        schema="km",
    )

    # Sources
    op.create_table(
        "sources",
        sa.Column("id", sa.String(36), nullable=False),
        sa.Column("tenant_id", sa.String(36), nullable=False),
        sa.Column("company_id", sa.String(36), nullable=False),
        sa.Column("title", sa.Text(), nullable=False),
        sa.Column(
            "source_type",
            sa.String(32),
            nullable=False,
        ),
        sa.Column("original_filename", sa.Text(), nullable=True),
        sa.Column("mime_type", sa.String(128), nullable=True),
        sa.Column("storage_path", sa.Text(), nullable=True),
        sa.Column(
            "department_id",
            sa.String(36),
            sa.ForeignKey("km.departments.id", ondelete="SET NULL"),
            nullable=True,
        ),
        sa.Column("visibility", sa.String(32), nullable=False, server_default="company_wide"),
        sa.Column("min_access_role", sa.String(32), nullable=False, server_default="OPERATOR"),
        sa.Column("status", sa.String(32), nullable=False, server_default="pending"),
        sa.Column("created_by", sa.String(36), nullable=False),
        sa.Column("updated_by", sa.String(36), nullable=False),
        sa.Column("meta", postgresql.JSONB(astext_type=sa.Text()), nullable=False, server_default="{}"),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.PrimaryKeyConstraint("id"),
        schema="km",
    )
    op.create_index("ix_km_sources_company_id", "sources", ["company_id"], schema="km")
    op.create_index("ix_km_sources_department_id", "sources", ["department_id"], schema="km")
    op.create_index("ix_km_sources_status", "sources", ["status"], schema="km")

    # Source versions
    op.create_table(
        "source_versions",
        sa.Column("id", sa.String(36), nullable=False),
        sa.Column(
            "source_id",
            sa.String(36),
            sa.ForeignKey("km.sources.id", ondelete="CASCADE"),
            nullable=False,
        ),
        sa.Column("version_number", sa.Integer(), nullable=False),
        sa.Column("storage_path", sa.Text(), nullable=False),
        sa.Column("change_summary", sa.Text(), nullable=True),
        sa.Column("created_by", sa.String(36), nullable=False),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("source_id", "version_number", name="uq_km_version_number"),
        schema="km",
    )

    # Chunks
    op.create_table(
        "chunks",
        sa.Column("id", sa.String(36), nullable=False),
        sa.Column(
            "source_id",
            sa.String(36),
            sa.ForeignKey("km.sources.id", ondelete="CASCADE"),
            nullable=False,
        ),
        sa.Column(
            "source_version_id",
            sa.String(36),
            sa.ForeignKey("km.source_versions.id", ondelete="CASCADE"),
            nullable=True,
        ),
        sa.Column("chunk_index", sa.Integer(), nullable=False),
        sa.Column("content", sa.Text(), nullable=False),
        sa.Column("summary", sa.Text(), nullable=True),
        sa.Column("token_count", sa.Integer(), nullable=True),
        sa.Column("meta", postgresql.JSONB(astext_type=sa.Text()), nullable=False, server_default="{}"),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.PrimaryKeyConstraint("id"),
        schema="km",
    )
    op.create_index("ix_km_chunks_source_id", "chunks", ["source_id"], schema="km")

    # Entities
    op.create_table(
        "entities",
        sa.Column("id", sa.String(36), nullable=False),
        sa.Column("tenant_id", sa.String(36), nullable=False),
        sa.Column("name", sa.Text(), nullable=False),
        sa.Column("entity_type", sa.String(64), nullable=False),
        sa.Column("meta", postgresql.JSONB(astext_type=sa.Text()), nullable=False, server_default="{}"),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.PrimaryKeyConstraint("id"),
        schema="km",
    )
    op.create_index("ix_km_entities_tenant_id", "entities", ["tenant_id"], schema="km")

    # Entity mentions
    op.create_table(
        "entity_mentions",
        sa.Column("id", sa.String(36), nullable=False),
        sa.Column(
            "entity_id",
            sa.String(36),
            sa.ForeignKey("km.entities.id", ondelete="CASCADE"),
            nullable=False,
        ),
        sa.Column(
            "chunk_id",
            sa.String(36),
            sa.ForeignKey("km.chunks.id", ondelete="CASCADE"),
            nullable=False,
        ),
        sa.Column("context_snippet", sa.Text(), nullable=True),
        sa.PrimaryKeyConstraint("id"),
        schema="km",
    )


def downgrade() -> None:
    op.drop_table("entity_mentions", schema="km")
    op.drop_table("entities", schema="km")
    op.drop_table("chunks", schema="km")
    op.drop_table("source_versions", schema="km")
    op.drop_table("sources", schema="km")
    op.drop_table("department_memberships", schema="km")
    op.drop_table("departments", schema="km")
    op.execute("DROP SCHEMA IF EXISTS km")
```

- [ ] **Step 3: Run migration against dev database**

```bash
cd apps/api && docker compose exec api alembic upgrade head
```
Expected: Migration applies cleanly, `km` schema created with all 7 tables.

- [ ] **Step 4: Verify tables exist**

```bash
docker compose exec db psql -U prescient -d prescient -c "\dt km.*"
```
Expected: Lists departments, department_memberships, sources, source_versions, chunks, entities, entity_mentions.

- [ ] **Step 5: Commit**

```bash
git add apps/api/alembic/versions/20260413_km_schema.py
git commit -m "feat(km): add knowledge management database schema and migration"
```

---

## Task 2: Domain Entities

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/domain/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/domain/entities/source.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/domain/entities/chunk.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/domain/entities/entity.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/domain/entities/department.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/domain/errors.py`

- [ ] **Step 1: Create package structure**

Create empty `__init__.py` files for:
- `apps/api/src/prescient/knowledge_mgmt/__init__.py`
- `apps/api/src/prescient/knowledge_mgmt/domain/__init__.py`
- `apps/api/src/prescient/knowledge_mgmt/domain/entities/__init__.py`

- [ ] **Step 2: Write domain entities — source.py**

Create `apps/api/src/prescient/knowledge_mgmt/domain/entities/source.py`:

```python
from __future__ import annotations

import enum
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class SourceType(str, enum.Enum):
    FILE_UPLOAD = "file_upload"
    AUTHORED = "authored"
    INTEGRATION = "integration"


class Visibility(str, enum.Enum):
    COMPANY_WIDE = "company_wide"
    DEPARTMENT_ONLY = "department_only"


class SourceStatus(str, enum.Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    READY = "ready"
    FAILED = "failed"


class Source(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    tenant_id: str
    company_id: str
    title: str
    source_type: SourceType
    original_filename: str | None = None
    mime_type: str | None = None
    storage_path: str | None = None
    department_id: str | None = None
    visibility: Visibility = Visibility.COMPANY_WIDE
    min_access_role: str = "OPERATOR"
    status: SourceStatus = SourceStatus.PENDING
    created_by: str
    updated_by: str
    meta: dict = Field(default_factory=dict)
    created_at: datetime | None = None
    updated_at: datetime | None = None


class SourceVersion(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    source_id: str
    version_number: int
    storage_path: str
    change_summary: str | None = None
    created_by: str
    created_at: datetime | None = None
```

- [ ] **Step 3: Write domain entities — chunk.py**

Create `apps/api/src/prescient/knowledge_mgmt/domain/entities/chunk.py`:

```python
from __future__ import annotations

from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class Chunk(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    source_id: str
    source_version_id: str | None = None
    chunk_index: int
    content: str
    summary: str | None = None
    token_count: int | None = None
    meta: dict = Field(default_factory=dict)
    created_at: datetime | None = None
```

- [ ] **Step 4: Write domain entities — entity.py**

Create `apps/api/src/prescient/knowledge_mgmt/domain/entities/entity.py`:

```python
from __future__ import annotations

from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class KMEntity(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    tenant_id: str
    name: str
    entity_type: str
    meta: dict = Field(default_factory=dict)
    created_at: datetime | None = None


class EntityMention(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    entity_id: str
    chunk_id: str
    context_snippet: str | None = None
```

- [ ] **Step 5: Write domain entities — department.py**

Create `apps/api/src/prescient/knowledge_mgmt/domain/entities/department.py`:

```python
from __future__ import annotations

from datetime import datetime

from pydantic import BaseModel, ConfigDict


class Department(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    tenant_id: str
    company_id: str
    name: str
    slug: str
    created_at: datetime | None = None


class DepartmentMembership(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    user_id: str
    department_id: str
    is_lead: bool = False
    created_at: datetime | None = None
```

- [ ] **Step 6: Write domain errors**

Create `apps/api/src/prescient/knowledge_mgmt/domain/errors.py`:

```python
class KMError(Exception):
    pass


class SourceNotFoundError(KMError):
    def __init__(self, source_id: str) -> None:
        self.source_id = source_id
        super().__init__(f"Source not found: {source_id}")


class AccessDeniedError(KMError):
    def __init__(self, source_id: str, user_id: str) -> None:
        self.source_id = source_id
        self.user_id = user_id
        super().__init__(f"Access denied to source {source_id} for user {user_id}")


class DepartmentNotFoundError(KMError):
    def __init__(self, department_id: str) -> None:
        self.department_id = department_id
        super().__init__(f"Department not found: {department_id}")


class UnsupportedFileTypeError(KMError):
    def __init__(self, mime_type: str) -> None:
        self.mime_type = mime_type
        super().__init__(f"Unsupported file type: {mime_type}")
```

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/
git commit -m "feat(km): add domain entities and error types"
```

---

## Task 3: SQLAlchemy Table Models

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/source.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/chunk.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/entity.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/department.py`
- Modify: `apps/api/src/prescient/shared/metadata.py` — register KM tables so Alembic sees them

- [ ] **Step 1: Create package structure**

Create empty `__init__.py` files for:
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/__init__.py`
- `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/__init__.py`

- [ ] **Step 2: Write table model — source.py**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/source.py`:

```python
from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Integer, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class SourceRow(Base):
    __tablename__ = "sources"
    __table_args__ = {"schema": "km"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    tenant_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    company_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    title: Mapped[str] = mapped_column(Text(), nullable=False)
    source_type: Mapped[str] = mapped_column(String(32), nullable=False)
    original_filename: Mapped[str | None] = mapped_column(Text(), nullable=True)
    mime_type: Mapped[str | None] = mapped_column(String(128), nullable=True)
    storage_path: Mapped[str | None] = mapped_column(Text(), nullable=True)
    department_id: Mapped[str | None] = mapped_column(
        String(36),
        ForeignKey("km.departments.id", ondelete="SET NULL"),
        nullable=True,
        index=True,
    )
    visibility: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="company_wide"
    )
    min_access_role: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="OPERATOR"
    )
    status: Mapped[str] = mapped_column(
        String(32), nullable=False, server_default="pending", index=True
    )
    created_by: Mapped[str] = mapped_column(String(36), nullable=False)
    updated_by: Mapped[str] = mapped_column(String(36), nullable=False)
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
    __table_args__ = {"schema": "km"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    source_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("km.sources.id", ondelete="CASCADE"),
        nullable=False,
    )
    version_number: Mapped[int] = mapped_column(Integer(), nullable=False)
    storage_path: Mapped[str] = mapped_column(Text(), nullable=False)
    change_summary: Mapped[str | None] = mapped_column(Text(), nullable=True)
    created_by: Mapped[str] = mapped_column(String(36), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
```

- [ ] **Step 3: Write table model — chunk.py**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/chunk.py`:

```python
from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Integer, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class ChunkRow(Base):
    __tablename__ = "chunks"
    __table_args__ = {"schema": "km"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
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
    meta: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="{}")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
```

- [ ] **Step 4: Write table model — entity.py**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/entity.py`:

```python
from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, Text, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class KMEntityRow(Base):
    __tablename__ = "entities"
    __table_args__ = {"schema": "km"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    tenant_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    name: Mapped[str] = mapped_column(Text(), nullable=False)
    entity_type: Mapped[str] = mapped_column(String(64), nullable=False)
    meta: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default="{}")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )


class EntityMentionRow(Base):
    __tablename__ = "entity_mentions"
    __table_args__ = {"schema": "km"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    entity_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("km.entities.id", ondelete="CASCADE"),
        nullable=False,
    )
    chunk_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("km.chunks.id", ondelete="CASCADE"),
        nullable=False,
    )
    context_snippet: Mapped[str | None] = mapped_column(Text(), nullable=True)
```

- [ ] **Step 5: Write table model — department.py**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/department.py`:

```python
from __future__ import annotations

from datetime import datetime

from sqlalchemy import Boolean, DateTime, ForeignKey, String, func
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class DepartmentRow(Base):
    __tablename__ = "departments"
    __table_args__ = {"schema": "km"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    tenant_id: Mapped[str] = mapped_column(String(36), nullable=False)
    company_id: Mapped[str] = mapped_column(String(36), nullable=False, index=True)
    name: Mapped[str] = mapped_column(String(128), nullable=False)
    slug: Mapped[str] = mapped_column(String(128), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )


class DepartmentMembershipRow(Base):
    __tablename__ = "department_memberships"
    __table_args__ = {"schema": "km"}

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    user_id: Mapped[str] = mapped_column(String(36), nullable=False)
    department_id: Mapped[str] = mapped_column(
        String(36),
        ForeignKey("km.departments.id", ondelete="CASCADE"),
        nullable=False,
    )
    is_lead: Mapped[bool] = mapped_column(
        Boolean(), nullable=False, server_default="false"
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
```

- [ ] **Step 6: Register KM tables in shared metadata**

Read `apps/api/src/prescient/shared/metadata.py` to find existing imports, then add:

```python
import prescient.knowledge_mgmt.infrastructure.tables.source  # noqa: F401
import prescient.knowledge_mgmt.infrastructure.tables.chunk  # noqa: F401
import prescient.knowledge_mgmt.infrastructure.tables.entity  # noqa: F401
import prescient.knowledge_mgmt.infrastructure.tables.department  # noqa: F401
```

This ensures Alembic's `target_metadata` picks up the new tables.

- [ ] **Step 7: Verify models compile**

```bash
cd apps/api && python -c "from prescient.knowledge_mgmt.infrastructure.tables.source import SourceRow, SourceVersionRow; print('OK')"
```

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/ apps/api/src/prescient/shared/metadata.py
git commit -m "feat(km): add SQLAlchemy table models for knowledge management"
```

---

## Task 4: Repositories

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/source_repository.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/chunk_repository.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/department_repository.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/entity_repository.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/__init__.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_source_repository.py`

- [ ] **Step 1: Write failing test for source repository**

Create `apps/api/tests/unit/knowledge_mgmt/__init__.py` (empty).

Create `apps/api/tests/unit/knowledge_mgmt/test_source_repository.py`:

```python
import pytest
from unittest.mock import AsyncMock, MagicMock

from prescient.knowledge_mgmt.infrastructure.repositories.source_repository import (
    SourceRepository,
)
from prescient.knowledge_mgmt.domain.entities.source import (
    Source,
    SourceType,
    Visibility,
    SourceStatus,
)


def _make_source(**overrides: object) -> Source:
    defaults = {
        "id": "src-001",
        "tenant_id": "t-1",
        "company_id": "c-1",
        "title": "Test SOP",
        "source_type": SourceType.FILE_UPLOAD,
        "visibility": Visibility.COMPANY_WIDE,
        "min_access_role": "OPERATOR",
        "status": SourceStatus.PENDING,
        "created_by": "u-1",
        "updated_by": "u-1",
    }
    defaults.update(overrides)
    return Source(**defaults)


@pytest.mark.asyncio
async def test_create_source_inserts_row() -> None:
    session = AsyncMock()
    repo = SourceRepository(session)
    source = _make_source()
    await repo.create(source)
    session.add.assert_called_once()


@pytest.mark.asyncio
async def test_get_by_id_returns_none_when_missing() -> None:
    session = AsyncMock()
    result_mock = MagicMock()
    result_mock.scalar_one_or_none.return_value = None
    session.execute.return_value = result_mock
    repo = SourceRepository(session)
    result = await repo.get_by_id("nonexistent")
    assert result is None
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_source_repository.py -v
```
Expected: FAIL — `ModuleNotFoundError: No module named 'prescient.knowledge_mgmt.infrastructure.repositories'`

- [ ] **Step 3: Write source repository**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/__init__.py` (empty).

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/source_repository.py`:

```python
from __future__ import annotations

import uuid
from datetime import datetime, timezone

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.knowledge_mgmt.domain.entities.source import Source, SourceVersion
from prescient.knowledge_mgmt.infrastructure.tables.source import (
    SourceRow,
    SourceVersionRow,
)


class SourceRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, source: Source) -> Source:
        row = SourceRow(
            id=source.id or str(uuid.uuid4()),
            tenant_id=source.tenant_id,
            company_id=source.company_id,
            title=source.title,
            source_type=source.source_type.value,
            original_filename=source.original_filename,
            mime_type=source.mime_type,
            storage_path=source.storage_path,
            department_id=source.department_id,
            visibility=source.visibility.value,
            min_access_role=source.min_access_role,
            status=source.status.value,
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
            stmt = stmt.where(SourceRow.status == status)
        result = await self._session.execute(stmt)
        return [self._row_to_entity(row) for row in result.scalars().all()]

    async def update_status(self, source_id: str, status: str, meta: dict | None = None) -> None:
        stmt = select(SourceRow).where(SourceRow.id == source_id)
        result = await self._session.execute(stmt)
        row = result.scalar_one_or_none()
        if row:
            row.status = status
            if meta:
                row.meta = {**row.meta, **meta}
            await self._session.flush()

    def _row_to_entity(self, row: SourceRow) -> Source:
        return Source(
            id=row.id,
            tenant_id=row.tenant_id,
            company_id=row.company_id,
            title=row.title,
            source_type=row.source_type,
            original_filename=row.original_filename,
            mime_type=row.mime_type,
            storage_path=row.storage_path,
            department_id=row.department_id,
            visibility=row.visibility,
            min_access_role=row.min_access_role,
            status=row.status,
            created_by=row.created_by,
            updated_by=row.updated_by,
            meta=row.meta,
            created_at=row.created_at,
            updated_at=row.updated_at,
        )
```

- [ ] **Step 4: Write chunk, entity, and department repositories**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/chunk_repository.py`:

```python
from __future__ import annotations

import uuid

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
                id=chunk.id or str(uuid.uuid4()),
                source_id=chunk.source_id,
                source_version_id=chunk.source_version_id,
                chunk_index=chunk.chunk_index,
                content=chunk.content,
                summary=chunk.summary,
                token_count=chunk.token_count,
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
                chunk_index=row.chunk_index,
                content=row.content,
                summary=row.summary,
                token_count=row.token_count,
                meta=row.meta,
                created_at=row.created_at,
            )
            for row in result.scalars().all()
        ]

    async def delete_by_source(self, source_id: str) -> None:
        stmt = delete(ChunkRow).where(ChunkRow.source_id == source_id)
        await self._session.execute(stmt)
```

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/entity_repository.py`:

```python
from __future__ import annotations

import uuid

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.knowledge_mgmt.domain.entities.entity import KMEntity, EntityMention
from prescient.knowledge_mgmt.infrastructure.tables.entity import (
    KMEntityRow,
    EntityMentionRow,
)


class EntityRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def upsert_entity(self, entity: KMEntity) -> KMEntity:
        stmt = select(KMEntityRow).where(
            KMEntityRow.tenant_id == entity.tenant_id,
            KMEntityRow.name == entity.name,
            KMEntityRow.entity_type == entity.entity_type,
        )
        result = await self._session.execute(stmt)
        existing = result.scalar_one_or_none()
        if existing:
            return KMEntity(
                id=existing.id,
                tenant_id=existing.tenant_id,
                name=existing.name,
                entity_type=existing.entity_type,
                meta=existing.meta,
                created_at=existing.created_at,
            )
        row = KMEntityRow(
            id=entity.id or str(uuid.uuid4()),
            tenant_id=entity.tenant_id,
            name=entity.name,
            entity_type=entity.entity_type,
            meta=entity.meta,
        )
        self._session.add(row)
        await self._session.flush()
        return KMEntity(
            id=row.id,
            tenant_id=row.tenant_id,
            name=row.name,
            entity_type=row.entity_type,
            meta=row.meta,
            created_at=row.created_at,
        )

    async def create_mention(self, mention: EntityMention) -> None:
        row = EntityMentionRow(
            id=mention.id or str(uuid.uuid4()),
            entity_id=mention.entity_id,
            chunk_id=mention.chunk_id,
            context_snippet=mention.context_snippet,
        )
        self._session.add(row)
        await self._session.flush()
```

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/department_repository.py`:

```python
from __future__ import annotations

import uuid

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.knowledge_mgmt.domain.entities.department import (
    Department,
    DepartmentMembership,
)
from prescient.knowledge_mgmt.infrastructure.tables.department import (
    DepartmentRow,
    DepartmentMembershipRow,
)


class DepartmentRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, dept: Department) -> Department:
        row = DepartmentRow(
            id=dept.id or str(uuid.uuid4()),
            tenant_id=dept.tenant_id,
            company_id=dept.company_id,
            name=dept.name,
            slug=dept.slug,
        )
        self._session.add(row)
        await self._session.flush()
        return Department(
            id=row.id,
            tenant_id=row.tenant_id,
            company_id=row.company_id,
            name=row.name,
            slug=row.slug,
            created_at=row.created_at,
        )

    async def list_by_company(self, company_id: str) -> list[Department]:
        stmt = (
            select(DepartmentRow)
            .where(DepartmentRow.company_id == company_id)
            .order_by(DepartmentRow.name)
        )
        result = await self._session.execute(stmt)
        return [
            Department(
                id=row.id,
                tenant_id=row.tenant_id,
                company_id=row.company_id,
                name=row.name,
                slug=row.slug,
                created_at=row.created_at,
            )
            for row in result.scalars().all()
        ]

    async def get_user_departments(self, user_id: str) -> list[str]:
        stmt = select(DepartmentMembershipRow.department_id).where(
            DepartmentMembershipRow.user_id == user_id
        )
        result = await self._session.execute(stmt)
        return [row for row in result.scalars().all()]

    async def add_member(self, membership: DepartmentMembership) -> None:
        row = DepartmentMembershipRow(
            id=membership.id or str(uuid.uuid4()),
            user_id=membership.user_id,
            department_id=membership.department_id,
            is_lead=membership.is_lead,
        )
        self._session.add(row)
        await self._session.flush()
```

- [ ] **Step 5: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_source_repository.py -v
```
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/repositories/ apps/api/tests/unit/knowledge_mgmt/
git commit -m "feat(km): add repository layer for all KM tables"
```

---

## Task 5: File Storage Service

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/file_storage.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_file_storage.py`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/knowledge_mgmt/test_file_storage.py`:

```python
import pytest
from pathlib import Path

from prescient.knowledge_mgmt.infrastructure.services.file_storage import (
    LocalFileStorage,
)


@pytest.fixture
def storage(tmp_path: Path) -> LocalFileStorage:
    return LocalFileStorage(base_dir=str(tmp_path))


@pytest.mark.asyncio
async def test_store_and_retrieve(storage: LocalFileStorage, tmp_path: Path) -> None:
    content = b"Hello, this is a test document."
    path = await storage.store(
        tenant_id="t-1",
        source_id="src-001",
        filename="test.pdf",
        content=content,
    )
    assert path.startswith("t-1/src-001/")
    retrieved = await storage.retrieve(path)
    assert retrieved == content


@pytest.mark.asyncio
async def test_retrieve_missing_file(storage: LocalFileStorage) -> None:
    result = await storage.retrieve("nonexistent/path.pdf")
    assert result is None


@pytest.mark.asyncio
async def test_delete(storage: LocalFileStorage) -> None:
    content = b"Delete me"
    path = await storage.store("t-1", "src-002", "delete.pdf", content)
    await storage.delete(path)
    assert await storage.retrieve(path) is None
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_file_storage.py -v
```
Expected: FAIL — module not found

- [ ] **Step 3: Write file storage service**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/__init__.py` (empty).

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/file_storage.py`:

```python
from __future__ import annotations

import os
from pathlib import Path
from typing import Protocol


class FileStorage(Protocol):
    async def store(
        self, tenant_id: str, source_id: str, filename: str, content: bytes
    ) -> str: ...

    async def retrieve(self, path: str) -> bytes | None: ...

    async def delete(self, path: str) -> None: ...


class LocalFileStorage:
    """Local filesystem storage. Replace with S3FileStorage for production."""

    def __init__(self, base_dir: str) -> None:
        self._base_dir = Path(base_dir)

    async def store(
        self, tenant_id: str, source_id: str, filename: str, content: bytes
    ) -> str:
        relative_path = f"{tenant_id}/{source_id}/{filename}"
        full_path = self._base_dir / relative_path
        full_path.parent.mkdir(parents=True, exist_ok=True)
        full_path.write_bytes(content)
        return relative_path

    async def retrieve(self, path: str) -> bytes | None:
        full_path = self._base_dir / path
        if not full_path.exists():
            return None
        return full_path.read_bytes()

    async def delete(self, path: str) -> None:
        full_path = self._base_dir / path
        if full_path.exists():
            full_path.unlink()
```

- [ ] **Step 4: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_file_storage.py -v
```
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/services/ apps/api/tests/unit/knowledge_mgmt/test_file_storage.py
git commit -m "feat(km): add local file storage service with protocol for S3 swap"
```

---

## Task 6: Text Extraction Service

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/text_extractor.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_text_extractor.py`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/knowledge_mgmt/test_text_extractor.py`:

```python
import pytest

from prescient.knowledge_mgmt.infrastructure.services.text_extractor import (
    TextExtractor,
    ExtractionResult,
)


@pytest.mark.asyncio
async def test_extract_plain_text() -> None:
    extractor = TextExtractor()
    content = b"This is a plain text document.\n\nSecond paragraph."
    result = await extractor.extract(content, mime_type="text/plain")
    assert isinstance(result, ExtractionResult)
    assert "plain text document" in result.text
    assert len(result.sections) >= 1


@pytest.mark.asyncio
async def test_extract_markdown() -> None:
    extractor = TextExtractor()
    content = b"# Heading One\n\nParagraph under heading.\n\n## Heading Two\n\nMore text."
    result = await extractor.extract(content, mime_type="text/markdown")
    assert "Heading One" in result.text
    assert any(s.heading == "Heading One" for s in result.sections)
    assert any(s.heading == "Heading Two" for s in result.sections)


@pytest.mark.asyncio
async def test_extract_unsupported_type() -> None:
    extractor = TextExtractor()
    with pytest.raises(ValueError, match="Unsupported"):
        await extractor.extract(b"data", mime_type="application/zip")
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_text_extractor.py -v
```
Expected: FAIL — module not found

- [ ] **Step 3: Write text extractor**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/text_extractor.py`:

```python
from __future__ import annotations

import io
import re
from dataclasses import dataclass, field


@dataclass
class Section:
    heading: str | None
    content: str
    page: int | None = None
    level: int = 1


@dataclass
class ExtractionResult:
    text: str
    sections: list[Section] = field(default_factory=list)


SUPPORTED_MIME_TYPES = {
    "text/plain",
    "text/markdown",
    "text/html",
    "application/pdf",
    "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
}


class TextExtractor:
    async def extract(self, content: bytes, *, mime_type: str) -> ExtractionResult:
        if mime_type not in SUPPORTED_MIME_TYPES:
            raise ValueError(f"Unsupported mime type: {mime_type}")

        if mime_type == "text/plain":
            return self._extract_plain_text(content)
        elif mime_type == "text/markdown":
            return self._extract_markdown(content)
        elif mime_type == "application/pdf":
            return self._extract_pdf(content)
        elif mime_type.endswith("wordprocessingml.document"):
            return self._extract_docx(content)
        elif mime_type == "text/html":
            return self._extract_html(content)
        raise ValueError(f"Unsupported mime type: {mime_type}")

    def _extract_plain_text(self, content: bytes) -> ExtractionResult:
        text = content.decode("utf-8", errors="replace")
        paragraphs = [p.strip() for p in text.split("\n\n") if p.strip()]
        sections = [Section(heading=None, content=p) for p in paragraphs]
        return ExtractionResult(text=text, sections=sections or [Section(heading=None, content=text)])

    def _extract_markdown(self, content: bytes) -> ExtractionResult:
        text = content.decode("utf-8", errors="replace")
        sections: list[Section] = []
        current_heading: str | None = None
        current_level: int = 1
        current_content: list[str] = []

        for line in text.split("\n"):
            heading_match = re.match(r"^(#{1,6})\s+(.+)$", line)
            if heading_match:
                if current_heading is not None or current_content:
                    sections.append(
                        Section(
                            heading=current_heading,
                            content="\n".join(current_content).strip(),
                            level=current_level,
                        )
                    )
                current_heading = heading_match.group(2).strip()
                current_level = len(heading_match.group(1))
                current_content = []
            else:
                current_content.append(line)

        if current_heading is not None or current_content:
            sections.append(
                Section(
                    heading=current_heading,
                    content="\n".join(current_content).strip(),
                    level=current_level,
                )
            )

        return ExtractionResult(text=text, sections=sections)

    def _extract_pdf(self, content: bytes) -> ExtractionResult:
        import pdfplumber

        sections: list[Section] = []
        all_text_parts: list[str] = []

        with pdfplumber.open(io.BytesIO(content)) as pdf:
            for page_num, page in enumerate(pdf.pages, start=1):
                page_text = page.extract_text() or ""
                all_text_parts.append(page_text)
                if page_text.strip():
                    sections.append(
                        Section(heading=f"Page {page_num}", content=page_text.strip(), page=page_num)
                    )

        full_text = "\n\n".join(all_text_parts)
        return ExtractionResult(text=full_text, sections=sections)

    def _extract_docx(self, content: bytes) -> ExtractionResult:
        from docx import Document

        doc = Document(io.BytesIO(content))
        sections: list[Section] = []
        current_heading: str | None = None
        current_content: list[str] = []
        all_text_parts: list[str] = []

        for para in doc.paragraphs:
            text = para.text.strip()
            if not text:
                continue
            all_text_parts.append(text)

            if para.style and para.style.name and para.style.name.startswith("Heading"):
                if current_heading is not None or current_content:
                    sections.append(
                        Section(heading=current_heading, content="\n".join(current_content))
                    )
                current_heading = text
                current_content = []
            else:
                current_content.append(text)

        if current_heading is not None or current_content:
            sections.append(
                Section(heading=current_heading, content="\n".join(current_content))
            )

        full_text = "\n\n".join(all_text_parts)
        return ExtractionResult(text=full_text, sections=sections)

    def _extract_html(self, content: bytes) -> ExtractionResult:
        from html.parser import HTMLParser

        text = content.decode("utf-8", errors="replace")
        # Simple strip-tags extraction
        class TagStripper(HTMLParser):
            def __init__(self) -> None:
                super().__init__()
                self.parts: list[str] = []
            def handle_data(self, data: str) -> None:
                self.parts.append(data)

        stripper = TagStripper()
        stripper.feed(text)
        plain = " ".join(stripper.parts)
        return ExtractionResult(
            text=plain, sections=[Section(heading=None, content=plain)]
        )
```

- [ ] **Step 4: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_text_extractor.py -v
```
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/services/text_extractor.py apps/api/tests/unit/knowledge_mgmt/test_text_extractor.py
git commit -m "feat(km): add text extraction service for PDF, Word, Markdown, HTML, plain text"
```

---

## Task 7: Semantic Chunking Service

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/chunker.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_chunker.py`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/knowledge_mgmt/test_chunker.py`:

```python
import pytest

from prescient.knowledge_mgmt.infrastructure.services.chunker import (
    SemanticChunker,
)
from prescient.knowledge_mgmt.infrastructure.services.text_extractor import (
    ExtractionResult,
    Section,
)


def test_chunks_respect_section_boundaries() -> None:
    chunker = SemanticChunker(target_tokens=50, overlap_tokens=10)
    result = ExtractionResult(
        text="Section one content. " * 20 + "Section two content. " * 20,
        sections=[
            Section(heading="Section One", content="Section one content. " * 20),
            Section(heading="Section Two", content="Section two content. " * 20),
        ],
    )
    chunks = chunker.chunk(result, source_id="src-001")
    assert len(chunks) >= 2
    assert all(c.source_id == "src-001" for c in chunks)
    assert chunks[0].meta.get("section_heading") == "Section One"


def test_small_document_produces_single_chunk() -> None:
    chunker = SemanticChunker(target_tokens=500, overlap_tokens=50)
    result = ExtractionResult(
        text="Short document.",
        sections=[Section(heading=None, content="Short document.")],
    )
    chunks = chunker.chunk(result, source_id="src-002")
    assert len(chunks) == 1
    assert chunks[0].content == "Short document."


def test_chunk_indices_are_sequential() -> None:
    chunker = SemanticChunker(target_tokens=30, overlap_tokens=5)
    result = ExtractionResult(
        text="Word " * 200,
        sections=[Section(heading=None, content="Word " * 200)],
    )
    chunks = chunker.chunk(result, source_id="src-003")
    indices = [c.chunk_index for c in chunks]
    assert indices == list(range(len(chunks)))
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_chunker.py -v
```
Expected: FAIL

- [ ] **Step 3: Write chunker**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/chunker.py`:

```python
from __future__ import annotations

import uuid

from prescient.knowledge_mgmt.domain.entities.chunk import Chunk
from prescient.knowledge_mgmt.infrastructure.services.text_extractor import (
    ExtractionResult,
    Section,
)


def _estimate_tokens(text: str) -> int:
    """Rough token estimate: ~4 chars per token for English text."""
    return max(1, len(text) // 4)


class SemanticChunker:
    def __init__(self, target_tokens: int = 500, overlap_tokens: int = 50) -> None:
        self._target_tokens = target_tokens
        self._overlap_tokens = overlap_tokens

    def chunk(self, extraction: ExtractionResult, *, source_id: str) -> list[Chunk]:
        chunks: list[Chunk] = []
        chunk_index = 0

        for section in extraction.sections:
            section_chunks = self._chunk_section(
                section, source_id=source_id, start_index=chunk_index
            )
            chunks.extend(section_chunks)
            chunk_index += len(section_chunks)

        return chunks

    def _chunk_section(
        self, section: Section, *, source_id: str, start_index: int
    ) -> list[Chunk]:
        content = section.content.strip()
        if not content:
            return []

        token_count = _estimate_tokens(content)
        if token_count <= self._target_tokens:
            return [
                Chunk(
                    id=str(uuid.uuid4()),
                    source_id=source_id,
                    chunk_index=start_index,
                    content=content,
                    token_count=token_count,
                    meta=self._build_meta(section),
                )
            ]

        paragraphs = [p.strip() for p in content.split("\n\n") if p.strip()]
        if len(paragraphs) <= 1:
            paragraphs = [p.strip() for p in content.split("\n") if p.strip()]
        if len(paragraphs) <= 1:
            paragraphs = content.split(". ")
            paragraphs = [p + "." if not p.endswith(".") else p for p in paragraphs if p.strip()]

        chunks: list[Chunk] = []
        current_parts: list[str] = []
        current_tokens = 0
        overlap_buffer: list[str] = []

        for para in paragraphs:
            para_tokens = _estimate_tokens(para)

            if current_tokens + para_tokens > self._target_tokens and current_parts:
                chunk_text = "\n\n".join(current_parts)
                chunks.append(
                    Chunk(
                        id=str(uuid.uuid4()),
                        source_id=source_id,
                        chunk_index=start_index + len(chunks),
                        content=chunk_text,
                        token_count=_estimate_tokens(chunk_text),
                        meta=self._build_meta(section),
                    )
                )
                overlap_buffer = current_parts[-1:]
                overlap_tokens = _estimate_tokens("\n\n".join(overlap_buffer))
                current_parts = list(overlap_buffer)
                current_tokens = overlap_tokens

            current_parts.append(para)
            current_tokens += para_tokens

        if current_parts:
            chunk_text = "\n\n".join(current_parts)
            chunks.append(
                Chunk(
                    id=str(uuid.uuid4()),
                    source_id=source_id,
                    chunk_index=start_index + len(chunks),
                    content=chunk_text,
                    token_count=_estimate_tokens(chunk_text),
                    meta=self._build_meta(section),
                )
            )

        return chunks

    def _build_meta(self, section: Section) -> dict:
        meta: dict = {}
        if section.heading:
            meta["section_heading"] = section.heading
        if section.page is not None:
            meta["page"] = section.page
        if section.level != 1:
            meta["heading_level"] = section.level
        return meta
```

- [ ] **Step 4: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_chunker.py -v
```
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/services/chunker.py apps/api/tests/unit/knowledge_mgmt/test_chunker.py
git commit -m "feat(km): add semantic chunking service with section-aware splitting"
```

---

## Task 8: Enrichment Service

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/enrichment.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_enrichment.py`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/knowledge_mgmt/test_enrichment.py`:

```python
import pytest
from unittest.mock import AsyncMock, patch

from prescient.knowledge_mgmt.infrastructure.services.enrichment import (
    EnrichmentService,
    EnrichmentResult,
)
from prescient.knowledge_mgmt.domain.entities.chunk import Chunk


def _make_chunk(content: str, index: int = 0) -> Chunk:
    return Chunk(
        id=f"chunk-{index}",
        source_id="src-001",
        chunk_index=index,
        content=content,
        token_count=len(content) // 4,
    )


@pytest.mark.asyncio
async def test_enrich_generates_embeddings() -> None:
    mock_client = AsyncMock()
    mock_client.embeddings.create.return_value = AsyncMock(
        data=[AsyncMock(embedding=[0.1] * 1536)]
    )
    mock_client.chat.completions.create.return_value = AsyncMock(
        choices=[AsyncMock(message=AsyncMock(content='{"summary": "A test summary", "entities": []}'))]
    )

    service = EnrichmentService(ai_client=mock_client)
    chunks = [_make_chunk("This is the onboarding procedure for new employees.")]
    result = await service.enrich_chunks(chunks, tenant_id="t-1")

    assert isinstance(result, EnrichmentResult)
    assert len(result.embeddings) == 1
    assert len(result.embeddings[0]) == 1536
    assert len(result.summaries) == 1
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_enrichment.py -v
```
Expected: FAIL

- [ ] **Step 3: Write enrichment service**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/enrichment.py`:

```python
from __future__ import annotations

import json
import uuid
from dataclasses import dataclass, field

from prescient.knowledge_mgmt.domain.entities.chunk import Chunk
from prescient.knowledge_mgmt.domain.entities.entity import KMEntity, EntityMention

EMBEDDING_MODEL = "text-embedding-3-small"
ENRICHMENT_MODEL = "claude-sonnet-4-5-20250514"


@dataclass
class EnrichmentResult:
    embeddings: list[list[float]] = field(default_factory=list)
    summaries: list[str] = field(default_factory=list)
    entities: list[KMEntity] = field(default_factory=list)
    mentions: list[EntityMention] = field(default_factory=list)
    source_meta: dict = field(default_factory=dict)


class EnrichmentService:
    def __init__(self, ai_client: object) -> None:
        self._client = ai_client

    async def enrich_chunks(
        self, chunks: list[Chunk], *, tenant_id: str
    ) -> EnrichmentResult:
        result = EnrichmentResult()

        # Generate embeddings in batches
        for chunk in chunks:
            embedding = await self._get_embedding(chunk.content)
            result.embeddings.append(embedding)

        # Generate summaries and extract entities
        for chunk in chunks:
            chunk_analysis = await self._analyze_chunk(chunk, tenant_id=tenant_id)
            result.summaries.append(chunk_analysis["summary"])
            for entity_data in chunk_analysis.get("entities", []):
                entity = KMEntity(
                    id=str(uuid.uuid4()),
                    tenant_id=tenant_id,
                    name=entity_data["name"],
                    entity_type=entity_data["type"],
                )
                result.entities.append(entity)
                result.mentions.append(
                    EntityMention(
                        id=str(uuid.uuid4()),
                        entity_id=entity.id,
                        chunk_id=chunk.id,
                        context_snippet=chunk.content[:200],
                    )
                )

        return result

    async def enrich_source_meta(self, full_text: str) -> dict:
        """Extract source-level metadata: suggested title, department, topics, doc type."""
        prompt = (
            "Analyze this document and return JSON with these fields:\n"
            '- "suggested_title": a concise title\n'
            '- "suggested_department": likely department (HR, Finance, Operations, IT, Legal, Sales, Marketing, or General)\n'
            '- "document_type": one of (policy, sop, procedure, guide, training, playbook, reference, other)\n'
            '- "topics": list of 3-5 key topics\n'
            '- "effective_date": date if mentioned, else null\n\n'
            f"Document text (first 3000 chars):\n{full_text[:3000]}"
        )
        response = await self._client.chat.completions.create(
            model=ENRICHMENT_MODEL,
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
        )
        return json.loads(response.choices[0].message.content)

    async def _get_embedding(self, text: str) -> list[float]:
        response = await self._client.embeddings.create(
            model=EMBEDDING_MODEL, input=text
        )
        return response.data[0].embedding

    async def _analyze_chunk(self, chunk: Chunk, *, tenant_id: str) -> dict:
        prompt = (
            "Analyze this text chunk and return JSON with:\n"
            '- "summary": 1-2 sentence summary\n'
            '- "entities": list of {{"name": "...", "type": "person|process|system|policy|team"}}\n\n'
            f"Text:\n{chunk.content}"
        )
        response = await self._client.chat.completions.create(
            model=ENRICHMENT_MODEL,
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
        )
        return json.loads(response.choices[0].message.content)
```

- [ ] **Step 4: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_enrichment.py -v
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/services/enrichment.py apps/api/tests/unit/knowledge_mgmt/test_enrichment.py
git commit -m "feat(km): add AI enrichment service for embeddings, summaries, entity extraction"
```

---

## Task 9: Access Policy Service

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_access_policy.py`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/knowledge_mgmt/test_access_policy.py`:

```python
import pytest

from prescient.knowledge_mgmt.infrastructure.services.access_policy import (
    KMAccessPolicy,
    UserContext,
)
from prescient.knowledge_mgmt.domain.entities.source import Source, SourceType, Visibility


def _make_source(**overrides: object) -> Source:
    defaults = {
        "id": "src-001",
        "tenant_id": "t-1",
        "company_id": "c-1",
        "title": "Test",
        "source_type": SourceType.FILE_UPLOAD,
        "visibility": Visibility.COMPANY_WIDE,
        "min_access_role": "OPERATOR",
        "status": "ready",
        "created_by": "u-1",
        "updated_by": "u-1",
    }
    defaults.update(overrides)
    return Source(**defaults)


def test_company_wide_operator_access() -> None:
    policy = KMAccessPolicy()
    source = _make_source(visibility=Visibility.COMPANY_WIDE, min_access_role="OPERATOR")
    user = UserContext(user_id="u-1", tenant_id="t-1", company_id="c-1", role="OPERATOR", department_ids=[])
    assert policy.can_access(user, source) is True


def test_company_wide_higher_role_required() -> None:
    policy = KMAccessPolicy()
    source = _make_source(visibility=Visibility.COMPANY_WIDE, min_access_role="OPERATING_PARTNER")
    user = UserContext(user_id="u-1", tenant_id="t-1", company_id="c-1", role="OPERATOR", department_ids=[])
    assert policy.can_access(user, source) is False


def test_department_only_member_access() -> None:
    policy = KMAccessPolicy()
    source = _make_source(
        visibility=Visibility.DEPARTMENT_ONLY,
        min_access_role="OPERATOR",
        department_id="dept-1",
    )
    user = UserContext(user_id="u-1", tenant_id="t-1", company_id="c-1", role="OPERATOR", department_ids=["dept-1"])
    assert policy.can_access(user, source) is True


def test_department_only_non_member_denied() -> None:
    policy = KMAccessPolicy()
    source = _make_source(
        visibility=Visibility.DEPARTMENT_ONLY,
        min_access_role="OPERATOR",
        department_id="dept-1",
    )
    user = UserContext(user_id="u-1", tenant_id="t-1", company_id="c-1", role="OPERATOR", department_ids=["dept-2"])
    assert policy.can_access(user, source) is False


def test_wrong_company_denied() -> None:
    policy = KMAccessPolicy()
    source = _make_source(company_id="c-1")
    user = UserContext(user_id="u-1", tenant_id="t-1", company_id="c-2", role="OPERATOR", department_ids=[])
    assert policy.can_access(user, source) is False
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_access_policy.py -v
```
Expected: FAIL

- [ ] **Step 3: Write access policy service**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py`:

```python
from __future__ import annotations

from dataclasses import dataclass

from prescient.knowledge_mgmt.domain.entities.source import Source, Visibility

# Role hierarchy: least → most privileged
ROLE_HIERARCHY: dict[str, int] = {
    "OPERATOR": 0,
    "PE_ANALYST": 1,
    "BOARD_MEMBER": 2,
    "OPERATING_PARTNER": 3,
}


@dataclass
class UserContext:
    user_id: str
    tenant_id: str
    company_id: str
    role: str
    department_ids: list[str]
    is_admin: bool = False


class KMAccessPolicy:
    """Centralized access checks for KM sources.

    Designed to swap backing logic from DB checks to WorkOS
    without changing callers.
    """

    def can_access(self, user: UserContext, source: Source) -> bool:
        if user.is_admin:
            return True

        if user.tenant_id != source.tenant_id:
            return False

        if user.company_id != source.company_id:
            return False

        if not self._meets_role_requirement(user.role, source.min_access_role):
            return False

        if source.visibility == Visibility.DEPARTMENT_ONLY:
            if source.department_id and source.department_id not in user.department_ids:
                return False

        return True

    def can_manage(self, user: UserContext, source: Source) -> bool:
        if user.is_admin:
            return True
        if user.user_id == source.created_by:
            return True
        return False

    def _meets_role_requirement(self, user_role: str, min_role: str) -> bool:
        user_level = ROLE_HIERARCHY.get(user_role, -1)
        min_level = ROLE_HIERARCHY.get(min_role, 0)
        return user_level >= min_level

    def build_opensearch_filters(self, user: UserContext) -> list[dict]:
        """Build OpenSearch filter clauses for access-controlled search."""
        filters: list[dict] = [
            {"term": {"tenant_id": user.tenant_id}},
            {"term": {"company_id": user.company_id}},
        ]

        user_level = ROLE_HIERARCHY.get(user.role, 0)
        allowed_roles = [
            role for role, level in ROLE_HIERARCHY.items() if level <= user_level
        ]
        filters.append({"terms": {"min_access_role": allowed_roles}})

        # Visibility filter: company_wide OR (department_only AND user is member)
        visibility_filter = {
            "bool": {
                "should": [
                    {"term": {"visibility": "company_wide"}},
                    {
                        "bool": {
                            "must": [
                                {"term": {"visibility": "department_only"}},
                                {"terms": {"department_id": user.department_ids}},
                            ]
                        }
                    },
                ],
                "minimum_should_match": 1,
            }
        }
        filters.append(visibility_filter)

        return filters
```

- [ ] **Step 4: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_access_policy.py -v
```
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py apps/api/tests/unit/knowledge_mgmt/test_access_policy.py
git commit -m "feat(km): add role-based access policy service with OpenSearch filter generation"
```

---

## Task 10: Celery Setup and Ingestion Pipeline

**Files:**
- Create: `apps/api/src/prescient/celery_app.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingestion.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_ingestion.py`

- [ ] **Step 1: Write Celery app factory**

Create `apps/api/src/prescient/celery_app.py`:

```python
from celery import Celery

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
    )
    app.autodiscover_tasks(
        ["prescient.knowledge_mgmt.infrastructure.tasks"]
    )
    return app


celery_app = create_celery_app()
```

- [ ] **Step 2: Write failing test for ingestion pipeline**

Create `apps/api/tests/unit/knowledge_mgmt/test_ingestion.py`:

```python
import pytest
from unittest.mock import AsyncMock, MagicMock, patch

from prescient.knowledge_mgmt.infrastructure.tasks.ingestion import (
    run_ingestion_pipeline,
)


@pytest.mark.asyncio
async def test_pipeline_updates_status_to_processing() -> None:
    mock_session = AsyncMock()
    mock_source_repo = AsyncMock()
    mock_file_storage = AsyncMock()
    mock_file_storage.retrieve.return_value = b"Test document content."

    mock_extractor = AsyncMock()
    mock_extractor.extract.return_value = MagicMock(
        text="Test document content.",
        sections=[MagicMock(heading=None, content="Test document content.", page=None, level=1)],
    )

    mock_chunker = MagicMock()
    mock_chunker.chunk.return_value = [
        MagicMock(
            id="chunk-1", source_id="src-001", chunk_index=0,
            content="Test document content.", summary=None,
            token_count=5, meta={},
        )
    ]

    mock_enrichment = AsyncMock()
    mock_enrichment.enrich_chunks.return_value = MagicMock(
        embeddings=[[0.1] * 1536],
        summaries=["A test document."],
        entities=[],
        mentions=[],
        source_meta={},
    )
    mock_enrichment.enrich_source_meta.return_value = {
        "suggested_title": "Test Doc",
        "topics": ["testing"],
    }

    mock_search = AsyncMock()
    mock_chunk_repo = AsyncMock()
    mock_entity_repo = AsyncMock()

    await run_ingestion_pipeline(
        source_id="src-001",
        tenant_id="t-1",
        session=mock_session,
        source_repo=mock_source_repo,
        chunk_repo=mock_chunk_repo,
        entity_repo=mock_entity_repo,
        file_storage=mock_file_storage,
        extractor=mock_extractor,
        chunker=mock_chunker,
        enrichment=mock_enrichment,
        search_service=mock_search,
    )

    # Verify status was updated to processing then ready
    calls = mock_source_repo.update_status.call_args_list
    assert calls[0].args == ("src-001", "processing")
    assert calls[-1].args[0:2] == ("src-001", "ready")
```

- [ ] **Step 3: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_ingestion.py -v
```
Expected: FAIL

- [ ] **Step 4: Write ingestion pipeline**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/__init__.py` (empty).

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingestion.py`:

```python
from __future__ import annotations

import logging

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.knowledge_mgmt.infrastructure.repositories.source_repository import (
    SourceRepository,
)
from prescient.knowledge_mgmt.infrastructure.repositories.chunk_repository import (
    ChunkRepository,
)
from prescient.knowledge_mgmt.infrastructure.repositories.entity_repository import (
    EntityRepository,
)
from prescient.knowledge_mgmt.infrastructure.services.file_storage import FileStorage
from prescient.knowledge_mgmt.infrastructure.services.text_extractor import (
    TextExtractor,
)
from prescient.knowledge_mgmt.infrastructure.services.chunker import SemanticChunker
from prescient.knowledge_mgmt.infrastructure.services.enrichment import (
    EnrichmentService,
)

logger = logging.getLogger(__name__)


async def run_ingestion_pipeline(
    *,
    source_id: str,
    tenant_id: str,
    session: AsyncSession,
    source_repo: SourceRepository,
    chunk_repo: ChunkRepository,
    entity_repo: EntityRepository,
    file_storage: FileStorage,
    extractor: TextExtractor,
    chunker: SemanticChunker,
    enrichment: EnrichmentService,
    search_service: object,
) -> None:
    """Execute the full ingestion pipeline for a source.

    Stages: intake → extract text → chunk → enrich → index
    """
    source = await source_repo.get_by_id(source_id)
    if source is None:
        logger.error("Source %s not found, skipping ingestion", source_id)
        return

    try:
        # Stage 1: Mark processing
        await source_repo.update_status(source_id, "processing")
        await session.commit()

        # Stage 2: Retrieve and extract text
        if source.storage_path:
            content = await file_storage.retrieve(source.storage_path)
            if content is None:
                raise FileNotFoundError(f"File not found: {source.storage_path}")
            extraction = await extractor.extract(
                content, mime_type=source.mime_type or "text/plain"
            )
        elif source.meta.get("content"):
            # Authored entry — content stored in meta
            from prescient.knowledge_mgmt.infrastructure.services.text_extractor import (
                ExtractionResult,
                Section,
            )
            text = source.meta["content"]
            extraction = ExtractionResult(
                text=text,
                sections=[Section(heading=None, content=text)],
            )
        else:
            raise ValueError(f"Source {source_id} has no storage_path or meta.content")

        # Stage 3: Chunk
        chunks = chunker.chunk(extraction, source_id=source_id)

        # Stage 4: Enrich
        enrichment_result = await enrichment.enrich_chunks(chunks, tenant_id=tenant_id)
        source_meta = await enrichment.enrich_source_meta(extraction.text)

        # Update chunk summaries from enrichment
        for i, chunk in enumerate(chunks):
            if i < len(enrichment_result.summaries):
                chunk.summary = enrichment_result.summaries[i]

        # Persist chunks
        await chunk_repo.delete_by_source(source_id)
        await chunk_repo.bulk_create(chunks)

        # Persist entities and mentions
        for entity in enrichment_result.entities:
            persisted = await entity_repo.upsert_entity(entity)
            # Update mention references to use persisted entity ID
            for mention in enrichment_result.mentions:
                if mention.entity_id == entity.id:
                    mention.entity_id = persisted.id

        for mention in enrichment_result.mentions:
            await entity_repo.create_mention(mention)

        # Stage 5: Index in OpenSearch
        await search_service.index_chunks(
            chunks=chunks,
            embeddings=enrichment_result.embeddings,
            source=source,
            tenant_slug=tenant_id,
        )

        # Mark ready with enriched metadata
        await source_repo.update_status(source_id, "ready", meta=source_meta)
        await session.commit()

        logger.info("Ingestion complete for source %s: %d chunks", source_id, len(chunks))

    except Exception:
        logger.exception("Ingestion failed for source %s", source_id)
        await session.rollback()
        await source_repo.update_status(
            source_id, "failed", meta={"error": "Ingestion pipeline failed"}
        )
        await session.commit()
        raise
```

- [ ] **Step 5: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_ingestion.py -v
```
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/celery_app.py apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ apps/api/tests/unit/knowledge_mgmt/test_ingestion.py
git commit -m "feat(km): add Celery app factory and async ingestion pipeline"
```

---

## Task 11: OpenSearch Hybrid Search Service

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py`
- Create: `apps/api/tests/unit/knowledge_mgmt/test_search.py`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/knowledge_mgmt/test_search.py`:

```python
import pytest
from unittest.mock import AsyncMock

from prescient.knowledge_mgmt.infrastructure.services.search import (
    KMSearchService,
    SearchResult,
    SearchHit,
)
from prescient.knowledge_mgmt.infrastructure.services.access_policy import UserContext


@pytest.mark.asyncio
async def test_search_returns_fused_results() -> None:
    mock_os = AsyncMock()

    # BM25 response
    bm25_response = {
        "hits": {
            "hits": [
                {
                    "_id": "chunk-1",
                    "_score": 10.5,
                    "_source": {
                        "content": "Onboarding procedure step 1",
                        "summary": "First step of onboarding",
                        "source_id": "src-001",
                        "source_title": "Onboarding SOP",
                        "department_id": "dept-1",
                        "section_heading": "Step 1",
                    },
                },
                {
                    "_id": "chunk-2",
                    "_score": 8.2,
                    "_source": {
                        "content": "Onboarding checklist",
                        "summary": "Checklist for new hires",
                        "source_id": "src-001",
                        "source_title": "Onboarding SOP",
                        "department_id": "dept-1",
                        "section_heading": "Checklist",
                    },
                },
            ]
        }
    }

    # k-NN response (overlapping + different results)
    knn_response = {
        "hits": {
            "hits": [
                {
                    "_id": "chunk-1",
                    "_score": 0.92,
                    "_source": bm25_response["hits"]["hits"][0]["_source"],
                },
                {
                    "_id": "chunk-3",
                    "_score": 0.85,
                    "_source": {
                        "content": "Employee handbook section on first day",
                        "summary": "First day guidance",
                        "source_id": "src-002",
                        "source_title": "Employee Handbook",
                        "department_id": "dept-1",
                        "section_heading": "First Day",
                    },
                },
            ]
        }
    }

    mock_os.search.side_effect = [bm25_response, knn_response]

    mock_embed_fn = AsyncMock(return_value=[0.1] * 1536)

    service = KMSearchService(opensearch=mock_os, embed_fn=mock_embed_fn)
    user = UserContext(
        user_id="u-1", tenant_id="t-1", company_id="c-1",
        role="OPERATOR", department_ids=["dept-1"],
    )
    result = await service.search(query="onboarding process", user=user, tenant_slug="acme")

    assert isinstance(result, SearchResult)
    assert len(result.hits) >= 2
    # chunk-1 appears in both results so should rank highest (RRF)
    assert result.hits[0].chunk_id == "chunk-1"
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_search.py -v
```
Expected: FAIL

- [ ] **Step 3: Write search service**

Create `apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py`:

```python
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Callable, Awaitable

from prescient.knowledge_mgmt.infrastructure.services.access_policy import (
    KMAccessPolicy,
    UserContext,
)

INDEX_TEMPLATE = "km_chunks__{slug}"
KNN_FIELD = "embedding"
KNN_DIMENSIONS = 1536
RRF_K = 60  # Standard RRF constant


@dataclass
class SearchHit:
    chunk_id: str
    content: str
    summary: str | None
    source_id: str
    source_title: str
    department_id: str | None
    section_heading: str | None
    score: float


@dataclass
class SearchResult:
    hits: list[SearchHit] = field(default_factory=list)
    query: str = ""


class KMSearchService:
    def __init__(
        self,
        opensearch: object,
        embed_fn: Callable[[str], Awaitable[list[float]]],
    ) -> None:
        self._os = opensearch
        self._embed_fn = embed_fn
        self._access_policy = KMAccessPolicy()

    async def search(
        self,
        *,
        query: str,
        user: UserContext,
        tenant_slug: str,
        department_id: str | None = None,
        source_type: str | None = None,
        top_k: int = 10,
    ) -> SearchResult:
        index = INDEX_TEMPLATE.format(slug=tenant_slug)
        access_filters = self._access_policy.build_opensearch_filters(user)

        if department_id:
            access_filters.append({"term": {"department_id": department_id}})
        if source_type:
            access_filters.append({"term": {"source_type": source_type}})

        # Run BM25 and k-NN in parallel conceptually (sequential here, parallel in prod with asyncio.gather)
        bm25_hits = await self._bm25_search(index, query, access_filters, top_k)
        query_embedding = await self._embed_fn(query)
        knn_hits = await self._knn_search(index, query_embedding, access_filters, top_k)

        # Reciprocal Rank Fusion
        fused = self._rrf_merge(bm25_hits, knn_hits, top_k)

        return SearchResult(hits=fused, query=query)

    async def index_chunks(
        self,
        *,
        chunks: list,
        embeddings: list[list[float]],
        source: object,
        tenant_slug: str,
    ) -> None:
        index = INDEX_TEMPLATE.format(slug=tenant_slug)

        # Ensure index exists
        if not await self._os.indices.exists(index=index):
            await self._create_index(index)

        for i, chunk in enumerate(chunks):
            doc = {
                "content": chunk.content,
                "summary": chunk.summary,
                "source_id": chunk.source_id,
                "source_title": source.title,
                "source_type": source.source_type if isinstance(source.source_type, str) else source.source_type.value,
                "department_id": source.department_id,
                "company_id": source.company_id,
                "tenant_id": source.tenant_id,
                "visibility": source.visibility if isinstance(source.visibility, str) else source.visibility.value,
                "min_access_role": source.min_access_role,
                "section_heading": chunk.meta.get("section_heading"),
                "chunk_index": chunk.chunk_index,
                KNN_FIELD: embeddings[i] if i < len(embeddings) else None,
            }
            await self._os.index(index=index, id=chunk.id, body=doc)

    async def delete_by_source(self, *, source_id: str, tenant_slug: str) -> None:
        index = INDEX_TEMPLATE.format(slug=tenant_slug)
        await self._os.delete_by_query(
            index=index,
            body={"query": {"term": {"source_id": source_id}}},
        )

    async def _bm25_search(
        self, index: str, query: str, filters: list[dict], top_k: int
    ) -> list[SearchHit]:
        body = {
            "size": top_k,
            "query": {
                "bool": {
                    "must": [
                        {
                            "multi_match": {
                                "query": query,
                                "fields": ["content", "summary", "source_title"],
                            }
                        }
                    ],
                    "filter": filters,
                }
            },
        }
        response = await self._os.search(index=index, body=body)
        return self._parse_hits(response)

    async def _knn_search(
        self, index: str, embedding: list[float], filters: list[dict], top_k: int
    ) -> list[SearchHit]:
        body = {
            "size": top_k,
            "query": {
                "bool": {
                    "must": [
                        {
                            "knn": {
                                KNN_FIELD: {
                                    "vector": embedding,
                                    "k": top_k,
                                }
                            }
                        }
                    ],
                    "filter": filters,
                }
            },
        }
        response = await self._os.search(index=index, body=body)
        return self._parse_hits(response)

    async def _create_index(self, index: str) -> None:
        body = {
            "settings": {
                "index": {"knn": True},
            },
            "mappings": {
                "properties": {
                    "content": {"type": "text", "analyzer": "standard"},
                    "summary": {"type": "text"},
                    "source_id": {"type": "keyword"},
                    "source_title": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                    "source_type": {"type": "keyword"},
                    "department_id": {"type": "keyword"},
                    "company_id": {"type": "keyword"},
                    "tenant_id": {"type": "keyword"},
                    "visibility": {"type": "keyword"},
                    "min_access_role": {"type": "keyword"},
                    "section_heading": {"type": "text"},
                    "chunk_index": {"type": "integer"},
                    KNN_FIELD: {
                        "type": "knn_vector",
                        "dimension": KNN_DIMENSIONS,
                        "method": {
                            "name": "hnsw",
                            "space_type": "cosinesimil",
                            "engine": "nmslib",
                        },
                    },
                }
            },
        }
        await self._os.indices.create(index=index, body=body)

    def _parse_hits(self, response: dict) -> list[SearchHit]:
        hits = []
        for hit in response.get("hits", {}).get("hits", []):
            src = hit["_source"]
            hits.append(
                SearchHit(
                    chunk_id=hit["_id"],
                    content=src.get("content", ""),
                    summary=src.get("summary"),
                    source_id=src.get("source_id", ""),
                    source_title=src.get("source_title", ""),
                    department_id=src.get("department_id"),
                    section_heading=src.get("section_heading"),
                    score=hit.get("_score", 0.0),
                )
            )
        return hits

    def _rrf_merge(
        self, bm25: list[SearchHit], knn: list[SearchHit], top_k: int
    ) -> list[SearchHit]:
        scores: dict[str, float] = {}
        hit_map: dict[str, SearchHit] = {}

        for rank, hit in enumerate(bm25):
            scores[hit.chunk_id] = scores.get(hit.chunk_id, 0) + 1.0 / (RRF_K + rank + 1)
            hit_map[hit.chunk_id] = hit

        for rank, hit in enumerate(knn):
            scores[hit.chunk_id] = scores.get(hit.chunk_id, 0) + 1.0 / (RRF_K + rank + 1)
            if hit.chunk_id not in hit_map:
                hit_map[hit.chunk_id] = hit

        sorted_ids = sorted(scores, key=lambda cid: scores[cid], reverse=True)[:top_k]
        result = []
        for cid in sorted_ids:
            hit = hit_map[cid]
            hit.score = scores[cid]
            result.append(hit)
        return result
```

- [ ] **Step 4: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_search.py -v
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py apps/api/tests/unit/knowledge_mgmt/test_search.py
git commit -m "feat(km): add hybrid BM25 + k-NN search service with RRF fusion"
```

---

## Task 12: API Routes

**Files:**
- Create: `apps/api/src/prescient/knowledge_mgmt/api/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/api/routes.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/application/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/application/use_cases/create_source.py`
- Create: `apps/api/src/prescient/knowledge_mgmt/application/use_cases/list_sources.py`
- Modify: `apps/api/src/prescient/app.py` — mount KM router

- [ ] **Step 1: Write use cases**

Create empty `__init__.py` files for `application/` and `application/use_cases/`.

Create `apps/api/src/prescient/knowledge_mgmt/application/use_cases/create_source.py`:

```python
from __future__ import annotations

import uuid
from dataclasses import dataclass

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.knowledge_mgmt.domain.entities.source import (
    Source,
    SourceType,
    SourceStatus,
    Visibility,
)
from prescient.knowledge_mgmt.infrastructure.repositories.source_repository import (
    SourceRepository,
)
from prescient.knowledge_mgmt.infrastructure.services.file_storage import FileStorage


@dataclass
class CreateSourceRequest:
    tenant_id: str
    company_id: str
    user_id: str
    title: str
    source_type: SourceType
    visibility: Visibility = Visibility.COMPANY_WIDE
    min_access_role: str = "OPERATOR"
    department_id: str | None = None
    # For file uploads
    filename: str | None = None
    mime_type: str | None = None
    file_content: bytes | None = None
    # For authored entries
    text_content: str | None = None


class CreateSource:
    def __init__(self, session: AsyncSession, file_storage: FileStorage) -> None:
        self._session = session
        self._repo = SourceRepository(session)
        self._storage = file_storage

    async def execute(self, request: CreateSourceRequest) -> Source:
        source_id = str(uuid.uuid4())
        storage_path: str | None = None
        meta: dict = {}

        if request.source_type == SourceType.FILE_UPLOAD and request.file_content:
            storage_path = await self._storage.store(
                tenant_id=request.tenant_id,
                source_id=source_id,
                filename=request.filename or "upload",
                content=request.file_content,
            )
        elif request.source_type == SourceType.AUTHORED and request.text_content:
            meta["content"] = request.text_content

        source = Source(
            id=source_id,
            tenant_id=request.tenant_id,
            company_id=request.company_id,
            title=request.title,
            source_type=request.source_type,
            original_filename=request.filename,
            mime_type=request.mime_type,
            storage_path=storage_path,
            department_id=request.department_id,
            visibility=request.visibility,
            min_access_role=request.min_access_role,
            status=SourceStatus.PENDING,
            created_by=request.user_id,
            updated_by=request.user_id,
            meta=meta,
        )
        return await self._repo.create(source)
```

Create `apps/api/src/prescient/knowledge_mgmt/application/use_cases/list_sources.py`:

```python
from __future__ import annotations

from sqlalchemy.ext.asyncio import AsyncSession

from prescient.knowledge_mgmt.domain.entities.source import Source
from prescient.knowledge_mgmt.infrastructure.repositories.source_repository import (
    SourceRepository,
)


class ListSources:
    def __init__(self, session: AsyncSession) -> None:
        self._repo = SourceRepository(session)

    async def execute(
        self,
        company_id: str,
        *,
        department_id: str | None = None,
        status: str | None = None,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Source]:
        return await self._repo.list_by_company(
            company_id,
            department_id=department_id,
            status=status,
            limit=limit,
            offset=offset,
        )
```

- [ ] **Step 2: Write API routes**

Create `apps/api/src/prescient/knowledge_mgmt/api/__init__.py` (empty).

Create `apps/api/src/prescient/knowledge_mgmt/api/routes.py`:

```python
from __future__ import annotations

from typing import Annotated

from fastapi import APIRouter, Depends, File, Form, UploadFile, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.auth.dependencies import get_request_context, RequestContext
from prescient.shared.dependencies import get_session
from prescient.knowledge_mgmt.application.use_cases.create_source import (
    CreateSource,
    CreateSourceRequest,
)
from prescient.knowledge_mgmt.application.use_cases.list_sources import ListSources
from prescient.knowledge_mgmt.domain.entities.source import (
    SourceType,
    Visibility,
)
from prescient.knowledge_mgmt.infrastructure.services.file_storage import LocalFileStorage
from prescient.config.base import get_settings

router = APIRouter(prefix="/knowledge", tags=["knowledge-management"])

CtxDep = Annotated[RequestContext, Depends(get_request_context)]


def _get_file_storage() -> LocalFileStorage:
    settings = get_settings()
    return LocalFileStorage(base_dir=settings.file_storage_path)


class SourceResponse(BaseModel):
    id: str
    title: str
    source_type: str
    status: str
    visibility: str
    department_id: str | None = None
    original_filename: str | None = None
    meta: dict = Field(default_factory=dict)
    created_at: str | None = None
    updated_at: str | None = None


class CreateAuthoredBody(BaseModel):
    title: str = Field(min_length=1, max_length=256)
    content: str = Field(min_length=1)
    visibility: Visibility = Visibility.COMPANY_WIDE
    min_access_role: str = "OPERATOR"
    department_id: str | None = None


class SourceListResponse(BaseModel):
    items: list[SourceResponse]
    count: int


@router.post("/sources/upload", response_model=SourceResponse, status_code=201)
async def upload_source(
    ctx: CtxDep,
    file: UploadFile = File(...),
    title: str = Form(...),
    visibility: str = Form("company_wide"),
    min_access_role: str = Form("OPERATOR"),
    department_id: str | None = Form(None),
    session: AsyncSession = Depends(get_session),
) -> SourceResponse:
    content = await file.read()
    storage = _get_file_storage()
    use_case = CreateSource(session=session, file_storage=storage)
    source = await use_case.execute(
        CreateSourceRequest(
            tenant_id=ctx.tenant_id,
            company_id=ctx.company_id,
            user_id=ctx.user_id,
            title=title,
            source_type=SourceType.FILE_UPLOAD,
            visibility=Visibility(visibility),
            min_access_role=min_access_role,
            department_id=department_id,
            filename=file.filename,
            mime_type=file.content_type,
            file_content=content,
        )
    )
    await session.commit()

    # TODO: Dispatch Celery ingestion task here
    # from prescient.knowledge_mgmt.infrastructure.tasks.ingestion import ingest_source
    # ingest_source.delay(source_id=source.id, tenant_id=ctx.tenant_id)

    return _to_response(source)


@router.post("/sources/authored", response_model=SourceResponse, status_code=201)
async def create_authored_source(
    ctx: CtxDep,
    body: CreateAuthoredBody,
    session: AsyncSession = Depends(get_session),
) -> SourceResponse:
    storage = _get_file_storage()
    use_case = CreateSource(session=session, file_storage=storage)
    source = await use_case.execute(
        CreateSourceRequest(
            tenant_id=ctx.tenant_id,
            company_id=ctx.company_id,
            user_id=ctx.user_id,
            title=body.title,
            source_type=SourceType.AUTHORED,
            visibility=body.visibility,
            min_access_role=body.min_access_role,
            department_id=body.department_id,
            text_content=body.content,
        )
    )
    await session.commit()

    # TODO: Dispatch Celery ingestion task here

    return _to_response(source)


@router.get("/sources", response_model=SourceListResponse)
async def list_sources(
    ctx: CtxDep,
    department_id: str | None = None,
    status: str | None = None,
    limit: int = 50,
    offset: int = 0,
    session: AsyncSession = Depends(get_session),
) -> SourceListResponse:
    use_case = ListSources(session=session)
    sources = await use_case.execute(
        ctx.company_id,
        department_id=department_id,
        status=status,
        limit=limit,
        offset=offset,
    )
    return SourceListResponse(
        items=[_to_response(s) for s in sources],
        count=len(sources),
    )


@router.get("/sources/{source_id}", response_model=SourceResponse)
async def get_source(
    ctx: CtxDep,
    source_id: str,
    session: AsyncSession = Depends(get_session),
) -> SourceResponse:
    from prescient.knowledge_mgmt.infrastructure.repositories.source_repository import (
        SourceRepository,
    )
    from prescient.knowledge_mgmt.infrastructure.services.access_policy import (
        KMAccessPolicy,
        UserContext,
    )
    from prescient.knowledge_mgmt.infrastructure.repositories.department_repository import (
        DepartmentRepository,
    )

    repo = SourceRepository(session)
    source = await repo.get_by_id(source_id)
    if source is None:
        raise HTTPException(status_code=404, detail="Source not found")

    dept_repo = DepartmentRepository(session)
    user_depts = await dept_repo.get_user_departments(ctx.user_id)
    user_ctx = UserContext(
        user_id=ctx.user_id,
        tenant_id=ctx.tenant_id,
        company_id=ctx.company_id,
        role=ctx.role,
        department_ids=user_depts,
    )
    policy = KMAccessPolicy()
    if not policy.can_access(user_ctx, source):
        raise HTTPException(status_code=403, detail="Access denied")

    return _to_response(source)


def _to_response(source) -> SourceResponse:
    return SourceResponse(
        id=source.id,
        title=source.title,
        source_type=source.source_type if isinstance(source.source_type, str) else source.source_type.value,
        status=source.status if isinstance(source.status, str) else source.status.value,
        visibility=source.visibility if isinstance(source.visibility, str) else source.visibility.value,
        department_id=source.department_id,
        original_filename=source.original_filename,
        meta=source.meta,
        created_at=str(source.created_at) if source.created_at else None,
        updated_at=str(source.updated_at) if source.updated_at else None,
    )
```

- [ ] **Step 3: Mount router in app**

Read `apps/api/src/prescient/app.py` to find where other routers are mounted, then add:

```python
from prescient.knowledge_mgmt.api.routes import router as km_router

app.include_router(km_router, prefix="/api")
```

Add this alongside the existing router includes.

- [ ] **Step 4: Add `file_storage_path` to settings**

Read `apps/api/src/prescient/config/base.py` and add:

```python
file_storage_path: str = Field(default="/tmp/prescient/uploads")
```

- [ ] **Step 5: Verify routes load**

```bash
cd apps/api && python -c "from prescient.knowledge_mgmt.api.routes import router; print(f'Routes: {len(router.routes)}')"
```
Expected: `Routes: 4`

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/api/ apps/api/src/prescient/knowledge_mgmt/application/ apps/api/src/prescient/app.py apps/api/src/prescient/config/base.py
git commit -m "feat(km): add API routes for source upload, authoring, listing, and retrieval"
```

---

## Task 13: Copilot Tool — search_knowledge_base

**Files:**
- Create: `apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py`
- Modify: `apps/api/src/prescient/intelligence/infrastructure/tools/registry.py` — register new tool
- Create: `apps/api/tests/unit/knowledge_mgmt/test_search_knowledge_base_tool.py`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/unit/knowledge_mgmt/test_search_knowledge_base_tool.py`:

```python
import pytest
from unittest.mock import AsyncMock, MagicMock

from prescient.intelligence.infrastructure.tools.search_knowledge_base import (
    SearchKnowledgeBaseTool,
    SearchKnowledgeBaseArguments,
)


@pytest.mark.asyncio
async def test_tool_returns_formatted_results() -> None:
    mock_search = AsyncMock()
    mock_search.search.return_value = MagicMock(
        hits=[
            MagicMock(
                chunk_id="chunk-1",
                content="The onboarding process starts with...",
                summary="Overview of onboarding steps",
                source_id="src-001",
                source_title="Onboarding SOP",
                department_id="dept-1",
                section_heading="Overview",
                score=0.95,
            )
        ]
    )

    tool = SearchKnowledgeBaseTool(search_service=mock_search)
    context = MagicMock()
    context.user_id = "u-1"
    context.tenant_id = "t-1"
    context.company_id = "c-1"
    context.primary_company_slug = "acme"
    context.role = "OPERATOR"
    context.user_department_ids = ["dept-1"]

    args = SearchKnowledgeBaseArguments(query="onboarding process")
    result = await tool.execute(context, args)

    assert result.success is True
    assert "Onboarding SOP" in result.content
    assert "onboarding process starts with" in result.content


@pytest.mark.asyncio
async def test_tool_returns_no_results_message() -> None:
    mock_search = AsyncMock()
    mock_search.search.return_value = MagicMock(hits=[])

    tool = SearchKnowledgeBaseTool(search_service=mock_search)
    context = MagicMock()
    context.user_id = "u-1"
    context.tenant_id = "t-1"
    context.company_id = "c-1"
    context.primary_company_slug = "acme"
    context.role = "OPERATOR"
    context.user_department_ids = []

    args = SearchKnowledgeBaseArguments(query="nonexistent topic")
    result = await tool.execute(context, args)

    assert result.success is True
    assert "No results" in result.content
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_search_knowledge_base_tool.py -v
```
Expected: FAIL

- [ ] **Step 3: Write the copilot tool**

Read `apps/api/src/prescient/intelligence/infrastructure/tools/base.py` to confirm the `ToolExecution` and base tool interface, then create `apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py`:

```python
from __future__ import annotations

from pydantic import BaseModel, ConfigDict, Field

from prescient.intelligence.infrastructure.tools.base import ToolExecution
from prescient.knowledge_mgmt.infrastructure.services.search import KMSearchService
from prescient.knowledge_mgmt.infrastructure.services.access_policy import UserContext


class SearchKnowledgeBaseArguments(BaseModel):
    model_config = ConfigDict(extra="forbid")

    query: str = Field(
        min_length=1,
        max_length=512,
        description="Natural language question to search the knowledge base",
    )
    department: str | None = Field(
        default=None,
        description="Optional department ID to scope results",
    )
    source_type: str | None = Field(
        default=None,
        description="Optional source type filter (file_upload, authored)",
    )


class SearchKnowledgeBaseTool:
    name = "search_knowledge_base"
    description = (
        "Search the company's internal knowledge base — policies, SOPs, procedures, "
        "training materials, guides, and other institutional knowledge. Use this when "
        "the user asks about internal processes, company policies, how to do something, "
        "or references internal documentation."
    )
    arguments_model = SearchKnowledgeBaseArguments

    def __init__(self, search_service: KMSearchService) -> None:
        self._search = search_service

    async def execute(
        self, context: object, arguments: SearchKnowledgeBaseArguments
    ) -> ToolExecution:
        user = UserContext(
            user_id=context.user_id,
            tenant_id=context.tenant_id,
            company_id=context.company_id,
            role=context.role,
            department_ids=getattr(context, "user_department_ids", []),
        )

        result = await self._search.search(
            query=arguments.query,
            user=user,
            tenant_slug=context.primary_company_slug,
            department_id=arguments.department,
            source_type=arguments.source_type,
        )

        if not result.hits:
            return ToolExecution.ok("No results found in the knowledge base for this query.")

        parts: list[str] = []
        for i, hit in enumerate(result.hits, 1):
            section_info = f" (Section: {hit.section_heading})" if hit.section_heading else ""
            parts.append(
                f"### Result {i}: {hit.source_title}{section_info}\n\n"
                f"{hit.content}\n\n"
                f"_Source: {hit.source_title} | ID: {hit.source_id}_"
            )

        content = "\n\n---\n\n".join(parts)
        return ToolExecution.ok(content)
```

- [ ] **Step 4: Register tool in registry**

Read `apps/api/src/prescient/intelligence/infrastructure/tools/registry.py` to find where tools are registered, then add the `SearchKnowledgeBaseTool` to the registration list. The exact code depends on how the registry builds its tool list — follow the pattern of existing tools like `SearchDocumentsTool`.

- [ ] **Step 5: Run tests**

```bash
cd apps/api && python -m pytest tests/unit/knowledge_mgmt/test_search_knowledge_base_tool.py -v
```
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py apps/api/src/prescient/intelligence/infrastructure/tools/registry.py apps/api/tests/unit/knowledge_mgmt/test_search_knowledge_base_tool.py
git commit -m "feat(km): add search_knowledge_base copilot tool with hybrid search"
```

---

## Task 14: Web — Knowledge Management Pages

**Files:**
- Create: `apps/web/src/app/(main)/knowledge/page.tsx`
- Create: `apps/web/src/components/km-source-list.tsx`
- Create: `apps/web/src/components/km-upload-form.tsx`
- Create: `apps/web/src/app/(main)/knowledge/upload/page.tsx`
- Create: `apps/web/src/app/(main)/knowledge/[sourceId]/page.tsx`
- Create: `apps/web/src/components/km-source-detail.tsx`
- Modify: `apps/web/src/components/nav.tsx` — add Knowledge nav link
- Modify: `apps/web/src/lib/api.ts` — add KM API client methods

- [ ] **Step 1: Add KM API client methods**

Read `apps/web/src/lib/api.ts` to understand the existing client pattern, then add:

```typescript
// Knowledge Management
async listKMSources(params?: {
  department_id?: string;
  status?: string;
  limit?: number;
  offset?: number;
}) {
  const searchParams = new URLSearchParams();
  if (params?.department_id) searchParams.set("department_id", params.department_id);
  if (params?.status) searchParams.set("status", params.status);
  if (params?.limit) searchParams.set("limit", params.limit.toString());
  if (params?.offset) searchParams.set("offset", params.offset.toString());
  const query = searchParams.toString();
  return this.get(`/knowledge/sources${query ? `?${query}` : ""}`);
},

async getKMSource(sourceId: string) {
  return this.get(`/knowledge/sources/${sourceId}`);
},

async uploadKMSource(formData: FormData) {
  return this.postFormData("/knowledge/sources/upload", formData);
},

async createAuthoredKMSource(body: {
  title: string;
  content: string;
  visibility?: string;
  min_access_role?: string;
  department_id?: string;
}) {
  return this.post("/knowledge/sources/authored", body);
},
```

Follow the exact pattern used by existing methods in the file.

- [ ] **Step 2: Add Knowledge link to nav**

Read `apps/web/src/components/nav.tsx`, then add a Knowledge link after the existing nav items:

```tsx
<Link href="/knowledge">Knowledge</Link>
```

- [ ] **Step 3: Create source list component**

Create `apps/web/src/components/km-source-list.tsx`:

```tsx
"use client";

import Link from "next/link";

interface KMSource {
  id: string;
  title: string;
  source_type: string;
  status: string;
  visibility: string;
  department_id: string | null;
  original_filename: string | null;
  created_at: string | null;
  updated_at: string | null;
}

interface KMSourceListProps {
  sources: KMSource[];
}

const STATUS_STYLES: Record<string, string> = {
  ready: "bg-green-100 text-green-800",
  pending: "bg-yellow-100 text-yellow-800",
  processing: "bg-blue-100 text-blue-800",
  failed: "bg-red-100 text-red-800",
};

export function KMSourceList({ sources }: KMSourceListProps) {
  if (sources.length === 0) {
    return (
      <div className="text-center py-12 text-gray-500">
        <p>No knowledge sources yet.</p>
        <Link
          href="/knowledge/upload"
          className="mt-4 inline-block text-blue-600 hover:underline"
        >
          Upload your first document
        </Link>
      </div>
    );
  }

  return (
    <table className="w-full text-sm">
      <thead>
        <tr className="border-b text-left text-gray-500">
          <th className="pb-2 font-medium">Title</th>
          <th className="pb-2 font-medium">Type</th>
          <th className="pb-2 font-medium">Status</th>
          <th className="pb-2 font-medium">Visibility</th>
          <th className="pb-2 font-medium">Updated</th>
        </tr>
      </thead>
      <tbody>
        {sources.map((source) => (
          <tr key={source.id} className="border-b hover:bg-gray-50">
            <td className="py-3">
              <Link
                href={`/knowledge/${source.id}`}
                className="text-blue-600 hover:underline font-medium"
              >
                {source.title}
              </Link>
              {source.original_filename && (
                <span className="ml-2 text-xs text-gray-400">
                  {source.original_filename}
                </span>
              )}
            </td>
            <td className="py-3 capitalize">
              {source.source_type.replace("_", " ")}
            </td>
            <td className="py-3">
              <span
                className={`px-2 py-0.5 rounded text-xs font-medium ${STATUS_STYLES[source.status] ?? "bg-gray-100"}`}
              >
                {source.status}
              </span>
            </td>
            <td className="py-3 capitalize">
              {source.visibility.replace("_", " ")}
            </td>
            <td className="py-3 text-gray-500">
              {source.updated_at
                ? new Date(source.updated_at).toLocaleDateString()
                : "—"}
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 4: Create knowledge landing page**

Create `apps/web/src/app/(main)/knowledge/page.tsx`:

```tsx
import Link from "next/link";
import { api } from "@/lib/api";
import { KMSourceList } from "@/components/km-source-list";

export default async function KnowledgePage() {
  const data = await api.listKMSources();

  return (
    <div>
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-2xl font-semibold">Knowledge Base</h1>
        <Link
          href="/knowledge/upload"
          className="rounded bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700"
        >
          Add Source
        </Link>
      </div>
      <KMSourceList sources={data.items} />
    </div>
  );
}
```

- [ ] **Step 5: Create upload page and form**

Create `apps/web/src/components/km-upload-form.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { api } from "@/lib/api";

export function KMUploadForm() {
  const router = useRouter();
  const [mode, setMode] = useState<"upload" | "author">("upload");
  const [title, setTitle] = useState("");
  const [content, setContent] = useState("");
  const [file, setFile] = useState<File | null>(null);
  const [visibility, setVisibility] = useState("company_wide");
  const [submitting, setSubmitting] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setSubmitting(true);

    try {
      if (mode === "upload" && file) {
        const formData = new FormData();
        formData.append("file", file);
        formData.append("title", title);
        formData.append("visibility", visibility);
        await api.uploadKMSource(formData);
      } else if (mode === "author") {
        await api.createAuthoredKMSource({ title, content, visibility });
      }
      router.push("/knowledge");
    } finally {
      setSubmitting(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} className="max-w-2xl space-y-6">
      <div className="flex gap-4">
        <button
          type="button"
          onClick={() => setMode("upload")}
          className={`px-4 py-2 rounded text-sm font-medium ${mode === "upload" ? "bg-blue-600 text-white" : "bg-gray-100"}`}
        >
          Upload File
        </button>
        <button
          type="button"
          onClick={() => setMode("author")}
          className={`px-4 py-2 rounded text-sm font-medium ${mode === "author" ? "bg-blue-600 text-white" : "bg-gray-100"}`}
        >
          Write Entry
        </button>
      </div>

      <div>
        <label className="block text-sm font-medium mb-1">Title</label>
        <input
          type="text"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          required
          className="w-full rounded border px-3 py-2 text-sm"
          placeholder="e.g., Employee Onboarding SOP"
        />
      </div>

      {mode === "upload" ? (
        <div>
          <label className="block text-sm font-medium mb-1">File</label>
          <input
            type="file"
            accept=".pdf,.docx,.doc,.txt,.md,.html"
            onChange={(e) => setFile(e.target.files?.[0] ?? null)}
            required
            className="w-full text-sm"
          />
          <p className="mt-1 text-xs text-gray-500">
            PDF, Word, text, Markdown, or HTML
          </p>
        </div>
      ) : (
        <div>
          <label className="block text-sm font-medium mb-1">Content</label>
          <textarea
            value={content}
            onChange={(e) => setContent(e.target.value)}
            required
            rows={12}
            className="w-full rounded border px-3 py-2 text-sm font-mono"
            placeholder="Write in Markdown..."
          />
        </div>
      )}

      <div>
        <label className="block text-sm font-medium mb-1">Visibility</label>
        <select
          value={visibility}
          onChange={(e) => setVisibility(e.target.value)}
          className="rounded border px-3 py-2 text-sm"
        >
          <option value="company_wide">Company Wide</option>
          <option value="department_only">Department Only</option>
        </select>
      </div>

      <button
        type="submit"
        disabled={submitting}
        className="rounded bg-blue-600 px-6 py-2 text-sm font-medium text-white hover:bg-blue-700 disabled:opacity-50"
      >
        {submitting ? "Submitting..." : "Add Source"}
      </button>
    </form>
  );
}
```

Create `apps/web/src/app/(main)/knowledge/upload/page.tsx`:

```tsx
import { KMUploadForm } from "@/components/km-upload-form";

export default function KnowledgeUploadPage() {
  return (
    <div>
      <h1 className="text-2xl font-semibold mb-6">Add Knowledge Source</h1>
      <KMUploadForm />
    </div>
  );
}
```

- [ ] **Step 6: Create source detail page**

Create `apps/web/src/components/km-source-detail.tsx`:

```tsx
"use client";

interface KMSourceDetailProps {
  source: {
    id: string;
    title: string;
    source_type: string;
    status: string;
    visibility: string;
    department_id: string | null;
    original_filename: string | null;
    meta: Record<string, unknown>;
    created_at: string | null;
    updated_at: string | null;
  };
}

export function KMSourceDetail({ source }: KMSourceDetailProps) {
  const topics = (source.meta?.topics as string[]) ?? [];
  const suggestedDept = source.meta?.suggested_department as string | undefined;
  const docType = source.meta?.document_type as string | undefined;

  return (
    <div className="space-y-6">
      <div className="grid grid-cols-2 gap-4 text-sm">
        <div>
          <span className="text-gray-500">Type:</span>{" "}
          <span className="capitalize">{source.source_type.replace("_", " ")}</span>
        </div>
        <div>
          <span className="text-gray-500">Status:</span> {source.status}
        </div>
        <div>
          <span className="text-gray-500">Visibility:</span>{" "}
          <span className="capitalize">{source.visibility.replace("_", " ")}</span>
        </div>
        {source.original_filename && (
          <div>
            <span className="text-gray-500">File:</span> {source.original_filename}
          </div>
        )}
        {docType && (
          <div>
            <span className="text-gray-500">Document Type:</span>{" "}
            <span className="capitalize">{docType}</span>
          </div>
        )}
        {suggestedDept && (
          <div>
            <span className="text-gray-500">Department (suggested):</span> {suggestedDept}
          </div>
        )}
      </div>

      {topics.length > 0 && (
        <div>
          <h3 className="text-sm font-medium text-gray-500 mb-2">Topics</h3>
          <div className="flex flex-wrap gap-2">
            {topics.map((topic) => (
              <span
                key={topic}
                className="rounded bg-gray-100 px-2 py-1 text-xs"
              >
                {topic}
              </span>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}
```

Create `apps/web/src/app/(main)/knowledge/[sourceId]/page.tsx`:

```tsx
import Link from "next/link";
import { api } from "@/lib/api";
import { KMSourceDetail } from "@/components/km-source-detail";

export default async function KnowledgeSourcePage({
  params,
}: {
  params: { sourceId: string };
}) {
  const source = await api.getKMSource(params.sourceId);

  return (
    <div>
      <Link
        href="/knowledge"
        className="text-sm text-blue-600 hover:underline mb-4 inline-block"
      >
        Back to Knowledge Base
      </Link>
      <h1 className="text-2xl font-semibold mb-6">{source.title}</h1>
      <KMSourceDetail source={source} />
    </div>
  );
}
```

- [ ] **Step 7: Commit**

```bash
git add apps/web/src/app/\(main\)/knowledge/ apps/web/src/components/km-source-list.tsx apps/web/src/components/km-upload-form.tsx apps/web/src/components/km-source-detail.tsx apps/web/src/components/nav.tsx apps/web/src/lib/api.ts
git commit -m "feat(km): add knowledge management web pages — list, upload, detail, nav link"
```

---

## Task 15: Omni Search Bar

**Files:**
- Create: `apps/web/src/components/omni-search.tsx`
- Modify: `apps/web/src/components/nav.tsx` — add OmniSearch to nav
- Modify: `apps/web/src/lib/api.ts` — add quick search endpoint

- [ ] **Step 1: Add quick search API method**

Add to `apps/web/src/lib/api.ts`:

```typescript
async quickSearchKM(query: string) {
  return this.get(`/knowledge/sources?limit=5&search=${encodeURIComponent(query)}`);
},
```

- [ ] **Step 2: Create omni search component**

Create `apps/web/src/components/omni-search.tsx`:

```tsx
"use client";

import { useState, useEffect, useRef, useCallback } from "react";
import Link from "next/link";
import { useRouter } from "next/navigation";
import { api } from "@/lib/api";

interface SearchHit {
  id: string;
  title: string;
  source_type: string;
  status: string;
}

export function OmniSearch() {
  const [open, setOpen] = useState(false);
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<SearchHit[]>([]);
  const [loading, setLoading] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);
  const router = useRouter();

  // Cmd+K handler
  useEffect(() => {
    function onKeyDown(e: KeyboardEvent) {
      if ((e.metaKey || e.ctrlKey) && e.key === "k") {
        e.preventDefault();
        setOpen((prev) => !prev);
      }
      if (e.key === "Escape") {
        setOpen(false);
      }
    }
    window.addEventListener("keydown", onKeyDown);
    return () => window.removeEventListener("keydown", onKeyDown);
  }, []);

  useEffect(() => {
    if (open) inputRef.current?.focus();
  }, [open]);

  // Debounced search
  useEffect(() => {
    if (!query.trim()) {
      setResults([]);
      return;
    }
    const timer = setTimeout(async () => {
      setLoading(true);
      try {
        const data = await api.quickSearchKM(query);
        setResults(data.items ?? []);
      } catch {
        setResults([]);
      } finally {
        setLoading(false);
      }
    }, 200);
    return () => clearTimeout(timer);
  }, [query]);

  const handleAskCopilot = useCallback(() => {
    setOpen(false);
    router.push(`/chat?q=${encodeURIComponent(query)}`);
  }, [query, router]);

  if (!open) {
    return (
      <button
        onClick={() => setOpen(true)}
        className="flex items-center gap-2 rounded border px-3 py-1.5 text-sm text-gray-500 hover:border-gray-400"
      >
        <span>Search...</span>
        <kbd className="rounded bg-gray-100 px-1.5 py-0.5 text-xs">
          {typeof navigator !== "undefined" &&
          navigator.platform?.includes("Mac")
            ? "\u2318"
            : "Ctrl+"}
          K
        </kbd>
      </button>
    );
  }

  return (
    <>
      <div
        className="fixed inset-0 z-40 bg-black/20"
        onClick={() => setOpen(false)}
      />
      <div className="fixed left-1/2 top-[20%] z-50 w-full max-w-lg -translate-x-1/2 rounded-lg bg-white shadow-xl border">
        <input
          ref={inputRef}
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search knowledge base..."
          className="w-full rounded-t-lg border-b px-4 py-3 text-sm outline-none"
          onKeyDown={(e) => {
            if (e.key === "Enter" && query.trim()) {
              handleAskCopilot();
            }
          }}
        />
        <div className="max-h-80 overflow-y-auto">
          {loading && (
            <div className="px-4 py-3 text-sm text-gray-500">Searching...</div>
          )}
          {!loading && results.length > 0 && (
            <ul>
              {results.map((hit) => (
                <li key={hit.id}>
                  <Link
                    href={`/knowledge/${hit.id}`}
                    onClick={() => setOpen(false)}
                    className="block px-4 py-2.5 text-sm hover:bg-gray-50"
                  >
                    <span className="font-medium">{hit.title}</span>
                    <span className="ml-2 text-xs text-gray-400 capitalize">
                      {hit.source_type.replace("_", " ")}
                    </span>
                  </Link>
                </li>
              ))}
            </ul>
          )}
          {query.trim() && (
            <button
              onClick={handleAskCopilot}
              className="w-full border-t px-4 py-3 text-left text-sm text-blue-600 hover:bg-blue-50"
            >
              Ask the copilot: &ldquo;{query}&rdquo;
            </button>
          )}
        </div>
      </div>
    </>
  );
}
```

- [ ] **Step 3: Add OmniSearch to nav**

Read `apps/web/src/components/nav.tsx` and add the OmniSearch component in the nav bar:

```tsx
import { OmniSearch } from "@/components/omni-search";

// Inside the nav, between the links and the portfolio dropdown:
<OmniSearch />
```

- [ ] **Step 4: Verify it compiles**

```bash
cd apps/web && npx next lint
```
Expected: No errors related to new files.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/components/omni-search.tsx apps/web/src/components/nav.tsx apps/web/src/lib/api.ts
git commit -m "feat(km): add Cmd+K omni search bar with quick results and copilot route"
```

---

## Task 16: Integration Test — Full Pipeline

**Files:**
- Create: `apps/api/tests/integration/knowledge_mgmt/__init__.py`
- Create: `apps/api/tests/integration/knowledge_mgmt/test_km_integration.py`

- [ ] **Step 1: Write integration test**

Create `apps/api/tests/integration/knowledge_mgmt/__init__.py` (empty).

Create `apps/api/tests/integration/knowledge_mgmt/test_km_integration.py`:

```python
import pytest
from httpx import AsyncClient

from prescient.app import create_app


@pytest.fixture
async def client():
    app = create_app()
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac


@pytest.mark.asyncio
async def test_upload_and_list_sources(client: AsyncClient) -> None:
    """End-to-end: upload a source, verify it appears in the list."""
    # Upload a text file
    response = await client.post(
        "/api/knowledge/sources/upload",
        files={"file": ("test.txt", b"This is test content for the knowledge base.", "text/plain")},
        data={"title": "Test Document", "visibility": "company_wide"},
    )
    assert response.status_code == 201
    source = response.json()
    assert source["title"] == "Test Document"
    assert source["status"] == "pending"

    # List sources
    response = await client.get("/api/knowledge/sources")
    assert response.status_code == 200
    data = response.json()
    assert data["count"] >= 1
    assert any(s["id"] == source["id"] for s in data["items"])


@pytest.mark.asyncio
async def test_create_authored_source(client: AsyncClient) -> None:
    """Create an authored entry and retrieve it."""
    response = await client.post(
        "/api/knowledge/sources/authored",
        json={
            "title": "Vendor Approval Process",
            "content": "All vendor approvals must go through the procurement team...",
            "visibility": "company_wide",
        },
    )
    assert response.status_code == 201
    source = response.json()
    assert source["source_type"] == "authored"

    # Get by ID
    response = await client.get(f"/api/knowledge/sources/{source['id']}")
    assert response.status_code == 200
    assert response.json()["title"] == "Vendor Approval Process"
```

- [ ] **Step 2: Run integration tests**

```bash
cd apps/api && python -m pytest tests/integration/knowledge_mgmt/ -v
```
Expected: Tests pass against dev database (or skip if DB not available).

- [ ] **Step 3: Commit**

```bash
git add apps/api/tests/integration/knowledge_mgmt/
git commit -m "test(km): add integration tests for upload, author, and list endpoints"
```

---

## Task 17: Department Management Page (Admin)

**Files:**
- Create: `apps/web/src/app/(main)/knowledge/departments/page.tsx`
- Create: `apps/web/src/components/km-department-manager.tsx`
- Modify: `apps/web/src/lib/api.ts` — add department API methods
- Create: `apps/api/src/prescient/knowledge_mgmt/api/department_routes.py` — department CRUD endpoints
- Modify: `apps/api/src/prescient/app.py` — mount department router

- [ ] **Step 1: Add department API endpoints**

Create `apps/api/src/prescient/knowledge_mgmt/api/department_routes.py`:

```python
from __future__ import annotations

from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.auth.dependencies import get_request_context, RequestContext
from prescient.shared.dependencies import get_session
from prescient.knowledge_mgmt.domain.entities.department import Department, DepartmentMembership
from prescient.knowledge_mgmt.infrastructure.repositories.department_repository import (
    DepartmentRepository,
)

import uuid
import re

router = APIRouter(prefix="/knowledge/departments", tags=["knowledge-management"])

CtxDep = Annotated[RequestContext, Depends(get_request_context)]


class CreateDepartmentBody(BaseModel):
    name: str = Field(min_length=1, max_length=128)


class DepartmentResponse(BaseModel):
    id: str
    name: str
    slug: str
    company_id: str
    created_at: str | None = None


class AddMemberBody(BaseModel):
    user_id: str
    is_lead: bool = False


@router.post("/", response_model=DepartmentResponse, status_code=201)
async def create_department(
    ctx: CtxDep,
    body: CreateDepartmentBody,
    session: AsyncSession = Depends(get_session),
) -> DepartmentResponse:
    repo = DepartmentRepository(session)
    slug = re.sub(r"[^a-z0-9]+", "-", body.name.lower()).strip("-")
    dept = await repo.create(
        Department(
            id=str(uuid.uuid4()),
            tenant_id=ctx.tenant_id,
            company_id=ctx.company_id,
            name=body.name,
            slug=slug,
        )
    )
    await session.commit()
    return DepartmentResponse(
        id=dept.id,
        name=dept.name,
        slug=dept.slug,
        company_id=dept.company_id,
        created_at=str(dept.created_at) if dept.created_at else None,
    )


@router.get("/", response_model=list[DepartmentResponse])
async def list_departments(
    ctx: CtxDep,
    session: AsyncSession = Depends(get_session),
) -> list[DepartmentResponse]:
    repo = DepartmentRepository(session)
    depts = await repo.list_by_company(ctx.company_id)
    return [
        DepartmentResponse(
            id=d.id,
            name=d.name,
            slug=d.slug,
            company_id=d.company_id,
            created_at=str(d.created_at) if d.created_at else None,
        )
        for d in depts
    ]


@router.post("/{department_id}/members", status_code=201)
async def add_member(
    ctx: CtxDep,
    department_id: str,
    body: AddMemberBody,
    session: AsyncSession = Depends(get_session),
) -> dict:
    repo = DepartmentRepository(session)
    await repo.add_member(
        DepartmentMembership(
            id=str(uuid.uuid4()),
            user_id=body.user_id,
            department_id=department_id,
            is_lead=body.is_lead,
        )
    )
    await session.commit()
    return {"status": "ok"}
```

- [ ] **Step 2: Mount department routes**

Add to `apps/api/src/prescient/app.py`:

```python
from prescient.knowledge_mgmt.api.department_routes import router as km_dept_router
app.include_router(km_dept_router, prefix="/api")
```

- [ ] **Step 3: Add department API client methods**

Add to `apps/web/src/lib/api.ts`:

```typescript
async listDepartments() {
  return this.get("/knowledge/departments");
},

async createDepartment(name: string) {
  return this.post("/knowledge/departments", { name });
},
```

- [ ] **Step 4: Create department manager component**

Create `apps/web/src/components/km-department-manager.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { api } from "@/lib/api";

interface Department {
  id: string;
  name: string;
  slug: string;
  created_at: string | null;
}

interface KMDepartmentManagerProps {
  departments: Department[];
}

export function KMDepartmentManager({ departments }: KMDepartmentManagerProps) {
  const router = useRouter();
  const [name, setName] = useState("");
  const [creating, setCreating] = useState(false);

  async function handleCreate(e: React.FormEvent) {
    e.preventDefault();
    if (!name.trim()) return;
    setCreating(true);
    try {
      await api.createDepartment(name.trim());
      setName("");
      router.refresh();
    } finally {
      setCreating(false);
    }
  }

  return (
    <div className="space-y-6">
      <form onSubmit={handleCreate} className="flex gap-3">
        <input
          type="text"
          value={name}
          onChange={(e) => setName(e.target.value)}
          placeholder="New department name"
          className="flex-1 rounded border px-3 py-2 text-sm"
        />
        <button
          type="submit"
          disabled={creating || !name.trim()}
          className="rounded bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700 disabled:opacity-50"
        >
          {creating ? "Creating..." : "Create"}
        </button>
      </form>

      {departments.length === 0 ? (
        <p className="text-sm text-gray-500">No departments created yet.</p>
      ) : (
        <table className="w-full text-sm">
          <thead>
            <tr className="border-b text-left text-gray-500">
              <th className="pb-2 font-medium">Name</th>
              <th className="pb-2 font-medium">Slug</th>
              <th className="pb-2 font-medium">Created</th>
            </tr>
          </thead>
          <tbody>
            {departments.map((dept) => (
              <tr key={dept.id} className="border-b">
                <td className="py-3 font-medium">{dept.name}</td>
                <td className="py-3 text-gray-500">{dept.slug}</td>
                <td className="py-3 text-gray-500">
                  {dept.created_at
                    ? new Date(dept.created_at).toLocaleDateString()
                    : "—"}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
    </div>
  );
}
```

- [ ] **Step 5: Create departments page**

Create `apps/web/src/app/(main)/knowledge/departments/page.tsx`:

```tsx
import { api } from "@/lib/api";
import { KMDepartmentManager } from "@/components/km-department-manager";

export default async function DepartmentsPage() {
  const departments = await api.listDepartments();

  return (
    <div>
      <h1 className="text-2xl font-semibold mb-6">Departments</h1>
      <KMDepartmentManager departments={departments} />
    </div>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/knowledge_mgmt/api/department_routes.py apps/api/src/prescient/app.py apps/web/src/app/\(main\)/knowledge/departments/ apps/web/src/components/km-department-manager.tsx apps/web/src/lib/api.ts
git commit -m "feat(km): add department management page and API endpoints"
```

---

## Dependencies to Add

Before starting implementation, add these Python packages to the API's dependencies:

```
celery[redis]
pdfplumber
python-docx
```

Add to `apps/api/pyproject.toml` in the dependencies section. Run `pip install -e .` or equivalent to install.

For OpenSearch k-NN, the plugin must be enabled on the cluster. Add to `docker-compose.yml` if not already present:

```yaml
opensearch:
  environment:
    - "plugins.security.disabled=true"
    - "plugins.knn.enabled=true"
```
