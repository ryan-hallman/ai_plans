# Chat Experience Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Polish the copilot chat interface with markdown rendering, rich source panel previews, company-scoped routing, and UX improvements.

**Architecture:** Three layers of change: (1) API — enrich citation data by deriving `source_type` from `tool_name` and extracting metadata from `ToolResult.result["records"]` in `_extract_citations`, (2) Frontend shared — add `react-markdown` renderer and extend `Citation` type, (3) Frontend pages — rebuild source panel with type-specific renderers, move chat to company-scoped route.

**Tech Stack:** react-markdown, remark-gfm, rehype-highlight, Next.js dynamic routes, FastAPI SSE streaming

---

### Task 1: Install markdown dependencies

**Files:**
- Modify: `apps/web/package.json`

- [ ] **Step 1: Install packages**

```bash
cd apps/web && npm install react-markdown remark-gfm rehype-highlight
```

- [ ] **Step 2: Verify installation**

```bash
cd apps/web && node -e "require('react-markdown'); require('remark-gfm'); require('rehype-highlight'); console.log('OK')"
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add apps/web/package.json apps/web/package-lock.json
git commit -m "feat(web): add react-markdown, remark-gfm, rehype-highlight"
```

---

### Task 2: Create MarkdownContent component

**Files:**
- Create: `apps/web/src/components/markdown-content.tsx`

- [ ] **Step 1: Create the component**

```tsx
// apps/web/src/components/markdown-content.tsx
"use client";

import ReactMarkdown from "react-markdown";
import remarkGfm from "remark-gfm";
import rehypeHighlight from "rehype-highlight";
import type { Components } from "react-markdown";

interface MarkdownContentProps {
  content: string;
}

const components: Components = {
  h1: ({ children }) => (
    <h1 className="mb-2 mt-4 text-lg font-semibold text-neutral-900 first:mt-0">{children}</h1>
  ),
  h2: ({ children }) => (
    <h2 className="mb-2 mt-3 text-base font-semibold text-neutral-900 first:mt-0">{children}</h2>
  ),
  h3: ({ children }) => (
    <h3 className="mb-1.5 mt-3 text-sm font-semibold text-neutral-900 first:mt-0">{children}</h3>
  ),
  h4: ({ children }) => (
    <h4 className="mb-1 mt-2 text-sm font-medium text-neutral-800 first:mt-0">{children}</h4>
  ),
  p: ({ children }) => <p className="mb-2 last:mb-0">{children}</p>,
  strong: ({ children }) => <strong className="font-semibold text-neutral-900">{children}</strong>,
  em: ({ children }) => <em className="italic">{children}</em>,
  ul: ({ children }) => <ul className="mb-2 ml-4 list-disc space-y-0.5 last:mb-0">{children}</ul>,
  ol: ({ children }) => (
    <ol className="mb-2 ml-4 list-decimal space-y-0.5 last:mb-0">{children}</ol>
  ),
  li: ({ children }) => <li className="pl-0.5">{children}</li>,
  a: ({ href, children }) => (
    <a
      href={href}
      target="_blank"
      rel="noopener noreferrer"
      className="text-indigo-600 underline hover:text-indigo-700"
    >
      {children}
    </a>
  ),
  code: ({ className, children }) => {
    const isBlock = className?.startsWith("language-") || className?.startsWith("hljs");
    if (isBlock) {
      return (
        <div className="group relative my-2 rounded-md border border-neutral-200 bg-neutral-900">
          <button
            type="button"
            className="absolute right-2 top-2 rounded border border-neutral-600 bg-neutral-800 px-2 py-0.5 text-xs text-neutral-400 opacity-0 transition-opacity group-hover:opacity-100 hover:text-neutral-200"
            onClick={() => {
              const text = typeof children === "string" ? children : "";
              navigator.clipboard.writeText(text);
            }}
          >
            Copy
          </button>
          <pre className="overflow-x-auto p-3 text-xs leading-relaxed">
            <code className={className}>{children}</code>
          </pre>
        </div>
      );
    }
    return (
      <code className="rounded bg-neutral-200 px-1 py-0.5 text-xs font-mono text-neutral-800">
        {children}
      </code>
    );
  },
  pre: ({ children }) => <>{children}</>,
  table: ({ children }) => (
    <div className="my-2 overflow-x-auto rounded-md border border-neutral-200">
      <table className="min-w-full text-xs">{children}</table>
    </div>
  ),
  thead: ({ children }) => <thead className="bg-neutral-100">{children}</thead>,
  th: ({ children }) => (
    <th className="px-3 py-1.5 text-left font-medium text-neutral-700">{children}</th>
  ),
  td: ({ children }) => (
    <td className="border-t border-neutral-200 px-3 py-1.5 text-neutral-700">{children}</td>
  ),
  hr: () => <hr className="my-3 border-neutral-200" />,
  blockquote: ({ children }) => (
    <blockquote className="my-2 border-l-2 border-neutral-300 pl-3 text-neutral-600 italic">
      {children}
    </blockquote>
  ),
};

export function MarkdownContent({ content }: MarkdownContentProps) {
  return (
    <ReactMarkdown remarkPlugins={[remarkGfm]} rehypePlugins={[rehypeHighlight]} components={components}>
      {content}
    </ReactMarkdown>
  );
}
```

