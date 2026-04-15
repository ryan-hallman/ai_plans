# Persistent Copilot Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the buried `/companies/[slug]/chat` page with a persistent slide-over copilot panel available on every page, grounded on the user's home organization.

**Architecture:** A Zustand store holds panel state, conversation messages, and page context. The panel lives in the main layout and survives navigation. The backend stream endpoint gets new optional fields (`focus_company_slug`, `page_context`, `selected_item_summary`) and resolves the user's home org for dual-scope grounding. The existing `ChatMessage`, `ChatInput`, and streaming infrastructure are reused.

**Tech Stack:** Next.js 14 (App Router), React, Zustand, Tailwind CSS, FastAPI, SQLAlchemy, Anthropic API (streaming SSE)

---

### Task 1: Copilot Zustand Store

**Files:**
- Create: `apps/web/src/stores/copilot-store.ts`
- Test: Manual verification (Zustand stores are tested through component integration)

This store holds all panel and conversation state. Other tasks depend on it.

- [ ] **Step 1: Create the copilot store**

```typescript
// apps/web/src/stores/copilot-store.ts
import { create } from "zustand";
import type { Citation, ToolCallLog } from "@/lib/chat-stream";

/* -------------------------------------------------------------------------- */
/*  Types                                                                     */
/* -------------------------------------------------------------------------- */

export type PageType =
  | "brief"
  | "triage"
  | "actions"
  | "board-prep"
  | "knowledge"
  | "strategy"
  | "company-detail"
  | "company-kpis";

export interface PageContext {
  pageType: PageType;
  companySlug?: string;
  selectedItemId?: string;
  selectedItemSummary?: string;
}

export interface ToolCall {
  id: string;
  tool: string;
  status: "running" | "done";
  summary?: string;
  recordCount?: number;
}

export interface CopilotMessage {
  role: "user" | "assistant";
  content: string;
  toolCalls?: ToolCall[];
  citations?: Citation[];
  groundingStatus?: string;
  iterations?: number;
  error?: string;
}

/* -------------------------------------------------------------------------- */
/*  Store                                                                     */
/* -------------------------------------------------------------------------- */

interface CopilotState {
  /* Panel */
  isOpen: boolean;
  toggle: () => void;
  open: () => void;
  close: () => void;

  /* Conversation */
  conversationId: string | undefined;
  messages: CopilotMessage[];
  isStreaming: boolean;
  lastQuestion: string;

  setConversationId: (id: string) => void;
  addUserMessage: (content: string) => void;
  addAssistantMessage: () => void;
  updateAssistant: (updater: (msg: CopilotMessage) => CopilotMessage) => void;
  setIsStreaming: (v: boolean) => void;
  setLastQuestion: (q: string) => void;
  newConversation: () => void;
  retryLast: () => void;

  /* Page context */
  pageContext: PageContext | null;
  setPageContext: (ctx: PageContext | null) => void;
}

export const useCopilotStore = create<CopilotState>((set, get) => ({
  /* Panel */
  isOpen: false,
  toggle: () => set((s) => ({ isOpen: !s.isOpen })),
  open: () => set({ isOpen: true }),
  close: () => set({ isOpen: false }),

  /* Conversation */
  conversationId: undefined,
  messages: [],
  isStreaming: false,
  lastQuestion: "",

  setConversationId: (id) => set({ conversationId: id }),
  addUserMessage: (content) =>
    set((s) => ({
      messages: [...s.messages, { role: "user", content }],
      lastQuestion: content,
    })),
  addAssistantMessage: () =>
    set((s) => ({
      messages: [...s.messages, { role: "assistant", content: "" }],
    })),
  updateAssistant: (updater) =>
    set((s) => {
      const last = s.messages.at(-1);
      if (!last || last.role !== "assistant") return s;
      return { messages: [...s.messages.slice(0, -1), updater(last)] };
    }),
  setIsStreaming: (v) => set({ isStreaming: v }),
  setLastQuestion: (q) => set({ lastQuestion: q }),
  newConversation: () =>
    set({ conversationId: undefined, messages: [], isStreaming: false, lastQuestion: "" }),
  retryLast: () => {
    const { lastQuestion } = get();
    if (!lastQuestion) return;
    set((s) => ({ messages: s.messages.slice(0, -2) }));
  },

  /* Page context */
  pageContext: null,
  setPageContext: (ctx) => set({ pageContext: ctx }),
}));
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit --pretty 2>&1 | head -20`
Expected: No errors related to `copilot-store.ts`

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/stores/copilot-store.ts
git commit -m "feat(copilot): add Zustand store for persistent copilot panel"
```

---

### Task 2: useCopilotContext Hook

**Files:**
- Create: `apps/web/src/hooks/use-copilot-context.ts`

This hook lets pages register their context with the copilot store. It sets context on mount and clears it on unmount.

- [ ] **Step 1: Create the hook**

```typescript
// apps/web/src/hooks/use-copilot-context.ts
import { useEffect } from "react";
import { useCopilotStore, type PageContext } from "@/stores/copilot-store";

/**
 * Register the current page's context with the copilot panel.
 * Context is set on mount and cleared on unmount.
 */
