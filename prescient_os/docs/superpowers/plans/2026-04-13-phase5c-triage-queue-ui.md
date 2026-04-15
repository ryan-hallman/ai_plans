# Phase 5c: Triage Queue UI + Operator Brainstorm Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the operator-facing triage queue page at `/triage` with a two-panel layout (queue list + detail), brainstorm chat integration, and triage item type-specific actions. Add triage nav link.

**Architecture:** Next.js client component page using the existing BFF pattern. Left panel shows ranked TriageItems, right panel shows detail with actions. Brainstorm opens the existing copilot chat with triage-specific system prompt. Uses the triage API from Phase 5b.

**Tech Stack:** Next.js 15, React 19, TypeScript, Tailwind CSS, shadcn/ui

**Depends on:** Phase 5b (triage API endpoints)

---

## File Structure

### New files:

```
apps/web/src/app/(main)/triage/page.tsx              — triage queue page
apps/web/src/components/triage-queue.tsx              — queue list component
apps/web/src/components/triage-detail.tsx             — detail panel component
apps/web/src/components/triage-question-detail.tsx    — sponsor question detail
apps/web/src/components/triage-anomaly-detail.tsx     — KPI anomaly detail
apps/web/src/components/triage-finding-detail.tsx     — monitoring finding detail
apps/web/src/components/triage-action-detail.tsx      — action item review detail
```

### Modified files:

```
apps/web/src/lib/api.ts                              — add triage API methods
apps/web/src/components/nav.tsx                       — add triage nav link
```

---

### Task 1: Triage API Client Methods

**Files:**
- Modify: `apps/web/src/lib/api.ts`

- [ ] **Step 1: Add triage types and API methods**

Open `apps/web/src/lib/api.ts` and add these types after the existing type definitions:

```typescript
export type TriageItemResponse = {
  id: string;
  organization_id: string;
  item_type: string;
  source_id: string;
  title: string;
  summary: string;
  priority: string;
  priority_signals: string[];
  status: string;
  assigned_to_user_id: string | null;
  created_at: string;
  resolved_at: string | null;
};

export type TriageQuestionResponse = {
  id: string;
  asker_user_id: string;
  asker_org_id: string;
  target_company_org_id: string;
  question_text: string;
  categories_detected: string[];
  status: string;
  partial_answer: string | null;
  escalation_reason: string | null;
  final_answer: string | null;
  priority: string;
  created_at: string;
  answered_at: string | null;
};

export type SharedItemResponse = {
  id: string;
  artifact_id: string;
  sponsor_org_id: string;
  shared_by_user_id: string;
  shared_at: string;
  triage_question_id: string | null;
};
```

Add these methods to the `api` object:

```typescript
  triageItems: (params?: { item_type?: string; priority?: string; status?: string }) => {
    const qs = new URLSearchParams();
    if (params?.item_type) qs.set("item_type", params.item_type);
    if (params?.priority) qs.set("priority", params.priority);
    if (params?.status) qs.set("status", params.status);
    const query = qs.toString() ? `?${qs.toString()}` : "";
    return getJSON<TriageItemResponse[]>(`/triage/items${query}`);
  },
  triageQuestion: (id: string) => getJSON<TriageQuestionResponse>(`/triage/questions/${id}`),
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/lib/api.ts
git commit -m "feat(web): add triage types and API client methods"
```

---

### Task 2: Triage Queue List Component

**Files:**
- Create: `apps/web/src/components/triage-queue.tsx`

- [ ] **Step 1: Implement triage queue list**