- [ ] **Step 2: Verify the component compiles**

```bash
cd apps/web && npx next build --no-lint 2>&1 | head -20
```

Expected: No TypeScript errors related to `markdown-content.tsx`. (Build may fail for other reasons — that's fine, we're just checking this file compiles.)

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/markdown-content.tsx
git commit -m "feat(web): add MarkdownContent component with GFM and syntax highlighting"
```

---

### Task 3: Wire markdown rendering into ChatMessage

**Files:**
- Modify: `apps/web/src/components/chat-message.tsx`

- [ ] **Step 1: Update imports**

Add the new import at the top of `chat-message.tsx`:

```tsx
import { MarkdownContent } from "@/components/markdown-content";
```

- [ ] **Step 2: Replace parseContent with markdown-aware version**

Replace the existing `parseContent` function (lines 22-44) with:

```tsx
function parseContent(
  content: string,
  citations: Citation[],
  onCitationClick: (marker: number) => void,
) {
  // Split on citation markers, render markdown for text segments
  const parts = content.split(/(\[\[\d+\]\])/g);
  return parts.map((part, i) => {
    const match = part.match(/^\[\[(\d+)\]\]$/);
    if (match && match[1]) {
      const marker = parseInt(match[1], 10);
      const citation = citations.find((c) => c.marker === marker);
      return (
        <CitationPill
          key={i}
          marker={marker}
          excerpt={citation?.excerpt ?? ""}
          onClick={onCitationClick}
        />
      );
    }
    if (!part) return null;
    return <MarkdownContent key={i} content={part} />;
  });
}
```

- [ ] **Step 3: Update the assistant bubble rendering**

Replace the assistant content rendering (the `<span className="whitespace-pre-wrap">` block, approximately lines 90-97) with:

```tsx
<div className="rounded-2xl rounded-bl-md bg-neutral-100 px-4 py-2.5 text-sm leading-relaxed text-neutral-800">
  {content ? (
    <div className="markdown-chat">
      {parseContent(content, citations ?? [], onCitationClick)}
    </div>
  ) : (
    <span className="inline-block h-4 w-1 animate-pulse bg-neutral-400" />
  )}
</div>
```

Note: changed from `<span className="whitespace-pre-wrap">` to `<div className="markdown-chat">`.

- [ ] **Step 4: Verify in browser**

Start the dev server (`cd apps/web && npm run dev`), navigate to `http://localhost:3000/chat`, and send a message. The response should render markdown formatting (bold, lists, headings) instead of raw markdown characters.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/components/chat-message.tsx
git commit -m "feat(web): render assistant chat messages as markdown"
```

---

### Task 4: Enrich citation data in the API

**Files:**
- Modify: `apps/api/src/prescient/intelligence/application/planner_stream.py`
- Modify: `apps/api/src/prescient/intelligence/application/planner.py`

This is the critical backend change. Currently `_extract_citations` in `planner.py` hardcodes `source_type="tool_result"` and uses `tr.result["summary"]` as the excerpt. We need to:
1. Derive `source_type` from `tool_name`
2. Extract the best excerpt from `tr.result["records"]` instead of the summary
3. Include structured metadata fields from the records

- [ ] **Step 1: Add a helper to derive source_type and metadata from tool results**

Add this function in `planner.py`, above `_extract_citations`:

```python
_TOOL_SOURCE_TYPE: dict[str, str] = {
    "search_documents": "document",
    "get_filing_section": "document",
    "search_artifacts": "artifact",
    "get_artifact": "artifact",
    "get_kpi_series": "kpi",
    "search_news": "news",
    "search_monitoring": "monitoring",
    "search_kpis": "kpi",
    "search_actions": "action_item",
    "get_company_profile": "profile",
}