export function useCopilotContext(ctx: PageContext): void {
  const setPageContext = useCopilotStore((s) => s.setPageContext);

  useEffect(() => {
    setPageContext(ctx);
    return () => setPageContext(null);
    // Re-register when any context field changes
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [ctx.pageType, ctx.companySlug, ctx.selectedItemId, ctx.selectedItemSummary, setPageContext]);
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit --pretty 2>&1 | head -20`
Expected: No errors related to `use-copilot-context.ts`

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/hooks/use-copilot-context.ts
git commit -m "feat(copilot): add useCopilotContext hook for page context registration"
```

---

### Task 3: Update `streamChat` to Accept New Fields

**Files:**
- Modify: `apps/web/src/lib/chat-stream.ts`

The streaming client needs to forward `focus_company_slug`, `page_context`, and `selected_item_summary` to the backend. The `company_slug` parameter becomes optional.

- [ ] **Step 1: Update the `streamChat` function signature and body**

In `apps/web/src/lib/chat-stream.ts`, replace the existing `streamChat` function (lines 44-95) with:

```typescript
export interface StreamChatOptions {
  question: string;
  conversationId?: string;
  authToken?: string;
  companySlug?: string;
  focusCompanySlug?: string;
  pageContext?: string;
  selectedItemSummary?: string;
}

export async function* streamChat(
  options: StreamChatOptions,
): AsyncGenerator<ChatEvent> {
  const headers: Record<string, string> = { "Content-Type": "application/json" };
  if (options.authToken) headers["Authorization"] = `Bearer ${options.authToken}`;

  const body: Record<string, unknown> = {
    question: options.question,
  };
  if (options.conversationId) body.conversation_id = options.conversationId;
  if (options.companySlug) body.company_slug = options.companySlug;
  if (options.focusCompanySlug) body.focus_company_slug = options.focusCompanySlug;
  if (options.pageContext) body.page_context = options.pageContext;
  if (options.selectedItemSummary) body.selected_item_summary = options.selectedItemSummary;

  const res = await fetch("/api/chat", {
    method: "POST",
    headers,
    body: JSON.stringify(body),
  });

  if (!res.ok || !res.body) {
    yield { type: "error", message: `HTTP ${res.status}` };
    return;
  }

  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split("\n");
    buffer = lines.pop() ?? "";

    let currentEvent = "";
    for (const line of lines) {
      if (line.startsWith("event: ")) {
        currentEvent = line.slice(7);
      } else if (line.startsWith("data: ") && currentEvent) {
        try {
          const data = JSON.parse(line.slice(6));
          yield { type: currentEvent, ...data } as ChatEvent;
        } catch {
          // skip malformed JSON
        }
        currentEvent = "";
      }
    }
  }
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit --pretty 2>&1 | head -20`
Expected: Compile errors in `companies/[slug]/chat/page.tsx` (old call signature) — this is expected and will be resolved when we remove that file in Task 7.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/lib/chat-stream.ts
git commit -m "feat(copilot): update streamChat to accept options object with new context fields"
```

---

### Task 4: Suggested Prompts Component

**Files:**
- Create: `apps/web/src/components/copilot/suggested-prompts.tsx`

Static mapping of page type to 2-3 suggested prompts.

- [ ] **Step 1: Create the component**

```typescript
// apps/web/src/components/copilot/suggested-prompts.tsx
"use client";

import type { PageType } from "@/stores/copilot-store";

const PROMPTS: Record<PageType, string[]> = {
  brief: [
    "What should I focus on today?",
    "Summarize this morning's key developments",
  ],
  triage: [
    "Which triage items are highest priority?",
    "Summarize the open questions",
  ],
  actions: [
    "Summarize my overdue items",
    "What should I prioritize this week?",
  ],
  "board-prep": [
    "What are the key themes for this quarter's board deck?",
    "Suggest talking points for the exec summary slide",
  ],
  knowledge: [
    "What documents were added recently?",
    "Search for competitive analysis",
  ],
  strategy: [
    "How are my strategic initiatives progressing?",
    "Which milestones are at risk?",
  ],
  "company-detail": [
    "What are the top risks for this company?",
    "Summarize the competitive landscape",
  ],
  "company-kpis": [
    "What KPI trends should I be aware of?",
    "Which metrics are underperforming?",
  ],
};

const DEFAULT_PROMPTS = [
  "What should I focus on today?",
  "How are my strategic initiatives progressing?",
];

interface SuggestedPromptsProps {
  pageType: PageType | null;
  onSelect: (prompt: string) => void;
  disabled: boolean;
}

export function SuggestedPrompts({ pageType, onSelect, disabled }: SuggestedPromptsProps) {
  const prompts = pageType ? (PROMPTS[pageType] ?? DEFAULT_PROMPTS) : DEFAULT_PROMPTS;

  return (
    <div className="flex flex-wrap gap-2">
      {prompts.map((prompt) => (
        <button
          key={prompt}
          onClick={() => onSelect(prompt)}
          disabled={disabled}
          className="rounded-full border border-neutral-200 bg-white px-3 py-1.5 text-xs text-neutral-600 transition-colors hover:border-neutral-300 hover:bg-neutral-50 disabled:opacity-50"
        >
          {prompt}
        </button>
      ))}
    </div>
  );
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit --pretty 2>&1 | grep suggested-prompts`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/copilot/suggested-prompts.tsx
git commit -m "feat(copilot): add SuggestedPrompts component with per-page prompt mapping"
```

---

### Task 5: Context Chip Component

**Files:**
- Create: `apps/web/src/components/copilot/context-chip.tsx`

Small indicator above the input showing the currently selected item.

- [ ] **Step 1: Create the component**

```typescript
// apps/web/src/components/copilot/context-chip.tsx
"use client";

interface ContextChipProps {
  summary: string;
  onClear: () => void;
}

export function ContextChip({ summary, onClear }: ContextChipProps) {
  return (
    <div className="flex items-center gap-2 rounded-md border border-indigo-200 bg-indigo-50 px-3 py-1.5 text-xs text-indigo-700">
      <span className="font-medium">Re:</span>
      <span className="min-w-0 truncate">{summary}</span>
      <button
        onClick={onClear}
        className="ml-auto flex-shrink-0 text-indigo-400 transition-colors hover:text-indigo-600"
        aria-label="Clear context"
      >
        <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
          <line x1="18" y1="6" x2="6" y2="18" />
          <line x1="6" y1="6" x2="18" y2="18" />
        </svg>
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/copilot/context-chip.tsx
git commit -m "feat(copilot): add ContextChip component for selected item display"
```

---

### Task 6: Copilot Panel and Toggle

**Files:**
- Create: `apps/web/src/components/copilot/copilot-panel.tsx`
- Create: `apps/web/src/components/copilot/copilot-toggle.tsx`

The main panel component wires the store to `ChatMessage`, `ChatInput`, `SuggestedPrompts`, and inline citations. The toggle is a floating button with keyboard shortcut.

- [ ] **Step 1: Create the toggle button**

```typescript
// apps/web/src/components/copilot/copilot-toggle.tsx
"use client";

import { useEffect } from "react";
import { useCopilotStore } from "@/stores/copilot-store";

export function CopilotToggle() {
  const toggle = useCopilotStore((s) => s.toggle);
  const isOpen = useCopilotStore((s) => s.isOpen);

  useEffect(() => {
    function handleKeyDown(e: KeyboardEvent) {
      if ((e.metaKey || e.ctrlKey) && e.key === "k") {
        e.preventDefault();
        toggle();
      }
    }
    window.addEventListener("keydown", handleKeyDown);
    return () => window.removeEventListener("keydown", handleKeyDown);
  }, [toggle]);

  if (isOpen) return null;

  return (
    <button
      onClick={toggle}
      className="fixed bottom-6 right-6 z-50 flex h-12 w-12 items-center justify-center rounded-full bg-indigo-600 text-white shadow-lg transition-transform hover:scale-105 hover:bg-indigo-700"
      aria-label="Open Copilot (Cmd+K)"
      title="Open Copilot (Cmd+K)"
    >
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z" />
      </svg>
    </button>
  );
}
```

- [ ] **Step 2: Create the copilot panel**

```typescript
// apps/web/src/components/copilot/copilot-panel.tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";

import { ChatInput } from "@/components/chat-input";
import { ChatMessage } from "@/components/chat-message";
import { Button } from "@/components/ui/button";
import { getToken } from "@/lib/auth";
import type { Citation } from "@/lib/chat-stream";
import { streamChat } from "@/lib/chat-stream";
import { useCopilotStore, type CopilotMessage } from "@/stores/copilot-store";
import { ContextChip } from "./context-chip";
import { SuggestedPrompts } from "./suggested-prompts";

/* -------------------------------------------------------------------------- */
/*  Inline citation detail                                                     */
/* -------------------------------------------------------------------------- */

function InlineCitationDetail({
  citation,
  onClose,
}: {
  citation: Citation;
  onClose: () => void;
}) {
  return (
    <div className="rounded-md border border-neutral-200 bg-neutral-50 p-3">
      <div className="flex items-center justify-between">
        <span className="text-xs font-semibold text-neutral-700">
          Source [{citation.marker}] — {citation.source_type}
        </span>
        <button onClick={onClose} className="text-neutral-400 hover:text-neutral-600">
          <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
            <line x1="18" y1="6" x2="6" y2="18" />
            <line x1="6" y1="6" x2="18" y2="18" />
          </svg>
        </button>
      </div>
      <p className="mt-1 text-xs text-neutral-500">
        Retrieved by {citation.tool_name.replace(/_/g, " ")}
      </p>
      <p className="mt-2 text-sm leading-relaxed text-neutral-700">{citation.excerpt}</p>
    </div>
  );
}

/* -------------------------------------------------------------------------- */
/*  Page context label                                                        */
/* -------------------------------------------------------------------------- */

const PAGE_LABELS: Record<string, string> = {
  brief: "Morning Brief",
  triage: "Triage",
  actions: "Actions",
  "board-prep": "Board Prep",
  knowledge: "Knowledge",
  strategy: "Strategy",
  "company-detail": "Company Overview",
  "company-kpis": "Company KPIs",
};

/* -------------------------------------------------------------------------- */
/*  Panel                                                                     */
/* -------------------------------------------------------------------------- */

export function CopilotPanel() {
  const isOpen = useCopilotStore((s) => s.isOpen);
  const close = useCopilotStore((s) => s.close);
  const messages = useCopilotStore((s) => s.messages);
  const isStreaming = useCopilotStore((s) => s.isStreaming);
  const pageContext = useCopilotStore((s) => s.pageContext);
  const conversationId = useCopilotStore((s) => s.conversationId);
  const newConversation = useCopilotStore((s) => s.newConversation);
  const addUserMessage = useCopilotStore((s) => s.addUserMessage);
  const addAssistantMessage = useCopilotStore((s) => s.addAssistantMessage);
  const updateAssistant = useCopilotStore((s) => s.updateAssistant);
  const setConversationId = useCopilotStore((s) => s.setConversationId);
  const setIsStreaming = useCopilotStore((s) => s.setIsStreaming);
  const setPageContext = useCopilotStore((s) => s.setPageContext);

  const scrollRef = useRef<HTMLDivElement>(null);
  const [expandedCitation, setExpandedCitation] = useState<Citation | null>(null);

  // Auto-scroll on new messages
  useEffect(() => {
    if (scrollRef.current) {
      scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
    }
  }, [messages]);

  const handleSend = useCallback(
    async (question: string) => {
      addUserMessage(question);
      addAssistantMessage();
      setIsStreaming(true);

      try {
        const token = getToken() ?? undefined;
        for await (const event of streamChat({
          question,
          conversationId,
          authToken: token,
          focusCompanySlug: pageContext?.companySlug,
          pageContext: pageContext?.pageType,
          selectedItemSummary: pageContext?.selectedItemSummary,
        })) {
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
                    ? { ...tc, status: "done" as const, summary: event.summary, recordCount: event.record_count }
                    : tc,
                ),
              }));
              break;
            case "delta":
              updateAssistant((msg) => ({ ...msg, content: msg.content + event.text }));
              break;
            case "citation":
              updateAssistant((msg) => ({
                ...msg,
                citations: [...(msg.citations ?? []), event],
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
              updateAssistant((msg) => ({ ...msg, content: "", error: event.message }));
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
    [conversationId, pageContext, addUserMessage, addAssistantMessage, updateAssistant, setConversationId, setIsStreaming],
  );

  const handleCitationClick = useCallback(
    (marker: number) => {
      for (const msg of messages) {
        const citation = msg.citations?.find((c) => c.marker === marker);
        if (citation) {
          setExpandedCitation(citation);
          return;
        }
      }
    },
    [messages],
  );

  // Build context label
  let contextLabel = "";
  if (pageContext) {
    const pageLabel = PAGE_LABELS[pageContext.pageType] ?? pageContext.pageType;
    if (pageContext.companySlug) {
      const companyName = pageContext.companySlug.replace(/_/g, " ").replace(/\b\w/g, (c) => c.toUpperCase());
      contextLabel = `${companyName} — ${pageLabel}`;
    } else {
      contextLabel = pageLabel;
    }
  }

  return (
    <div
      className={`fixed right-0 top-0 z-40 flex h-full w-[440px] flex-col border-l border-neutral-200 bg-white shadow-lg transition-transform duration-200 ${
        isOpen ? "translate-x-0" : "translate-x-full"
      }`}
    >
      {/* Header */}
      <div className="flex items-center justify-between border-b border-neutral-200 px-4 py-3">
        <div className="min-w-0">
          <h2 className="text-sm font-semibold text-neutral-900">Copilot</h2>
          {contextLabel && (
            <p className="truncate text-xs text-neutral-500">{contextLabel}</p>
          )}
        </div>
        <div className="flex items-center gap-1">
          <Button
            variant="ghost"
            size="sm"
            onClick={newConversation}
            className="text-xs text-neutral-500"
            disabled={messages.length === 0}
          >
            New
          </Button>
          <Button variant="ghost" size="icon" onClick={close} className="h-7 w-7">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
              <line x1="18" y1="6" x2="6" y2="18" />
              <line x1="6" y1="6" x2="18" y2="18" />
            </svg>
          </Button>
        </div>
      </div>

      {/* Messages */}
      <div ref={scrollRef} className="flex-1 overflow-y-auto px-4 py-4">
        {messages.length === 0 ? (
          <div className="flex h-full flex-col items-center justify-center text-center">
            <p className="text-sm font-medium text-neutral-900">How can I help?</p>
            <p className="mt-1 max-w-xs text-xs text-neutral-500">
              Ask questions about your portfolio, strategy, or anything on this page.
            </p>
            <div className="mt-4">
              <SuggestedPrompts
                pageType={pageContext?.pageType ?? null}
                onSelect={handleSend}
                disabled={isStreaming}
              />
            </div>
          </div>
        ) : (
          <div className="space-y-4">
            {messages.map((msg, i) => {
              if (msg.error) {
                return (
                  <div key={i} className="flex justify-start">
                    <div className="max-w-[90%] rounded-lg border border-red-200 bg-red-50 px-3 py-2">
                      <p className="text-sm text-red-700">{msg.error}</p>
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

        {/* Inline citation detail */}
        {expandedCitation && (
          <div className="mt-3">
            <InlineCitationDetail
              citation={expandedCitation}
              onClose={() => setExpandedCitation(null)}
            />
          </div>
        )}
      </div>

      {/* Context chip + Input */}
      <div>
        {pageContext?.selectedItemSummary && (
          <div className="px-4 pt-2">
            <ContextChip
              summary={pageContext.selectedItemSummary}
              onClear={() =>
                setPageContext(
                  pageContext
                    ? { pageType: pageContext.pageType, companySlug: pageContext.companySlug }
                    : null,
                )
              }
            />
          </div>
        )}
        <ChatInput onSend={handleSend} disabled={isStreaming} />
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: Possible errors in `companies/[slug]/chat/page.tsx` (old `streamChat` call) — expected, resolved in Task 7.

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/components/copilot/copilot-panel.tsx apps/web/src/components/copilot/copilot-toggle.tsx
git commit -m "feat(copilot): add CopilotPanel slide-over and CopilotToggle floating button"
```

---

### Task 7: Wire Panel into Layout and Remove Old Chat Page

**Files:**
- Modify: `apps/web/src/app/(main)/layout.tsx`
- Remove: `apps/web/src/app/(main)/companies/[slug]/chat/page.tsx`

The layout becomes a flex container. When the copilot panel is open, the main content area shrinks. The old chat page is deleted.

- [ ] **Step 1: Update the layout**

Replace the entire contents of `apps/web/src/app/(main)/layout.tsx` with:

```typescript
import type { ReactNode } from "react";

import { Nav } from "@/components/nav";
import { RouteGuard } from "@/components/route-guard";
import { CopilotPanel } from "@/components/copilot/copilot-panel";
import { CopilotToggle } from "@/components/copilot/copilot-toggle";

export default function MainLayout({ children }: { children: ReactNode }) {
  return (
    <>
      <RouteGuard />
      <Nav />
      <div className="flex">
        <main className="mx-auto max-w-6xl flex-1 px-6 py-8">{children}</main>
        <CopilotPanel />
      </div>
      <CopilotToggle />
    </>
  );
}
```

- [ ] **Step 2: Delete the old chat page**

```bash
rm apps/web/src/app/\(main\)/companies/\[slug\]/chat/page.tsx
```

Check if the `chat/` directory is now empty and remove it:

```bash
rmdir apps/web/src/app/\(main\)/companies/\[slug\]/chat/ 2>/dev/null || true
```

- [ ] **Step 3: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit --pretty 2>&1 | head -20`
Expected: PASS (no errors — the old chat page was the only consumer of the old `streamChat` signature)

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/app/\(main\)/layout.tsx
git rm apps/web/src/app/\(main\)/companies/\[slug\]/chat/page.tsx
git commit -m "feat(copilot): wire panel into layout, remove old chat page"
```

---

### Task 8: Add useCopilotContext to All Pages

**Files:**
- Modify: `apps/web/src/app/(main)/brief/page.tsx`
- Modify: `apps/web/src/app/(main)/triage/page.tsx`
- Modify: `apps/web/src/app/(main)/actions/page.tsx`
- Modify: `apps/web/src/app/(main)/board-prep/page.tsx`
- Modify: `apps/web/src/app/(main)/knowledge/page.tsx`
- Modify: `apps/web/src/app/(main)/strategy/page.tsx`
- Modify: `apps/web/src/app/(main)/companies/[slug]/page.tsx`

Add one `useCopilotContext` call near the top of each page component.

- [ ] **Step 1: Add hook to each page**

For each page, add the import and hook call inside the default export function, before any other hooks or logic.

**`brief/page.tsx`** — Add after the existing imports:
```typescript
import { useCopilotContext } from "@/hooks/use-copilot-context";
```
Add as the first line inside `export default function BriefPage() {`:
```typescript
useCopilotContext({ pageType: "brief" });
```

**`triage/page.tsx`** — Same pattern:
```typescript
import { useCopilotContext } from "@/hooks/use-copilot-context";
```
First line inside `export default function TriagePage() {`:
```typescript
useCopilotContext({ pageType: "triage" });
```

**`actions/page.tsx`** — Same pattern:
```typescript
import { useCopilotContext } from "@/hooks/use-copilot-context";
```
First line inside `export default function ActionsPage() {`:
```typescript
useCopilotContext({ pageType: "actions" });
```

**`board-prep/page.tsx`** — Same pattern:
```typescript
import { useCopilotContext } from "@/hooks/use-copilot-context";
```
First line inside `export default function BoardPrepPage() {`:
```typescript
useCopilotContext({ pageType: "board-prep" });
```

**`knowledge/page.tsx`** — Same pattern:
```typescript
import { useCopilotContext } from "@/hooks/use-copilot-context";
```
First line inside `export default function KnowledgePage() {`:
```typescript
useCopilotContext({ pageType: "knowledge" });
```

**`strategy/page.tsx`** — Same pattern:
```typescript
import { useCopilotContext } from "@/hooks/use-copilot-context";
```
First line inside the default export function:
```typescript
useCopilotContext({ pageType: "strategy" });
```

**`companies/[slug]/page.tsx`** — This page has a `slug` param. Add:
```typescript
import { useCopilotContext } from "@/hooks/use-copilot-context";
```
After the slug is resolved from params, add:
```typescript
useCopilotContext({ pageType: "company-detail", companySlug: slug });
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd apps/web && npx tsc --noEmit --pretty 2>&1 | head -20`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/\(main\)/brief/page.tsx apps/web/src/app/\(main\)/triage/page.tsx apps/web/src/app/\(main\)/actions/page.tsx apps/web/src/app/\(main\)/board-prep/page.tsx apps/web/src/app/\(main\)/knowledge/page.tsx apps/web/src/app/\(main\)/strategy/page.tsx apps/web/src/app/\(main\)/companies/\[slug\]/page.tsx
git commit -m "feat(copilot): register page context on all main pages"
```

---

### Task 9: Backend — Make `company_slug` Optional, Add New Fields

**Files:**
- Modify: `apps/api/src/prescient/intelligence/api/copilot.py`
- Modify: `apps/api/src/prescient/intelligence/api/copilot_stream.py`
- Test: `apps/api/tests/unit/intelligence/test_copilot_stream_request.py`

The stream endpoint accepts optional `focus_company_slug`, `page_context`, and `selected_item_summary`. The `company_slug` field becomes optional (defaults to `None`).

- [ ] **Step 1: Write the failing test**

Create `apps/api/tests/unit/intelligence/test_copilot_stream_request.py`:

```python
"""Tests for the updated StreamRequest model."""

import pytest
from pydantic import ValidationError


def test_stream_request_minimal():
    """StreamRequest with only question should be valid."""
    from prescient.intelligence.api.copilot_stream import StreamRequest

    req = StreamRequest(question="What are the top risks?")
    assert req.question == "What are the top risks?"
    assert req.company_slug is None
    assert req.focus_company_slug is None
    assert req.page_context is None
    assert req.selected_item_summary is None


def test_stream_request_with_focus_company():
    """StreamRequest with focus_company_slug passes through."""
    from prescient.intelligence.api.copilot_stream import StreamRequest

    req = StreamRequest(
        question="Summarize KPIs",
        focus_company_slug="peloton",
        page_context="company-kpis",
    )
    assert req.focus_company_slug == "peloton"
    assert req.page_context == "company-kpis"


def test_stream_request_backward_compat():
    """StreamRequest with company_slug (old field) still works."""
    from prescient.intelligence.api.copilot_stream import StreamRequest

    req = StreamRequest(question="Hello", company_slug="peloton")
    assert req.company_slug == "peloton"


def test_ask_request_minimal():
    """AskRequest with only question should be valid."""
    from prescient.intelligence.api.copilot import AskRequest

    req = AskRequest(question="What are the top risks?")
    assert req.question == "What are the top risks?"
    assert req.company_slug is None


def test_ask_request_with_all_fields():
    """AskRequest with all new fields passes through."""
    from prescient.intelligence.api.copilot import AskRequest

    req = AskRequest(
        question="Summarize",
        focus_company_slug="peloton",
        page_context="actions",
        selected_item_summary="Overdue: Review margin analysis",
    )
    assert req.focus_company_slug == "peloton"
    assert req.page_context == "actions"
    assert req.selected_item_summary == "Overdue: Review margin analysis"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && python -m pytest tests/unit/intelligence/test_copilot_stream_request.py -v`
Expected: FAIL — `company_slug` is currently required (min_length=1), and new fields don't exist.

- [ ] **Step 3: Update `StreamRequest` in `copilot_stream.py`**

In `apps/api/src/prescient/intelligence/api/copilot_stream.py`, replace the `StreamRequest` class (lines 45-51) with:

```python
class StreamRequest(BaseModel):
    model_config = ConfigDict(frozen=True)

    question: str = Field(min_length=1, max_length=4096)
    conversation_id: UUID | None = None
    company_slug: str | None = Field(default=None, max_length=128)
    focus_company_slug: str | None = Field(default=None, max_length=128)
    page_context: str | None = Field(default=None, max_length=64)
    selected_item_summary: str | None = Field(default=None, max_length=512)
```

- [ ] **Step 4: Update `AskRequest` in `copilot.py`**

In `apps/api/src/prescient/intelligence/api/copilot.py`, replace the `AskRequest` class (lines 58-64) with:

```python
class AskRequest(BaseModel):
    model_config = ConfigDict(frozen=True)

    question: str = Field(min_length=1, max_length=4096)
    conversation_id: UUID | None = None
    company_slug: str | None = Field(default=None, max_length=128)
    focus_company_slug: str | None = Field(default=None, max_length=128)
    page_context: str | None = Field(default=None, max_length=64)
    selected_item_summary: str | None = Field(default=None, max_length=512)
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && python -m pytest tests/unit/intelligence/test_copilot_stream_request.py -v`
Expected: PASS (5 tests)

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/intelligence/api/copilot.py apps/api/src/prescient/intelligence/api/copilot_stream.py apps/api/tests/unit/intelligence/test_copilot_stream_request.py
git commit -m "feat(copilot): make company_slug optional, add focus_company_slug and page_context fields"
```

---

### Task 10: Backend — Resolve Scope from Home Org or Focus Company

**Files:**
- Modify: `apps/api/src/prescient/intelligence/api/copilot.py`
- Modify: `apps/api/src/prescient/intelligence/api/copilot_stream.py`
- Test: `apps/api/tests/unit/intelligence/test_scope_resolution.py`

When `company_slug` is not provided, the endpoint resolves the user's home org (fund) as the primary scope. When `focus_company_slug` is provided, it layers in that company's data via `visible_company_ids`.

- [ ] **Step 1: Write the failing test**

Create `apps/api/tests/unit/intelligence/test_scope_resolution.py`:

```python
"""Tests for the scope resolution helper."""

from uuid import UUID, uuid4

import pytest

from prescient.intelligence.domain.entities.copilot_conversation import ConversationScope
from prescient.shared.types import CompanyId


def test_home_org_scope_has_no_focus_company():
    """When no focus company, scope has only the home company."""
    from prescient.intelligence.api.copilot import _build_home_scope

    home_company_id = CompanyId(uuid4())
    scope = _build_home_scope(home_company_id, focus_company_id=None)
    assert scope.primary_company_id == home_company_id
    assert scope.visible_company_ids == ()


def test_home_org_scope_with_focus_company():
    """When focus company provided, it appears in visible_company_ids."""
    from prescient.intelligence.api.copilot import _build_home_scope

    home_company_id = CompanyId(uuid4())
    focus_company_id = CompanyId(uuid4())
    scope = _build_home_scope(home_company_id, focus_company_id=focus_company_id)
    assert scope.primary_company_id == home_company_id
    assert focus_company_id in scope.visible_company_ids


def test_home_org_scope_deduplicates_same_company():
    """When focus company is same as home, no duplicate in visible."""
    from prescient.intelligence.api.copilot import _build_home_scope

    company_id = CompanyId(uuid4())
    scope = _build_home_scope(company_id, focus_company_id=company_id)
    assert scope.primary_company_id == company_id
    # Should not appear in visible_company_ids since it's already primary
    assert company_id not in scope.visible_company_ids
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && python -m pytest tests/unit/intelligence/test_scope_resolution.py -v`
Expected: FAIL — `_build_home_scope` doesn't exist yet.

- [ ] **Step 3: Add `_build_home_scope` to `copilot.py`**

In `apps/api/src/prescient/intelligence/api/copilot.py`, add after the existing `_build_scope` function (after line 155):

```python
def _build_home_scope(
    home_company_id: CompanyId,
    focus_company_id: CompanyId | None = None,
) -> ConversationScope:
    """Build a scope anchored to the user's home org.

    If ``focus_company_id`` is given and differs from ``home_company_id``,
    it is added to ``visible_company_ids`` so retrieval tools can access
    both the home org's and the focus company's data.
    """
    visible: tuple[CompanyId, ...] = ()
    if focus_company_id and focus_company_id != home_company_id:
        visible = (focus_company_id,)
    return ConversationScope(
        primary_company_id=home_company_id,
        visible_company_ids=visible,
    )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/api && python -m pytest tests/unit/intelligence/test_scope_resolution.py -v`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/intelligence/api/copilot.py apps/api/tests/unit/intelligence/test_scope_resolution.py
git commit -m "feat(copilot): add _build_home_scope for dual-scope grounding"
```

---

### Task 11: Backend — System Prompt Page Context Augmentation

**Files:**
- Modify: `apps/api/src/prescient/intelligence/prompts/__init__.py`
- Modify: `apps/api/src/prescient/intelligence/prompts/system.md`
- Test: `apps/api/tests/unit/intelligence/test_system_prompt.py`

Add a `{page_context}` template variable to the system prompt. When provided, it tells the copilot what page the user is viewing.

- [ ] **Step 1: Write the failing test**

Create `apps/api/tests/unit/intelligence/test_system_prompt.py`:

```python
"""Tests for system prompt loading with page context."""


def test_load_system_prompt_with_page_context():
    """Page context appears in the rendered system prompt."""
    from prescient.intelligence.prompts import load_system_prompt

    prompt = load_system_prompt(
        primary_company_name="Peloton",
        primary_company_slug="peloton",
        tool_descriptions="(tools here)",
        strategy_context="",
        page_context="The user is currently viewing the Actions inbox.",
    )
    assert "The user is currently viewing the Actions inbox." in prompt


def test_load_system_prompt_without_page_context():
    """Empty page_context produces no extra block."""
    from prescient.intelligence.prompts import load_system_prompt

    prompt = load_system_prompt(
        primary_company_name="Peloton",
        primary_company_slug="peloton",
        tool_descriptions="(tools here)",
        strategy_context="",
        page_context="",
    )
    # The placeholder should be cleanly absent
    assert "{page_context}" not in prompt


def test_load_system_prompt_backward_compat():
    """Calling without page_context kwarg works (default empty)."""
    from prescient.intelligence.prompts import load_system_prompt

    prompt = load_system_prompt(
        primary_company_name="Peloton",
        primary_company_slug="peloton",
        tool_descriptions="(tools here)",
    )
    assert "Peloton" in prompt
    assert "{page_context}" not in prompt
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && python -m pytest tests/unit/intelligence/test_system_prompt.py -v`
Expected: FAIL — `page_context` parameter doesn't exist on `load_system_prompt`.

- [ ] **Step 3: Add `{page_context}` to the system prompt template**

In `apps/api/src/prescient/intelligence/prompts/system.md`, add after the `{strategy_context}` line (line 19):

```
{page_context}
```

- [ ] **Step 4: Update `load_system_prompt` to accept `page_context`**

In `apps/api/src/prescient/intelligence/prompts/__init__.py`, replace the `load_system_prompt` function (lines 10-24) with:

```python
def load_system_prompt(
    *,
    primary_company_name: str,
    primary_company_slug: str,
    tool_descriptions: str,
    strategy_context: str = "",
    page_context: str = "",
) -> str:
    """Load and interpolate the system prompt template."""
    template = (_PROMPTS_DIR / "system.md").read_text()
    return template.format(
        primary_company_name=primary_company_name,
        primary_company_slug=primary_company_slug,
        tool_descriptions=tool_descriptions,
        strategy_context=strategy_context,
        page_context=page_context,
    )
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && python -m pytest tests/unit/intelligence/test_system_prompt.py -v`
Expected: PASS (3 tests)

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/intelligence/prompts/__init__.py apps/api/src/prescient/intelligence/prompts/system.md apps/api/tests/unit/intelligence/test_system_prompt.py
git commit -m "feat(copilot): add page_context to system prompt template"
```

---

### Task 12: Backend — Wire New Fields Through Planner and Route Handlers

**Files:**
- Modify: `apps/api/src/prescient/intelligence/application/planner.py`
- Modify: `apps/api/src/prescient/intelligence/application/planner_stream.py`
- Modify: `apps/api/src/prescient/intelligence/api/copilot.py`
- Modify: `apps/api/src/prescient/intelligence/api/copilot_stream.py`

Pass `page_context` through the planner to `load_system_prompt`. Update route handlers to build page context string and resolve scope using `_build_home_scope` when no `company_slug` is provided.

- [ ] **Step 1: Add `page_context` parameter to `run_planner`**

In `apps/api/src/prescient/intelligence/application/planner.py`, update the `run_planner` function signature (line 67) to add `page_context: str = ""` after `strategy_context: str = ""`:

```python
async def run_planner(
    *,
    llm: LLMProviderPort,
    tools: ToolRegistryPort,
    question: str,
    conversation_messages: list[dict[str, Any]],
    primary_company_name: str,
    primary_company_slug: str,
    model: str,
    conversation_id: UUID,
    strategy_context: str = "",
    page_context: str = "",
) -> Answer:
```

Update the `load_system_prompt` call inside `run_planner` (around line 116-121) to pass `page_context`:

```python
    system_prompt = load_system_prompt(
        primary_company_name=primary_company_name,
        primary_company_slug=primary_company_slug,
        tool_descriptions=tool_descriptions,
        strategy_context=strategy_context,
        page_context=page_context,
    )
```

- [ ] **Step 2: Add `page_context` parameter to `run_planner_stream`**

In `apps/api/src/prescient/intelligence/application/planner_stream.py`, update the `run_planner_stream` function signature (line 69) to add `page_context: str = ""` after `strategy_context: str = ""`:

```python
async def run_planner_stream(
    *,
    llm: LLMProviderPort,
    tools: ToolRegistryPort,
    question: str,
    conversation_messages: list[dict[str, Any]],
    primary_company_name: str,
    primary_company_slug: str,
    model: str,
    conversation_id: UUID,
    max_iterations: int = 15,
    strategy_context: str = "",
    page_context: str = "",
) -> AsyncIterator[StreamEvent]:
```

Update the `load_system_prompt` call inside `run_planner_stream` (around line 91-96) to pass `page_context`:

```python
    system_prompt = load_system_prompt(
        primary_company_name=primary_company_name,
        primary_company_slug=primary_company_slug,
        tool_descriptions=tool_descriptions,
        strategy_context=strategy_context,
        page_context=page_context,
    )
```

- [ ] **Step 3: Add `_build_page_context_prompt` helper to `copilot.py`**

In `apps/api/src/prescient/intelligence/api/copilot.py`, add after the `_build_home_scope` function:

```python
_PAGE_LABELS: dict[str, str] = {
    "brief": "the Morning Brief",
    "triage": "the Triage queue",
    "actions": "the Actions inbox",
    "board-prep": "the Board Prep workspace",
    "knowledge": "the Knowledge management page",
    "strategy": "the Strategy initiatives page",
    "company-detail": "a portfolio company overview",
    "company-kpis": "a portfolio company's KPI dashboard",
}


def _build_page_context_prompt(
    page_context: str | None,
    selected_item_summary: str | None,
) -> str:
    """Build a page context block for the system prompt.

    Returns an empty string if no context is provided, so the template
    renders cleanly without a blank section.
    """
    parts: list[str] = []
    if page_context:
        label = _PAGE_LABELS.get(page_context, page_context)
        parts.append(f"## Current page\n\nThe user is currently viewing {label}.")
    if selected_item_summary:
        parts.append(f"They are focused on: {selected_item_summary}")
    return "\n".join(parts)
```

- [ ] **Step 4: Update the `stream_chat` route handler in `copilot_stream.py`**

In `apps/api/src/prescient/intelligence/api/copilot_stream.py`, update the `stream_chat` function (starting at line 103) to handle both old-style `company_slug` and new-style `focus_company_slug` / no-company requests.

Replace the route function body (lines 120-162) with:

```python
async def stream_chat(
    body: StreamRequest,
    ctx: CtxDep,
    llm: LLMDep,
    opensearch: OpenSearchDep,
    registry: ToolRegistryDep,
    company_repo: CompanyRepoDep,
    relationship_repo: RelationshipRepoDep,
    engagement_repo: EngagementRepoDep,
) -> StreamingResponse:
    """Stream a copilot conversation as Server-Sent Events."""
    settings = get_settings()

    # Resolve the effective company slug: prefer focus_company_slug, fall back to company_slug
    effective_slug = body.focus_company_slug or body.company_slug

    if effective_slug:
        company = await _resolve_company(
            company_repo, engagement_repo, effective_slug, ctx.user.fund_id
        )
        scope = await _build_scope(relationship_repo, CompanyId(company.id))
        company_name = company.name
        company_slug = company.slug
    else:
        # No company context — use fund-level scope.
        # Import here to avoid circular dependency at module level.
        from prescient.intelligence.api.copilot import _build_home_scope

        scope = _build_home_scope(CompanyId(ctx.user.fund_id))
        company_name = "your portfolio"
        company_slug = ""

    tool_context = ToolContext(
        tenant_id=ctx.user.fund_id,
        user_id=ctx.user.user_id,
        scope=scope,
        primary_company_slug=company_slug,
        session=ctx.session,
        opensearch=opensearch,
    )
    bound_tools = registry.bind(tool_context)

    conversation_id = body.conversation_id or uuid4()

    messages: list[dict[str, Any]] = [
        {"role": "user", "content": body.question},
    ]

    # Build strategy context for the system prompt.
    from prescient.intelligence.api.copilot import _build_strategy_context, _build_page_context_prompt

    strategy_ctx = await _build_strategy_context(ctx.session, str(ctx.user.fund_id))
    page_ctx = _build_page_context_prompt(body.page_context, body.selected_item_summary)

    return StreamingResponse(
        _event_generator(
            llm=llm,
            tools=bound_tools,
            question=body.question,
            conversation_messages=messages,
            primary_company_name=company_name,
            primary_company_slug=company_slug,
            model=settings.anthropic_model,
            conversation_id=conversation_id,
            strategy_context=strategy_ctx,
            page_context=page_ctx,
        ),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

Also update the `_event_generator` function signature (line 65) to accept `page_context`:

```python
async def _event_generator(
    *,
    llm: Any,
    tools: Any,
    question: str,
    conversation_messages: list[dict[str, Any]],
    primary_company_name: str,
    primary_company_slug: str,
    model: str,
    conversation_id: UUID,
    strategy_context: str = "",
    page_context: str = "",
) -> AsyncIterator[str]:
```

And pass `page_context` to `run_planner_stream` inside `_event_generator` (around line 83):

```python
        async for event in run_planner_stream(
            llm=llm,
            tools=tools,
            question=question,
            conversation_messages=conversation_messages,
            primary_company_name=primary_company_name,
            primary_company_slug=primary_company_slug,
            model=model,
            conversation_id=conversation_id,
            strategy_context=strategy_context,
            page_context=page_context,
        ):
```

- [ ] **Step 5: Update the `ask` route handler in `copilot.py`**

In `apps/api/src/prescient/intelligence/api/copilot.py`, apply the same pattern to the `ask` function. Replace the route function body (lines 216-284) with the same dual-path logic:

```python
async def ask(
    body: AskRequest,
    ctx: CtxDep,
    llm: LLMDep,
    opensearch: OpenSearchDep,
    registry: ToolRegistryDep,
    company_repo: CompanyRepoDep,
    relationship_repo: RelationshipRepoDep,
    engagement_repo: EngagementRepoDep,
) -> AskResponse:
    """Accept a user question and return a grounded answer."""
    settings = get_settings()

    effective_slug = body.focus_company_slug or body.company_slug

    if effective_slug:
        company = await _resolve_company(
            company_repo, engagement_repo, effective_slug, ctx.user.fund_id
        )
        scope = await _build_scope(relationship_repo, CompanyId(company.id))
        company_name = company.name
        company_slug = company.slug
    else:
        scope = _build_home_scope(CompanyId(ctx.user.fund_id))
        company_name = "your portfolio"
        company_slug = ""

    tool_context = ToolContext(
        tenant_id=ctx.user.fund_id,
        user_id=ctx.user.user_id,
        scope=scope,
        primary_company_slug=company_slug,
        session=ctx.session,
        opensearch=opensearch,
    )
    bound_tools: ToolRegistryPort = registry.bind(tool_context)

    conversation_id = body.conversation_id or uuid4()

    messages: list[dict[str, Any]] = [
        {"role": "user", "content": body.question},
    ]

    strategy_ctx = await _build_strategy_context(ctx.session, str(ctx.user.fund_id))
    page_ctx = _build_page_context_prompt(body.page_context, body.selected_item_summary)

    try:
        answer = await run_planner(
            llm=llm,
            tools=bound_tools,
            question=body.question,
            conversation_messages=messages,
            primary_company_name=company_name,
            primary_company_slug=company_slug,
            model=settings.anthropic_model,
            conversation_id=conversation_id,
            strategy_context=strategy_ctx,
            page_context=page_ctx,
        )
    except PlannerLoopExceededError:
        raise ValidationError(
            "The copilot could not produce an answer within the iteration limit. "
            "Please try a more specific question."
        ) from None

    return AskResponse(
        conversation_id=conversation_id,
        answer=answer.text,
        citations=[
            CitationResponse(
                marker=c.marker,
                tool_name=c.tool_name,
                source_type=c.source_type,
                excerpt=c.excerpt,
            )
            for c in answer.citations
        ],
        grounding_status=answer.grounding_status.value,
        tool_call_count=len(answer.tool_calls),
        iterations=answer.iterations,
    )
```

- [ ] **Step 6: Run all tests**

Run: `cd apps/api && python -m pytest tests/unit/intelligence/ -v`
Expected: All tests pass, including the ones from Tasks 9-11.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/prescient/intelligence/application/planner.py apps/api/src/prescient/intelligence/application/planner_stream.py apps/api/src/prescient/intelligence/api/copilot.py apps/api/src/prescient/intelligence/api/copilot_stream.py
git commit -m "feat(copilot): wire page_context through planner, resolve home org scope in route handlers"
```

---

### Task 13: Browser Validation

**Files:** None (manual testing)

Start the dev environment and validate the copilot panel end-to-end.

- [ ] **Step 1: Start the dev environment**

Run: `docker compose up -d` (or however the dev containers are started)

- [ ] **Step 2: Validate panel basics**

1. Navigate to `http://localhost:3000/brief`
2. Verify the floating copilot button appears in the bottom-right corner
3. Click it — panel should slide in from the right, main content should shrink
4. Verify header says "Copilot" with "Morning Brief" context label
5. Verify suggested prompts appear (e.g., "What should I focus on today?")
6. Click the X button — panel should close, floating button reappears
7. Press `Cmd+K` — panel should reopen

- [ ] **Step 3: Validate conversation persistence across navigation**

1. Open the panel on `/brief`
2. Type a message (e.g., "Hello") and send
3. Navigate to `/actions` using the top nav
4. Verify the panel stays open with the same message
5. Verify the context label changes to "Actions"
6. Verify suggested prompts are NOT shown (conversation is active)

- [ ] **Step 4: Validate new conversation**

1. Click the "New" button in the panel header
2. Verify messages are cleared
3. Verify suggested prompts appear again

- [ ] **Step 5: Validate company page context**

1. Navigate to a company detail page (e.g., `/companies/peloton`)
2. Verify the context label shows "Peloton — Company Overview"

- [ ] **Step 6: Validate old chat page is gone**

1. Navigate to `/companies/peloton/chat`
2. Verify it shows a 404 page

- [ ] **Step 7: Commit any fixes if needed**

```bash
git add -A
git commit -m "fix(copilot): browser validation fixes"
```
