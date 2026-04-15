# Chat Experience Polish — Design Spec

**Date**: 2026-04-13
**Status**: Approved

---

## 1. Problem

The copilot chat interface has two major gaps:

1. **No markdown rendering** — assistant responses are rendered as plain text with `whitespace-pre-wrap`. Any markdown the LLM produces (headings, bold, lists, tables, code blocks) displays as literal characters.
2. **Source panel is a dead end** — clicking a citation shows only `tool_name`, `source_type`, and a short text excerpt. The API has rich structured data in `ToolExecution.records` (document titles, artifact summaries, KPI values, section names, confidence labels) but none of it reaches the frontend.

Secondary issues:
- Chat is hardcoded to the "peloton" company slug — not company-scoped.
- No hover preview on citation pills.
- Thin cursor as streaming indicator gives no context on what's happening.

---

## 2. Decisions

| Question | Decision |
|---|---|
| Source panel behavior | Rich inline preview with type-specific renderers + deep link to full source |
| Markdown rendering level | Full: headings, bold/italic, lists, code blocks with syntax highlighting, tables, horizontal rules |
| Chat scoping | Company-scoped at `/companies/{slug}/chat`, accessed from company pages |
| Chat layout | Dedicated page (not drawer or split view) |
| Conversation persistence | Out of scope — state lives in React useState |
| Global copilot | Future work — architecture won't block it |

---

## 3. Markdown Rendering

### Dependencies

Install `react-markdown` with plugins:
- `remark-gfm` — GitHub-flavored markdown (tables, strikethrough, task lists)
- `rehype-highlight` — syntax-highlighted code blocks

### Component: `<MarkdownContent>`

A new component wrapping `react-markdown` with:
- Tailwind prose styling (`prose prose-sm prose-neutral`)
- Custom element renderers for headings (h1-h4), bold, italic, inline code, bulleted/numbered lists, code blocks, tables, horizontal rules
- Code blocks get a copy-to-clipboard button
- Links open in new tabs

### Integration with ChatMessage

- Assistant messages: replace the current `whitespace-pre-wrap` span with `<MarkdownContent>`
- User messages: stay as plain text (no markdown rendering)
- Citation markers (`[[N]]`) are pre-processed out of the raw markdown string before rendering. The content is split on `/(\[\[\d+\]\])/g`, markdown segments are rendered through `<MarkdownContent>`, and citation markers are replaced with inline `<CitationPill>` components.

---

## 4. Source Panel — Rich Inline Preview

### API Changes

The `citation` SSE event currently sends:
```json
{"marker": 1, "tool_result_id": "...", "tool_name": "search_documents", "source_type": "filing", "excerpt": "..."}
```

Expand it to include structured metadata from `ToolExecution.records`. The planner stream already has access to `all_tool_results` (a list of `ToolResult` objects) when it builds citations via `_extract_citations()`. Each `ToolResult.result` contains `{"summary": ..., "records": [...]}` where `records` holds per-hit structured data from tool execution. The citation's `tool_result_id` maps to a specific `ToolResult`. When yielding citation events, look up the matching `ToolResult`, extract relevant metadata from its `records`, and include it in the event payload.

**New fields by source type:**

| Source type | Additional fields |
|---|---|
| `filing`, `transcript`, `document` | `document_title`, `section_name`, `published_at`, `document_type`, `document_id` |
| `artifact` | `artifact_id`, `artifact_summary`, `artifact_function`, `confidence_label`, `version_number` |
| `kpi` | `kpi_id`, `period_label`, `value`, `unit`, `sparkline_points` (last 6 values) |

These fields are optional — the panel renders gracefully when they're missing (falls back to the current excerpt-only view).

### Frontend Citation type

Extend the `Citation` type in `chat-stream.ts`:
```typescript
export type Citation = {
  marker: number;
  tool_name: string;
  source_type: string;
  excerpt: string;
  // Rich metadata (optional, type-dependent)
  document_title?: string;
  section_name?: string;
  published_at?: string;
  document_type?: string;
  document_id?: string;
  artifact_id?: string;
  artifact_summary?: string;
  artifact_function?: string;
  confidence_label?: string;
  version_number?: number;
  kpi_id?: string;
  period_label?: string;
  value?: number;
  unit?: string;
  sparkline_points?: { period_label: string; value: number }[];
};
```

