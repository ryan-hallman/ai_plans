# Board Prep Interactive Deck Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Transform the Board Prep page from a static section list into a high-fidelity slide editor with an AI assistant panel, inline AI actions, slide management, and PDF/PPTX export.

**Architecture:** Three-zone layout (filmstrip | canvas | AI panel) backed by Zustand state, new backend persistence endpoints, and AI chat/inline-action endpoints. The existing `generate_deck` endpoint stays as-is for initial generation; new endpoints handle CRUD, AI, and export.

**Tech Stack:** Next.js 15, React 19, TypeScript, Zustand, Tailwind, shadcn/Radix, `@dnd-kit/sortable` (drag reorder), `html2canvas` + `jsPDF` (PDF export), `python-pptx` (PPTX export server-side), FastAPI, SQLAlchemy, Anthropic SDK.

**Worktree:** `.worktrees/board-prep-interactive` on branch `feature/board-prep-interactive-deck`

---

## File Structure

### Frontend — New Files
| File | Responsibility |
|------|---------------|
| `apps/web/src/stores/deck-store.ts` | Zustand store: slides, active index, undo/redo, dirty state, chat history |
| `apps/web/src/app/(main)/board-prep/page.tsx` | Rewrite: three-zone layout, wire up all components |
| `apps/web/src/components/board-prep/slide-filmstrip.tsx` | Slide thumbnail list, drag-to-reorder, add/delete |
| `apps/web/src/components/board-prep/slide-canvas.tsx` | High-fidelity slide renderer with WYSIWYG editing |
| `apps/web/src/components/board-prep/slide-header.tsx` | Colored banner with editable title |
| `apps/web/src/components/board-prep/slide-content.tsx` | Editable markdown content area with citation rendering |
| `apps/web/src/components/board-prep/ai-panel.tsx` | Chat panel: input, messages, context-aware AI |
| `apps/web/src/components/board-prep/inline-action-toolbar.tsx` | Floating toolbar on text selection |
| `apps/web/src/components/board-prep/export-dropdown.tsx` | Export button with PDF/PPTX options |
| `apps/web/src/components/board-prep/deck-toolbar.tsx` | Top toolbar: quarter selector, generate, export, approve/share |

### Frontend — Modified Files
| File | Change |
|------|--------|
| `apps/web/src/components/board-section.tsx` | No longer used by board-prep page (kept for now, unused) |

### Backend — New Files
| File | Responsibility |
|------|---------------|
| `apps/api/src/prescient/board_prep/infrastructure/tables/board_deck.py` | SQLAlchemy table: `board_prep.board_decks` |
| `apps/api/src/prescient/board_prep/infrastructure/repository.py` | Repository: CRUD for board decks |
| `apps/api/src/prescient/board_prep/application/chat_service.py` | AI chat: context-aware slide editing via LLM |
| `apps/api/src/prescient/board_prep/application/inline_action_service.py` | Inline AI actions: rewrite, concise, expand, tone |
| `apps/api/src/prescient/board_prep/application/export_service.py` | PPTX generation via python-pptx |
| `apps/api/alembic/versions/20260413_board_prep_schema.py` | Migration: create `board_prep` schema + `board_decks` table |

### Backend — Modified Files
| File | Change |
|------|--------|
| `apps/api/src/prescient/board_prep/api/routes.py` | Add CRUD, chat, inline-action, and export endpoints |
| `apps/api/src/prescient/board_prep/domain/entities/board_deck.py` | Make `BoardDeck` mutable, add slide management methods |

---

## Task 1: Database Schema & Migration

**Files:**
- Create: `apps/api/alembic/versions/20260413_board_prep_schema.py`
- Create: `apps/api/src/prescient/board_prep/infrastructure/__init__.py`
- Create: `apps/api/src/prescient/board_prep/infrastructure/tables/__init__.py`
- Create: `apps/api/src/prescient/board_prep/infrastructure/tables/board_deck.py`

- [ ] **Step 1: Create the migration file**

Create `apps/api/alembic/versions/20260413_board_prep_schema.py`:

```python
"""board prep persistence — board_decks table

Revision ID: 0007_board_prep_schema
Revises: 0006_km_schema
Create Date: 2026-04-13

Adds the `board_prep` schema with a `board_decks` table for persisting
generated and edited board decks.
"""

from __future__ import annotations

from collections.abc import Sequence

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision: str = "0007_board_prep_schema"
down_revision: str | None = "0006_km_schema"
branch_labels: str | Sequence[str] | None = None
depends_on: str | Sequence[str] | None = None


def upgrade() -> None:
    op.execute("CREATE SCHEMA IF NOT EXISTS board_prep")

    op.create_table(
        "board_decks",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("owner_tenant_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("company_slug", sa.String(128), nullable=False),
        sa.Column("quarter", sa.String(16), nullable=False),
        sa.Column("slides", postgresql.JSONB(astext_type=sa.Text()), nullable=False),
        sa.Column(
            "status",
            sa.String(16),
            nullable=False,
            server_default="draft",
        ),
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
        schema="board_prep",
    )

    op.create_index(
        "ix_board_prep_board_decks_tenant",
        "board_decks",
        ["owner_tenant_id"],
        schema="board_prep",
    )
    op.create_index(
        "ix_board_prep_board_decks_company",
        "board_decks",
        ["company_slug"],
        schema="board_prep",
    )


def downgrade() -> None:
    op.drop_table("board_decks", schema="board_prep")
    op.execute("DROP SCHEMA IF EXISTS board_prep CASCADE")
```

- [ ] **Step 2: Create the SQLAlchemy table model**

Create empty `__init__.py` files:
- `apps/api/src/prescient/board_prep/infrastructure/__init__.py`
- `apps/api/src/prescient/board_prep/infrastructure/tables/__init__.py`

Create `apps/api/src/prescient/board_prep/infrastructure/tables/board_deck.py`:

```python
"""`board_prep.board_decks` — persisted board decks with editable slides."""

from __future__ import annotations

from datetime import datetime
from uuid import UUID

from sqlalchemy import DateTime, String, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from sqlalchemy.orm import Mapped, mapped_column

from prescient.shared.db_base import Base


class BoardDeckTable(Base):
    __tablename__ = "board_decks"
    __table_args__ = {"schema": "board_prep"}  # noqa: RUF012

    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True)
    owner_tenant_id: Mapped[UUID] = mapped_column(
        PgUUID(as_uuid=True), nullable=False, index=True
    )
    company_slug: Mapped[str] = mapped_column(String(128), nullable=False, index=True)
    quarter: Mapped[str] = mapped_column(String(16), nullable=False)
    slides: Mapped[dict] = mapped_column(JSONB, nullable=False)
    status: Mapped[str] = mapped_column(String(16), nullable=False, server_default="draft")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )
```

- [ ] **Step 3: Run the migration**

```bash
cd apps/api && make docker-migrate
```

Expected: Migration applies successfully, `board_prep.board_decks` table created.

- [ ] **Step 4: Commit**

```bash
git add apps/api/alembic/versions/20260413_board_prep_schema.py \
  apps/api/src/prescient/board_prep/infrastructure/__init__.py \
  apps/api/src/prescient/board_prep/infrastructure/tables/__init__.py \
  apps/api/src/prescient/board_prep/infrastructure/tables/board_deck.py
git commit -m "feat(board-prep): add board_decks persistence schema and table"
```

---

## Task 2: Backend Repository & Domain Updates

**Files:**
- Create: `apps/api/src/prescient/board_prep/infrastructure/repository.py`
- Modify: `apps/api/src/prescient/board_prep/domain/entities/board_deck.py`

- [ ] **Step 1: Update domain entities to support mutability and JSON serialization**

Replace the contents of `apps/api/src/prescient/board_prep/domain/entities/board_deck.py`:

```python
"""Board deck domain entities — mutable value objects for a generated/edited deck."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID


@dataclass
class BoardSection:
    """One slide of a board deck with citation-annotated markdown content."""

    number: int
    title: str
    content: str  # Markdown with [[n]] citations
    citations: list[dict] = field(default_factory=list)

    def to_dict(self) -> dict:
        return {
            "number": self.number,
            "title": self.title,
            "content": self.content,
            "citations": self.citations,
        }

    @classmethod
    def from_dict(cls, data: dict) -> BoardSection:
        return cls(
            number=data["number"],
            title=data["title"],
            content=data["content"],
            citations=data.get("citations", []),
        )


@dataclass
class BoardDeck:
    """A fully assembled quarterly board deck."""

    id: UUID
    company_slug: str
    quarter: str
    sections: list[BoardSection]
    generated_at: datetime
    status: str = "draft"
    owner_tenant_id: UUID | None = None

    def slides_to_json(self) -> list[dict]:
        return [s.to_dict() for s in self.sections]

    @classmethod
    def sections_from_json(cls, data: list[dict]) -> list[BoardSection]:
        return [BoardSection.from_dict(d) for d in data]
```

- [ ] **Step 2: Create the repository**

Create `apps/api/src/prescient/board_prep/infrastructure/repository.py`:

```python
"""Board deck repository — CRUD for persisted board decks."""

from __future__ import annotations

from datetime import UTC, datetime
from uuid import UUID

from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.board_prep.domain.entities.board_deck import BoardDeck, BoardSection
from prescient.board_prep.infrastructure.tables.board_deck import BoardDeckTable


class BoardDeckRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def save(self, deck: BoardDeck) -> None:
        """Insert or update a board deck."""
        existing = await self._session.get(BoardDeckTable, deck.id)
        if existing:
            existing.slides = deck.slides_to_json()
            existing.status = deck.status
            existing.updated_at = datetime.now(UTC)
        else:
            row = BoardDeckTable(
                id=deck.id,
                owner_tenant_id=deck.owner_tenant_id,
                company_slug=deck.company_slug,
                quarter=deck.quarter,
                slides=deck.slides_to_json(),
                status=deck.status,
            )
            self._session.add(row)
        await self._session.flush()

    async def get(self, deck_id: UUID) -> BoardDeck | None:
        """Load a board deck by ID."""
        row = await self._session.get(BoardDeckTable, deck_id)
        if row is None:
            return None
        return BoardDeck(
            id=row.id,
            company_slug=row.company_slug,
            quarter=row.quarter,
            sections=BoardDeck.sections_from_json(row.slides),
            generated_at=row.created_at,
            status=row.status,
            owner_tenant_id=row.owner_tenant_id,
        )

    async def update_slides(self, deck_id: UUID, slides: list[dict]) -> None:
        """Update just the slides JSON for a deck."""
        await self._session.execute(
            update(BoardDeckTable)
            .where(BoardDeckTable.id == deck_id)
            .values(slides=slides, updated_at=datetime.now(UTC))
        )
        await self._session.flush()

    async def update_status(self, deck_id: UUID, status: str) -> None:
        """Update the deck status."""
        await self._session.execute(
            update(BoardDeckTable)
            .where(BoardDeckTable.id == deck_id)
            .values(status=status, updated_at=datetime.now(UTC))
        )
        await self._session.flush()
```