def _enrich_citation(
    tool_name: str,
    tool_result: "ToolResult",
) -> dict[str, Any]:
    """Extract source_type and structured metadata from a tool result.

    Returns a dict with ``source_type`` (str) and optional metadata keys
    that the frontend can use for rich source-panel rendering.
    """
    source_type = _TOOL_SOURCE_TYPE.get(tool_name, "tool_result")
    records: list[dict[str, Any]] = tool_result.result.get("records", [])
    meta: dict[str, Any] = {"source_type": source_type}

    if not records:
        meta["excerpt"] = tool_result.result.get("summary", "")[:512]
        return meta

    first = records[0]

    if source_type == "document":
        meta["excerpt"] = first.get("excerpt", "")[:512]
        meta["document_title"] = first.get("title")
        meta["section_name"] = first.get("section")
        meta["published_at"] = first.get("published_at")
        meta["document_type"] = first.get("document_type")
        meta["document_id"] = first.get("document_id")

    elif source_type == "artifact":
        meta["excerpt"] = first.get("summary") or first.get("title", "")
        meta["artifact_id"] = first.get("artifact_id")
        meta["artifact_summary"] = first.get("summary")
        meta["artifact_function"] = first.get("function")
        meta["confidence_label"] = first.get("confidence_label")
        meta["version_number"] = first.get("version_number")

    elif source_type == "kpi":
        meta["excerpt"] = tool_result.result.get("summary", "")[:512]
        meta["kpi_id"] = first.get("label") or first.get("kpi_id")
        meta["period_label"] = first.get("period_label")
        meta["value"] = first.get("value")
        meta["unit"] = first.get("unit")
        # Include last 6 data points for sparkline
        meta["sparkline_points"] = [
            {"period_label": r.get("period_label", ""), "value": r.get("value", 0)}
            for r in records[:6]
        ]

    else:
        meta["excerpt"] = tool_result.result.get("summary", "")[:512]

    return meta
```

- [ ] **Step 2: Update _extract_citations to use the enrichment helper**

In `planner.py`, find the `_extract_citations` function. Replace the citation construction loop. The current code builds each `Citation` with hardcoded `source_type="tool_result"` and `excerpt=summary[:256]`. Update it to:

```python
        enrichment = _enrich_citation(tool_name, tr)
        citations.append(
            Citation(
                marker=marker,
                tool_result_id=result_id,
                tool_name=tool_name,
                source_type=enrichment.pop("source_type", "tool_result"),
                excerpt=enrichment.pop("excerpt", summary[:256]) if enrichment.get("excerpt") else (summary[:256] or "tool result"),
            )
        )
```

Also store the enrichment metadata alongside citations for the streaming layer. The simplest approach: return both citations and a parallel metadata list. Change the return type and add:

```python
    return tuple(citations), tuple(enrichments)
```

Where `enrichments` is a list built alongside `citations`:

```python
    enrichments: list[dict[str, Any]] = []
    # ... inside the loop, after building enrichment:
    enrichments.append(enrichment)
```

- [ ] **Step 3: Update planner_stream.py to forward enrichment metadata in citation events**

In `planner_stream.py`, the call to `_extract_citations` currently returns just citations:

```python
citations = _extract_citations(current_text, citation_map, all_tool_calls, all_tool_results)
```

Update to unpack both:

```python
citations, citation_metadata = _extract_citations(current_text, citation_map, all_tool_calls, all_tool_results)
```

Then update the citation event yielding loop (around line 336) to merge in the metadata:

```python
        for c, meta in zip(citations, citation_metadata):
            yield StreamEvent(
                "citation",
                {
                    "marker": c.marker,
                    "tool_result_id": str(c.tool_result_id),
                    "tool_name": c.tool_name,
                    "source_type": c.source_type,
                    "excerpt": c.excerpt,
                    **meta,
                },
            )
```

And update the `done` event's citations array the same way:

```python
                "citations": [
                    {
                        "marker": c.marker,
                        "tool_result_id": str(c.tool_result_id),
                        "tool_name": c.tool_name,
                        "source_type": c.source_type,
                        "excerpt": c.excerpt,
                        **meta,
                    }
                    for c, meta in zip(citations, citation_metadata)
                ],
```

- [ ] **Step 4: Update the synchronous planner.py to also use the new return type**

In `planner.py`, find where `_extract_citations` is called (the synchronous `run_planner` function). Update to unpack both values:

```python
citations, _metadata = _extract_citations(...)
```

The synchronous endpoint doesn't need the metadata (it returns citations in a JSON response that uses the Citation model), so we discard it.

- [ ] **Step 5: Add the `Any` import if missing**

Make sure `from typing import Any` is imported in `planner.py` (it likely already is — check before adding).

- [ ] **Step 6: Run existing tests**

```bash
cd apps/api && python -m pytest tests/unit/intelligence/ -v
```

Expected: All existing tests pass. The changes are additive — `_extract_citations` returns an extra tuple element, but existing code that only unpacks the first element will break, which is why Step 4 is critical.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/intelligence/application/planner.py apps/api/src/prescient/intelligence/application/planner_stream.py
git commit -m "feat(api): enrich citation events with source_type and structured metadata"
```

