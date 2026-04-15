# Phase 4c: Artifact Viewer, Document Viewer & Company Overview Enrichment

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the artifact detail view with version history and cross-references, a document viewer for SEC filings and transcripts, a provenance panel for chat messages, and enrich the company overview page with activity timeline, competitor sidebar, and "Ask about this company" chat link.

**Architecture:** Primarily frontend work leveraging existing API endpoints (`GET /artifact-store/{id}`, `GET /companies/{slug}/artifacts`, `GET /companies/{slug}/documents`). One new API endpoint for fetching a single document by ID. The artifact detail page renders blocks with inline citations. The document viewer shows filing/transcript content with excerpt highlighting. The provenance panel is added to the chat message component.

**Tech Stack:** Next.js 15, React 19, Tailwind CSS, shadcn/ui. Existing FastAPI endpoints.

**Spec:** `docs/superpowers/specs/2026-04-12-phase4-demo-design.md` — Section 4

**Depends on:** Phase 4b (chat page must exist for provenance panel)

---

## File Structure

### New Files

```
apps/web/src/
├── app/(main)/companies/[slug]/artifacts/[id]/
│   └── page.tsx                    # Artifact detail view
├── app/(main)/companies/[slug]/documents/[id]/
│   └── page.tsx                    # Document viewer page
├── components/
│   ├── artifact-block.tsx          # Single content block with citations
│   ├── version-sidebar.tsx         # Version list with state badges
│   ├── cross-references.tsx        # Related artifacts section
│   ├── activity-timeline.tsx       # Recent activity for company overview
│   ├── competitor-sidebar.tsx      # Watched competitors with latest findings
│   └── provenance-panel.tsx        # "Show reasoning" for chat messages

apps/api/src/prescient/
├── documents/
│   └── api/
│       └── routes.py               # GET /documents/{id} endpoint
```

### Modified Files

```
apps/web/src/
├── app/(main)/companies/[slug]/page.tsx     # Add activity, competitors, chat link
├── components/chat-message.tsx              # Add provenance toggle
├── components/source-panel.tsx              # Enhance with full artifact/document view

apps/api/src/prescient/
├── main.py                                  # Register documents router
```

---

## Task 1: Document Detail API Endpoint

**Files:**
- Create: `apps/api/src/prescient/documents/api/__init__.py`
- Create: `apps/api/src/prescient/documents/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`

The existing `GET /companies/{slug}/documents` returns a list of document summaries. We need a detail endpoint that returns the full document content by ID so the frontend can display filing sections and transcript segments.

- [ ] **Step 1: Create the documents API router**

Read the existing `documents/public_queries.py` to understand the `DocumentTable` structure. Create a simple `GET /documents/{document_id}` endpoint that returns the full document row including content/storage_path.

The endpoint should:
- Accept a `document_id` (UUID)
- Query `documents.documents` table by ID
- Return: id, document_type, title, filing_id, section, published_at, source_url, content (from meta or storage), subject_company_id
- Return 404 if not found

Follow the pattern in `companies/api/routes.py` — use `RequestContext` for auth, direct SQLAlchemy queries.

- [ ] **Step 2: Register the router in main.py**

```python
from prescient.documents.api.routes import router as documents_router
app.include_router(documents_router)
```

- [ ] **Step 3: Test the endpoint**

```bash
curl http://localhost:8000/documents/{some-document-id} | python3 -m json.tool
```

- [ ] **Step 4: Commit**

```bash
git commit -m "feat: add GET /documents/{id} detail endpoint"
```

---

## Task 2: Artifact Detail Page

**Files:**
- Create: `apps/web/src/app/(main)/companies/[slug]/artifacts/[id]/page.tsx`
- Create: `apps/web/src/components/artifact-block.tsx`
- Create: `apps/web/src/components/version-sidebar.tsx`
- Create: `apps/web/src/components/cross-references.tsx`

- [ ] **Step 1: Create artifact-block.tsx**

Renders a single content block from an artifact version. Each block has a heading and body. The body may contain `[[n]]` citation markers that should be rendered as `CitationPill` components (reuse from chat).

```tsx
interface ArtifactBlockProps {
  block: { block_id: string; heading?: string; body: string; kind?: string };
  citations?: Array<{ id: string; block_id: string; source_type: string; excerpt: string; document_id?: string }>;
  onCitationClick?: (citationId: string) => void;
}
```

- Render heading as `<h3>` with semibold styling
- Render body as paragraphs (split on `\n\n`)
- Parse `[[n]]` markers and replace with CitationPill components
- For `kind: "priority"` blocks, render with a priority number badge and metadata (owner, horizon, linked_metrics)
- For `kind: "out_of_scope"` blocks, render with a muted/de-emphasized style

