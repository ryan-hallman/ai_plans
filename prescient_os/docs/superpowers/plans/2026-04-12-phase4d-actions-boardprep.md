# Phase 4d: Insight-to-Action & Board Prep Deck Generation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the insight-to-action flow (create action items and decisions from chat/briefing) and the board prep deck generation (10-section board package assembled from accumulated data with LLM, section-by-section operator curation).

**Architecture:** The insight-to-action flow adds UI components to chat messages and briefing items that call the existing action items/decisions API. The board prep is a new bounded context with a `BoardPrepAssembler` service that gathers data across contexts and calls the LLM per section. A new frontend page renders the generated deck with section editing.

**Tech Stack:** Python 3.12, FastAPI, Anthropic SDK, SQLAlchemy 2.0 (async), Next.js 15, React 19, Tailwind, shadcn/ui.

**Spec:** `docs/superpowers/specs/2026-04-12-phase4-demo-design.md` — Sections 5 and 6

**Depends on:** Phases 4a-4c (seed data, chat, artifact viewer)

---

## File Structure

### New Files (API)

```
apps/api/src/prescient/
├── board_prep/
│   ├── __init__.py
│   ├── domain/
│   │   ├── __init__.py
│   │   └── entities/
│   │       ├── __init__.py
│   │       └── board_deck.py           # BoardDeck, BoardSection entities
│   ├── application/
│   │   ├── __init__.py
│   │   ├── assembler.py                # BoardPrepAssembler — gathers data per section
│   │   ├── section_templates.py        # Prompt templates for each of 10 sections
│   │   └── use_cases/
│   │       ├── __init__.py
│   │       └── generate_deck.py        # Orchestrates full deck generation
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   └── tables/
│   │       ├── __init__.py
│   │       └── board_deck.py           # Persisted deck snapshots
│   └── api/
│       ├── __init__.py
│       └── routes.py                   # POST /board-prep/generate, GET /board-prep/{id}
```

### New Files (Frontend)

```
apps/web/src/
├── app/(main)/
│   ├── board-prep/
│   │   └── page.tsx                    # Board prep generation + review page
├── components/
│   ├── action-bar.tsx                  # "Create action item" / "Log decision" bar for chat
│   ├── action-item-dialog.tsx          # Modal form for creating action items
│   ├── decision-dialog.tsx             # Modal form for logging decisions
│   └── board-section.tsx               # Single board deck section (editable)
```

### Modified Files

```
apps/web/src/
├── components/chat-message.tsx         # Add action bar below assistant messages
├── components/briefing-item.tsx        # Add "Take action" button for Decide items
├── components/nav.tsx                  # Wire Board Prep link

apps/api/src/prescient/
├── main.py                            # Register board_prep router
├── shared/metadata.py                 # Register board_prep tables
```

---

## Task 1: Action Bar and Dialogs for Chat

**Files:**
- Create: `apps/web/src/components/action-bar.tsx`
- Create: `apps/web/src/components/action-item-dialog.tsx`
- Create: `apps/web/src/components/decision-dialog.tsx`
- Modify: `apps/web/src/components/chat-message.tsx`

- [ ] **Step 1: Create action-item-dialog.tsx**

A modal/dialog form for creating an action item. Uses shadcn Dialog component.

```tsx
interface ActionItemDialogProps {
  open: boolean;
  onClose: () => void;
  defaultTitle?: string;
  defaultOwner?: string;
  sourceContext?: string;  // conversation excerpt for provenance
}
```

Form fields:
- Title (pre-filled from `defaultTitle`)
- Owner (text input, pre-filled from `defaultOwner`)
- Priority (select: high/medium/low)
- Due date (date input, defaults to 7 days from now)

On submit: POST to `/actions/items` via `apiFetch` with the form data plus `organization_id` from auth storage. Close dialog on success, show error on failure.

- [ ] **Step 2: Create decision-dialog.tsx**

A modal form for logging a decision record.

```tsx
interface DecisionDialogProps {
  open: boolean;
  onClose: () => void;
  defaultContext?: string;  // conversation excerpt
}
```

Form fields:
- Title (the decision)
- Rationale (textarea)
- Context (pre-filled, read-only or editable)

On submit: POST to `/actions/decisions` via `apiFetch`. Close on success.

- [ ] **Step 3: Create action-bar.tsx**

A subtle action bar that appears below assistant messages.

```tsx
interface ActionBarProps {
  messageContent: string;
  onCreateAction: () => void;
  onLogDecision: () => void;
}
```

Renders two small buttons: "Create action item" and "Log decision". Styled as text buttons or ghost buttons, minimal visual weight.