---

### Task 5: Extend frontend Citation type

**Files:**
- Modify: `apps/web/src/lib/chat-stream.ts`

- [ ] **Step 1: Update the Citation type**

Replace the existing `Citation` type:

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

- [ ] **Step 2: Update the ChatEvent citation type to match**

The `citation` variant in `ChatEvent` should use the same shape. Replace:

```typescript
  | { type: "citation"; marker: number; tool_name: string; source_type: string; excerpt: string }
```

with:

```typescript
  | (Citation & { type: "citation" })
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/lib/chat-stream.ts
git commit -m "feat(web): extend Citation type with optional rich metadata fields"
```

---

### Task 6: Rebuild the source panel with type-specific renderers

**Files:**
- Modify: `apps/web/src/components/source-panel.tsx`

- [ ] **Step 1: Update the SourceCitation interface and imports**

Replace the existing `SourceCitation` interface and update imports:

```tsx
"use client";

import Link from "next/link";

import type { Citation } from "@/lib/chat-stream";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";

interface SourcePanelProps {
  citation: Citation | null;
  companySlug: string;
  onClose: () => void;
}
```

This replaces the old `SourceCitation` interface with the rich `Citation` type from `chat-stream.ts`.

- [ ] **Step 2: Keep the existing badge and helper functions, update for undefined safety**

Keep `sourceTypeBadge`, `getBadgeConfig`, `humanizeToolName` as they are (they already handle `undefined` from the earlier fix). Remove `getNavigationLink` — we'll replace it with type-specific rendering.

- [ ] **Step 3: Add type-specific renderer components**

Add these components inside `source-panel.tsx`, above the `SourcePanel` export:

```tsx
function formatKpiLabel(id: string): string {
  return id.replace(/_/g, " ").replace(/\b\w/g, (c) => c.toUpperCase());
}

function MiniSparkline({ points }: { points: { period_label: string; value: number }[] }) {
  if (points.length < 2) return null;
  const values = points.map((p) => p.value);
  const min = Math.min(...values);
  const max = Math.max(...values);
  const range = max - min || 1;
  const width = 200;
  const height = 40;
  const coords = values.map((v, i) => {
    const x = (i / (values.length - 1)) * width;
    const y = height - ((v - min) / range) * height;
    return `${x},${y}`;
  });
  return (
    <svg width={width} height={height} className="mt-2 overflow-visible">
      <polyline
        points={coords.join(" ")}
        fill="none"
        stroke="#6366f1"
        strokeWidth={2}
        strokeLinejoin="round"
        strokeLinecap="round"
      />
    </svg>
  );
}

function DocumentDetail({ citation, companySlug }: { citation: Citation; companySlug: string }) {
  return (
    <>
      {citation.document_title && (
        <div className="mb-4">
          <p className="mb-1 text-xs font-medium uppercase tracking-wider text-neutral-400">
            Document
          </p>
          <p className="text-sm font-medium text-neutral-900">{citation.document_title}</p>
          <div className="mt-1.5 flex flex-wrap items-center gap-2">
            {citation.document_type && (
              <Badge className="bg-blue-100 text-xs text-blue-700 hover:bg-blue-100">
                {citation.document_type}
              </Badge>
            )}
            {citation.section_name && (
              <span className="text-xs text-neutral-500">{citation.section_name}</span>
            )}
            {citation.published_at && (
              <span className="text-xs text-neutral-400">
                {new Date(citation.published_at).toLocaleDateString()}
              </span>
            )}
          </div>
        </div>
      )}
      <p className="mb-1 text-xs font-medium uppercase tracking-wider text-neutral-400">Excerpt</p>
      <blockquote className="rounded-md border-l-2 border-indigo-300 bg-neutral-50 p-4 text-sm leading-relaxed text-neutral-700">
        {citation.excerpt}
      </blockquote>
      {citation.document_id && (
        <div className="mt-4">
          <Link
            href={`/companies/${companySlug}/documents/${citation.document_id}`}
            className="inline-flex items-center gap-1.5 text-sm font-medium text-indigo-600 hover:text-indigo-700"
          >
            View full document &rarr;
          </Link>
        </div>
      )}
    </>
  );
}

function ArtifactDetail({ citation, companySlug }: { citation: Citation; companySlug: string }) {
  return (
    <>
      {(citation.artifact_function || citation.confidence_label) && (
        <div className="mb-4 flex flex-wrap items-center gap-2">
          {citation.artifact_function && (
            <Badge className="bg-indigo-100 text-xs text-indigo-700 hover:bg-indigo-100">
              {citation.artifact_function}
            </Badge>
          )}
          {citation.confidence_label && (
            <Badge className="bg-neutral-100 text-xs text-neutral-600 hover:bg-neutral-100">
              {citation.confidence_label}
            </Badge>
          )}
          {citation.version_number && (
            <span className="text-xs text-neutral-400">v{citation.version_number}</span>
          )}
        </div>
      )}
      {citation.artifact_summary && (
        <div className="mb-4">
          <p className="mb-1 text-xs font-medium uppercase tracking-wider text-neutral-400">
            Summary
          </p>
          <p className="text-sm leading-relaxed text-neutral-700">{citation.artifact_summary}</p>
        </div>
      )}
      <p className="mb-1 text-xs font-medium uppercase tracking-wider text-neutral-400">Excerpt</p>
      <div className="rounded-md border border-neutral-200 bg-neutral-50 p-4">
        <p className="text-sm leading-relaxed text-neutral-700">{citation.excerpt}</p>
      </div>
      {citation.artifact_id && (
        <div className="mt-4">
          <Link
            href={`/companies/${companySlug}/artifacts/${citation.artifact_id}`}
            className="inline-flex items-center gap-1.5 text-sm font-medium text-indigo-600 hover:text-indigo-700"
          >
            View full artifact &rarr;
          </Link>
        </div>
      )}
    </>
  );
}

function KpiDetail({ citation }: { citation: Citation }) {
  return (
    <>
      {citation.kpi_id && (
        <div className="mb-4">
          <p className="mb-1 text-xs font-medium uppercase tracking-wider text-neutral-400">
            Metric
          </p>
          <p className="text-lg font-semibold text-neutral-900">
            {formatKpiLabel(citation.kpi_id)}
          </p>
        </div>
      )}
      {citation.value != null && (
        <div className="mb-4 rounded-md border border-neutral-200 bg-neutral-50 p-4">
          <p className="text-2xl font-bold text-neutral-900">
            {typeof citation.value === "number" ? citation.value.toLocaleString() : citation.value}
            {citation.unit && (
              <span className="ml-1 text-sm font-normal text-neutral-500">{citation.unit}</span>
            )}
          </p>
          {citation.period_label && (
            <p className="mt-1 text-xs text-neutral-500">{citation.period_label}</p>
          )}
          {citation.sparkline_points && citation.sparkline_points.length >= 2 && (
            <MiniSparkline points={citation.sparkline_points} />
          )}
        </div>
      )}
      <p className="mb-1 text-xs font-medium uppercase tracking-wider text-neutral-400">Context</p>
      <div className="rounded-md border border-neutral-200 bg-neutral-50 p-4">
        <p className="text-sm leading-relaxed text-neutral-700">{citation.excerpt}</p>
      </div>
    </>
  );
}

function FallbackDetail({ citation }: { citation: Citation }) {
  return (
    <>
      <p className="mb-1 text-xs font-medium uppercase tracking-wider text-neutral-400">Excerpt</p>
      <div className="rounded-md border border-neutral-200 bg-neutral-50 p-4">
        <p className="whitespace-pre-wrap text-sm leading-relaxed text-neutral-800">
          {citation.excerpt}
        </p>
      </div>
    </>
  );
}
```

- [ ] **Step 4: Rewrite the SourcePanel component**

Replace the entire `SourcePanel` export:

```tsx
export function SourcePanel({ citation, companySlug, onClose }: SourcePanelProps) {
  const badge = citation ? getBadgeConfig(citation.source_type) : null;
  const sourceType = citation?.source_type?.toLowerCase() ?? "";

  return (
    <div
      className={`fixed right-0 top-0 z-40 h-full w-[480px] border-l border-neutral-200 bg-white shadow-lg transition-transform duration-200 ${
        citation ? "translate-x-0" : "translate-x-full"
      }`}
    >
      {citation && badge && (
        <div className="flex h-full flex-col">
          {/* Header */}
          <div className="flex items-center justify-between border-b border-neutral-200 px-5 py-3">
            <div className="flex items-center gap-2">
              <span className="text-sm font-semibold text-neutral-900">
                Source [{citation.marker}]
              </span>
              <Badge className={`text-xs ${badge.className}`}>{badge.label}</Badge>
            </div>
            <Button variant="ghost" size="icon" onClick={onClose} className="h-7 w-7">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                strokeWidth="2"
                strokeLinecap="round"
                strokeLinejoin="round"
              >
                <line x1="18" y1="6" x2="6" y2="18" />
                <line x1="6" y1="6" x2="18" y2="18" />
              </svg>
            </Button>
          </div>

          {/* Content */}
          <div className="flex-1 overflow-y-auto px-5 py-5">
            {/* Tool name */}
            <p className="mb-1 text-xs font-medium uppercase tracking-wider text-neutral-400">
              Retrieved by
            </p>
            <p className="mb-4 text-sm font-medium text-neutral-700">
              {humanizeToolName(citation.tool_name)}
            </p>

            {/* Type-specific content */}
            {sourceType.includes("document") || sourceType.includes("filing") || sourceType.includes("transcript") ? (
              <DocumentDetail citation={citation} companySlug={companySlug} />
            ) : sourceType.includes("artifact") ? (
              <ArtifactDetail citation={citation} companySlug={companySlug} />
            ) : sourceType.includes("kpi") ? (
              <KpiDetail citation={citation} />
            ) : (
              <FallbackDetail citation={citation} />
            )}
          </div>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 5: Verify in browser**

Navigate to `http://localhost:3000/chat`, ask a question, wait for a cited response, click a citation marker. The source panel should now show structured metadata (document title, dates, etc.) if the API returned enriched data, or fall back gracefully to just the excerpt.

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/components/source-panel.tsx
git commit -m "feat(web): rebuild source panel with type-specific renderers for documents, artifacts, and KPIs"
```

---

### Task 7: Add tooltip to citation pills

**Files:**
- Modify: `apps/web/src/components/citation-pill.tsx`

- [ ] **Step 1: Add Radix tooltip**

The project already has `@radix-ui/react-tooltip` installed. Replace the entire file:

```tsx
"use client";