- [ ] **Step 2: Create version-sidebar.tsx**

Shows the version history of an artifact. For the demo, most artifacts will have 1 version.

```tsx
interface VersionSidebarProps {
  versions: Array<{
    id: string;
    version_number: number;
    state: string;
    confidence_label: string;
    approved_at?: string;
    approved_by?: string;
  }>;
  activeVersionId: string;
  onVersionSelect: (versionId: string) => void;
}
```

- List of versions, newest first
- Each shows: version number, state badge (draft/active/approved/archived with color), confidence label, approved date
- Active version highlighted
- Click to select a different version

- [ ] **Step 3: Create cross-references.tsx**

Shows related artifacts linked via `artifact_references`.

```tsx
interface CrossReferencesProps {
  references: Array<{
    id: string;
    reference_type: string;
    target_artifact_id: string;
    target_title: string;
    target_type: string;
  }>;
  companySlug: string;
}
```

- Group by reference_type (INFORMS, SPAWNED, SUMMARIZES, RELATES_TO)
- Each reference is a clickable link to `/companies/{slug}/artifacts/{target_id}`
- Show type badge and title
- If no references, show "No related artifacts"

- [ ] **Step 4: Create the artifact detail page**

`apps/web/src/app/(main)/companies/[slug]/artifacts/[id]/page.tsx`

This page:
1. Fetches the artifact detail from `GET /artifact-store/{id}`
2. Renders the header: title, type badge, confidence label, last updated
3. Renders blocks via `ArtifactBlock` components
4. Shows `VersionSidebar` on the right
5. Shows `CrossReferences` at the bottom

The page should use the existing `apiFetch` from `@/lib/api-client`. Read the response shape from `GET /artifact-store/{id}` (check `artifacts/api/routes.py`) to type the data correctly.

Use a two-column layout: main content (left, wider) + version sidebar (right, narrow).

- [ ] **Step 5: Commit**

```bash
git commit -m "feat: add artifact detail page with blocks, versions, and cross-references"
```

---

## Task 3: Document Viewer Page

**Files:**
- Create: `apps/web/src/app/(main)/companies/[slug]/documents/[id]/page.tsx`

- [ ] **Step 1: Create the document viewer page**

Displays a single SEC filing section, earnings transcript segment, or news article.

The page:
1. Fetches the document from `GET /documents/{id}` (the endpoint from Task 1)
2. Renders:
   - Header: title, document_type badge (filing/transcript/news), published_at date
   - Filing metadata: filing_id, section name
   - Source URL as external link ("View on EDGAR" for SEC filings)
   - Full content text with the ability to highlight a specific excerpt (passed via URL query param `?highlight=...`)
3. For transcript segments: show speaker name and quarter
4. For SEC filings: show section name and filing reference

Highlight implementation: if `?highlight=some+text` is in the URL, find that text in the content and wrap it in a `<mark>` tag with a yellow background.

Keep it simple — this is a read-only viewer. No editing, no state management beyond the fetch.

- [ ] **Step 2: Commit**

```bash
git commit -m "feat: add document viewer page with excerpt highlighting"
```

---

## Task 4: Enhanced Source Panel

**Files:**
- Modify: `apps/web/src/components/source-panel.tsx`
- Modify: `apps/web/src/app/(main)/chat/page.tsx`

- [ ] **Step 1: Enhance source-panel.tsx**

Currently the source panel shows just the citation excerpt and metadata. Enhance it to:
- Show the full excerpt with better formatting
- Add a "View full artifact" or "View full document" link that navigates to the artifact/document detail page
- Show source_type as a colored badge (filing → blue, transcript → purple, kpi → green, artifact → indigo)
- Add the tool_name that produced the citation
- Make the panel wider (w-[480px]) with better content structure

The panel needs to know the company slug for linking. Add `companySlug` to `SourcePanelProps`.

- [ ] **Step 2: Update chat page to pass companySlug to SourcePanel**