### Panel Renderers

The `SourcePanel` component switches on `source_type` and renders a type-specific view:

**Document/Filing sources:**
- Header: document title + published date
- Badge: document type (filing, transcript)
- Section name if available
- Excerpt in a styled blockquote
- "View document" link to `/companies/{slug}/documents/{document_id}`

**Artifact sources:**
- Header: artifact name
- Badges: function tag (e.g. "Growth Strategy"), confidence label
- Version number
- Summary paragraph
- "View artifact" link to `/companies/{slug}/artifacts/{artifact_id}`

**KPI sources:**
- Header: KPI name (title-cased from snake_case id)
- Current value displayed prominently with unit
- Mini sparkline of recent values (reuse existing `Sparkline` component pattern)
- "View KPIs" link to company KPI page

**Fallback (unrecognized source type):**
- Current behavior: badge, tool name, excerpt, no deep link

---

## 5. Company-Scoped Chat Routing

### Route Change

- **New**: `app/(main)/companies/[slug]/chat/page.tsx`
- **Remove**: `app/(main)/chat/page.tsx` (or redirect to a default company)
- The page reads `params.slug`, fetches company name for the header, passes slug to `streamChat()` and `<SourcePanel>`

### Navigation

- Company overview page: update the existing "Ask about this company" button to link to `/companies/{slug}/chat`
- Remove "Copilot" from the top nav (chat is now accessed from company context)
- Source panel deep links use the slug from the route

### API Integration

`streamChat()` in `chat-stream.ts` already accepts `companySlug` — it passes through to the Next.js BFF proxy at `/api/chat`, which forwards to the FastAPI backend. No API changes needed for routing.

### Future: Global Copilot

The architecture supports adding a `/chat` global route later. That route would let the user pick a company context or have the LLM infer it from the question. The company-scoped version and global version share all components — the only difference is where the slug comes from.

---

## 6. Chat UX Polish

### Empty State

Keep the suggested prompt pills from recent polish. Make them company-aware:
- Header: "Ask anything about {companyName}"
- Prompts reference the company contextually (these can be generic operational prompts like "What are our top risks this quarter?")

### Streaming Indicator

- When tools are running (between `tool_start` and the first `delta`): show "Researching..." label
- When text is streaming: show the current blinking cursor

### Citation Pill Tooltip

Add a hover tooltip on `<CitationPill>` showing the first ~80 characters of the excerpt. Gives a preview before clicking to open the full source panel.

---

## 7. Files Affected

### New Files
- `apps/web/src/components/markdown-content.tsx` — markdown renderer component
- `apps/web/src/app/(main)/companies/[slug]/chat/page.tsx` — company-scoped chat route

### Modified Files
- `apps/web/src/components/chat-message.tsx` — swap plain text for `<MarkdownContent>`
- `apps/web/src/components/source-panel.tsx` — type-specific renderers, richer data display
- `apps/web/src/components/citation-pill.tsx` — add hover tooltip
- `apps/web/src/lib/chat-stream.ts` — extend Citation type with optional metadata fields
- `apps/web/src/app/(main)/companies/[slug]/page.tsx` — update "Ask about this company" link
- `apps/web/src/components/nav.tsx` — remove global Copilot link
- `apps/api/src/prescient/intelligence/application/planner_stream.py` — enrich citation event data
- `apps/web/package.json` — add react-markdown, remark-gfm, rehype-highlight

### Removed Files
- `apps/web/src/app/(main)/chat/page.tsx` — replaced by company-scoped route

---

## 8. Non-Goals

- Conversation persistence across page reloads
- Global copilot (future work)
- Multiple conversations per session
- File/image upload in chat
- Mid-stream citation rendering (citations arrive after response completion)
- Code block execution