import * as Tooltip from "@radix-ui/react-tooltip";

interface CitationPillProps {
  marker: number;
  excerpt: string;
  onClick: (marker: number) => void;
}

export function CitationPill({ marker, excerpt, onClick }: CitationPillProps) {
  const preview = excerpt.length > 80 ? excerpt.slice(0, 80) + "..." : excerpt;

  return (
    <Tooltip.Provider delayDuration={300}>
      <Tooltip.Root>
        <Tooltip.Trigger asChild>
          <button
            type="button"
            onClick={() => onClick(marker)}
            className="mx-0.5 inline-flex h-5 min-w-5 items-center justify-center rounded-full bg-indigo-600 px-1.5 text-xs font-medium text-white transition-colors hover:bg-indigo-700"
          >
            {marker}
          </button>
        </Tooltip.Trigger>
        {preview && (
          <Tooltip.Portal>
            <Tooltip.Content
              side="top"
              sideOffset={4}
              className="z-50 max-w-xs rounded-md bg-neutral-900 px-3 py-2 text-xs leading-relaxed text-neutral-100 shadow-lg animate-in fade-in-0 zoom-in-95"
            >
              {preview}
              <Tooltip.Arrow className="fill-neutral-900" />
            </Tooltip.Content>
          </Tooltip.Portal>
        )}
      </Tooltip.Root>
    </Tooltip.Provider>
  );
}
```

- [ ] **Step 2: Verify in browser**

Hover over a citation pill `[1]` in a chat response. A dark tooltip should appear showing the first ~80 chars of the source excerpt.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/citation-pill.tsx
git commit -m "feat(web): add hover tooltip to citation pills showing excerpt preview"
```

---

### Task 8: Move chat to company-scoped route

**Files:**
- Create: `apps/web/src/app/(main)/companies/[slug]/chat/page.tsx`
- Modify: `apps/web/src/app/(main)/companies/[slug]/page.tsx`
- Modify: `apps/web/src/components/nav.tsx`
- Remove: `apps/web/src/app/(main)/chat/page.tsx`

- [ ] **Step 1: Create the company-scoped chat page**