Add `companySlug="peloton"` to the `<SourcePanel>` in the chat page.

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: enhance source panel with full content and navigation links"
```

---

## Task 5: Provenance Panel for Chat Messages

**Files:**
- Create: `apps/web/src/components/provenance-panel.tsx`
- Modify: `apps/web/src/components/chat-message.tsx`
- Modify: `apps/web/src/app/(main)/chat/page.tsx`

- [ ] **Step 1: Create provenance-panel.tsx**

A collapsible panel that shows the full reasoning chain for an assistant message.

```tsx
interface ProvenancePanelProps {
  toolCalls: Array<{
    tool: string;
    arguments?: Record<string, unknown>;
    summary?: string;
    record_count?: number;
    status?: string;
    latency_ms?: number;
  }>;
  citations: Array<{
    marker: number;
    tool_name: string;
    source_type: string;
    excerpt: string;
  }>;
  groundingStatus?: string;
  iterations?: number;
}
```

Renders:
- "Show reasoning" toggle button (collapsed by default)
- When expanded:
  - **Tool calls** in chronological order: tool name (humanized), arguments (collapsed JSON), result summary, record count, latency
  - **Citations** mapped to their source tool calls: "Citation [[1]] → get_kpi_series: Churn: 8.2%..."
  - **Summary footer**: total iterations, grounding status badge

Style: muted background (neutral-50), smaller text (text-xs), monospace for arguments.

- [ ] **Step 2: Add provenance toggle to chat-message.tsx**

Add the `ProvenancePanel` below assistant messages. Pass the toolCalls, citations, groundingStatus from the message props.

The `ChatMessageProps` already has `toolCalls`, `citations`, and `groundingStatus`. Also add `iterations` as an optional prop (passed from the `done` event data).

- [ ] **Step 3: Update chat page to pass iterations to messages**

When the `done` event arrives, store `iterations` on the assistant message alongside `groundingStatus`. Pass it through to `ChatMessage`.

- [ ] **Step 4: Commit**

```bash
git commit -m "feat: add provenance panel to chat messages — show reasoning chain"
```

---

## Task 6: Company Overview Enrichment

**Files:**
- Create: `apps/web/src/components/activity-timeline.tsx`
- Create: `apps/web/src/components/competitor-sidebar.tsx`
- Modify: `apps/web/src/app/(main)/companies/[slug]/page.tsx`

- [ ] **Step 1: Create activity-timeline.tsx**

A vertical timeline showing recent activity for a company: intelligence signals, decisions, action items, monitoring findings.

```tsx
interface ActivityItem {
  id: string;
  type: "decision" | "action_item" | "signal" | "finding";
  title: string;
  summary?: string;
  status?: string;
  created_at: string;
}

interface ActivityTimelineProps {
  items: ActivityItem[];
}
```

Renders:
- Vertical line with dots for each item
- Each item: type icon/badge, title, summary (truncated), relative timestamp
- Type colors: decision → indigo, action_item → amber, signal → blue, finding → neutral
- Max 10 items, "View all" link at bottom

- [ ] **Step 2: Create competitor-sidebar.tsx**

Shows watched competitors with their latest monitoring finding.

```tsx
interface CompetitorInfo {
  slug: string;
  name: string;
  relationship_type: string;
  latest_finding?: {
    title: string;
    severity: string;
    discovered_at: string;
  };
}

interface CompetitorSidebarProps {
  competitors: CompetitorInfo[];
  companySlug: string;
}
```

Renders:
- List of competitor cards
- Each card: company name, relationship type badge (competitor/ecosystem_threat/cautionary_parallel), latest finding title + severity badge
- Click navigates to competitor company page

- [ ] **Step 3: Enrich the company overview page**

Read the existing `apps/web/src/app/(main)/companies/[slug]/page.tsx` fully. Then add:

1. **"Ask about this company" button** — a prominent button that links to `/chat` (or opens chat scoped to this company). Place it in the header area.

2. **Activity timeline section** — below the artifacts list. Fetch recent action items from `GET /actions/items?organization_id={fundId}` and recent monitoring findings from the headspace digest. Combine into `ActivityItem[]` and pass to `ActivityTimeline`.

3. **Competitor sidebar** — right column alongside the main content. Fetch from `GET /companies/{slug}/relationships` to get competitor companies. For each competitor, show the name and relationship type. Latest finding can be pulled from the headspace digest data if available, or omitted for now.

4. **Layout adjustment** — restructure to a two-column layout: main content (left) with KPIs + artifacts + activity timeline, and a narrower right column with competitor sidebar.

- [ ] **Step 4: Commit**

```bash
git commit -m "feat: enrich company overview with activity timeline, competitors, and chat link"
```

---

## Summary

| Task | What it does | Key files |
|------|-------------|-----------|
| 1 | Document detail API endpoint | `documents/api/routes.py`, `main.py` |
| 2 | Artifact detail page (blocks, versions, cross-refs) | `artifacts/[id]/page.tsx`, 3 components |
| 3 | Document viewer page with highlighting | `documents/[id]/page.tsx` |
| 4 | Enhanced source panel with navigation links | `source-panel.tsx`, `chat/page.tsx` |
| 5 | Provenance panel for chat messages | `provenance-panel.tsx`, `chat-message.tsx` |
| 6 | Company overview enrichment | `activity-timeline.tsx`, `competitor-sidebar.tsx`, company page |