```typescript
// apps/web/src/components/triage-queue.tsx
"use client";

import { cn } from "@/lib/utils";
import type { TriageItemResponse } from "@/lib/api";

const priorityColor: Record<string, string> = {
  critical: "bg-red-600 text-white",
  high: "bg-orange-500 text-white",
  medium: "bg-yellow-500 text-white",
  low: "bg-gray-400 text-white",
};

const typeIcon: Record<string, string> = {
  sponsor_question: "💬",
  kpi_anomaly: "📊",
  monitoring_finding: "🛡️",
  action_review: "✅",
  system_alert: "⚠️",
};

function timeAgo(dateStr: string): string {
  const diff = Date.now() - new Date(dateStr).getTime();
  const hours = Math.floor(diff / 3600000);
  if (hours < 1) return "just now";
  if (hours < 24) return `${hours}h ago`;
  const days = Math.floor(hours / 24);
  return `${days}d ago`;
}

interface TriageQueueProps {
  items: TriageItemResponse[];
  selectedId: string | null;
  onSelect: (id: string) => void;
  filter: { type: string | null; priority: string | null; status: string | null };
  onFilterChange: (filter: { type: string | null; priority: string | null; status: string | null }) => void;
}

export function TriageQueue({ items, selectedId, onSelect, filter, onFilterChange }: TriageQueueProps) {
  return (
    <div className="flex flex-col h-full">
      {/* Filters */}
      <div className="flex gap-2 p-3 border-b border-neutral-200">
        <select
          className="text-xs rounded border border-neutral-300 px-2 py-1"
          value={filter.type ?? ""}
          onChange={(e) => onFilterChange({ ...filter, type: e.target.value || null })}
        >
          <option value="">All types</option>
          <option value="sponsor_question">Questions</option>
          <option value="kpi_anomaly">KPI Anomalies</option>
          <option value="monitoring_finding">Findings</option>
          <option value="action_review">Action Items</option>
        </select>
        <select
          className="text-xs rounded border border-neutral-300 px-2 py-1"
          value={filter.priority ?? ""}
          onChange={(e) => onFilterChange({ ...filter, priority: e.target.value || null })}
        >
          <option value="">All priorities</option>
          <option value="critical">Critical</option>
          <option value="high">High</option>
          <option value="medium">Medium</option>
          <option value="low">Low</option>
        </select>
        <select
          className="text-xs rounded border border-neutral-300 px-2 py-1"
          value={filter.status ?? ""}
          onChange={(e) => onFilterChange({ ...filter, status: e.target.value || null })}
        >
          <option value="">Pending + In Progress</option>
          <option value="pending">Pending</option>
          <option value="in_progress">In Progress</option>
          <option value="resolved">Resolved</option>
          <option value="dismissed">Dismissed</option>
        </select>
      </div>

      {/* Item list */}
      <div className="flex-1 overflow-y-auto">
        {items.length === 0 && (
          <div className="p-8 text-center text-neutral-400 text-sm">No items in queue</div>
        )}
        {items.map((item) => (
          <button
            key={item.id}
            onClick={() => onSelect(item.id)}
            className={cn(
              "w-full text-left p-3 border-b border-neutral-100 hover:bg-neutral-50 transition-colors",
              selectedId === item.id && "bg-blue-50 border-l-2 border-l-blue-500"
            )}
          >
            <div className="flex items-start gap-2">
              <span className="text-lg" title={item.item_type}>
                {typeIcon[item.item_type] ?? "📋"}
              </span>
              <div className="flex-1 min-w-0">
                <div className="flex items-center gap-2 mb-0.5">
                  <span
                    className={cn(
                      "text-[10px] font-bold uppercase px-1.5 py-0.5 rounded",
                      priorityColor[item.priority] ?? "bg-gray-300"
                    )}
                  >
                    {item.priority}
                  </span>
                  <span className="text-[10px] text-neutral-400">{timeAgo(item.created_at)}</span>
                  <span
                    className={cn(
                      "ml-auto text-[10px] px-1.5 py-0.5 rounded",
                      item.status === "pending"
                        ? "bg-amber-100 text-amber-700"
                        : item.status === "in_progress"
                        ? "bg-blue-100 text-blue-700"
                        : item.status === "resolved"
                        ? "bg-green-100 text-green-700"
                        : "bg-neutral-100 text-neutral-500"
                    )}
                  >
                    {item.status.replace("_", " ")}
                  </span>
                </div>
                <p className="text-sm font-medium text-neutral-800 truncate">{item.title}</p>
                <p className="text-xs text-neutral-500 truncate mt-0.5">{item.summary}</p>
                {item.priority_signals.length > 0 && (
                  <p className="text-[10px] text-neutral-400 mt-0.5">
                    {item.priority_signals.join(" · ")}
                  </p>
                )}
              </div>
            </div>
          </button>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/triage-queue.tsx
git commit -m "feat(web): add TriageQueue list component with filters and priority badges"
```

---

### Task 3: Triage Detail Components

**Files:**
- Create: `apps/web/src/components/triage-detail.tsx`
- Create: `apps/web/src/components/triage-question-detail.tsx`

- [ ] **Step 1: Implement triage detail router**