- [ ] **Step 4: Add action bar to chat-message.tsx**

Read the existing `chat-message.tsx`. Below assistant messages (after the content and provenance panel), add:
- The `ActionBar` component
- State for dialog open/close
- `ActionItemDialog` and `DecisionDialog` (rendered conditionally)
- Pre-fill the dialog defaults from the message content (first 100 chars as title suggestion, infer owner from content keywords if possible)

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: add insight-to-action flow — action items and decisions from chat"
```

---

## Task 2: Briefing "Take Action" Button

**Files:**
- Modify: `apps/web/src/components/briefing-item.tsx`

- [ ] **Step 1: Add "Take action" to Decide briefing items**

Read the existing `briefing-item.tsx`. For items with `action_type === "decide"`, add a "Take action" button that opens the `ActionItemDialog`.

The briefing item component needs:
- Import `ActionItemDialog`
- State for dialog open/close
- The button styled consistent with the briefing card design
- Pre-fill the action item title from the briefing item title

- [ ] **Step 2: Commit**

```bash
git commit -m "feat: add Take action button to Decide briefing items"
```

---

## Task 3: Board Prep Domain and Assembler (Backend)

**Files:**
- Create: `apps/api/src/prescient/board_prep/__init__.py`
- Create: `apps/api/src/prescient/board_prep/domain/__init__.py`
- Create: `apps/api/src/prescient/board_prep/domain/entities/__init__.py`
- Create: `apps/api/src/prescient/board_prep/domain/entities/board_deck.py`
- Create: `apps/api/src/prescient/board_prep/application/__init__.py`
- Create: `apps/api/src/prescient/board_prep/application/assembler.py`
- Create: `apps/api/src/prescient/board_prep/application/section_templates.py`

- [ ] **Step 1: Create domain entities**

`board_deck.py`:
```python
@dataclass(frozen=True)
class BoardSection:
    number: int
    title: str
    content: str          # Markdown with [[n]] citations
    citations: list[dict]  # Citation metadata
    data_sources: list[str]  # What was queried to produce this section

@dataclass(frozen=True)
class BoardDeck:
    id: UUID
    company_slug: str
    quarter: str           # e.g. "FY2024 Q3"
    sections: list[BoardSection]
    generated_at: datetime
    status: str            # "draft" | "approved"
```

- [ ] **Step 2: Create section templates**

`section_templates.py`: A list of 10 section templates, each with:
- `number`: 1-10
- `title`: section name
- `data_query`: what data the assembler should gather (KPIs, artifacts, actions, findings, etc.)
- `prompt_template`: the system prompt for the LLM call, with `{data}` placeholder

Define all 10 sections from the spec:
1. Executive Summary
2. Financial Performance
3. Subscriber / Customer Metrics
4. Pipeline & Go-Get
5. Key Initiative Updates
6. Competitive Landscape
7. Risk & Legal
8. People & Organization
9. What Not to Worry About
10. Decisions Needed

Each template should specify what data it needs and include a prompt that instructs the LLM to produce markdown with `[[n]]` citations.

- [ ] **Step 3: Create the assembler**

`assembler.py`: `BoardPrepAssembler` class with:

```python
class BoardPrepAssembler:
    def __init__(self, session: AsyncSession, llm: LLMProviderPort):
        ...

    async def gather_section_data(self, template: SectionTemplate, company_slug: str, quarter: str) -> dict:
        """Query relevant data for this section type."""
        # Based on template.data_query, fetch from:
        # - KPI values (documents.kpi_values via public_queries)
        # - KPI anomalies (kpis.kpi_anomalies)
        # - Action items (action_items table)
        # - Decision records (artifacts with type decision_record)
        # - Monitoring findings
        # - Strategic artifacts (knowledge.artifacts)
        # - Observations

    async def generate_section(self, template: SectionTemplate, data: dict, model: str) -> BoardSection:
        """Call LLM with template prompt + gathered data to produce section content."""
        prompt = template.prompt_template.format(data=json.dumps(data, default=str))
        response = await self.llm.send(
            system="You are a board deck section writer...",
            messages=[{"role": "user", "content": prompt}],
            tools=[],
            model=model,
        )
        text = response["content"][0]["text"]
        # Parse citations from text
        return BoardSection(number=template.number, title=template.title, content=text, ...)

    async def generate_deck(self, company_slug: str, quarter: str, model: str) -> BoardDeck:
        """Generate all 10 sections."""
        sections = []
        for template in SECTION_TEMPLATES:
            data = await self.gather_section_data(template, company_slug, quarter)
            section = await self.generate_section(template, data, model)
            sections.append(section)
        return BoardDeck(id=uuid4(), company_slug=company_slug, quarter=quarter, sections=sections, ...)