```tsx
// apps/web/src/app/(main)/companies/[slug]/chat/page.tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";
import { useParams } from "next/navigation";

import { ChatInput } from "@/components/chat-input";
import { ChatMessage } from "@/components/chat-message";
import { SourcePanel } from "@/components/source-panel";
import type { ToolCall } from "@/components/tool-call-chips";
import { Button } from "@/components/ui/button";
import { getToken } from "@/lib/auth";
import type { Citation } from "@/lib/chat-stream";
import { streamChat } from "@/lib/chat-stream";

interface Message {
  role: "user" | "assistant";
  content: string;
  toolCalls?: ToolCall[];
  citations?: Citation[];
  groundingStatus?: string;
  iterations?: number;
  error?: string;
}

function formatCompanyName(slug: string): string {
  return slug
    .replace(/_/g, " ")
    .replace(/\b\w/g, (c) => c.toUpperCase());
}

const SUGGESTED_PROMPTS = [
  "What are our top risks this quarter?",
  "Summarize the competitive landscape",
  "How are key initiatives progressing?",
  "What KPI trends should I be aware of?",
];

export default function CompanyChatPage() {
  const params = useParams<{ slug: string }>();
  const slug = params.slug;
  const companyName = formatCompanyName(slug);

  const [messages, setMessages] = useState<Message[]>([]);
  const [isStreaming, setIsStreaming] = useState(false);
  const [activeSource, setActiveSource] = useState<Citation | null>(null);
  const [conversationId, setConversationId] = useState<string | undefined>();
  const [lastQuestion, setLastQuestion] = useState<string>("");
  const scrollRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (scrollRef.current) {
      scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
    }
  }, [messages]);

  const handleSend = useCallback(
    async (question: string) => {
      setLastQuestion(question);

      const userMessage: Message = { role: "user", content: question };
      const assistantMessage: Message = { role: "assistant", content: "" };

      setMessages((prev) => [...prev, userMessage, assistantMessage]);
      setIsStreaming(true);

      const updateAssistant = (updater: (prev: Message) => Message) => {
        setMessages((prev) => {
          const last = prev.at(-1);
          if (!last || last.role !== "assistant") return prev;
          return [...prev.slice(0, -1), updater(last)];
        });
      };

      try {
        const token = getToken() ?? undefined;
        for await (const event of streamChat(slug, question, conversationId, token)) {
          switch (event.type) {
            case "init":
              setConversationId(event.conversation_id);
              break;

            case "tool_start":
              updateAssistant((msg) => ({
                ...msg,
                toolCalls: [
                  ...(msg.toolCalls ?? []),
                  { id: event.tool_use_id, tool: event.tool, status: "running" as const },
                ],
              }));
              break;

            case "tool_end":
              updateAssistant((msg) => ({
                ...msg,
                toolCalls: (msg.toolCalls ?? []).map((tc) =>
                  tc.id === event.tool_use_id
                    ? {
                        ...tc,
                        status: "done" as const,
                        summary: event.summary,
                        recordCount: event.record_count,
                      }
                    : tc,
                ),
              }));
              break;

            case "delta":
              updateAssistant((msg) => ({
                ...msg,
                content: msg.content + event.text,
              }));
              break;

            case "citation":
              updateAssistant((msg) => ({
                ...msg,
                citations: [
                  ...(msg.citations ?? []),
                  event,
                ],
              }));
              break;

            case "done":
              updateAssistant((msg) => ({
                ...msg,
                groundingStatus: event.grounding_status,
                iterations: event.iterations,
              }));
              setIsStreaming(false);
              break;

            case "error":
              updateAssistant((msg) => ({
                ...msg,
                content: "",
                error: event.message,
              }));
              setIsStreaming(false);
              break;
          }
        }
      } catch (err) {
        updateAssistant((msg) => ({
          ...msg,
          content: "",
          error: err instanceof Error ? err.message : "Connection failed",
        }));
        setIsStreaming(false);
      }
    },
    [slug, conversationId],
  );

  const handleRetry = useCallback(() => {
    if (lastQuestion) {
      setMessages((prev) => prev.slice(0, -2));
      handleSend(lastQuestion);
    }
  }, [lastQuestion, handleSend]);

  const handleCitationClick = useCallback(
    (marker: number) => {
      for (const msg of messages) {
        const citation = msg.citations?.find((c) => c.marker === marker);
        if (citation) {
          setActiveSource(citation);
          return;
        }
      }
    },
    [messages],
  );

  return (
    <div className="flex h-[calc(100vh-80px)]">
      {/* Main chat area */}
      <div className="flex flex-1 flex-col">
        {/* Header */}
        <div className="border-b border-neutral-200 px-6 py-4">
          <h1 className="text-lg font-semibold tracking-tight text-neutral-900">Copilot</h1>
          <p className="text-sm text-neutral-500">{companyName}</p>
        </div>

        {/* Messages */}
        <div ref={scrollRef} className="flex-1 overflow-y-auto px-4 py-4">
          {messages.length === 0 ? (
            <div className="flex h-full flex-col items-center justify-center text-center">
              <p className="text-lg font-semibold text-neutral-900">
                Ask anything about {companyName}
              </p>
              <p className="mt-2 max-w-md text-sm text-neutral-500">
                The copilot searches company artifacts, KPIs, monitoring data, and action items to
                answer your questions with cited sources.
              </p>
              <div className="mt-6 flex flex-wrap justify-center gap-2">
                {SUGGESTED_PROMPTS.map((prompt) => (
                  <button
                    key={prompt}
                    onClick={() => handleSend(prompt)}
                    disabled={isStreaming}
                    className="rounded-full border border-neutral-200 bg-white px-4 py-2 text-sm text-neutral-600 transition-colors hover:border-neutral-300 hover:bg-neutral-50"
                  >
                    {prompt}
                  </button>
                ))}
              </div>
            </div>
          ) : (
            <div className="space-y-4">
              {messages.map((msg, i) => {
                if (msg.error) {
                  return (
                    <div key={i} className="flex justify-start">
                      <div className="max-w-[80%] rounded-lg border border-red-200 bg-red-50 px-4 py-3">
                        <p className="text-sm text-red-700">{msg.error}</p>
                        <Button variant="outline" size="sm" className="mt-2" onClick={handleRetry}>
                          Retry
                        </Button>
                      </div>
                    </div>
                  );
                }
                return (
                  <ChatMessage
                    key={i}
                    role={msg.role}
                    content={msg.content}
                    toolCalls={msg.toolCalls}
                    citations={msg.citations}
                    groundingStatus={msg.groundingStatus}
                    iterations={msg.iterations}
                    onCitationClick={handleCitationClick}
                  />
                );
              })}
            </div>
          )}
        </div>

        {/* Input */}
        <ChatInput onSend={handleSend} disabled={isStreaming} />
      </div>

      {/* Source panel */}
      <SourcePanel
        citation={activeSource}
        companySlug={slug}
        onClose={() => setActiveSource(null)}
      />
    </div>
  );
}
```

