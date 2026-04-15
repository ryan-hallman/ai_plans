# Board Prep: Interactive Deck Preview with AI Assistant

**Date:** 2026-04-13
**Status:** Approved
**Author:** Ryan + Claude

## Overview

Redesign the Board Prep experience from a static section list into a high-fidelity, interactive slide editor with an integrated AI assistant. The result should feel like a polished presentation editor (Google Slides / Canva) purpose-built for board deck preparation, with AI deeply integrated for content creation and refinement.

## Goals

- Deliver a visually impressive, demo-ready board deck experience
- Enable direct WYSIWYG editing of slide content
- Provide AI assistance via both a conversational chat panel and inline quick actions
- Support slide management (add, remove, reorder)
- Enable export to PDF and PowerPoint formats
- Persist deck edits to the backend

## Layout

Three-zone layout:

### 1. Slide Filmstrip (left, ~180px)

- Vertical strip of slide thumbnails showing slide number and title
- Click to navigate between slides
- Drag-to-reorder slides
- Right-click or hover menu for add/delete actions
- "Add Slide" button at the bottom
- When adding a slide, user picks from existing section templates or starts with a blank slide

### 2. Slide Canvas (center, fills remaining space)

- High-fidelity rendering of the active slide in 16:9 aspect ratio
- Centered with letterboxing on non-standard screen sizes
- Presentation-quality styling:
  - Colored header banner with slide number and title (editable)
  - Clean typography and structured content areas
  - Consistent branding across all slides
- WYSIWYG editing:
  - Click any text block to enter edit mode (subtle border/highlight)
  - Rich text basics: bold, italic, bullet lists, headers
  - Auto-save on blur (no explicit save/cancel buttons)
- Citations render as subtle superscript pills with hover excerpts (preserving existing citation behavior)

### 3. AI Assistant Panel (right, ~350px, collapsible)

- Chat interface for conversational AI help
- Context-aware: knows which slide is active and its content
- Toggle button to show/hide the panel
- Chat history persists within the session

### Top Toolbar

- Quarter selector (existing FY-based quarterly picker)
- Generate Deck button
- Export dropdown (PDF / PowerPoint)
- Approve and Share actions

## Inline AI Actions

When the user selects text on the slide canvas, a floating toolbar appears above the selection with quick actions:

- **Rewrite** — rephrase the selected text
- **Make concise** — shorten while preserving meaning
- **Expand** — add more detail
- **Change tone** — dropdown with options: formal, conversational, executive

Each action fires a quick AI call and replaces the selected text with the result. User can undo if they don't like the output.

## AI Assistant Chat

### Behavior

- Context updates when the user switches slides — the assistant always knows the active slide and full deck structure
- Can handle both slide-specific and cross-deck requests
- Responses that modify content apply directly to the slide canvas — user sees changes happen
- Can create new slides, reorder slides, and modify content across multiple slides

### Example Interactions

- "Make this slide more concise" — rewrites active slide content
- "Add a new slide about customer expansion revenue" — creates and populates a new slide after the current one
- "Compare our revenue growth to the competitor data" — pulls from knowledge sources and updates content with citations
- "Move the risk section before decisions needed" — reorders slides
- "Does the executive summary align with the financial section?" — cross-slide analysis without modifications

### Scope Boundaries (Demo)

- The assistant operates on deck content — it does not fetch new external data or trigger a full regeneration
- It works with what's already been generated plus the existing knowledge base
- No multi-user collaboration or real-time sync — single user editing

## Export

### PDF

- Client-side generation (e.g., `html2canvas` + `jsPDF` or similar)
- Each slide rendered as a landscape (16:9) page
- Preserves high-fidelity visual styling: colors, typography, layout
- Citations rendered inline with a references appendix page at the end

### PowerPoint (.pptx)

- Server-side generation using `python-pptx`
- One slide per section, mapping content structure to PowerPoint elements (title, body text, bullet lists)
- Clean template with consistent branding/colors
- Polished, editable starting point — not pixel-perfect match to in-app preview
- Backend already has the deck data model, so this fits naturally

### Export Trigger

Dropdown button in the top toolbar: "Export" with PDF and PowerPoint as options. PDF downloads immediately (client-side). PPTX triggers a backend call then downloads.

## Data & State Management

### Frontend State (Zustand)

- Deck state: slides array with order, content, citations, metadata
- Undo/redo stack for all edits (manual and AI-generated)
- Dirty state tracking — warn before navigating away with unsaved changes
- Active slide index
- AI chat history

### Backend Persistence

The current backend generates a deck but does not persist edits. This redesign adds persistence.

**New/modified endpoints:**

| Method | Path | Purpose |
|--------|------|---------|
| `PUT` | `/board-prep/{deck_id}` | Save full deck state (slides, order, content) |
| `POST` | `/board-prep/{deck_id}/slides` | Add a slide |
| `DELETE` | `/board-prep/{deck_id}/slides/{slide_number}` | Remove a slide |
| `PUT` | `/board-prep/{deck_id}/slides/{slide_number}` | Update a single slide |
| `POST` | `/board-prep/{deck_id}/chat` | AI assistant chat message |
| `POST` | `/board-prep/{deck_id}/inline-action` | Inline AI action on selected text |
| `GET` | `/board-prep/{deck_id}/export/{format}` | Export deck as PDF or PPTX |

**Deck status values:** `draft` | `approved` | `shared`

### AI Assistant Backend

**Chat endpoint receives:**
- User message
- Active slide content
- Full deck summary (all slide titles + abbreviated content)
- Conversation history

**Chat endpoint returns:**
- Assistant response text
- Optional `mutations` array: content updates, slide additions, reorders
- Frontend applies mutations to the deck state

**Inline action endpoint receives:**
- Selected text
- Action type (rewrite, concise, expand, tone change)
- Slide context (full slide content for surrounding context)

**Inline action endpoint returns:**
- Replacement text

## Slide Sections

Default generated deck retains the existing 10-section structure:

1. Executive Summary
2. Financial Performance
3. Subscriber/Customer Metrics
4. Pipeline & Go-Get
5. Key Initiative Updates
6. Competitive Landscape
7. Risk & Legal
8. People & Organization
9. What Not to Worry About
10. Decisions Needed

Users can add, remove, and reorder these freely after generation. New slides can use any of the above as templates or start blank.

## Technical Notes

- Existing `board-prep/generate` endpoint continues to work for initial deck generation
- Deck generation creates a persisted deck record; subsequent edits update that record
- The current `board-section.tsx` component will be replaced by the new slide canvas
- Undo/redo is frontend-only (Zustand state history), not persisted to backend
- AI chat and inline actions use the same Claude model as the existing generation pipeline