- [ ] **Step 3: Verify imports work**

```bash
cd apps/api && python -c "from prescient.board_prep.infrastructure.repository import BoardDeckRepository; print('OK')"
```

Expected: `OK`

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/board_prep/domain/entities/board_deck.py \
  apps/api/src/prescient/board_prep/infrastructure/repository.py
git commit -m "feat(board-prep): add deck repository and mutable domain entities"
```

---

## Task 3: Backend CRUD & Generate Endpoints

**Files:**
- Modify: `apps/api/src/prescient/board_prep/api/routes.py`

- [ ] **Step 1: Rewrite routes with CRUD endpoints**

Replace `apps/api/src/prescient/board_prep/api/routes.py` with:

```python
"""Board prep API routes — generate, persist, and manage board decks."""

from __future__ import annotations

from functools import lru_cache
from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, ConfigDict, Field

from prescient.board_prep.application.assembler import BoardPrepAssembler
from prescient.board_prep.infrastructure.repository import BoardDeckRepository
from prescient.config import get_settings
from prescient.context import RequestContext, get_request_context
from prescient.intelligence.domain.ports import LLMProviderPort
from prescient.intelligence.infrastructure.anthropic_adapter import AnthropicAdapter
from prescient.shared.errors import ValidationError

router = APIRouter(prefix="/board-prep", tags=["board-prep"])

CtxDep = Annotated[RequestContext, Depends(get_request_context)]


# ---- LLM dependency --------------------------------------------------------


@lru_cache(maxsize=1)
def get_llm_provider() -> LLMProviderPort:
    settings = get_settings()
    if not settings.anthropic_api_key:
        raise ValidationError("ANTHROPIC_API_KEY is not configured")
    return AnthropicAdapter(
        api_key=settings.anthropic_api_key,
        default_model=settings.anthropic_model,
    )


LLMDep = Annotated[LLMProviderPort, Depends(get_llm_provider)]


# ---- request / response models ---------------------------------------------


class GenerateDeckRequest(BaseModel):
    model_config = ConfigDict(frozen=True)
    company_slug: str = Field(min_length=1, max_length=128)
    quarter: str = Field(min_length=1, max_length=16, examples=["FY2026-Q3"])


class SlidePayload(BaseModel):
    model_config = ConfigDict(frozen=True)
    number: int
    title: str
    content: str
    citations: list[dict] = Field(default_factory=list)


class DeckResponse(BaseModel):
    model_config = ConfigDict(frozen=True)
    id: str
    company_slug: str
    quarter: str
    slides: list[SlidePayload]
    status: str
    generated_at: str


class UpdateSlidesRequest(BaseModel):
    slides: list[SlidePayload]


class UpdateSlideRequest(BaseModel):
    title: str | None = None
    content: str | None = None
    citations: list[dict] | None = None


class AddSlideRequest(BaseModel):
    title: str = "New Slide"
    content: str = ""
    after_number: int | None = None  # Insert after this slide number; None = append


class UpdateStatusRequest(BaseModel):
    status: str = Field(pattern=r"^(draft|approved|shared)$")


# ---- helpers ----------------------------------------------------------------


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
            )
            for s in deck.sections
        ],
        status=deck.status,
        generated_at=deck.generated_at.isoformat(),
    )


async def _get_deck_or_404(ctx: RequestContext, deck_id: UUID):
    repo = BoardDeckRepository(ctx.session)
    deck = await repo.get(deck_id)
    if deck is None:
        raise HTTPException(status_code=404, detail="Deck not found")
    return deck, repo


# ---- routes -----------------------------------------------------------------


@router.post("/generate", response_model=DeckResponse)
async def generate_deck(
    body: GenerateDeckRequest, ctx: CtxDep, llm: LLMDep
) -> DeckResponse:
    """Generate a board deck and persist it."""
    settings = get_settings()
    assembler = BoardPrepAssembler(session=ctx.session, llm=llm, fund_id=ctx.user.fund_id)
    deck = await assembler.generate_deck(
        company_slug=body.company_slug,
        quarter=body.quarter,
        model=settings.anthropic_model,
    )
    deck.owner_tenant_id = ctx.user.fund_id

    repo = BoardDeckRepository(ctx.session)
    await repo.save(deck)
    await ctx.session.commit()

    return _deck_to_response(deck)


@router.get("/{deck_id}", response_model=DeckResponse)
async def get_deck(deck_id: UUID, ctx: CtxDep) -> DeckResponse:
    """Retrieve a persisted board deck."""
    deck, _ = await _get_deck_or_404(ctx, deck_id)
    return _deck_to_response(deck)


@router.put("/{deck_id}/slides", response_model=DeckResponse)
async def update_slides(
    deck_id: UUID, body: UpdateSlidesRequest, ctx: CtxDep
) -> DeckResponse:
    """Replace all slides in a deck (full save)."""
    deck, repo = await _get_deck_or_404(ctx, deck_id)
    slides_json = [s.model_dump() for s in body.slides]
    await repo.update_slides(deck_id, slides_json)
    await ctx.session.commit()

    deck = await repo.get(deck_id)
    return _deck_to_response(deck)


@router.put("/{deck_id}/slides/{slide_number}", response_model=DeckResponse)
async def update_slide(
    deck_id: UUID, slide_number: int, body: UpdateSlideRequest, ctx: CtxDep
) -> DeckResponse:
    """Update a single slide by number."""
    deck, repo = await _get_deck_or_404(ctx, deck_id)

    slides = deck.slides_to_json()
    target = next((s for s in slides if s["number"] == slide_number), None)
    if target is None:
        raise HTTPException(status_code=404, detail=f"Slide {slide_number} not found")

    if body.title is not None:
        target["title"] = body.title
    if body.content is not None:
        target["content"] = body.content
    if body.citations is not None:
        target["citations"] = body.citations

    await repo.update_slides(deck_id, slides)
    await ctx.session.commit()

    deck = await repo.get(deck_id)
    return _deck_to_response(deck)


@router.post("/{deck_id}/slides", response_model=DeckResponse)
async def add_slide(
    deck_id: UUID, body: AddSlideRequest, ctx: CtxDep
) -> DeckResponse:
    """Add a new slide to the deck."""
    deck, repo = await _get_deck_or_404(ctx, deck_id)
    slides = deck.slides_to_json()

    # Determine insert position
    if body.after_number is not None:
        idx = next(
            (i for i, s in enumerate(slides) if s["number"] == body.after_number),
            len(slides) - 1,
        )
        insert_at = idx + 1
    else:
        insert_at = len(slides)

    new_slide = {
        "number": 0,  # Will be renumbered
        "title": body.title,
        "content": body.content,
        "citations": [],
    }
    slides.insert(insert_at, new_slide)

    # Renumber all slides sequentially
    for i, s in enumerate(slides):
        s["number"] = i + 1

    await repo.update_slides(deck_id, slides)
    await ctx.session.commit()

    deck = await repo.get(deck_id)
    return _deck_to_response(deck)


@router.delete("/{deck_id}/slides/{slide_number}", response_model=DeckResponse)
async def delete_slide(
    deck_id: UUID, slide_number: int, ctx: CtxDep
) -> DeckResponse:
    """Remove a slide from the deck."""
    deck, repo = await _get_deck_or_404(ctx, deck_id)
    slides = deck.slides_to_json()
    slides = [s for s in slides if s["number"] != slide_number]

    if not slides:
        raise HTTPException(status_code=400, detail="Cannot delete the last slide")

    # Renumber
    for i, s in enumerate(slides):
        s["number"] = i + 1

    await repo.update_slides(deck_id, slides)
    await ctx.session.commit()

    deck = await repo.get(deck_id)
    return _deck_to_response(deck)


@router.put("/{deck_id}/status")
async def update_deck_status(
    deck_id: UUID, body: UpdateStatusRequest, ctx: CtxDep
) -> dict:
    """Update deck status (draft, approved, shared)."""
    _, repo = await _get_deck_or_404(ctx, deck_id)
    await repo.update_status(deck_id, body.status)
    await ctx.session.commit()
    return {"status": body.status}
```

- [ ] **Step 2: Verify the server starts**

```bash
cd apps/api && make docker-up
```

Check logs for import errors. Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/board_prep/api/routes.py
git commit -m "feat(board-prep): add CRUD endpoints for deck persistence"
```

---

## Task 4: Backend AI Chat & Inline Action Endpoints

**Files:**
- Create: `apps/api/src/prescient/board_prep/application/chat_service.py`
- Create: `apps/api/src/prescient/board_prep/application/inline_action_service.py`
- Modify: `apps/api/src/prescient/board_prep/api/routes.py`

- [ ] **Step 1: Create the chat service**

Create `apps/api/src/prescient/board_prep/application/chat_service.py`:

```python
"""Board prep chat service — context-aware AI assistant for deck editing."""

from __future__ import annotations

import json
import logging
import re
from typing import Any

from prescient.intelligence.domain.ports import LLMProviderPort

logger = logging.getLogger(__name__)


async def handle_chat_message(
    *,
    llm: LLMProviderPort,
    model: str,
    user_message: str,
    active_slide: dict | None,
    all_slides: list[dict],
    conversation_history: list[dict],
) -> dict:
    """Process a chat message and return response + optional mutations.

    Returns:
        {
            "response": str,       # Assistant's text reply
            "mutations": [         # Optional content changes
                {"type": "update_slide", "slide_number": int, "content": str},
                {"type": "add_slide", "after_number": int, "title": str, "content": str},
                {"type": "reorder", "order": [int, ...]},
            ]
        }
    """
    deck_summary = "\n".join(
        f"Slide {s['number']}: {s['title']}" for s in all_slides
    )

    active_context = ""
    if active_slide:
        active_context = (
            f"\n\nCurrently viewing Slide {active_slide['number']}: "
            f"{active_slide['title']}\n\nContent:\n{active_slide['content']}"
        )

    system_prompt = (
        "You are an AI assistant helping prepare a board deck. "
        "You can view and edit slide content. When the user asks you to modify "
        "content, respond with your explanation AND include a JSON mutations block "
        "at the end of your response.\n\n"
        "Mutations block format (place at the very end, on its own line):\n"
        "```mutations\n"
        '[{"type": "update_slide", "slide_number": 1, "content": "new content...", "title": "optional new title"}]\n'
        "```\n\n"
        "Mutation types:\n"
        '- {"type": "update_slide", "slide_number": N, "content": "...", "title": "..."}\n'
        '- {"type": "add_slide", "after_number": N, "title": "...", "content": "..."}\n'
        '- {"type": "reorder", "order": [3, 1, 2, ...]}\n'
        '- {"type": "delete_slide", "slide_number": N}\n\n'
        "Only include mutations when the user explicitly asks you to change something. "
        "For questions or analysis, just respond with text.\n\n"
        f"Deck structure:\n{deck_summary}{active_context}"
    )

    messages = list(conversation_history)
    messages.append({"role": "user", "content": user_message})

    response = await llm.send(
        system=system_prompt,
        messages=messages,
        tools=[],
        model=model,
    )

    # Extract text content
    response_text = ""
    for block in response.get("content", []):
        if isinstance(block, dict) and block.get("type") == "text":
            response_text += block["text"]
        elif isinstance(block, str):
            response_text += block

    # Parse mutations block if present
    mutations = []
    mutations_match = re.search(
        r"```mutations\n(.+?)\n```", response_text, re.DOTALL
    )
    if mutations_match:
        try:
            mutations = json.loads(mutations_match.group(1))
        except json.JSONDecodeError:
            logger.warning("Failed to parse mutations JSON from AI response")

        # Remove mutations block from visible response
        response_text = response_text[: mutations_match.start()].strip()

    return {
        "response": response_text,
        "mutations": mutations,
    }