```

The assembler queries data using the same tables the existing seed pipeline writes to. Use the public_queries pattern (direct SQLAlchemy queries on legacy tables) for KPIs and documents. Use the new tables for action items and anomalies.

- [ ] **Step 4: Commit**

```bash
git commit -m "feat: add board prep domain entities, section templates, and assembler"
```

---

## Task 4: Board Prep API

**Files:**
- Create: `apps/api/src/prescient/board_prep/api/__init__.py`
- Create: `apps/api/src/prescient/board_prep/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`

- [ ] **Step 1: Create the board prep API**

Two endpoints:

```python
POST /board-prep/generate
```
Request: `{ company_slug: str, quarter: str }`
Response: SSE stream (like copilot) OR JSON with full deck.

For the demo, use a **non-streaming JSON response** (simpler). The frontend can show a loading spinner while it generates. Each section takes 2-5 seconds, total ~30-60 seconds. Return the full deck as JSON:

```python
class GenerateDeckResponse(BaseModel):
    id: str
    company_slug: str
    quarter: str
    sections: list[SectionResponse]
    generated_at: str

class SectionResponse(BaseModel):
    number: int
    title: str
    content: str
    citations: list[dict]
```

The endpoint:
1. Validates company exists
2. Creates the assembler with session + LLM adapter
3. Calls `assembler.generate_deck()`
4. Returns the result

```python
GET /board-prep/{deck_id}
```
Returns a previously generated deck. For the demo, we can skip persistence and just return from the generation endpoint. If persistence is needed, store as a JSONB row.

Use the same auth pattern (CtxDep) and LLM provider pattern (LLMDep) from the copilot routes.

- [ ] **Step 2: Register in main.py**

```python
from prescient.board_prep.api.routes import router as board_prep_router
app.include_router(board_prep_router)
```

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: add POST /board-prep/generate endpoint"
```

---

## Task 5: Board Prep Frontend Page

**Files:**
- Create: `apps/web/src/components/board-section.tsx`
- Create: `apps/web/src/app/(main)/board-prep/page.tsx`
- Modify: `apps/web/src/components/nav.tsx`

- [ ] **Step 1: Create board-section.tsx**

A single section of the board deck, rendered as an editable card.

```tsx
interface BoardSectionProps {
  number: number;
  title: string;
  content: string;
  citations: Array<{ marker: number; source_type: string; excerpt: string }>;
  onEdit: (content: string) => void;
  onRegenerate: () => void;
}
```

Renders:
- Section number and title header
- Content rendered as markdown (split on `\n\n` for paragraphs, `[[n]]` as citation pills)
- "Edit" button toggles between view and edit mode (textarea for editing)
- "Regenerate" button (calls onRegenerate)
- Styled as a card with subtle border, full-width

- [ ] **Step 2: Create board-prep page**

`apps/web/src/app/(main)/board-prep/page.tsx`

A "use client" page with:

**State:**
- `quarter` — selected quarter (default: current quarter)
- `sections` — array of generated sections
- `isGenerating` — loading state
- `editingSection` — which section is in edit mode

**Flow:**
1. Page shows a "Prepare Board Deck" form: company name (Peloton, read-only), quarter selector
2. "Generate" button triggers POST to `/board-prep/generate` via `apiFetch`
3. While generating: loading spinner with "Generating board deck..." message
4. When complete: render all sections as `BoardSection` cards in order
5. Each section editable: click Edit → textarea, save changes locally
6. "Approve Deck" button at the bottom (for demo, just shows a success toast)

**Quarter selector:** Simple dropdown with the last 4 quarters based on Peloton's fiscal calendar.

- [ ] **Step 3: Wire Board Prep link in nav**

Read the existing `nav.tsx`. The "Board Prep" item is currently a disabled `<span>`. Change it to an active `<Link href="/board-prep">`.

- [ ] **Step 4: Commit**

```bash
git commit -m "feat: add board prep page with section generation and editing"
```

---

## Summary

| Task | What it does | Key files |
|------|-------------|-----------|
| 1 | Action bar + dialogs for chat | `action-bar.tsx`, `action-item-dialog.tsx`, `decision-dialog.tsx`, `chat-message.tsx` |
| 2 | Briefing "Take action" button | `briefing-item.tsx` |
| 3 | Board prep domain + assembler (backend) | `board_deck.py`, `assembler.py`, `section_templates.py` |
| 4 | Board prep API endpoint | `routes.py`, `main.py` |
| 5 | Board prep frontend page | `board-section.tsx`, `board-prep/page.tsx`, `nav.tsx` |