```typescript
// apps/web/src/components/triage-detail.tsx
"use client";

import { useEffect, useState } from "react";
import type { TriageItemResponse, TriageQuestionResponse } from "@/lib/api";
import { getToken } from "@/lib/auth";
import { TriageQuestionDetail } from "./triage-question-detail";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

interface TriageDetailProps {
  item: TriageItemResponse;
  onStatusChange: () => void;
}

export function TriageDetail({ item, onStatusChange }: TriageDetailProps) {
  if (item.item_type === "sponsor_question") {
    return (
      <TriageQuestionDetail
        item={item}
        onStatusChange={onStatusChange}
      />
    );
  }

  // Generic detail for other types
  return (
    <div className="p-6">
      <div className="flex items-center gap-3 mb-4">
        <h2 className="text-lg font-semibold text-neutral-800">{item.title}</h2>
        <span className="text-xs px-2 py-0.5 bg-neutral-100 rounded capitalize">
          {item.item_type.replace("_", " ")}
        </span>
      </div>
      <p className="text-sm text-neutral-600 mb-6">{item.summary}</p>

      <div className="flex gap-2">
        {item.status === "pending" && (
          <>
            <ActionButton
              label="Start"
              action="start"
              itemId={item.id}
              onDone={onStatusChange}
              variant="secondary"
            />
            <ActionButton
              label="Resolve"
              action="resolve"
              itemId={item.id}
              onDone={onStatusChange}
              variant="primary"
            />
            <ActionButton
              label="Dismiss"
              action="dismiss"
              itemId={item.id}
              onDone={onStatusChange}
              variant="ghost"
            />
          </>
        )}
        {item.status === "in_progress" && (
          <>
            <ActionButton
              label="Resolve"
              action="resolve"
              itemId={item.id}
              onDone={onStatusChange}
              variant="primary"
            />
            <ActionButton
              label="Dismiss"
              action="dismiss"
              itemId={item.id}
              onDone={onStatusChange}
              variant="ghost"
            />
          </>
        )}
      </div>
    </div>
  );
}

function ActionButton({
  label,
  action,
  itemId,
  onDone,
  variant,
}: {
  label: string;
  action: string;
  itemId: string;
  onDone: () => void;
  variant: "primary" | "secondary" | "ghost";
}) {
  const [loading, setLoading] = useState(false);

  const handleClick = async () => {
    setLoading(true);
    try {
      const token = getToken();
      await fetch(`${API}/triage/items/${itemId}`, {
        method: "PATCH",
        headers: {
          "Content-Type": "application/json",
          ...(token ? { Authorization: `Bearer ${token}` } : {}),
        },
        body: JSON.stringify({ action }),
      });
      onDone();
    } finally {
      setLoading(false);
    }
  };

  const cls =
    variant === "primary"
      ? "bg-blue-600 text-white hover:bg-blue-700"
      : variant === "secondary"
      ? "bg-neutral-200 text-neutral-800 hover:bg-neutral-300"
      : "text-neutral-500 hover:text-neutral-700";

  return (
    <button
      onClick={handleClick}
      disabled={loading}
      className={`px-3 py-1.5 text-sm rounded font-medium transition-colors disabled:opacity-50 ${cls}`}
    >
      {loading ? "..." : label}
    </button>
  );
}

export { ActionButton };
```

- [ ] **Step 2: Implement sponsor question detail**

