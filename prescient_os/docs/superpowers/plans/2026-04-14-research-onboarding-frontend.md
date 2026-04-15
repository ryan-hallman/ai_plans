# Research Engine & Smart Onboarding Frontend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build shared chat primitives, onboarding interview page with concurrent research, standalone `/research` page with split-view and source controls, and supporting changes (notification polling, dual-path onboarding entry).

**Architecture:** Extract reusable hooks (`useStreamChat`, `useResearchPoller`) and components (`ChatMessageList`, `ChatInputBar`) from the existing `CopilotPanel`. Refactor the panel to compose from these primitives. Build two new pages — `/onboarding/interview` (full-page chat with background research) and `/research` (split-view with results + chat + controls). Add Next.js API route proxies for new backend endpoints.

**Tech Stack:** Next.js 15 (app router), TypeScript, React 19, Zustand, React Query, Tailwind CSS, existing UI components (Card, Button, Input from Radix/CVA)

**Spec:** `docs/superpowers/specs/2026-04-14-research-onboarding-frontend-design.md`

---

## File Structure

### New: Shared hooks

```
apps/web/src/hooks/
├── use-stream-chat.ts          # Streaming chat hook (extracted from CopilotPanel)
└── use-research-poller.ts      # Research task polling hook
```

### New: Shared chat components

```
apps/web/src/components/chat/
├── chat-message-list.tsx       # Reusable scrollable message list
└── chat-input-bar.tsx          # Reusable chat input with optional context
```

### New: Research components

```
apps/web/src/components/research/
├── source-progress-bar.tsx     # Source completion indicator
├── research-controls.tsx       # Depth preset + source toggles
├── research-result-card.tsx    # Completed research result
└── research-progress-card.tsx  # In-flight research indicator
```

### New: Pages

```
apps/web/src/app/(main)/
├── onboarding/interview/
│   └── page.tsx                # Onboarding interview page
└── research/
    └── page.tsx                # Standalone research page
```

### New: API route proxies

```
apps/web/src/app/api/
├── research/
│   ├── sources/
│   │   └── route.ts            # GET → /research/sources
│   ├── [taskId]/
│   │   └── route.ts            # GET → /research/{taskId}
│   └── route.ts                # POST → /research
└── onboarding/
    └── interview/
        └── complete/
            └── route.ts        # POST → /onboarding/interview/complete
```

### Modified

```
apps/web/src/components/copilot/copilot-panel.tsx  # Refactor to use shared primitives
apps/web/src/components/notifications-bell.tsx     # Add polling interval
apps/web/src/app/(main)/onboarding/page.tsx        # Dual-path entry
```

---

## Task 1: `useStreamChat` Hook

**Files:**
- Create: `apps/web/src/hooks/use-stream-chat.ts`

- [ ] **Step 1: Create the hook**

Create `apps/web/src/hooks/use-stream-chat.ts`:

```typescript
"use client";

import { useCallback, useRef, useState } from "react";

import type { ChatEvent, Citation, StreamChatOptions } from "@/lib/chat-stream";
import { streamChat } from "@/lib/chat-stream";
import { getToken } from "@/lib/auth";

export interface ToolCall {
  id: string;
  tool: string;
  status: "running" | "done";
  summary?: string;
  recordCount?: number;
}

export interface ChatMessage {
  role: "user" | "assistant";
  content: string;
  toolCalls?: ToolCall[];
  citations?: Citation[];
  groundingStatus?: string;
  iterations?: number;
  error?: string;
}

export interface UseStreamChatOptions {
  /** Company ID for scoped queries */
  companyId?: string;
  /** Focus company ID */
  focusCompanyId?: string;
  /** Page context string */
  pageContext?: string;
  /** Extra fields merged into the request body (e.g., mode, research_context) */
  extraBody?: Record<string, unknown>;
}

export interface UseStreamChatReturn {
  messages: ChatMessage[];
  isStreaming: boolean;
  conversationId: string | undefined;
  send: (question: string) => Promise<void>;
  newConversation: () => void;
}

export function useStreamChat(options: UseStreamChatOptions = {}): UseStreamChatReturn {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [isStreaming, setIsStreaming] = useState(false);
  const [conversationId, setConversationId] = useState<string | undefined>(undefined);
  const optionsRef = useRef(options);
  optionsRef.current = options;

  const updateLastAssistant = useCallback(
    (updater: (msg: ChatMessage) => ChatMessage) => {
      setMessages((prev) => {
        const last = prev.at(-1);
        if (!last || last.role !== "assistant") return prev;
        return [...prev.slice(0, -1), updater(last)];
      });
    },
    [],
  );

  const send = useCallback(
    async (question: string) => {
      setMessages((prev) => [
        ...prev,
        { role: "user", content: question },
        { role: "assistant", content: "" },
      ]);
      setIsStreaming(true);

      try {
        const token = getToken() ?? undefined;
        const opts = optionsRef.current;

        for await (const event of streamChat({
          question,
          conversationId,
          authToken: token,
          companyId: opts.companyId,
          focusCompanyId: opts.focusCompanyId,
          pageContext: opts.pageContext,
          ...opts.extraBody,
        } as StreamChatOptions)) {
          switch (event.type) {
            case "init":
              setConversationId(event.conversation_id);
              break;
            case "tool_start":
              updateLastAssistant((msg) => ({
                ...msg,
                toolCalls: [
                  ...(msg.toolCalls ?? []),
                  { id: event.tool_use_id, tool: event.tool, status: "running" as const },
                ],
              }));
              break;
            case "tool_end":
              updateLastAssistant((msg) => ({
                ...msg,
                toolCalls: (msg.toolCalls ?? []).map((tc) =>
                  tc.id === event.tool_use_id
                    ? { ...tc, status: "done" as const, summary: event.summary, recordCount: event.record_count }
                    : tc,
                ),
              }));
              break;
            case "delta":
              updateLastAssistant((msg) => ({ ...msg, content: msg.content + event.text }));
              break;
            case "citation":
              updateLastAssistant((msg) => ({
                ...msg,
                citations: [...(msg.citations ?? []), event],
              }));
              break;
            case "done":
              updateLastAssistant((msg) => ({
                ...msg,
                groundingStatus: event.grounding_status,
                iterations: event.iterations,
              }));
              setIsStreaming(false);
              break;
            case "error":
              updateLastAssistant((msg) => ({ ...msg, content: "", error: event.message }));
              setIsStreaming(false);
              break;
          }
        }
      } catch (err) {
        updateLastAssistant((msg) => ({
          ...msg,
          content: "",
          error: err instanceof Error ? err.message : "Connection failed",
        }));
        setIsStreaming(false);
      }
    },
    [conversationId, updateLastAssistant],
  );

  const newConversation = useCallback(() => {
    setMessages([]);
    setConversationId(undefined);
    setIsStreaming(false);
  }, []);

  return { messages, isStreaming, conversationId, send, newConversation };
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cd apps/web && npx tsc --noEmit src/hooks/use-stream-chat.ts 2>&1 | head -20`
If there are import resolution issues with `--noEmit` on a single file, just run the full project check:
Run: `cd apps/web && npx next lint 2>&1 | tail -10`

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/hooks/use-stream-chat.ts
git commit -m "feat(web): add useStreamChat hook for reusable chat streaming"
```

---

## Task 2: `ChatMessageList` and `ChatInputBar` Components

**Files:**
- Create: `apps/web/src/components/chat/chat-message-list.tsx`
- Create: `apps/web/src/components/chat/chat-input-bar.tsx`

- [ ] **Step 1: Create ChatMessageList**

Create `apps/web/src/components/chat/chat-message-list.tsx`:

```typescript
"use client";