```

- [ ] **Step 2: Create the inline action service**

Create `apps/api/src/prescient/board_prep/application/inline_action_service.py`:

```python
"""Inline AI actions — quick text transformations for selected slide content."""

from __future__ import annotations

from prescient.intelligence.domain.ports import LLMProviderPort

_ACTION_PROMPTS = {
    "rewrite": "Rewrite the following text, keeping the same meaning but with fresh phrasing. Maintain any [[n]] citation markers exactly as they appear.",
    "concise": "Make the following text more concise while preserving all key information and [[n]] citation markers.",
    "expand": "Expand the following text with more detail, analysis, and context. Preserve any [[n]] citation markers.",
    "tone_formal": "Rewrite the following text in a formal, board-appropriate executive tone. Preserve any [[n]] citation markers.",
    "tone_conversational": "Rewrite the following text in a conversational, accessible tone while keeping it professional. Preserve any [[n]] citation markers.",
    "tone_executive": "Rewrite the following text in a crisp, executive summary style — short sentences, action-oriented. Preserve any [[n]] citation markers.",
}


async def apply_inline_action(
    *,
    llm: LLMProviderPort,
    model: str,
    selected_text: str,
    action: str,
    slide_context: str,
) -> str:
    """Apply an inline AI action to selected text and return the replacement.

    Args:
        selected_text: The text the user selected on the slide.
        action: One of: rewrite, concise, expand, tone_formal, tone_conversational, tone_executive.
        slide_context: The full slide content for surrounding context.

    Returns:
        The replacement text.
    """
    action_prompt = _ACTION_PROMPTS.get(action)
    if action_prompt is None:
        raise ValueError(f"Unknown inline action: {action}")

    system_prompt = (
        "You are editing a board deck slide. Return ONLY the rewritten text — "
        "no preamble, no explanation, no markdown code fences. "
        "The text should be a direct replacement for the selected text."
    )

    user_message = (
        f"{action_prompt}\n\n"
        f"Selected text:\n{selected_text}\n\n"
        f"Surrounding slide context:\n{slide_context}"
    )

    response = await llm.send(
        system=system_prompt,
        messages=[{"role": "user", "content": user_message}],
        tools=[],
        model=model,
    )

    result = ""
    for block in response.get("content", []):
        if isinstance(block, dict) and block.get("type") == "text":
            result += block["text"]
        elif isinstance(block, str):
            result += block

    return result.strip()
```

- [ ] **Step 3: Add chat and inline-action endpoints to routes**

Add these imports at the top of `apps/api/src/prescient/board_prep/api/routes.py`:

```python
from prescient.board_prep.application.chat_service import handle_chat_message
from prescient.board_prep.application.inline_action_service import apply_inline_action
```

Add these request models after the existing ones:

```python
class ChatRequest(BaseModel):
    message: str
    active_slide_number: int | None = None
    conversation_history: list[dict] = Field(default_factory=list)


class ChatResponse(BaseModel):
    response: str
    mutations: list[dict] = Field(default_factory=list)


class InlineActionRequest(BaseModel):
    selected_text: str
    action: str = Field(pattern=r"^(rewrite|concise|expand|tone_formal|tone_conversational|tone_executive)$")
    slide_number: int


class InlineActionResponse(BaseModel):
    replacement_text: str
```

Add these route handlers at the end of the routes file:

```python
@router.post("/{deck_id}/chat", response_model=ChatResponse)
async def chat_with_deck(
    deck_id: UUID, body: ChatRequest, ctx: CtxDep, llm: LLMDep
) -> ChatResponse:
    """Send a message to the AI assistant about the deck."""
    deck, repo = await _get_deck_or_404(ctx, deck_id)
    slides = deck.slides_to_json()

    active_slide = None
    if body.active_slide_number is not None:
        active_slide = next(
            (s for s in slides if s["number"] == body.active_slide_number), None
        )

    settings = get_settings()
    result = await handle_chat_message(
        llm=llm,
        model=settings.anthropic_model,
        user_message=body.message,
        active_slide=active_slide,
        all_slides=slides,
        conversation_history=body.conversation_history,
    )

    # Apply mutations to persisted deck if any
    if result["mutations"]:
        for mutation in result["mutations"]:
            if mutation["type"] == "update_slide":
                target = next(
                    (s for s in slides if s["number"] == mutation["slide_number"]),
                    None,
                )
                if target:
                    if "content" in mutation:
                        target["content"] = mutation["content"]
                    if "title" in mutation:
                        target["title"] = mutation["title"]
            elif mutation["type"] == "add_slide":
                after = mutation.get("after_number", len(slides))
                idx = next(
                    (i for i, s in enumerate(slides) if s["number"] == after),
                    len(slides) - 1,
                )
                slides.insert(idx + 1, {
                    "number": 0,
                    "title": mutation.get("title", "New Slide"),
                    "content": mutation.get("content", ""),
                    "citations": [],
                })
                for i, s in enumerate(slides):
                    s["number"] = i + 1
            elif mutation["type"] == "delete_slide":
                slides = [s for s in slides if s["number"] != mutation["slide_number"]]
                for i, s in enumerate(slides):
                    s["number"] = i + 1
            elif mutation["type"] == "reorder":
                order = mutation["order"]
                by_number = {s["number"]: s for s in slides}
                slides = [by_number[n] for n in order if n in by_number]
                for i, s in enumerate(slides):
                    s["number"] = i + 1

        await repo.update_slides(deck_id, slides)
        await ctx.session.commit()

    return ChatResponse(
        response=result["response"],
        mutations=result["mutations"],
    )


@router.post("/{deck_id}/inline-action", response_model=InlineActionResponse)
async def inline_action(
    deck_id: UUID, body: InlineActionRequest, ctx: CtxDep, llm: LLMDep
) -> InlineActionResponse:
    """Apply an inline AI action to selected text on a slide."""
    deck, _ = await _get_deck_or_404(ctx, deck_id)
    slides = deck.slides_to_json()

    slide = next(
        (s for s in slides if s["number"] == body.slide_number), None
    )
    if slide is None:
        raise HTTPException(status_code=404, detail=f"Slide {body.slide_number} not found")

    settings = get_settings()
    replacement = await apply_inline_action(
        llm=llm,
        model=settings.anthropic_model,
        selected_text=body.selected_text,
        action=body.action,
        slide_context=slide["content"],
    )

    return InlineActionResponse(replacement_text=replacement)
```

- [ ] **Step 4: Verify server starts cleanly**

```bash
cd apps/api && make docker-up
```

Expected: No import errors.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/board_prep/application/chat_service.py \
  apps/api/src/prescient/board_prep/application/inline_action_service.py \
  apps/api/src/prescient/board_prep/api/routes.py
git commit -m "feat(board-prep): add AI chat and inline action endpoints"
```

---

## Task 5: Backend PPTX Export Endpoint

**Files:**
- Create: `apps/api/src/prescient/board_prep/application/export_service.py`
- Modify: `apps/api/src/prescient/board_prep/api/routes.py`

- [ ] **Step 1: Install python-pptx**

```bash
cd apps/api && pip install python-pptx
```

Add `python-pptx` to the project's dependencies (check `pyproject.toml` for the dependency list format and add it there).

- [ ] **Step 2: Create the export service**

Create `apps/api/src/prescient/board_prep/application/export_service.py`:

```python
"""Board deck export — generate PowerPoint files from deck data."""

from __future__ import annotations

import io
import re

from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN


def generate_pptx(slides: list[dict], title: str = "Board Deck") -> bytes:
    """Generate a PowerPoint file from slide data.

    Args:
        slides: List of slide dicts with number, title, content, citations.
        title: Deck title for the title slide.

    Returns:
        The .pptx file as bytes.
    """
    prs = Presentation()
    prs.slide_width = Inches(13.333)  # 16:9
    prs.slide_height = Inches(7.5)

    # Color scheme
    header_bg = RGBColor(0x1E, 0x1B, 0x4B)  # indigo-950
    header_text_color = RGBColor(0xFF, 0xFF, 0xFF)
    body_text_color = RGBColor(0x1F, 0x2A, 0x37)  # neutral-800
    accent_color = RGBColor(0x4F, 0x46, 0xE5)  # indigo-600

    # Title slide
    slide_layout = prs.slide_layouts[6]  # Blank
    title_slide = prs.slides.add_slide(slide_layout)

    # Title slide background
    bg = title_slide.background
    fill = bg.fill
    fill.solid()
    fill.fore_color.rgb = header_bg

    # Title text
    txBox = title_slide.shapes.add_textbox(
        Inches(1), Inches(2.5), Inches(11), Inches(2)
    )
    tf = txBox.text_frame
    tf.word_wrap = True
    p = tf.paragraphs[0]
    p.text = title
    p.font.size = Pt(44)
    p.font.color.rgb = header_text_color
    p.font.bold = True
    p.alignment = PP_ALIGN.CENTER

    # Content slides
    for slide_data in slides:
        slide = prs.slides.add_slide(slide_layout)

        # Header bar
        header_shape = slide.shapes.add_shape(
            1,  # Rectangle
            Emu(0), Emu(0),
            prs.slide_width, Inches(1.2),
        )
        header_shape.fill.solid()
        header_shape.fill.fore_color.rgb = header_bg
        header_shape.line.fill.background()

        # Slide number + title in header
        header_text = slide.shapes.add_textbox(
            Inches(0.5), Inches(0.2), Inches(12), Inches(0.8)
        )
        htf = header_text.text_frame
        htf.word_wrap = True
        hp = htf.paragraphs[0]
        hp.text = f"{slide_data['number']}. {slide_data['title']}"
        hp.font.size = Pt(28)
        hp.font.color.rgb = header_text_color
        hp.font.bold = True

        # Content area
        content_box = slide.shapes.add_textbox(
            Inches(0.75), Inches(1.6), Inches(11.5), Inches(5.5)
        )
        ctf = content_box.text_frame
        ctf.word_wrap = True

        # Parse markdown content into paragraphs
        content = slide_data.get("content", "")
        # Strip citation markers for cleaner PPTX
        content = re.sub(r"\[\[\d+\]\]", "", content)

        for line in content.split("\n"):
            line = line.strip()
            if not line:
                continue

            p = ctf.add_paragraph()

            # Handle bullet points
            if line.startswith("- ") or line.startswith("* "):
                line = line[2:]
                p.level = 0
                p.space_before = Pt(4)

            # Handle bold markers
            clean_line = re.sub(r"\*\*(.+?)\*\*", r"\1", line)
            # Handle headers
            if line.startswith("###"):
                clean_line = clean_line.lstrip("# ").strip()
                p.font.size = Pt(16)
                p.font.bold = True
                p.font.color.rgb = accent_color
                p.space_before = Pt(12)
            elif line.startswith("##"):
                clean_line = clean_line.lstrip("# ").strip()
                p.font.size = Pt(18)
                p.font.bold = True
                p.font.color.rgb = accent_color
                p.space_before = Pt(14)
            else:
                p.font.size = Pt(14)
                p.font.color.rgb = body_text_color

            p.text = clean_line

    # Write to bytes
    buffer = io.BytesIO()
    prs.save(buffer)
    buffer.seek(0)
    return buffer.read()
```