```typescript
// apps/web/src/components/triage-question-detail.tsx
"use client";

import { useEffect, useState } from "react";
import type { TriageItemResponse, TriageQuestionResponse } from "@/lib/api";
import { getToken } from "@/lib/auth";
import { ActionButton } from "./triage-detail";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

interface TriageQuestionDetailProps {
  item: TriageItemResponse;
  onStatusChange: () => void;
}

export function TriageQuestionDetail({ item, onStatusChange }: TriageQuestionDetailProps) {
  const [question, setQuestion] = useState<TriageQuestionResponse | null>(null);
  const [responding, setResponding] = useState(false);
  const [responseText, setResponseText] = useState("");

  useEffect(() => {
    const fetchQuestion = async () => {
      const token = getToken();
      const res = await fetch(`${API}/triage/questions/${item.source_id}`, {
        headers: token ? { Authorization: `Bearer ${token}` } : {},
      });
      if (res.ok) {
        setQuestion(await res.json());
      }
    };
    fetchQuestion();
  }, [item.source_id]);

  const handleRespond = async () => {
    if (!responseText.trim()) return;
    setResponding(true);
    try {
      const token = getToken();
      await fetch(`${API}/triage/questions/${item.source_id}/respond`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          ...(token ? { Authorization: `Bearer ${token}` } : {}),
        },
        body: JSON.stringify({
          final_answer: responseText,
          final_citations: [],
        }),
      });
      onStatusChange();
    } finally {
      setResponding(false);
    }
  };

  if (!question) {
    return <div className="p-6 text-neutral-400 text-sm">Loading question...</div>;
  }

  return (
    <div className="p-6 space-y-4">
      {/* Header */}
      <div>
        <div className="flex items-center gap-2 mb-2">
          <span className="text-xs px-2 py-0.5 bg-purple-100 text-purple-700 rounded">
            Sponsor Question
          </span>
          <span className="text-xs text-neutral-400 capitalize">
            {question.status.replace("_", " ")}
          </span>
        </div>
        <h2 className="text-lg font-semibold text-neutral-800">
          {question.question_text}
        </h2>
        <p className="text-xs text-neutral-400 mt-1">
          From {question.asker_user_id} · {new Date(question.created_at).toLocaleDateString()}
        </p>
      </div>

      {/* Categories */}
      {question.categories_detected.length > 0 && (
        <div>
          <p className="text-xs font-medium text-neutral-500 mb-1">Categories detected:</p>
          <div className="flex gap-1 flex-wrap">
            {question.categories_detected.map((c) => (
              <span key={c} className="text-[10px] px-1.5 py-0.5 bg-neutral-100 rounded">
                {c.replace("_", " ")}
              </span>
            ))}
          </div>
        </div>
      )}

      {/* Partial answer */}
      {question.partial_answer && (
        <div className="bg-green-50 border border-green-200 rounded p-3">
          <p className="text-xs font-medium text-green-700 mb-1">Partial answer (shared with sponsor):</p>
          <p className="text-sm text-green-900">{question.partial_answer}</p>
        </div>
      )}

      {/* Escalation reason */}
      {question.escalation_reason && (
        <div className="bg-amber-50 border border-amber-200 rounded p-3">
          <p className="text-xs font-medium text-amber-700 mb-1">Escalation reason:</p>
          <p className="text-sm text-amber-900">{question.escalation_reason}</p>
        </div>
      )}

      {/* Final answer */}
      {question.final_answer && (
        <div className="bg-blue-50 border border-blue-200 rounded p-3">
          <p className="text-xs font-medium text-blue-700 mb-1">Final answer (sent to sponsor):</p>
          <p className="text-sm text-blue-900">{question.final_answer}</p>
        </div>
      )}

      {/* Actions */}
      {(question.status === "escalated" || question.status === "partially_answered") && (
        <div className="space-y-3 pt-2 border-t border-neutral-200">
          <div className="flex gap-2">
            <a
              href={`/chat?triage=${item.source_id}`}
              className="px-3 py-1.5 text-sm rounded font-medium bg-purple-600 text-white hover:bg-purple-700 transition-colors"
            >
              Open Brainstorm
            </a>
          </div>

          <div>
            <label className="text-xs font-medium text-neutral-600 block mb-1">
              Or respond directly:
            </label>
            <textarea
              value={responseText}
              onChange={(e) => setResponseText(e.target.value)}
              placeholder="Type your response..."
              rows={4}
              className="w-full rounded border border-neutral-300 p-2 text-sm focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
            />
            <button
              onClick={handleRespond}
              disabled={responding || !responseText.trim()}
              className="mt-2 px-3 py-1.5 text-sm rounded font-medium bg-blue-600 text-white hover:bg-blue-700 disabled:opacity-50 transition-colors"
            >
              {responding ? "Sending..." : "Approve & Send Response"}
            </button>
          </div>
        </div>
      )}

      {/* Generic actions for non-question states */}
      {item.status === "pending" && question.status !== "escalated" && question.status !== "partially_answered" && (
        <div className="flex gap-2 pt-2 border-t border-neutral-200">
          <ActionButton label="Resolve" action="resolve" itemId={item.id} onDone={onStatusChange} variant="primary" />
          <ActionButton label="Dismiss" action="dismiss" itemId={item.id} onDone={onStatusChange} variant="ghost" />
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/triage-detail.tsx apps/web/src/components/triage-question-detail.tsx
git commit -m "feat(web): add triage detail components — question detail with brainstorm + respond"
```