Key differences from the old chat page:
- `slug` comes from `useParams()` instead of being hardcoded
- `companyName` is derived from slug via `formatCompanyName`
- Citation event handling stores the full event object (which now includes rich metadata) instead of picking fields
- `companySlug={slug}` on `SourcePanel` uses the dynamic slug
- Suggested prompts are generic (not Peloton-specific)

- [ ] **Step 2: Update the company overview page link**

In `apps/web/src/app/(main)/companies/[slug]/page.tsx`, update the "Ask about this company" button (around line 84):

Replace:
```tsx
        <Button variant="outline" size="sm" asChild>
          <Link href="/chat">Ask about this company</Link>
        </Button>
```

With:
```tsx
        <Button variant="outline" size="sm" asChild>
          <Link href={`/companies/${slug}/chat`}>Ask about this company</Link>
        </Button>
```

- [ ] **Step 3: Update the nav**

In `apps/web/src/components/nav.tsx`, remove the Chat link from `NAV_LINKS`. Change:

```typescript
const NAV_LINKS = [
  { href: "/brief",      label: "Morning Brief" },
  { href: "/onboarding", label: "Setup" },
  { href: "/triage",     label: "Triage" },
  { href: "/actions",    label: "Actions" },
  { href: "/chat",       label: "Chat" },
  { href: "/board-prep", label: "Board Prep" },
];
```

To:

```typescript
const NAV_LINKS = [
  { href: "/brief",      label: "Morning Brief" },
  { href: "/onboarding", label: "Setup" },
  { href: "/triage",     label: "Triage" },
  { href: "/actions",    label: "Actions" },
  { href: "/board-prep", label: "Board Prep" },
];
```

- [ ] **Step 4: Delete the old chat page**

```bash
rm apps/web/src/app/\(main\)/chat/page.tsx
```

If there's a `chat/` directory with only `page.tsx`, remove the directory:

```bash
rmdir apps/web/src/app/\(main\)/chat/ 2>/dev/null || true
```

- [ ] **Step 5: Verify in browser**

1. Navigate to `http://localhost:3000/companies/peloton` — the "Ask about this company" button should link to `/companies/peloton/chat`
2. Click through to `/companies/peloton/chat` — the header should say "Copilot" / "Peloton"
3. The top nav should no longer have a "Chat" link
4. Send a message — should stream and render correctly

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/app/\(main\)/companies/\[slug\]/chat/page.tsx apps/web/src/app/\(main\)/companies/\[slug\]/page.tsx apps/web/src/components/nav.tsx
git rm apps/web/src/app/\(main\)/chat/page.tsx
git commit -m "feat(web): move chat to company-scoped route at /companies/{slug}/chat"
```

---

### Task 9: Improve streaming indicator

**Files:**
- Modify: `apps/web/src/components/chat-message.tsx`

- [ ] **Step 1: Add a "Researching..." label for the pre-text streaming phase**

In `chat-message.tsx`, the assistant bubble currently shows a thin blinking cursor when `content` is empty:

```tsx
<span className="inline-block h-4 w-1 animate-pulse bg-neutral-400" />
```

Replace it with a more informative indicator:

```tsx
<span className="inline-flex items-center gap-1.5 text-neutral-500">
  <span className="inline-block h-1.5 w-1.5 animate-pulse rounded-full bg-indigo-500" />
  Researching...
</span>
```

- [ ] **Step 2: Verify in browser**

Send a message. During the tool-execution phase (before text starts streaming), the bubble should show "Researching..." with a pulsing dot instead of just a thin cursor.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/chat-message.tsx
git commit -m "feat(web): show 'Researching...' indicator during tool execution phase"
```