- [ ] **Step 3: Add export endpoint to routes**

Add this import at the top of `routes.py`:

```python
from fastapi.responses import Response
from prescient.board_prep.application.export_service import generate_pptx
```

Add this route handler:

```python
@router.get("/{deck_id}/export/{format}")
async def export_deck(
    deck_id: UUID, format: str, ctx: CtxDep
) -> Response:
    """Export deck as PPTX."""
    if format not in ("pptx",):
        raise HTTPException(status_code=400, detail=f"Unsupported export format: {format}")

    deck, _ = await _get_deck_or_404(ctx, deck_id)
    slides = deck.slides_to_json()

    title = f"Board Deck — {deck.company_slug} — {deck.quarter}"
    pptx_bytes = generate_pptx(slides, title=title)

    filename = f"board-deck-{deck.company_slug}-{deck.quarter}.pptx"
    return Response(
        content=pptx_bytes,
        media_type="application/vnd.openxmlformats-officedocument.presentationml.presentation",
        headers={"Content-Disposition": f'attachment; filename="{filename}"'},
    )
```

- [ ] **Step 4: Verify import**

```bash
cd apps/api && python -c "from prescient.board_prep.application.export_service import generate_pptx; print('OK')"
```

Expected: `OK`

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/board_prep/application/export_service.py \
  apps/api/src/prescient/board_prep/api/routes.py \
  apps/api/pyproject.toml
git commit -m "feat(board-prep): add PPTX export endpoint"
```

---

## Task 6: Frontend Zustand Store

**Files:**
- Create: `apps/web/src/stores/deck-store.ts`

- [ ] **Step 1: Install @dnd-kit packages for drag-and-drop**

```bash
cd apps/web && npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```

- [ ] **Step 2: Create the deck store**

Create `apps/web/src/stores/deck-store.ts`:

```typescript
import { create } from "zustand";

/* -------------------------------------------------------------------------- */
/*  Types                                                                     */
/* -------------------------------------------------------------------------- */

export interface Slide {
  number: number;
  title: string;
  content: string;
  citations: Array<{ marker?: number; source_type?: string; excerpt?: string }>;
}

export interface ChatMessage {
  role: "user" | "assistant";
  content: string;
  mutations?: Array<Record<string, unknown>>;
}

interface DeckState {
  /* Deck metadata */
  deckId: string | null;
  companySlug: string | null;
  quarter: string;
  status: "draft" | "approved" | "shared";
  generatedAt: string | null;

  /* Slides */
  slides: Slide[];
  activeSlideIndex: number;

  /* Undo/redo */
  history: Slide[][];
  historyIndex: number;

  /* UI state */
  isGenerating: boolean;
  isSaving: boolean;
  isDirty: boolean;
  error: string | null;
  aiPanelOpen: boolean;

  /* Chat */
  chatMessages: ChatMessage[];

  /* Actions */
  setDeck: (deck: {
    id: string;
    company_slug: string;
    quarter: string;
    slides: Slide[];
    status: string;
    generated_at: string;
  }) => void;
  setSlides: (slides: Slide[]) => void;
  setActiveSlideIndex: (index: number) => void;
  updateSlideContent: (index: number, content: string) => void;
  updateSlideTitle: (index: number, title: string) => void;
  addSlide: (afterIndex: number, title?: string, content?: string) => void;
  deleteSlide: (index: number) => void;
  reorderSlides: (fromIndex: number, toIndex: number) => void;
  undo: () => void;
  redo: () => void;
  setIsGenerating: (v: boolean) => void;
  setIsSaving: (v: boolean) => void;
  setError: (e: string | null) => void;
  setAiPanelOpen: (v: boolean) => void;
  addChatMessage: (msg: ChatMessage) => void;
  clearChat: () => void;
  setStatus: (s: "draft" | "approved" | "shared") => void;
  setCompanySlug: (slug: string | null) => void;
  setQuarter: (q: string) => void;
  reset: () => void;
}

/* -------------------------------------------------------------------------- */
/*  Helpers                                                                   */
/* -------------------------------------------------------------------------- */

function renumber(slides: Slide[]): Slide[] {
  return slides.map((s, i) => ({ ...s, number: i + 1 }));
}

function pushHistory(state: DeckState): Pick<DeckState, "history" | "historyIndex" | "isDirty"> {
  const newHistory = state.history.slice(0, state.historyIndex + 1);
  newHistory.push(state.slides.map((s) => ({ ...s })));
  return {
    history: newHistory.slice(-50), // Keep last 50 states
    historyIndex: Math.min(newHistory.length - 1, 49),
    isDirty: true,
  };
}

/* -------------------------------------------------------------------------- */
/*  Store                                                                     */
/* -------------------------------------------------------------------------- */

export const useDeckStore = create<DeckState>((set) => ({
  deckId: null,
  companySlug: null,
  quarter: "",
  status: "draft",
  generatedAt: null,

  slides: [],
  activeSlideIndex: 0,

  history: [],
  historyIndex: -1,

  isGenerating: false,
  isSaving: false,
  isDirty: false,
  error: null,
  aiPanelOpen: true,

  chatMessages: [],

  setDeck: (deck) =>
    set({
      deckId: deck.id,
      companySlug: deck.company_slug,
      quarter: deck.quarter,
      slides: deck.slides,
      status: deck.status as DeckState["status"],
      generatedAt: deck.generated_at,
      activeSlideIndex: 0,
      isDirty: false,
      error: null,
      history: [deck.slides.map((s) => ({ ...s }))],
      historyIndex: 0,
    }),

  setSlides: (slides) =>
    set((state) => ({
      slides,
      ...pushHistory(state),
    })),

  setActiveSlideIndex: (index) => set({ activeSlideIndex: index }),

  updateSlideContent: (index, content) =>
    set((state) => {
      const slides = state.slides.map((s, i) =>
        i === index ? { ...s, content } : s
      );
      return { slides, ...pushHistory(state) };
    }),

  updateSlideTitle: (index, title) =>
    set((state) => {
      const slides = state.slides.map((s, i) =>
        i === index ? { ...s, title } : s
      );
      return { slides, ...pushHistory(state) };
    }),

  addSlide: (afterIndex, title = "New Slide", content = "") =>
    set((state) => {
      const newSlide: Slide = { number: 0, title, content, citations: [] };
      const slides = [...state.slides];
      slides.splice(afterIndex + 1, 0, newSlide);
      return {
        slides: renumber(slides),
        activeSlideIndex: afterIndex + 1,
        ...pushHistory(state),
      };
    }),

  deleteSlide: (index) =>
    set((state) => {
      if (state.slides.length <= 1) return state;
      const slides = renumber(state.slides.filter((_, i) => i !== index));
      const newIndex = Math.min(state.activeSlideIndex, slides.length - 1);
      return {
        slides,
        activeSlideIndex: newIndex,
        ...pushHistory(state),
      };
    }),

  reorderSlides: (fromIndex, toIndex) =>
    set((state) => {
      const slides = [...state.slides];
      const [moved] = slides.splice(fromIndex, 1);
      slides.splice(toIndex, 0, moved);
      return {
        slides: renumber(slides),
        activeSlideIndex: toIndex,
        ...pushHistory(state),
      };
    }),

  undo: () =>
    set((state) => {
      if (state.historyIndex <= 0) return state;
      const newIndex = state.historyIndex - 1;
      return {
        slides: state.history[newIndex].map((s) => ({ ...s })),
        historyIndex: newIndex,
        isDirty: true,
      };
    }),

  redo: () =>
    set((state) => {
      if (state.historyIndex >= state.history.length - 1) return state;
      const newIndex = state.historyIndex + 1;
      return {
        slides: state.history[newIndex].map((s) => ({ ...s })),
        historyIndex: newIndex,
        isDirty: true,
      };
    }),

  setIsGenerating: (v) => set({ isGenerating: v }),
  setIsSaving: (v) => set({ isSaving: v }),
  setError: (e) => set({ error: e }),
  setAiPanelOpen: (v) => set({ aiPanelOpen: v }),
  addChatMessage: (msg) =>
    set((state) => ({ chatMessages: [...state.chatMessages, msg] })),
  clearChat: () => set({ chatMessages: [] }),
  setStatus: (s) => set({ status: s }),
  setCompanySlug: (slug) => set({ companySlug: slug }),
  setQuarter: (q) => set({ quarter: q }),
  reset: () =>
    set({
      deckId: null,
      slides: [],
      activeSlideIndex: 0,
      history: [],
      historyIndex: -1,
      isGenerating: false,
      isSaving: false,
      isDirty: false,
      error: null,
      chatMessages: [],
      status: "draft",
      generatedAt: null,
    }),
}));
```

- [ ] **Step 3: Verify it compiles**

```bash
cd apps/web && npx tsc --noEmit src/stores/deck-store.ts 2>&1 | head -20
```

Expected: No errors (or only unrelated existing errors).

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/stores/deck-store.ts apps/web/package.json apps/web/package-lock.json
git commit -m "feat(board-prep): add Zustand deck store with undo/redo"
```