import { useEffect, useRef } from "react";

import { ChatMessage as ChatMessageComponent } from "@/components/chat-message";
import type { ChatMessage } from "@/hooks/use-stream-chat";

interface ChatMessageListProps {
  messages: ChatMessage[];
  isStreaming: boolean;
  /** Whether to show tool call chips (default: true) */
  showToolCalls?: boolean;
  /** Callback when a citation marker is clicked */
  onCitationClick?: (marker: number) => void;
  /** Optional empty state content */
  emptyState?: React.ReactNode;
  /** Additional className for the scroll container */
  className?: string;
}

export function ChatMessageList({
  messages,
  isStreaming,
  showToolCalls = true,
  onCitationClick,
  emptyState,
  className = "",
}: ChatMessageListProps) {
  const scrollRef = useRef<HTMLDivElement>(null);

  // Auto-scroll on new content
  useEffect(() => {
    if (scrollRef.current) {
      scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
    }
  }, [messages]);

  const handleCitationClick = onCitationClick ?? (() => {});

  return (
    <div ref={scrollRef} className={`flex-1 overflow-y-auto px-4 py-4 ${className}`}>
      {messages.length === 0 ? (
        emptyState ?? null
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
              <ChatMessageComponent
                key={i}
                role={msg.role}
                content={msg.content}
                toolCalls={showToolCalls ? msg.toolCalls : undefined}
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
  );
}
```

- [ ] **Step 2: Create ChatInputBar**

Create `apps/web/src/components/chat/chat-input-bar.tsx`:

```typescript
"use client";

import { ChatInput } from "@/components/chat-input";

interface ChatInputBarProps {
  onSend: (message: string) => void;
  disabled?: boolean;
  /** Optional context label shown above the input */
  contextLabel?: string;
  onClearContext?: () => void;
}

export function ChatInputBar({
  onSend,
  disabled = false,
  contextLabel,
  onClearContext,
}: ChatInputBarProps) {
  return (
    <div>
      {contextLabel && (
        <div className="px-4 pt-2">
          <div className="flex items-center gap-2 rounded-md bg-blue-50 px-3 py-1.5 text-xs text-blue-700">
            <span className="truncate">Re: {contextLabel}</span>
            {onClearContext && (
              <button onClick={onClearContext} className="shrink-0 text-blue-400 hover:text-blue-600">
                <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
                  <line x1="18" y1="6" x2="6" y2="18" />
                  <line x1="6" y1="6" x2="18" y2="18" />
                </svg>
              </button>
            )}
          </div>
        </div>
      )}
      <ChatInput onSend={onSend} disabled={disabled} />
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/chat/
git commit -m "feat(web): add ChatMessageList and ChatInputBar shared components"
```

---

## Task 3: `useResearchPoller` Hook

**Files:**
- Create: `apps/web/src/hooks/use-research-poller.ts`

- [ ] **Step 1: Create the hook**

Create `apps/web/src/hooks/use-research-poller.ts`:

```typescript
"use client";

import { useCallback, useEffect, useRef, useState } from "react";

export interface ResearchPollResult {
  status: string;
  sourcesCompleted: string[];
  sourcesTotal: string[];
  synthesizedResult: Record<string, unknown> | null;
  isComplete: boolean;
}

interface UseResearchPollerOptions {
  taskId: string | null;
  intervalMs?: number;
  enabled?: boolean;
}

export function useResearchPoller({
  taskId,
  intervalMs = 3000,
  enabled = true,
}: UseResearchPollerOptions): ResearchPollResult {
  const [result, setResult] = useState<ResearchPollResult>({
    status: "pending",
    sourcesCompleted: [],
    sourcesTotal: [],
    synthesizedResult: null,
    isComplete: false,
  });

  const isCompleteRef = useRef(false);

  const poll = useCallback(async () => {
    if (!taskId || isCompleteRef.current) return;

    try {
      const res = await fetch(`/api/research/${encodeURIComponent(taskId)}`);
      if (!res.ok) return;

      const data = await res.json();
      const sourceResults: Record<string, { status: string }> = data.source_results ?? {};

      const sourcesTotal = Object.keys(sourceResults);
      const sourcesCompleted = sourcesTotal.filter(
        (name) => sourceResults[name]?.status === "succeeded",
      );

      const isComplete = data.status === "completed" || data.status === "failed";
      isCompleteRef.current = isComplete;

      setResult({
        status: data.status,
        sourcesCompleted,
        sourcesTotal,
        synthesizedResult: data.synthesized_result ?? null,
        isComplete,
      });
    } catch {
      // Silently fail — next poll will retry
    }
  }, [taskId]);

  useEffect(() => {
    if (!taskId || !enabled) return;

    // Initial poll immediately
    poll();

    const interval = setInterval(poll, intervalMs);
    return () => clearInterval(interval);
  }, [taskId, intervalMs, enabled, poll]);

  // Reset when taskId changes
  useEffect(() => {
    isCompleteRef.current = false;
    setResult({
      status: "pending",
      sourcesCompleted: [],
      sourcesTotal: [],
      synthesizedResult: null,
      isComplete: false,
    });
  }, [taskId]);

  return result;
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/hooks/use-research-poller.ts
git commit -m "feat(web): add useResearchPoller hook for research task polling"
```

---

## Task 4: API Route Proxies

**Files:**
- Create: `apps/web/src/app/api/research/route.ts`
- Create: `apps/web/src/app/api/research/sources/route.ts`
- Create: `apps/web/src/app/api/research/[taskId]/route.ts`
- Create: `apps/web/src/app/api/onboarding/interview/complete/route.ts`

- [ ] **Step 1: Create POST /api/research proxy**

Create `apps/web/src/app/api/research/route.ts`:

```typescript
import type { NextRequest } from "next/server";

const API_BASE = process.env.PRESCIENT_API_URL ?? "http://localhost:8000";

export async function POST(req: NextRequest) {
  const body = await req.json();

  const headers: Record<string, string> = { "Content-Type": "application/json" };
  const authHeader = req.headers.get("authorization");
  if (authHeader) headers["Authorization"] = authHeader;

  const upstream = await fetch(`${API_BASE}/research`, {
    method: "POST",
    headers,
    body: JSON.stringify(body),
  });

  const data = await upstream.json();
  return Response.json(data, { status: upstream.status });
}
```

- [ ] **Step 2: Create GET /api/research/sources proxy**

Create `apps/web/src/app/api/research/sources/route.ts`:

```typescript
import type { NextRequest } from "next/server";

const API_BASE = process.env.PRESCIENT_API_URL ?? "http://localhost:8000";

export async function GET(req: NextRequest) {
  const headers: Record<string, string> = {};
  const authHeader = req.headers.get("authorization");
  if (authHeader) headers["Authorization"] = authHeader;

  const upstream = await fetch(`${API_BASE}/research/sources`, { headers });

  const data = await upstream.json();
  return Response.json(data, { status: upstream.status });
}
```

- [ ] **Step 3: Create GET /api/research/[taskId] proxy**

Create `apps/web/src/app/api/research/[taskId]/route.ts`:

```typescript
import type { NextRequest } from "next/server";

const API_BASE = process.env.PRESCIENT_API_URL ?? "http://localhost:8000";

export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ taskId: string }> },
) {
  const { taskId } = await params;

  const headers: Record<string, string> = {};
  const authHeader = req.headers.get("authorization");
  if (authHeader) headers["Authorization"] = authHeader;

  const upstream = await fetch(`${API_BASE}/research/${encodeURIComponent(taskId)}`, { headers });

  const data = await upstream.json();
  return Response.json(data, { status: upstream.status });
}
```

- [ ] **Step 4: Create POST /api/onboarding/interview/complete proxy**

Create `apps/web/src/app/api/onboarding/interview/complete/route.ts`:

```typescript
import type { NextRequest } from "next/server";

const API_BASE = process.env.PRESCIENT_API_URL ?? "http://localhost:8000";

export async function POST(req: NextRequest) {
  const body = await req.json();

  const headers: Record<string, string> = { "Content-Type": "application/json" };
  const authHeader = req.headers.get("authorization");
  if (authHeader) headers["Authorization"] = authHeader;

  const upstream = await fetch(`${API_BASE}/onboarding/interview/complete`, {
    method: "POST",
    headers,
    body: JSON.stringify(body),
  });

  const data = await upstream.json();
  return Response.json(data, { status: upstream.status });
}
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/app/api/research/ apps/web/src/app/api/onboarding/
git commit -m "feat(web): add API route proxies for research and onboarding interview"
```

---

## Task 5: Source Progress Bar Component

**Files:**
- Create: `apps/web/src/components/research/source-progress-bar.tsx`

- [ ] **Step 1: Create the component**

Create `apps/web/src/components/research/source-progress-bar.tsx`:

```typescript
"use client";

import { useEffect, useState } from "react";

const SOURCE_LABELS: Record<string, string> = {
  website_scraper: "Website",
  search_engine: "Search",
  dns_lookup: "DNS",
  news_aggregator: "News",
  edgar: "SEC Filings",
};

interface SourceProgressBarProps {
  sourcesCompleted: string[];
  sourcesTotal: string[];
  isComplete: boolean;
}

export function SourceProgressBar({
  sourcesCompleted,
  sourcesTotal,
  isComplete,
}: SourceProgressBarProps) {
  const [hidden, setHidden] = useState(false);

  // Fade out after completion
  useEffect(() => {
    if (isComplete) {
      const timer = setTimeout(() => setHidden(true), 2000);
      return () => clearTimeout(timer);
    }
    setHidden(false);
  }, [isComplete]);

  if (hidden || sourcesTotal.length === 0) return null;

  return (
    <div
      className={`flex items-center gap-3 rounded-lg border border-neutral-200 bg-neutral-50 px-4 py-2 text-xs transition-opacity duration-500 ${
        isComplete ? "opacity-0" : "opacity-100"
      }`}
    >
      <span className="shrink-0 font-medium text-neutral-500">Researching:</span>
      <div className="flex flex-wrap gap-2">
        {sourcesTotal.map((name) => {
          const done = sourcesCompleted.includes(name);
          const label = SOURCE_LABELS[name] ?? name.replace(/_/g, " ");
          return (
            <span
              key={name}
              className={`inline-flex items-center gap-1 ${done ? "text-green-600" : "text-neutral-500"}`}
            >
              {done ? (
                <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="3">
                  <polyline points="20 6 9 17 4 12" />
                </svg>
              ) : (
                <span className="inline-block h-1.5 w-1.5 animate-pulse rounded-full bg-indigo-400" />
              )}
              {label}
            </span>
          );
        })}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/research/source-progress-bar.tsx
git commit -m "feat(web): add SourceProgressBar component for research progress"
```

---

## Task 6: Onboarding Interview Page

**Files:**
- Create: `apps/web/src/app/(main)/onboarding/interview/page.tsx`

- [ ] **Step 1: Create the interview page**

Create `apps/web/src/app/(main)/onboarding/interview/page.tsx`:

```typescript
"use client";

import { useCallback, useEffect, useRef, useState } from "react";
import { useRouter } from "next/navigation";

import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { ChatInputBar } from "@/components/chat/chat-input-bar";
import { ChatMessageList } from "@/components/chat/chat-message-list";
import { SourceProgressBar } from "@/components/research/source-progress-bar";
import { useResearchPoller } from "@/hooks/use-research-poller";
import { useStreamChat } from "@/hooks/use-stream-chat";
import { getToken } from "@/lib/auth";

type PageState = "form" | "interview" | "completing";

export default function OnboardingInterviewPage() {
  const router = useRouter();
  const [pageState, setPageState] = useState<PageState>("form");

  // Form fields
  const [name, setName] = useState("");
  const [role, setRole] = useState("");
  const [companyName, setCompanyName] = useState("");
  const [companyWebsite, setCompanyWebsite] = useState("");

  // Research task tracking
  const [researchTaskId, setResearchTaskId] = useState<string | null>(null);
  const researchContextSent = useRef(false);

  const research = useResearchPoller({
    taskId: researchTaskId,
    intervalMs: 3000,
    enabled: pageState === "interview",
  });

  const chat = useStreamChat({
    pageContext: "interview",
    extraBody: { mode: "interview" },
  });

  // Submit form → kick off research + start interview
  const handleFormSubmit = useCallback(
    async (e: React.FormEvent) => {
      e.preventDefault();

      // Kick off quick research
      try {
        const token = getToken();
        const headers: Record<string, string> = { "Content-Type": "application/json" };
        if (token) headers["Authorization"] = `Bearer ${token}`;

        const res = await fetch("/api/research", {
          method: "POST",
          headers,
          body: JSON.stringify({
            subject_type: "company",
            subject_data: {
              name: companyName,
              website: companyWebsite || undefined,
            },
            mode: "quick",
          }),
        });

        if (res.ok) {
          const data = await res.json();
          setResearchTaskId(data.task_id);
        }
      } catch {
        // Research failing shouldn't block the interview
      }

      setPageState("interview");

      // Send initial greeting to start the conversation
      await chat.send(
        `Hi, I'm ${name}. I'm ${role} at ${companyName}.`,
      );
    },
    [name, role, companyName, companyWebsite, chat],
  );

  // Inject research context when findings arrive
  useEffect(() => {
    if (
      research.isComplete &&
      research.synthesizedResult &&
      !researchContextSent.current &&
      !chat.isStreaming
    ) {
      researchContextSent.current = true;
      // The next message sent will include research context
      // The backend uses this to enrich the interview system prompt
    }
  }, [research.isComplete, research.synthesizedResult, chat.isStreaming]);

  // Complete interview
  const handleComplete = useCallback(async () => {
    setPageState("completing");

    try {
      const token = getToken();
      const headers: Record<string, string> = { "Content-Type": "application/json" };
      if (token) headers["Authorization"] = `Bearer ${token}`;

      await fetch("/api/onboarding/interview/complete", {
        method: "POST",
        headers,
        body: JSON.stringify({ conversation_id: chat.conversationId }),
      });

      router.push("/brief");
    } catch {
      // If completion fails, redirect anyway — data was captured in the chat
      router.push("/brief");
    }
  }, [chat.conversationId, router]);

  // ---------- Form State ----------
  if (pageState === "form") {
    return (
      <div className="mx-auto flex min-h-[60vh] max-w-md items-center">
        <Card className="w-full">
          <CardHeader>
            <CardTitle className="text-lg">Let us learn about you</CardTitle>
            <p className="text-sm text-neutral-500">
              We'll research your company and ask a few questions to personalize your experience.
              Takes about 2-3 minutes.
            </p>
          </CardHeader>
          <CardContent>
            <form onSubmit={handleFormSubmit} className="space-y-4">
              <div>
                <label className="mb-1 block text-sm font-medium text-neutral-700">Your name</label>
                <input
                  type="text"
                  value={name}
                  onChange={(e) => setName(e.target.value)}
                  required
                  className="w-full rounded-md border border-neutral-300 px-3 py-2 text-sm focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
                />
              </div>
              <div>
                <label className="mb-1 block text-sm font-medium text-neutral-700">Your role</label>
                <input
                  type="text"
                  value={role}
                  onChange={(e) => setRole(e.target.value)}
                  required
                  placeholder="e.g., VP of Strategy, Portfolio Manager"
                  className="w-full rounded-md border border-neutral-300 px-3 py-2 text-sm focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
                />
              </div>
              <div>
                <label className="mb-1 block text-sm font-medium text-neutral-700">Company name</label>
                <input
                  type="text"
                  value={companyName}
                  onChange={(e) => setCompanyName(e.target.value)}
                  required
                  className="w-full rounded-md border border-neutral-300 px-3 py-2 text-sm focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
                />
              </div>
              <div>
                <label className="mb-1 block text-sm font-medium text-neutral-700">
                  Company website <span className="text-neutral-400">(optional)</span>
                </label>
                <input
                  type="url"
                  value={companyWebsite}
                  onChange={(e) => setCompanyWebsite(e.target.value)}
                  placeholder="https://..."
                  className="w-full rounded-md border border-neutral-300 px-3 py-2 text-sm focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
                />
              </div>
              <Button
                type="submit"
                className="w-full bg-indigo-600 hover:bg-indigo-700"
                disabled={!name.trim() || !role.trim() || !companyName.trim()}
              >
                Start interview
              </Button>
            </form>
          </CardContent>
        </Card>
      </div>
    );
  }

  // ---------- Completing State ----------
  if (pageState === "completing") {
    return (
      <div className="mx-auto flex min-h-[60vh] max-w-md items-center justify-center">
        <div className="text-center">
          <div className="mx-auto h-8 w-8 animate-spin rounded-full border-2 border-indigo-600 border-t-transparent" />
          <p className="mt-4 text-sm text-neutral-500">Setting things up...</p>
        </div>
      </div>
    );
  }

  // ---------- Interview State ----------
  const showFinishButton =
    !chat.isStreaming &&
    chat.messages.length >= 6 &&
    chat.messages.at(-1)?.role === "assistant" &&
    (chat.messages.at(-1)?.content?.length ?? 0) > 100;

  return (
    <div className="mx-auto flex h-[calc(100vh-120px)] max-w-2xl flex-col">
      {/* Source progress bar */}
      {researchTaskId && (
        <div className="mb-3">
          <SourceProgressBar
            sourcesCompleted={research.sourcesCompleted}
            sourcesTotal={research.sourcesTotal}
            isComplete={research.isComplete}
          />
        </div>
      )}

      {/* Chat messages */}
      <ChatMessageList
        messages={chat.messages}
        isStreaming={chat.isStreaming}
        showToolCalls={false}
        className="rounded-lg border border-neutral-200"
      />

      {/* Finish button */}
      {showFinishButton && (
        <div className="flex justify-center py-3">
          <Button
            onClick={handleComplete}
            className="bg-indigo-600 hover:bg-indigo-700"
          >
            Finish setup
          </Button>
        </div>
      )}

      {/* Chat input */}
      <ChatInputBar onSend={chat.send} disabled={chat.isStreaming} />
    </div>
  );
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cd apps/web && npx next build 2>&1 | tail -20`
Or for faster feedback: `cd apps/web && npx next lint 2>&1 | tail -10`

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/\\(main\\)/onboarding/interview/
git commit -m "feat(web): add onboarding interview page with concurrent research"
```

---

## Task 7: Research Controls Component

**Files:**
- Create: `apps/web/src/components/research/research-controls.tsx`

- [ ] **Step 1: Create the component**

Create `apps/web/src/components/research/research-controls.tsx`:

```typescript
"use client";

import { useEffect, useState } from "react";
import { Button } from "@/components/ui/button";

export interface SourceDescriptor {
  name: string;
  label: string;
  tier: string;
  timeout_seconds: number;
}

interface ResearchControlsProps {
  depth: string;
  onDepthChange: (depth: string) => void;
  enabledSources: string[];
  onSourcesChange: (sources: string[]) => void;
  availableSources: SourceDescriptor[];
}

const TIER_RANK: Record<string, number> = { quick: 0, standard: 1, deep: 2 };

export function ResearchControls({
  depth,
  onDepthChange,
  enabledSources,
  onSourcesChange,
  availableSources,
}: ResearchControlsProps) {
  const [isCollapsed, setIsCollapsed] = useState(false);

  // When depth changes, auto-select sources at that tier and below
  const handleDepthChange = (newDepth: string) => {
    onDepthChange(newDepth);
    const depthRank = TIER_RANK[newDepth] ?? 0;
    const autoSelected = availableSources
      .filter((s) => (TIER_RANK[s.tier] ?? 0) <= depthRank)
      .map((s) => s.name);
    onSourcesChange(autoSelected);
  };

  const toggleSource = (name: string) => {
    if (enabledSources.includes(name)) {
      onSourcesChange(enabledSources.filter((s) => s !== name));
    } else {
      onSourcesChange([...enabledSources, name]);
    }
  };

  if (isCollapsed) {
    return (
      <button
        onClick={() => setIsCollapsed(false)}
        className="flex w-full items-center gap-2 rounded-lg border border-neutral-200 bg-neutral-50 px-4 py-2 text-xs text-neutral-500 hover:bg-neutral-100"
      >
        <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
          <polyline points="6 9 12 15 18 9" />
        </svg>
        Research settings: {depth} mode, {enabledSources.length} sources
      </button>
    );
  }

  return (
    <div className="rounded-lg border border-neutral-200 bg-neutral-50 p-4">
      <div className="flex items-center justify-between">
        <div className="flex items-center gap-4">
          {/* Depth presets */}
          <div className="flex items-center gap-1">
            <span className="mr-2 text-xs font-medium text-neutral-500">Depth:</span>
            {["quick", "standard", "deep"].map((d) => (
              <button
                key={d}
                onClick={() => handleDepthChange(d)}
                className={`rounded-md px-3 py-1 text-xs font-medium transition-colors ${
                  depth === d
                    ? "bg-indigo-600 text-white"
                    : "bg-white text-neutral-600 hover:bg-neutral-100"
                }`}
              >
                {d.charAt(0).toUpperCase() + d.slice(1)}
              </button>
            ))}
          </div>

          {/* Source toggles */}
          <div className="flex items-center gap-2 border-l border-neutral-300 pl-4">
            {availableSources.map((source) => (
              <label
                key={source.name}
                className="flex items-center gap-1.5 text-xs text-neutral-600"
              >
                <input
                  type="checkbox"
                  checked={enabledSources.includes(source.name)}
                  onChange={() => toggleSource(source.name)}
                  className="h-3.5 w-3.5 rounded border-neutral-300 text-indigo-600 focus:ring-indigo-500"
                />
                {source.label}
              </label>
            ))}
          </div>
        </div>

        {/* Collapse button */}
        <button
          onClick={() => setIsCollapsed(true)}
          className="text-neutral-400 hover:text-neutral-600"
        >
          <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
            <polyline points="18 15 12 9 6 15" />
          </svg>
        </button>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/research/research-controls.tsx
git commit -m "feat(web): add ResearchControls component with depth presets and source toggles"
```

---

## Task 8: Research Result Card and Progress Card

**Files:**
- Create: `apps/web/src/components/research/research-result-card.tsx`
- Create: `apps/web/src/components/research/research-progress-card.tsx`

- [ ] **Step 1: Create ResearchResultCard**

Create `apps/web/src/components/research/research-result-card.tsx`:

```typescript
"use client";

import { useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { MarkdownContent } from "@/components/markdown-content";

interface ResearchBlock {
  block_id: string;
  heading: string;
  body: string;
  order: number;
}

interface ResearchResultCardProps {
  subjectName: string;
  subjectType: "company" | "person" | "market";
  blocks: ResearchBlock[];
  summary: string | null;
  onWatch?: () => void;
  onDeeper?: () => void;
}

const TYPE_BADGE: Record<string, { bg: string; text: string }> = {
  company: { bg: "bg-blue-100", text: "text-blue-700" },
  person: { bg: "bg-purple-100", text: "text-purple-700" },
  market: { bg: "bg-green-100", text: "text-green-700" },
};

export function ResearchResultCard({
  subjectName,
  subjectType,
  blocks,
  summary,
  onWatch,
  onDeeper,
}: ResearchResultCardProps) {
  const [expandedBlocks, setExpandedBlocks] = useState<Set<string>>(
    new Set(blocks.slice(0, 2).map((b) => b.block_id)),
  );

  const badge = TYPE_BADGE[subjectType] ?? TYPE_BADGE.company;
  const sortedBlocks = [...blocks].sort((a, b) => a.order - b.order);

  const toggleBlock = (blockId: string) => {
    setExpandedBlocks((prev) => {
      const next = new Set(prev);
      if (next.has(blockId)) next.delete(blockId);
      else next.add(blockId);
      return next;
    });
  };

  return (
    <Card className="overflow-hidden">
      <CardHeader className="pb-2">
        <div className="flex items-center gap-2">
          <span className={`rounded-full px-2 py-0.5 text-xs font-medium ${badge.bg} ${badge.text}`}>
            {subjectType.charAt(0).toUpperCase() + subjectType.slice(1)}
          </span>
          <CardTitle className="text-base">{subjectName}</CardTitle>
        </div>
        {summary && (
          <p className="mt-1 text-sm text-neutral-500">{summary}</p>
        )}
      </CardHeader>
      <CardContent className="space-y-2 pt-0">
        {sortedBlocks.map((block) => (
          <div key={block.block_id} className="border-t border-neutral-100 pt-2">
            <button
              onClick={() => toggleBlock(block.block_id)}
              className="flex w-full items-center justify-between text-left"
            >
              <span className="text-sm font-medium text-neutral-700">{block.heading}</span>
              <svg
                width="14"
                height="14"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                strokeWidth="2"
                className={`text-neutral-400 transition-transform ${expandedBlocks.has(block.block_id) ? "rotate-180" : ""}`}
              >
                <polyline points="6 9 12 15 18 9" />
              </svg>
            </button>
            {expandedBlocks.has(block.block_id) && (
              <div className="mt-1 text-sm text-neutral-600">
                <MarkdownContent content={block.body} />
              </div>
            )}
          </div>
        ))}

        {/* Actions */}
        <div className="flex gap-2 border-t border-neutral-100 pt-3">
          {onWatch && (
            <Button variant="outline" size="sm" onClick={onWatch} className="text-xs">
              Watch
            </Button>
          )}
          {onDeeper && (
            <Button variant="outline" size="sm" onClick={onDeeper} className="text-xs">
              Go deeper
            </Button>
          )}
        </div>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 2: Create ResearchProgressCard**

Create `apps/web/src/components/research/research-progress-card.tsx`:

```typescript
"use client";

import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { SourceProgressBar } from "@/components/research/source-progress-bar";

interface ResearchProgressCardProps {
  subjectName: string;
  sourcesCompleted: string[];
  sourcesTotal: string[];
}

export function ResearchProgressCard({
  subjectName,
  sourcesCompleted,
  sourcesTotal,
}: ResearchProgressCardProps) {
  return (
    <Card className="overflow-hidden">
      <CardHeader className="pb-2">
        <div className="flex items-center gap-2">
          <span className="rounded-full bg-amber-100 px-2 py-0.5 text-xs font-medium text-amber-700">
            In progress
          </span>
          <CardTitle className="text-base">{subjectName}</CardTitle>
        </div>
      </CardHeader>
      <CardContent className="pt-0">
        <SourceProgressBar
          sourcesCompleted={sourcesCompleted}
          sourcesTotal={sourcesTotal}
          isComplete={false}
        />
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/research/research-result-card.tsx apps/web/src/components/research/research-progress-card.tsx
git commit -m "feat(web): add ResearchResultCard and ResearchProgressCard components"
```

---

## Task 9: `/research` Page

**Files:**
- Create: `apps/web/src/app/(main)/research/page.tsx`

- [ ] **Step 1: Create the research page**

Create `apps/web/src/app/(main)/research/page.tsx`:

```typescript
"use client";

import { useCallback, useEffect, useState } from "react";

import { ChatInputBar } from "@/components/chat/chat-input-bar";
import { ChatMessageList } from "@/components/chat/chat-message-list";
import {
  ResearchControls,
  type SourceDescriptor,
} from "@/components/research/research-controls";
import { ResearchResultCard } from "@/components/research/research-result-card";
import { useStreamChat } from "@/hooks/use-stream-chat";

const SUGGESTED_PROMPTS = [
  "Research a company — tell me the name and I'll look them up",
  "Analyze a market — which industry should I explore?",
  "Look up a person — who would you like to know about?",
];

export default function ResearchPage() {
  // Research controls state
  const [depth, setDepth] = useState("standard");
  const [enabledSources, setEnabledSources] = useState<string[]>([]);
  const [availableSources, setAvailableSources] = useState<SourceDescriptor[]>([]);

  // Research results (from chat tool calls)
  const [results, setResults] = useState<
    Array<{
      subjectName: string;
      subjectType: "company" | "person" | "market";
      blocks: Array<{ block_id: string; heading: string; body: string; order: number }>;
      summary: string | null;
      taskId: string;
    }>
  >([]);

  // Chat
  const chat = useStreamChat({
    pageContext: JSON.stringify({ scope: "research", depth, enabled_sources: enabledSources }),
  });

  // Fetch available sources on mount
  useEffect(() => {
    fetch("/api/research/sources")
      .then((r) => (r.ok ? r.json() : []))
      .then((data: SourceDescriptor[]) => {
        setAvailableSources(data);
        // Default: select sources at standard tier and below
        const standardRank = 1;
        const tierRank: Record<string, number> = { quick: 0, standard: 1, deep: 2 };
        const defaults = data
          .filter((s) => (tierRank[s.tier] ?? 0) <= standardRank)
          .map((s) => s.name);
        setEnabledSources(defaults);
      })
      .catch(() => {});
  }, []);

  // Send a message (used by suggested prompts and card actions)
  const handleAction = useCallback(
    (message: string) => {
      chat.send(message);
    },
    [chat],
  );

  // Empty state for chat
  const chatEmptyState = (
    <div className="flex h-full flex-col items-center justify-center text-center">
      <p className="text-sm font-medium text-neutral-900">Research anything</p>
      <p className="mt-1 max-w-xs text-xs text-neutral-500">
        Ask me to research a company, analyze a market, or look up a person.
      </p>
      <div className="mt-4 space-y-2">
        {SUGGESTED_PROMPTS.map((prompt) => (
          <button
            key={prompt}
            onClick={() => handleAction(prompt)}
            disabled={chat.isStreaming}
            className="block w-full rounded-lg border border-neutral-200 px-4 py-2 text-left text-xs text-neutral-600 transition-colors hover:border-indigo-300 hover:bg-indigo-50 disabled:opacity-50"
          >
            {prompt}
          </button>
        ))}
      </div>
    </div>
  );

  return (
    <div className="flex h-[calc(100vh-120px)] flex-col">
      {/* Controls */}
      <div className="mb-4">
        <ResearchControls
          depth={depth}
          onDepthChange={setDepth}
          enabledSources={enabledSources}
          onSourcesChange={setEnabledSources}
          availableSources={availableSources}
        />
      </div>

      {/* Split view */}
      <div className="flex min-h-0 flex-1 gap-4">
        {/* Results panel (60%) */}
        <div className="flex w-3/5 flex-col overflow-y-auto">
          {results.length === 0 ? (
            <div className="flex flex-1 items-center justify-center rounded-lg border border-dashed border-neutral-300">
              <p className="text-sm text-neutral-400">
                Research results will appear here as you ask questions.
              </p>
            </div>
          ) : (
            <div className="space-y-4">
              {results.map((r, i) => (
                <ResearchResultCard
                  key={i}
                  subjectName={r.subjectName}
                  subjectType={r.subjectType}
                  blocks={r.blocks}
                  summary={r.summary}
                  onWatch={() => handleAction(`Watch ${r.subjectName}`)}
                  onDeeper={() => handleAction(`Do a deep dive on ${r.subjectName}`)}
                />
              ))}
            </div>
          )}
        </div>

        {/* Chat panel (40%) */}
        <div className="flex w-2/5 flex-col rounded-lg border border-neutral-200">
          <ChatMessageList
            messages={chat.messages}
            isStreaming={chat.isStreaming}
            emptyState={chatEmptyState}
          />
          <ChatInputBar onSend={chat.send} disabled={chat.isStreaming} />
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cd apps/web && npx next lint 2>&1 | tail -10`

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/\\(main\\)/research/
git commit -m "feat(web): add /research page with split-view, controls, and chat"
```

---

## Task 10: Update Onboarding Landing Page

**Files:**
- Modify: `apps/web/src/app/(main)/onboarding/page.tsx`

- [ ] **Step 1: Add dual-path entry and rename step 2**

Replace the entire file `apps/web/src/app/(main)/onboarding/page.tsx`:

```typescript
"use client";

import Link from "next/link";
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Button } from "@/components/ui/button";

const STEPS = [
  {
    title: "Company Setup",
    description:
      "Tell us about your firm — name and industry — so we can personalize your intelligence.",
    href: "/onboarding/company",
    step: 1,
  },
  {
    title: "Companies to Watch",
    description:
      "Add the companies you track. We'll research each one and build live intelligence profiles.",
    href: "/onboarding/competitors",
    step: 2,
  },
  {
    title: "Interests",
    description: "Select the topics that matter to your team so your Morning Brief stays relevant.",
    href: "/onboarding/interests",
    step: 3,
  },
];

export default function OnboardingPage() {
  return (
    <div className="mx-auto max-w-2xl space-y-8">
      <div>
        <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">
          Welcome to Prescient OS
        </h1>
        <p className="mt-2 text-sm text-neutral-500">
          Choose how you'd like to get started.
        </p>
      </div>

      {/* Interview path */}
      <Card className="border-indigo-200 bg-indigo-50/50 transition-shadow hover:shadow-md">
        <CardHeader className="pb-3">
          <CardTitle className="text-base text-indigo-900">Let us learn about you</CardTitle>
        </CardHeader>
        <CardContent className="flex items-center justify-between">
          <CardDescription className="max-w-sm text-indigo-700/70">
            We'll research your company and ask a few questions to personalize your experience.
            Takes about 2-3 minutes.
          </CardDescription>
          <Button asChild size="sm" className="ml-4 shrink-0 bg-indigo-600 hover:bg-indigo-700">
            <Link href="/onboarding/interview">Start interview</Link>
          </Button>
        </CardContent>
      </Card>

      {/* Divider */}
      <div className="flex items-center gap-4">
        <div className="h-px flex-1 bg-neutral-200" />
        <span className="text-xs text-neutral-400">Or set up manually</span>
        <div className="h-px flex-1 bg-neutral-200" />
      </div>

      {/* Manual wizard steps */}
      <div className="space-y-4">
        {STEPS.map((step) => (
          <Card key={step.step} className="transition-shadow hover:shadow-md">
            <CardHeader className="pb-3">
              <div className="flex items-center gap-3">
                <span className="flex h-7 w-7 shrink-0 items-center justify-center rounded-full bg-neutral-100 text-sm font-semibold text-neutral-700">
                  {step.step}
                </span>
                <CardTitle className="text-base">{step.title}</CardTitle>
              </div>
            </CardHeader>
            <CardContent className="flex items-center justify-between">
              <CardDescription className="max-w-sm">{step.description}</CardDescription>
              <Button asChild variant="outline" size="sm" className="ml-4 shrink-0">
                <Link href={step.href}>Begin</Link>
              </Button>
            </CardContent>
          </Card>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/app/\\(main\\)/onboarding/page.tsx
git commit -m "feat(web): add dual-path onboarding entry with interview option"
```

---

## Task 11: Notification Polling

**Files:**
- Modify: `apps/web/src/components/notifications-bell.tsx`

- [ ] **Step 1: Add polling interval**

In `apps/web/src/components/notifications-bell.tsx`, update the `useEffect` that fetches the count to add a 30-second polling interval:

Replace the existing `useEffect`:

```typescript
  useEffect(() => {
    const userId = getUserId();
    if (!userId) return;

    fetch(`/api/notifications/count?recipient_id=${encodeURIComponent(userId)}`)
      .then((r) => (r.ok ? r.json() : Promise.reject(r)))
      .then((data: CountResponse) => setCount(data.total))
      .catch(() => {});
  }, []);
```

With:

```typescript
  useEffect(() => {
    const userId = getUserId();
    if (!userId) return;

    const fetchCount = () => {
      fetch(`/api/notifications/count?recipient_id=${encodeURIComponent(userId)}`)
        .then((r) => (r.ok ? r.json() : Promise.reject(r)))
        .then((data: CountResponse) => setCount(data.total))
        .catch(() => {});
    };

    // Initial fetch
    fetchCount();

    // Poll every 30 seconds
    const interval = setInterval(fetchCount, 30_000);
    return () => clearInterval(interval);
  }, []);
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/notifications-bell.tsx
git commit -m "feat(web): add 30-second polling to notification bell"
```

---

## Task 12: Refactor CopilotPanel to Use Shared Primitives

**Files:**
- Modify: `apps/web/src/components/copilot/copilot-panel.tsx`

- [ ] **Step 1: Refactor to use ChatMessageList and ChatInputBar**

Replace the messages rendering section and input section in `copilot-panel.tsx` with the shared components. The panel keeps its Zustand store integration but delegates rendering to shared primitives.

Replace the entire file `apps/web/src/components/copilot/copilot-panel.tsx`:

```typescript
"use client";

import { useCallback, useEffect, useRef, useState } from "react";

import { ChatInputBar } from "@/components/chat/chat-input-bar";
import { ChatMessageList } from "@/components/chat/chat-message-list";
import { Button } from "@/components/ui/button";
import { getToken } from "@/lib/auth";
import type { Citation } from "@/lib/chat-stream";
import { streamChat } from "@/lib/chat-stream";
import { useCopilotStore } from "@/stores/copilot-store";
import { SuggestedPrompts } from "./suggested-prompts";

/* -------------------------------------------------------------------------- */
/*  Inline citation detail                                                     */
/* -------------------------------------------------------------------------- */

function InlineCitationDetail({ citation, onClose }: { citation: Citation; onClose: () => void }) {
  return (
    <div className="rounded-md border border-neutral-200 bg-neutral-50 p-3">
      <div className="flex items-center justify-between">
        <span className="text-xs font-semibold text-neutral-700">
          Source [{citation.marker}] — {citation.source_type}
        </span>
        <button onClick={onClose} className="text-neutral-400 hover:text-neutral-600">
          <svg
            width="12"
            height="12"
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

  const pendingQuestion = useCopilotStore((s) => s.pendingQuestion);
  const clearPending = useCopilotStore((s) => s.setPendingQuestion);

  const [expandedCitation, setExpandedCitation] = useState<Citation | null>(null);

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
          focusCompanyId: pageContext?.companyId,
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
    [
      conversationId,
      pageContext,
      addUserMessage,
      addAssistantMessage,
      updateAssistant,
      setConversationId,
      setIsStreaming,
    ],
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

  // Consume pending question from omni-search or other triggers
  const pendingConsumed = useRef(false);
  useEffect(() => {
    if (isOpen && pendingQuestion && !pendingConsumed.current) {
      pendingConsumed.current = true;
      clearPending(null);
      const t = setTimeout(() => {
        void handleSend(pendingQuestion);
      }, 100);
      return () => clearTimeout(t);
    }
    if (!pendingQuestion) {
      pendingConsumed.current = false;
    }
  }, [isOpen, pendingQuestion, clearPending, handleSend]);

  // Build context label
  let contextLabel = "";
  if (pageContext) {
    // eslint-disable-next-line security/detect-object-injection
    const pageLabel = PAGE_LABELS[pageContext.pageType] ?? pageContext.pageType;
    if (pageContext.companyId) {
      contextLabel = `Company ${pageContext.companyId.slice(0, 8)} — ${pageLabel}`;
    } else {
      contextLabel = pageLabel;
    }
  }

  // Empty state with suggested prompts
  const emptyState = (
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
  );

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
          {contextLabel && <p className="truncate text-xs text-neutral-500">{contextLabel}</p>}
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
            <svg
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
      </div>

      {/* Messages — now using shared ChatMessageList */}
      <ChatMessageList
        messages={messages}
        isStreaming={isStreaming}
        onCitationClick={handleCitationClick}
        emptyState={emptyState}
      />

      {/* Inline citation detail */}
      {expandedCitation && (
        <div className="px-4 pb-2">
          <InlineCitationDetail
            citation={expandedCitation}
            onClose={() => setExpandedCitation(null)}
          />
        </div>
      )}

      {/* Input — now using shared ChatInputBar */}
      <ChatInputBar
        onSend={handleSend}
        disabled={isStreaming}
        contextLabel={pageContext?.selectedItemSummary}
        onClearContext={() =>
          setPageContext(
            pageContext
              ? { pageType: pageContext.pageType, companyId: pageContext.companyId }
              : null,
          )
        }
      />
    </div>
  );
}
```

- [ ] **Step 2: Verify the app still compiles and the panel works**

Run: `cd apps/web && npx next build 2>&1 | tail -20`

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/copilot/copilot-panel.tsx
git commit -m "refactor(web): CopilotPanel uses shared ChatMessageList and ChatInputBar"
```

---

## Task 13: Final Verification

- [ ] **Step 1: Run build**

Run: `cd apps/web && npx next build 2>&1 | tail -30`
Expected: Build succeeds with no errors.

- [ ] **Step 2: Run lint**

Run: `cd apps/web && npx next lint 2>&1 | tail -10`
Expected: No lint errors.

- [ ] **Step 3: Fix any issues found**

Address any build or lint errors.

- [ ] **Step 4: Final commit if fixes were needed**

```bash
git add -A
git commit -m "fix: resolve build/lint issues from frontend implementation"
```

---

## Summary

| Layer | Tasks | What It Delivers |
|-------|-------|-----------------|
| **Shared Primitives** | Tasks 1-3 | `useStreamChat`, `useResearchPoller`, `ChatMessageList`, `ChatInputBar` |
| **API Routes** | Task 4 | Proxies for research and onboarding endpoints |
| **Research Components** | Tasks 5, 7-8 | `SourceProgressBar`, `ResearchControls`, `ResearchResultCard`, `ResearchProgressCard` |
| **Pages** | Tasks 6, 9 | `/onboarding/interview`, `/research` |
| **Updates** | Tasks 10-12 | Dual-path onboarding, notification polling, CopilotPanel refactor |
| **Verification** | Task 13 | Build + lint clean |

**Total tasks:** 13
**Estimated commits:** 13-15