---

### Task 4: Triage Queue Page

**Files:**
- Create: `apps/web/src/app/(main)/triage/page.tsx`

- [ ] **Step 1: Implement triage page**

```typescript
// apps/web/src/app/(main)/triage/page.tsx
"use client";

import { useCallback, useEffect, useState } from "react";
import { getToken, getOrgId } from "@/lib/auth";
import type { TriageItemResponse } from "@/lib/api";
import { TriageQueue } from "@/components/triage-queue";
import { TriageDetail } from "@/components/triage-detail";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

export default function TriagePage() {
  const [items, setItems] = useState<TriageItemResponse[]>([]);
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const [filter, setFilter] = useState<{
    type: string | null;
    priority: string | null;
    status: string | null;
  }>({ type: null, priority: null, status: null });

  const fetchItems = useCallback(async () => {
    const token = getToken();
    const params = new URLSearchParams();
    if (filter.type) params.set("item_type", filter.type);
    if (filter.priority) params.set("priority", filter.priority);
    if (filter.status) params.set("status", filter.status);
    const query = params.toString() ? `?${params.toString()}` : "";

    try {
      const res = await fetch(`${API}/triage/items${query}`, {
        headers: token ? { Authorization: `Bearer ${token}` } : {},
      });
      if (res.ok) {
        const data = await res.json();
        setItems(data);
      }
    } finally {
      setLoading(false);
    }
  }, [filter]);

  useEffect(() => {
    fetchItems();
  }, [fetchItems]);

  const selectedItem = items.find((i) => i.id === selectedId) ?? null;

  return (
    <div className="flex h-[calc(100vh-4rem)]">
      {/* Left panel: queue */}
      <div className="w-[400px] border-r border-neutral-200 bg-white flex flex-col">
        <div className="p-3 border-b border-neutral-200">
          <h1 className="text-lg font-semibold text-neutral-800">Triage Queue</h1>
          <p className="text-xs text-neutral-400 mt-0.5">
            {items.length} item{items.length !== 1 ? "s" : ""} needing attention
          </p>
        </div>
        {loading ? (
          <div className="p-8 text-center text-neutral-400 text-sm">Loading...</div>
        ) : (
          <TriageQueue
            items={items}
            selectedId={selectedId}
            onSelect={setSelectedId}
            filter={filter}
            onFilterChange={setFilter}
          />
        )}
      </div>

      {/* Right panel: detail */}
      <div className="flex-1 bg-white overflow-y-auto">
        {selectedItem ? (
          <TriageDetail item={selectedItem} onStatusChange={fetchItems} />
        ) : (
          <div className="flex items-center justify-center h-full text-neutral-400 text-sm">
            Select an item to view details
          </div>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/app/\(main\)/triage/
git commit -m "feat(web): add triage queue page — two-panel layout with filters and detail"
```

---

### Task 5: Navigation Link

**Files:**
- Modify: `apps/web/src/components/nav.tsx`

- [ ] **Step 1: Add triage link to navigation**

Open `apps/web/src/components/nav.tsx` and add a "Triage" link alongside the existing navigation items. Find the nav link pattern (likely a list of `<Link>` components) and add:

```typescript
{ href: "/triage", label: "Triage" },
```

Place it after "Brief" and before "Actions" in the navigation order, since triage is the operator's attention management layer that sits between morning brief and action tracking.

- [ ] **Step 2: Verify in browser**

Start the dev server (`cd infrastructure && docker compose up`) and navigate to `http://localhost:3000/triage`. Verify:
- Triage link appears in nav
- Page loads with queue layout
- Empty state shows "No items in queue"

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/nav.tsx
git commit -m "feat(web): add triage link to main navigation"
```

---

### Task 6: Verify Full Integration

**Files:** None (verification only)

- [ ] **Step 1: TypeScript check**

Run: `cd apps/web && npx tsc --noEmit`
Expected: No type errors in new files.

- [ ] **Step 2: Lint check**

Run: `cd apps/web && npx eslint src/app/\(main\)/triage/ src/components/triage-*.tsx`
Expected: No lint errors.

- [ ] **Step 3: Fix any issues and commit**

```bash
git add -A
git commit -m "fix: resolve lint/type issues from Phase 5c"
```

---