---

## Task 7: Slide Canvas Components

**Files:**
- Create: `apps/web/src/components/board-prep/slide-header.tsx`
- Create: `apps/web/src/components/board-prep/slide-content.tsx`
- Create: `apps/web/src/components/board-prep/slide-canvas.tsx`

- [ ] **Step 1: Create slide header component**

Create `apps/web/src/components/board-prep/slide-header.tsx`:

```tsx
"use client";

import { useCallback, useRef, useState } from "react";

interface SlideHeaderProps {
  number: number;
  title: string;
  onTitleChange: (title: string) => void;
}

export function SlideHeader({ number, title, onTitleChange }: SlideHeaderProps) {
  const [isEditing, setIsEditing] = useState(false);
  const [draft, setDraft] = useState(title);
  const inputRef = useRef<HTMLInputElement>(null);

  const handleDoubleClick = useCallback(() => {
    setDraft(title);
    setIsEditing(true);
    setTimeout(() => inputRef.current?.focus(), 0);
  }, [title]);

  const handleBlur = useCallback(() => {
    setIsEditing(false);
    if (draft.trim() && draft !== title) {
      onTitleChange(draft.trim());
    }
  }, [draft, title, onTitleChange]);

  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      if (e.key === "Enter") {
        (e.target as HTMLInputElement).blur();
      } else if (e.key === "Escape") {
        setDraft(title);
        setIsEditing(false);
      }
    },
    [title]
  );

  return (
    <div className="flex items-center gap-3 rounded-t-lg bg-indigo-950 px-6 py-4">
      <span className="flex h-8 w-8 shrink-0 items-center justify-center rounded-full bg-indigo-600 text-sm font-bold text-white">
        {number}
      </span>
      {isEditing ? (
        <input
          ref={inputRef}
          value={draft}
          onChange={(e) => setDraft(e.target.value)}
          onBlur={handleBlur}
          onKeyDown={handleKeyDown}
          className="flex-1 rounded bg-indigo-900 px-2 py-1 text-lg font-semibold text-white outline-none ring-1 ring-indigo-400"
        />
      ) : (
        <h2
          onDoubleClick={handleDoubleClick}
          className="flex-1 cursor-text text-lg font-semibold text-white"
          title="Double-click to edit"
        >
          {title}
        </h2>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Create slide content component**

Create `apps/web/src/components/board-prep/slide-content.tsx`:

```tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";

import { CitationPill } from "@/components/citation-pill";

interface SlideContentProps {
  content: string;
  citations: Array<{ marker?: number; source_type?: string; excerpt?: string }>;
  onContentChange: (content: string) => void;
  onTextSelect?: (text: string, rect: DOMRect) => void;
}

function renderContentWithCitations(
  content: string,
  citations: SlideContentProps["citations"]
) {
  const parts = content.split(/(\[\[\d+\]\])/g);
  return parts.map((part, i) => {
    const match = part.match(/^\[\[(\d+)\]\]$/);
    if (match) {
      const marker = parseInt(match[1], 10);
      const citation = citations.find((c) => c.marker === marker);
      return (
        <CitationPill
          key={i}
          marker={marker}
          excerpt={citation?.excerpt}
        />
      );
    }
    return <span key={i}>{part}</span>;
  });
}

export function SlideContent({
  content,
  citations,
  onContentChange,
  onTextSelect,
}: SlideContentProps) {
  const [isEditing, setIsEditing] = useState(false);
  const [draft, setDraft] = useState(content);
  const textareaRef = useRef<HTMLTextAreaElement>(null);
  const contentRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    setDraft(content);
  }, [content]);

  const handleDoubleClick = useCallback(() => {
    setDraft(content);
    setIsEditing(true);
    setTimeout(() => {
      textareaRef.current?.focus();
      autoResize();
    }, 0);
  }, [content]);

  const handleBlur = useCallback(() => {
    setIsEditing(false);
    if (draft !== content) {
      onContentChange(draft);
    }
  }, [draft, content, onContentChange]);

  const autoResize = useCallback(() => {
    const el = textareaRef.current;
    if (el) {
      el.style.height = "auto";
      el.style.height = el.scrollHeight + "px";
    }
  }, []);

  const handleMouseUp = useCallback(() => {
    if (isEditing || !onTextSelect) return;
    const selection = window.getSelection();
    if (selection && selection.toString().trim().length > 0) {
      const range = selection.getRangeAt(0);
      const rect = range.getBoundingClientRect();
      onTextSelect(selection.toString(), rect);
    }
  }, [isEditing, onTextSelect]);

  if (isEditing) {
    return (
      <div className="p-6">
        <textarea
          ref={textareaRef}
          value={draft}
          onChange={(e) => {
            setDraft(e.target.value);
            autoResize();
          }}
          onBlur={handleBlur}
          onKeyDown={(e) => {
            if (e.key === "Escape") {
              setDraft(content);
              setIsEditing(false);
            }
          }}
          className="w-full resize-none rounded border border-indigo-200 bg-white p-3 text-sm leading-relaxed text-neutral-800 outline-none ring-1 ring-indigo-300 focus:ring-2 focus:ring-indigo-500"
          style={{ minHeight: "200px" }}
        />
      </div>
    );
  }

  return (
    <div
      ref={contentRef}
      onDoubleClick={handleDoubleClick}
      onMouseUp={handleMouseUp}
      className="cursor-text p-6 text-sm leading-relaxed text-neutral-800"
      title="Double-click to edit"
    >
      <div className="prose prose-sm max-w-none prose-headings:text-indigo-900 prose-strong:text-neutral-900">
        {content.split("\n").map((line, i) => {
          if (!line.trim()) return <br key={i} />;
          const isBullet = line.trimStart().startsWith("- ") || line.trimStart().startsWith("* ");
          const isHeader = line.trimStart().startsWith("#");

          if (isHeader) {
            const level = line.match(/^#+/)?.[0].length ?? 1;
            const text = line.replace(/^#+\s*/, "");
            const Tag = level <= 2 ? "h3" : "h4";
            return (
              <Tag key={i} className="mt-3 mb-1 font-semibold text-indigo-900">
                {renderContentWithCitations(text, citations)}
              </Tag>
            );
          }

          if (isBullet) {
            const text = line.replace(/^\s*[-*]\s*/, "");
            return (
              <div key={i} className="ml-4 flex gap-2">
                <span className="mt-1 text-indigo-400">&#x2022;</span>
                <span>{renderContentWithCitations(text, citations)}</span>
              </div>
            );
          }

          return (
            <p key={i} className="mb-2">
              {renderContentWithCitations(line, citations)}
            </p>
          );
        })}
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Create slide canvas component**

Create `apps/web/src/components/board-prep/slide-canvas.tsx`:

```tsx
"use client";

import { useCallback, useState } from "react";

import { useDeckStore } from "@/stores/deck-store";
import { SlideHeader } from "./slide-header";
import { SlideContent } from "./slide-content";
import { InlineActionToolbar } from "./inline-action-toolbar";

export function SlideCanvas() {
  const slides = useDeckStore((s) => s.slides);
  const activeSlideIndex = useDeckStore((s) => s.activeSlideIndex);
  const updateSlideContent = useDeckStore((s) => s.updateSlideContent);
  const updateSlideTitle = useDeckStore((s) => s.updateSlideTitle);

  const [selectionInfo, setSelectionInfo] = useState<{
    text: string;
    rect: DOMRect;
  } | null>(null);

  const activeSlide = slides[activeSlideIndex];

  const handleTextSelect = useCallback((text: string, rect: DOMRect) => {
    setSelectionInfo({ text, rect });
  }, []);

  const handleDismissToolbar = useCallback(() => {
    setSelectionInfo(null);
  }, []);

  if (!activeSlide) {
    return (
      <div className="flex flex-1 items-center justify-center text-sm text-neutral-400">
        No slides to display. Generate a deck to get started.
      </div>
    );
  }

  return (
    <div className="flex flex-1 items-center justify-center bg-neutral-100 p-8">
      {/* 16:9 slide container */}
      <div className="relative w-full max-w-4xl" style={{ aspectRatio: "16/9" }}>
        <div className="absolute inset-0 flex flex-col overflow-hidden rounded-lg bg-white shadow-xl">
          <SlideHeader
            number={activeSlide.number}
            title={activeSlide.title}
            onTitleChange={(title) => updateSlideTitle(activeSlideIndex, title)}
          />
          <div className="flex-1 overflow-y-auto">
            <SlideContent
              content={activeSlide.content}
              citations={activeSlide.citations}
              onContentChange={(content) =>
                updateSlideContent(activeSlideIndex, content)
              }
              onTextSelect={handleTextSelect}
            />
          </div>
          {/* Slide counter */}
          <div className="border-t border-neutral-100 px-6 py-2 text-right text-xs text-neutral-400">
            {activeSlide.number} / {slides.length}
          </div>
        </div>

        {/* Inline action toolbar */}
        {selectionInfo && (
          <InlineActionToolbar
            selectedText={selectionInfo.text}
            rect={selectionInfo.rect}
            slideNumber={activeSlide.number}
            onDismiss={handleDismissToolbar}
          />
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/components/board-prep/slide-header.tsx \
  apps/web/src/components/board-prep/slide-content.tsx \
  apps/web/src/components/board-prep/slide-canvas.tsx
git commit -m "feat(board-prep): add slide canvas with WYSIWYG editing"
```

---

## Task 8: Slide Filmstrip Component

**Files:**
- Create: `apps/web/src/components/board-prep/slide-filmstrip.tsx`

- [ ] **Step 1: Create the filmstrip component**

Create `apps/web/src/components/board-prep/slide-filmstrip.tsx`:

```tsx
"use client";

import { useCallback } from "react";
import {
  DndContext,
  closestCenter,
  PointerSensor,
  useSensor,
  useSensors,
  type DragEndEvent,
} from "@dnd-kit/core";
import {
  SortableContext,
  useSortable,
  verticalListSortingStrategy,
} from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import { Plus, Trash2, GripVertical } from "lucide-react";

import { useDeckStore, type Slide } from "@/stores/deck-store";

/* -------------------------------------------------------------------------- */
/*  Sortable thumbnail                                                        */
/* -------------------------------------------------------------------------- */

function SlideThumbnail({
  slide,
  index,
  isActive,
  onClick,
  onDelete,
}: {
  slide: Slide;
  index: number;
  isActive: boolean;
  onClick: () => void;
  onDelete: () => void;
}) {
  const { attributes, listeners, setNodeRef, transform, transition } =
    useSortable({ id: slide.number });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
  };

  return (
    <div
      ref={setNodeRef}
      style={style}
      className={`group relative cursor-pointer rounded-md border-2 p-2 transition-colors ${
        isActive
          ? "border-indigo-500 bg-indigo-50"
          : "border-transparent bg-white hover:border-neutral-300"
      }`}
      onClick={onClick}
    >
      {/* Drag handle */}
      <div
        {...attributes}
        {...listeners}
        className="absolute left-0 top-1/2 -translate-y-1/2 cursor-grab opacity-0 group-hover:opacity-100"
      >
        <GripVertical className="h-4 w-4 text-neutral-400" />
      </div>

      {/* Thumbnail preview */}
      <div className="ml-4">
        <div className="flex items-center gap-1.5">
          <span className="flex h-5 w-5 shrink-0 items-center justify-center rounded-full bg-indigo-950 text-[10px] font-bold text-white">
            {slide.number}
          </span>
          <span className="truncate text-xs font-medium text-neutral-700">
            {slide.title}
          </span>
        </div>
        <p className="mt-1 line-clamp-2 text-[10px] leading-tight text-neutral-400">
          {slide.content.slice(0, 80)}
        </p>
      </div>

      {/* Delete button */}
      <button
        onClick={(e) => {
          e.stopPropagation();
          onDelete();
        }}
        className="absolute right-1 top-1 rounded p-0.5 opacity-0 transition-opacity hover:bg-red-50 group-hover:opacity-100"
        title="Delete slide"
      >
        <Trash2 className="h-3 w-3 text-red-400 hover:text-red-600" />
      </button>
    </div>
  );
}

/* -------------------------------------------------------------------------- */
/*  Filmstrip                                                                 */
/* -------------------------------------------------------------------------- */

export function SlideFilmstrip() {
  const slides = useDeckStore((s) => s.slides);
  const activeSlideIndex = useDeckStore((s) => s.activeSlideIndex);
  const setActiveSlideIndex = useDeckStore((s) => s.setActiveSlideIndex);
  const reorderSlides = useDeckStore((s) => s.reorderSlides);
  const deleteSlide = useDeckStore((s) => s.deleteSlide);
  const addSlide = useDeckStore((s) => s.addSlide);

  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } })
  );

  const handleDragEnd = useCallback(
    (event: DragEndEvent) => {
      const { active, over } = event;
      if (!over || active.id === over.id) return;

      const fromIndex = slides.findIndex((s) => s.number === active.id);
      const toIndex = slides.findIndex((s) => s.number === over.id);
      if (fromIndex !== -1 && toIndex !== -1) {
        reorderSlides(fromIndex, toIndex);
      }
    },
    [slides, reorderSlides]
  );

  return (
    <div className="flex w-48 flex-col border-r border-neutral-200 bg-neutral-50">
      <div className="border-b border-neutral-200 px-3 py-2">
        <span className="text-xs font-medium uppercase tracking-wider text-neutral-500">
          Slides
        </span>
      </div>

      <div className="flex-1 space-y-1 overflow-y-auto p-2">
        <DndContext
          sensors={sensors}
          collisionDetection={closestCenter}
          onDragEnd={handleDragEnd}
        >
          <SortableContext
            items={slides.map((s) => s.number)}
            strategy={verticalListSortingStrategy}
          >
            {slides.map((slide, index) => (
              <SlideThumbnail
                key={slide.number}
                slide={slide}
                index={index}
                isActive={index === activeSlideIndex}
                onClick={() => setActiveSlideIndex(index)}
                onDelete={() => deleteSlide(index)}
              />
            ))}
          </SortableContext>
        </DndContext>
      </div>

      <div className="border-t border-neutral-200 p-2">
        <button
          onClick={() => addSlide(slides.length - 1)}
          className="flex w-full items-center justify-center gap-1.5 rounded-md border border-dashed border-neutral-300 px-3 py-2 text-xs font-medium text-neutral-500 transition-colors hover:border-indigo-400 hover:text-indigo-600"
        >
          <Plus className="h-3.5 w-3.5" />
          Add Slide
        </button>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/board-prep/slide-filmstrip.tsx
git commit -m "feat(board-prep): add slide filmstrip with drag-to-reorder"
```

---

## Task 9: Inline Action Toolbar Component

**Files:**
- Create: `apps/web/src/components/board-prep/inline-action-toolbar.tsx`

- [ ] **Step 1: Create the inline action toolbar**

Create `apps/web/src/components/board-prep/inline-action-toolbar.tsx`:

```tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";
import { Wand2 } from "lucide-react";

import { apiFetch } from "@/lib/api-client";
import { useDeckStore } from "@/stores/deck-store";

interface InlineActionToolbarProps {
  selectedText: string;
  rect: DOMRect;
  slideNumber: number;
  onDismiss: () => void;
}

const ACTIONS = [
  { key: "rewrite", label: "Rewrite" },
  { key: "concise", label: "Concise" },
  { key: "expand", label: "Expand" },
  { key: "tone_formal", label: "Formal" },
  { key: "tone_conversational", label: "Casual" },
  { key: "tone_executive", label: "Executive" },
] as const;

export function InlineActionToolbar({
  selectedText,
  rect,
  slideNumber,
  onDismiss,
}: InlineActionToolbarProps) {
  const deckId = useDeckStore((s) => s.deckId);
  const slides = useDeckStore((s) => s.slides);
  const activeSlideIndex = useDeckStore((s) => s.activeSlideIndex);
  const updateSlideContent = useDeckStore((s) => s.updateSlideContent);

  const [isLoading, setIsLoading] = useState(false);
  const toolbarRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    function handleClickOutside(e: MouseEvent) {
      if (
        toolbarRef.current &&
        !toolbarRef.current.contains(e.target as Node)
      ) {
        onDismiss();
      }
    }
    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, [onDismiss]);

  const handleAction = useCallback(
    async (action: string) => {
      if (!deckId) return;
      setIsLoading(true);
      try {
        const result = await apiFetch<{ replacement_text: string }>(
          `/board-prep/${deckId}/inline-action`,
          {
            method: "POST",
            body: JSON.stringify({
              selected_text: selectedText,
              action,
              slide_number: slideNumber,
            }),
          }
        );

        // Replace the selected text in the slide content
        const slide = slides[activeSlideIndex];
        if (slide) {
          const newContent = slide.content.replace(selectedText, result.replacement_text);
          updateSlideContent(activeSlideIndex, newContent);
        }
        onDismiss();
      } catch {
        // Silently fail — user can try again
      } finally {
        setIsLoading(false);
      }
    },
    [deckId, selectedText, slideNumber, slides, activeSlideIndex, updateSlideContent, onDismiss]
  );

  // Position the toolbar above the selection
  const top = rect.top - 48;
  const left = rect.left + rect.width / 2;

  return (
    <div
      ref={toolbarRef}
      className="fixed z-50 -translate-x-1/2 rounded-lg border border-neutral-200 bg-white px-1 py-1 shadow-lg"
      style={{ top: `${top}px`, left: `${left}px` }}
    >
      {isLoading ? (
        <div className="flex items-center gap-2 px-3 py-1">
          <Wand2 className="h-3.5 w-3.5 animate-pulse text-indigo-500" />
          <span className="text-xs text-neutral-500">Applying...</span>
        </div>
      ) : (
        <div className="flex items-center gap-0.5">
          <Wand2 className="mx-1 h-3.5 w-3.5 text-indigo-500" />
          {ACTIONS.map((action) => (
            <button
              key={action.key}
              onClick={() => handleAction(action.key)}
              className="rounded px-2 py-1 text-xs font-medium text-neutral-600 transition-colors hover:bg-indigo-50 hover:text-indigo-700"
            >
              {action.label}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/board-prep/inline-action-toolbar.tsx
git commit -m "feat(board-prep): add inline AI action toolbar"
```

---

## Task 10: AI Assistant Panel Component

**Files:**
- Create: `apps/web/src/components/board-prep/ai-panel.tsx`

- [ ] **Step 1: Create the AI panel**

Create `apps/web/src/components/board-prep/ai-panel.tsx`:

```tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";
import { MessageSquare, Send, X } from "lucide-react";

import { apiFetch } from "@/lib/api-client";
import { useDeckStore } from "@/stores/deck-store";

export function AiPanel() {
  const deckId = useDeckStore((s) => s.deckId);
  const slides = useDeckStore((s) => s.slides);
  const activeSlideIndex = useDeckStore((s) => s.activeSlideIndex);
  const aiPanelOpen = useDeckStore((s) => s.aiPanelOpen);
  const setAiPanelOpen = useDeckStore((s) => s.setAiPanelOpen);
  const chatMessages = useDeckStore((s) => s.chatMessages);
  const addChatMessage = useDeckStore((s) => s.addChatMessage);
  const setSlides = useDeckStore((s) => s.setSlides);

  const [input, setInput] = useState("");
  const [isLoading, setIsLoading] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const textareaRef = useRef<HTMLTextAreaElement>(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [chatMessages]);

  const handleSend = useCallback(async () => {
    const message = input.trim();
    if (!message || !deckId || isLoading) return;

    setInput("");
    addChatMessage({ role: "user", content: message });
    setIsLoading(true);

    try {
      const activeSlide = slides[activeSlideIndex];
      const conversationHistory = chatMessages.map((m) => ({
        role: m.role,
        content: m.content,
      }));

      const result = await apiFetch<{
        response: string;
        mutations: Array<Record<string, unknown>>;
      }>(`/board-prep/${deckId}/chat`, {
        method: "POST",
        body: JSON.stringify({
          message,
          active_slide_number: activeSlide?.number ?? null,
          conversation_history: conversationHistory,
        }),
      });

      addChatMessage({
        role: "assistant",
        content: result.response,
        mutations: result.mutations,
      });

      // If there were mutations, refetch the deck to get updated slides
      if (result.mutations.length > 0) {
        const updatedDeck = await apiFetch<{
          slides: Array<{
            number: number;
            title: string;
            content: string;
            citations: Array<{ marker?: number; source_type?: string; excerpt?: string }>;
          }>;
        }>(`/board-prep/${deckId}`);
        setSlides(updatedDeck.slides);
      }
    } catch {
      addChatMessage({
        role: "assistant",
        content: "Sorry, I encountered an error. Please try again.",
      });
    } finally {
      setIsLoading(false);
    }
  }, [input, deckId, isLoading, slides, activeSlideIndex, chatMessages, addChatMessage, setSlides]);

  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      if (e.key === "Enter" && (e.metaKey || e.ctrlKey)) {
        e.preventDefault();
        handleSend();
      }
    },
    [handleSend]
  );

  if (!aiPanelOpen) {
    return (
      <button
        onClick={() => setAiPanelOpen(true)}
        className="fixed right-4 top-20 z-40 rounded-full bg-indigo-600 p-3 text-white shadow-lg transition-colors hover:bg-indigo-700"
        title="Open AI Assistant"
      >
        <MessageSquare className="h-5 w-5" />
      </button>
    );
  }

  const activeSlide = slides[activeSlideIndex];

  return (
    <div className="flex w-[350px] flex-col border-l border-neutral-200 bg-white">
      {/* Header */}
      <div className="flex items-center justify-between border-b border-neutral-200 px-4 py-3">
        <div className="flex items-center gap-2">
          <MessageSquare className="h-4 w-4 text-indigo-600" />
          <span className="text-sm font-medium text-neutral-900">
            AI Assistant
          </span>
        </div>
        <button
          onClick={() => setAiPanelOpen(false)}
          className="rounded p-1 transition-colors hover:bg-neutral-100"
        >
          <X className="h-4 w-4 text-neutral-400" />
        </button>
      </div>

      {/* Context indicator */}
      {activeSlide && (
        <div className="border-b border-neutral-100 bg-indigo-50 px-4 py-2">
          <span className="text-xs text-indigo-600">
            Viewing: Slide {activeSlide.number} — {activeSlide.title}
          </span>
        </div>
      )}

      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4">
        {chatMessages.length === 0 ? (
          <div className="py-8 text-center">
            <MessageSquare className="mx-auto mb-2 h-8 w-8 text-neutral-300" />
            <p className="text-sm text-neutral-400">
              Ask me to edit slides, add content, or analyze your deck.
            </p>
          </div>
        ) : (
          <div className="space-y-4">
            {chatMessages.map((msg, i) => (
              <div
                key={i}
                className={`flex ${
                  msg.role === "user" ? "justify-end" : "justify-start"
                }`}
              >
                <div
                  className={`max-w-[85%] rounded-lg px-3 py-2 text-sm ${
                    msg.role === "user"
                      ? "bg-indigo-600 text-white"
                      : "bg-neutral-100 text-neutral-800"
                  }`}
                >
                  <p className="whitespace-pre-wrap">{msg.content}</p>
                  {msg.mutations && msg.mutations.length > 0 && (
                    <p className="mt-1 text-xs opacity-70">
                      Applied {msg.mutations.length} change
                      {msg.mutations.length > 1 ? "s" : ""} to deck
                    </p>
                  )}
                </div>
              </div>
            ))}
            {isLoading && (
              <div className="flex justify-start">
                <div className="rounded-lg bg-neutral-100 px-3 py-2">
                  <div className="flex gap-1">
                    <span className="h-2 w-2 animate-bounce rounded-full bg-neutral-400" />
                    <span className="h-2 w-2 animate-bounce rounded-full bg-neutral-400 [animation-delay:0.1s]" />
                    <span className="h-2 w-2 animate-bounce rounded-full bg-neutral-400 [animation-delay:0.2s]" />
                  </div>
                </div>
              </div>
            )}
            <div ref={messagesEndRef} />
          </div>
        )}
      </div>

      {/* Input */}
      <div className="border-t border-neutral-200 p-3">
        <div className="flex gap-2">
          <textarea
            ref={textareaRef}
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={handleKeyDown}
            placeholder="Ask about your deck..."
            rows={2}
            className="flex-1 resize-none rounded-md border border-neutral-200 px-3 py-2 text-sm outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500"
          />
          <button
            onClick={handleSend}
            disabled={!input.trim() || isLoading}
            className="self-end rounded-md bg-indigo-600 p-2 text-white transition-colors hover:bg-indigo-700 disabled:cursor-not-allowed disabled:opacity-50"
          >
            <Send className="h-4 w-4" />
          </button>
        </div>
        <p className="mt-1 text-[10px] text-neutral-400">
          {"\u2318"}+Enter to send
        </p>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/board-prep/ai-panel.tsx
git commit -m "feat(board-prep): add AI assistant chat panel"
```

---

## Task 11: Deck Toolbar & Export Dropdown

**Files:**
- Create: `apps/web/src/components/board-prep/deck-toolbar.tsx`
- Create: `apps/web/src/components/board-prep/export-dropdown.tsx`

- [ ] **Step 1: Install jspdf and html2canvas for PDF export**

```bash
cd apps/web && npm install jspdf html2canvas
```

- [ ] **Step 2: Create the export dropdown**

Create `apps/web/src/components/board-prep/export-dropdown.tsx`:

```tsx
"use client";

import { useCallback, useState } from "react";
import { Download, FileText, Presentation } from "lucide-react";
import html2canvas from "html2canvas";
import jsPDF from "jspdf";

import { useDeckStore } from "@/stores/deck-store";

export function ExportDropdown() {
  const deckId = useDeckStore((s) => s.deckId);
  const slides = useDeckStore((s) => s.slides);
  const companySlug = useDeckStore((s) => s.companySlug);
  const quarter = useDeckStore((s) => s.quarter);
  const [open, setOpen] = useState(false);
  const [isExporting, setIsExporting] = useState(false);

  const handleExportPDF = useCallback(async () => {
    if (!slides.length) return;
    setIsExporting(true);
    setOpen(false);

    try {
      const pdf = new jsPDF({ orientation: "landscape", unit: "in", format: [13.333, 7.5] });
      const canvas = document.querySelector("[data-slide-canvas]") as HTMLElement;

      if (canvas) {
        const rendered = await html2canvas(canvas, { scale: 2, useCORS: true });
        const imgData = rendered.toDataURL("image/png");
        pdf.addImage(imgData, "PNG", 0, 0, 13.333, 7.5);
      }

      pdf.save(`board-deck-${companySlug}-${quarter}.pdf`);
    } finally {
      setIsExporting(false);
    }
  }, [slides, companySlug, quarter]);

  const handleExportPPTX = useCallback(async () => {
    if (!deckId) return;
    setIsExporting(true);
    setOpen(false);

    try {
      const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:8000";
      const token = localStorage.getItem("prescient_token");
      const headers: Record<string, string> = {};
      if (token) headers["Authorization"] = `Bearer ${token}`;

      const res = await fetch(`${API_BASE}/board-prep/${deckId}/export/pptx`, {
        headers,
      });

      if (!res.ok) throw new Error("Export failed");

      const blob = await res.blob();
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = `board-deck-${companySlug}-${quarter}.pptx`;
      a.click();
      URL.revokeObjectURL(url);
    } finally {
      setIsExporting(false);
    }
  }, [deckId, companySlug, quarter]);

  return (
    <div className="relative">
      <button
        onClick={() => setOpen(!open)}
        disabled={isExporting || !slides.length}
        className="flex items-center gap-1.5 rounded-md border border-neutral-200 bg-white px-3 py-2 text-sm font-medium text-neutral-700 transition-colors hover:bg-neutral-50 disabled:cursor-not-allowed disabled:opacity-50"
      >
        <Download className="h-4 w-4" />
        {isExporting ? "Exporting..." : "Export"}
      </button>

      {open && (
        <>
          <div className="fixed inset-0 z-40" onClick={() => setOpen(false)} />
          <div className="absolute right-0 top-full z-50 mt-1 w-48 rounded-md border border-neutral-200 bg-white py-1 shadow-lg">
            <button
              onClick={handleExportPDF}
              className="flex w-full items-center gap-2 px-3 py-2 text-sm text-neutral-700 hover:bg-neutral-50"
            >
              <FileText className="h-4 w-4 text-red-500" />
              Export as PDF
            </button>
            <button
              onClick={handleExportPPTX}
              className="flex w-full items-center gap-2 px-3 py-2 text-sm text-neutral-700 hover:bg-neutral-50"
            >
              <Presentation className="h-4 w-4 text-orange-500" />
              Export as PowerPoint
            </button>
          </div>
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Create the deck toolbar**

Create `apps/web/src/components/board-prep/deck-toolbar.tsx`:

```tsx
"use client";

import { useCallback, useEffect, useState } from "react";
import { Undo2, Redo2, Save, Check, Share2 } from "lucide-react";

import { apiFetch } from "@/lib/api-client";
import { useDeckStore } from "@/stores/deck-store";
import { ExportDropdown } from "./export-dropdown";
import { ShareWithSponsor } from "@/components/share-with-sponsor";

interface DeckToolbarProps {
  quarterOptions: { label: string; value: string }[];
  onGenerate: () => void;
}

export function DeckToolbar({ quarterOptions, onGenerate }: DeckToolbarProps) {
  const quarter = useDeckStore((s) => s.quarter);
  const setQuarter = useDeckStore((s) => s.setQuarter);
  const isGenerating = useDeckStore((s) => s.isGenerating);
  const slides = useDeckStore((s) => s.slides);
  const deckId = useDeckStore((s) => s.deckId);
  const isDirty = useDeckStore((s) => s.isDirty);
  const isSaving = useDeckStore((s) => s.isSaving);
  const setIsSaving = useDeckStore((s) => s.setIsSaving);
  const status = useDeckStore((s) => s.status);
  const setStatus = useDeckStore((s) => s.setStatus);
  const undo = useDeckStore((s) => s.undo);
  const redo = useDeckStore((s) => s.redo);
  const historyIndex = useDeckStore((s) => s.historyIndex);
  const history = useDeckStore((s) => s.history);

  const canUndo = historyIndex > 0;
  const canRedo = historyIndex < history.length - 1;

  const handleSave = useCallback(async () => {
    if (!deckId || !isDirty) return;
    setIsSaving(true);
    try {
      const slidesPayload = useDeckStore.getState().slides;
      await apiFetch(`/board-prep/${deckId}/slides`, {
        method: "PUT",
        body: JSON.stringify({ slides: slidesPayload }),
      });
    } finally {
      setIsSaving(false);
    }
  }, [deckId, isDirty, setIsSaving]);

  const handleApprove = useCallback(async () => {
    if (!deckId) return;
    await apiFetch(`/board-prep/${deckId}/status`, {
      method: "PUT",
      body: JSON.stringify({ status: "approved" }),
    });
    setStatus("approved");
  }, [deckId, setStatus]);

  // Keyboard shortcuts
  useEffect(() => {
    function handleKeyDown(e: KeyboardEvent) {
      if ((e.metaKey || e.ctrlKey) && e.key === "z" && !e.shiftKey) {
        e.preventDefault();
        undo();
      }
      if ((e.metaKey || e.ctrlKey) && e.key === "z" && e.shiftKey) {
        e.preventDefault();
        redo();
      }
      if ((e.metaKey || e.ctrlKey) && e.key === "s") {
        e.preventDefault();
        handleSave();
      }
    }
    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [undo, redo, handleSave]);

  return (
    <div className="flex items-center justify-between border-b border-neutral-200 bg-white px-4 py-2">
      <div className="flex items-center gap-3">
        <h1 className="text-lg font-semibold text-neutral-900">Board Prep</h1>

        <select
          value={quarter}
          onChange={(e) => setQuarter(e.target.value)}
          className="rounded-md border border-neutral-200 bg-neutral-50 px-3 py-1.5 text-sm text-neutral-700 focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
        >
          {quarterOptions.map((opt) => (
            <option key={opt.value} value={opt.value}>
              {opt.label}
            </option>
          ))}
        </select>

        <button
          onClick={onGenerate}
          disabled={isGenerating}
          className="rounded-md bg-indigo-600 px-4 py-1.5 text-sm font-medium text-white transition-colors hover:bg-indigo-700 disabled:cursor-not-allowed disabled:opacity-50"
        >
          {isGenerating ? "Generating..." : "Generate Deck"}
        </button>
      </div>

      {slides.length > 0 && (
        <div className="flex items-center gap-2">
          {/* Undo/Redo */}
          <button
            onClick={undo}
            disabled={!canUndo}
            className="rounded p-1.5 text-neutral-500 transition-colors hover:bg-neutral-100 disabled:opacity-30"
            title="Undo (Cmd+Z)"
          >
            <Undo2 className="h-4 w-4" />
          </button>
          <button
            onClick={redo}
            disabled={!canRedo}
            className="rounded p-1.5 text-neutral-500 transition-colors hover:bg-neutral-100 disabled:opacity-30"
            title="Redo (Cmd+Shift+Z)"
          >
            <Redo2 className="h-4 w-4" />
          </button>

          <div className="mx-1 h-5 w-px bg-neutral-200" />

          {/* Save */}
          <button
            onClick={handleSave}
            disabled={!isDirty || isSaving}
            className="flex items-center gap-1.5 rounded-md border border-neutral-200 px-3 py-1.5 text-sm font-medium text-neutral-700 transition-colors hover:bg-neutral-50 disabled:opacity-50"
          >
            <Save className="h-4 w-4" />
            {isSaving ? "Saving..." : "Save"}
          </button>

          <ExportDropdown />

          <div className="mx-1 h-5 w-px bg-neutral-200" />

          {/* Approve */}
          {status === "approved" ? (
            <span className="flex items-center gap-1.5 rounded-md bg-green-50 px-3 py-1.5 text-sm font-medium text-green-700">
              <Check className="h-4 w-4" />
              Approved
            </span>
          ) : (
            <button
              onClick={handleApprove}
              className="rounded-md bg-green-600 px-4 py-1.5 text-sm font-medium text-white transition-colors hover:bg-green-700"
            >
              Approve
            </button>
          )}

          <ShareWithSponsor artifactId="board-deck" />
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/components/board-prep/export-dropdown.tsx \
  apps/web/src/components/board-prep/deck-toolbar.tsx \
  apps/web/package.json apps/web/package-lock.json
git commit -m "feat(board-prep): add deck toolbar with export, undo/redo, and approve"
```

---

## Task 12: Rewrite Board Prep Page

**Files:**
- Modify: `apps/web/src/app/(main)/board-prep/page.tsx`

- [ ] **Step 1: Rewrite the page with three-zone layout**

Replace the entire contents of `apps/web/src/app/(main)/board-prep/page.tsx`:

```tsx
"use client";

import { useEffect } from "react";

import { apiFetch } from "@/lib/api-client";
import { useDeckStore, type Slide } from "@/stores/deck-store";
import { SlideFilmstrip } from "@/components/board-prep/slide-filmstrip";
import { SlideCanvas } from "@/components/board-prep/slide-canvas";
import { AiPanel } from "@/components/board-prep/ai-panel";
import { DeckToolbar } from "@/components/board-prep/deck-toolbar";

/* -------------------------------------------------------------------------- */
/*  Quarter helpers — Peloton FY ends June 30                                 */
/* -------------------------------------------------------------------------- */

function buildQuarterOptions(): { label: string; value: string }[] {
  const now = new Date();
  const month = now.getMonth();
  const year = now.getFullYear();

  let fyYear: number;
  let fyQ: number;
  if (month >= 6) {
    fyYear = year + 1;
    fyQ = month >= 9 ? 2 : 1;
  } else {
    fyYear = year;
    fyQ = month >= 3 ? 4 : 3;
  }

  const quarters: { label: string; value: string }[] = [];
  let y = fyYear;
  let q = fyQ;

  for (let i = 0; i < 4; i++) {
    const rangeLabels = new Map<number, string>([
      [1, "Jul-Sep"],
      [2, "Oct-Dec"],
      [3, "Jan-Mar"],
      [4, "Apr-Jun"],
    ]);
    quarters.push({
      label: `FY${y} Q${q} (${rangeLabels.get(q) ?? ""})`,
      value: `FY${y}-Q${q}`,
    });
    q -= 1;
    if (q === 0) {
      q = 4;
      y -= 1;
    }
  }

  return quarters;
}

/* -------------------------------------------------------------------------- */
/*  API response type                                                         */
/* -------------------------------------------------------------------------- */

interface DeckApiResponse {
  id: string;
  company_slug: string;
  quarter: string;
  slides: Slide[];
  status: string;
  generated_at: string;
}

/* -------------------------------------------------------------------------- */
/*  Page component                                                            */
/* -------------------------------------------------------------------------- */

export default function BoardPrepPage() {
  const quarterOptions = buildQuarterOptions();

  const setDeck = useDeckStore((s) => s.setDeck);
  const setCompanySlug = useDeckStore((s) => s.setCompanySlug);
  const setQuarter = useDeckStore((s) => s.setQuarter);
  const setIsGenerating = useDeckStore((s) => s.setIsGenerating);
  const setError = useDeckStore((s) => s.setError);
  const companySlug = useDeckStore((s) => s.companySlug);
  const quarter = useDeckStore((s) => s.quarter);
  const isGenerating = useDeckStore((s) => s.isGenerating);
  const error = useDeckStore((s) => s.error);
  const slides = useDeckStore((s) => s.slides);

  // Initialize quarter
  useEffect(() => {
    if (!quarter && quarterOptions.length > 0) {
      setQuarter(quarterOptions[0].value);
    }
  }, [quarter, quarterOptions, setQuarter]);

  // Resolve company slug
  useEffect(() => {
    async function resolve() {
      try {
        const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:8000";
        const token = localStorage.getItem("prescient_token");
        const orgId = localStorage.getItem("prescient_org_id") ?? "";
        const headers: Record<string, string> = {};
        if (token) headers["Authorization"] = `Bearer ${token}`;
        const res = await fetch(`${API_BASE}/companies`, { headers });
        if (res.ok) {
          const companies: { slug: string }[] = await res.json();
          const match = companies.find((c) => orgId.includes(c.slug));
          setCompanySlug(match?.slug ?? companies[0]?.slug ?? null);
        }
      } catch {
        // ignore
      }
    }
    void resolve();
  }, [setCompanySlug]);

  async function handleGenerate() {
    if (!companySlug || !quarter) return;
    setIsGenerating(true);
    setError(null);
    try {
      const data = await apiFetch<DeckApiResponse>("/board-prep/generate", {
        method: "POST",
        body: JSON.stringify({ company_slug: companySlug, quarter }),
      });
      setDeck(data);
    } catch (err) {
      if (err instanceof Error && err.message !== "Unauthorized") {
        setError(err.message);
      }
    } finally {
      setIsGenerating(false);
    }
  }

  return (
    <div className="flex h-[calc(100vh-4rem)] flex-col">
      {/* Top toolbar */}
      <DeckToolbar quarterOptions={quarterOptions} onGenerate={handleGenerate} />

      {/* Error banner */}
      {error && (
        <div className="border-b border-red-200 bg-red-50 px-4 py-2 text-sm text-red-700">
          {error}
        </div>
      )}

      {/* Main content area */}
      {slides.length === 0 ? (
        <div className="flex flex-1 flex-col items-center justify-center">
          {isGenerating ? (
            <div className="flex items-center gap-3">
              <svg
                className="h-5 w-5 animate-spin text-indigo-600"
                viewBox="0 0 24 24"
                fill="none"
              >
                <circle
                  className="opacity-25"
                  cx="12"
                  cy="12"
                  r="10"
                  stroke="currentColor"
                  strokeWidth="4"
                />
                <path
                  className="opacity-75"
                  fill="currentColor"
                  d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"
                />
              </svg>
              <span className="text-sm font-medium text-indigo-700">
                Generating board deck... This may take up to a minute.
              </span>
            </div>
          ) : (
            <div className="text-center">
              <p className="text-sm text-neutral-400">
                Select a quarter and generate your first deck.
              </p>
            </div>
          )}
        </div>
      ) : (
        <div className="flex flex-1 overflow-hidden">
          {/* Filmstrip */}
          <SlideFilmstrip />

          {/* Canvas */}
          <SlideCanvas />

          {/* AI Panel */}
          <AiPanel />
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Start the dev server and verify the page loads**

```bash
cd apps/web && npm run dev
```

Navigate to `/board-prep` in the browser. Verify:
- Toolbar renders with quarter selector and Generate button
- Empty state shows when no deck is generated
- After generating, three-zone layout appears (filmstrip | canvas | AI panel)

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/\(main\)/board-prep/page.tsx
git commit -m "feat(board-prep): rewrite page with three-zone slide editor layout"
```

---

## Task 13: Browser Smoke Test

**Files:** None (testing only)

- [ ] **Step 1: Start both servers**

```bash
# In Docker:
make docker-up
```

- [ ] **Step 2: Test the complete flow**

Navigate to `/board-prep`. Verify each feature:

1. **Generate deck** — Click Generate Deck, wait for completion, confirm slides appear
2. **Filmstrip** — Click different thumbnails, verify canvas updates
3. **Drag reorder** — Drag a thumbnail to a new position, verify renumbering
4. **Edit title** — Double-click a slide title in the header, edit, blur to save
5. **Edit content** — Double-click slide content, edit in textarea, blur to save
6. **Add slide** — Click "Add Slide" button, verify new blank slide appears
7. **Delete slide** — Hover a thumbnail, click trash icon, verify removal
8. **Undo/Redo** — Make an edit, Cmd+Z to undo, Cmd+Shift+Z to redo
9. **AI chat** — Type a message in the AI panel, verify response
10. **Inline actions** — Select text on a slide, verify toolbar appears, click an action
11. **Save** — Make edits, click Save or Cmd+S
12. **Export PDF** — Click Export > PDF, verify download
13. **Export PPTX** — Click Export > PowerPoint, verify download
14. **Approve** — Click Approve button, verify green confirmation

- [ ] **Step 3: Fix any issues found during testing**

Address any bugs discovered during the smoke test.

- [ ] **Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix(board-prep): address smoke test issues"
```
