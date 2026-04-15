# Research Engine & Smart Onboarding — Frontend Design Spec

> Frontend companion to the Phase 2 backend spec (`2026-04-14-research-onboarding-phase2-design.md`). The backend delivers async research infrastructure, source tiers, research tools, and data model updates. This spec covers the frontend: shared chat primitives, onboarding interview UI, standalone `/research` page, and supporting changes.

## Overview

Four workstreams:

1. **Shared chat primitives** — extract reusable hooks and components from `CopilotPanel` so both the interview page and research page can compose their own chat UIs
2. **Onboarding interview page** — full-page concurrent research + chat interview with source-level progress
3. **Standalone `/research` page** — split-view with results cards, depth/source controls, and chat
4. **Supporting changes** — onboarding landing page dual path, notification polling, SSE event handling

---

## 1. Shared Chat Primitives

### 1.1 `useStreamChat` Hook

Wraps `streamChat()` from `lib/chat-stream.ts` with React state management.

**Interface:**

```typescript
interface UseStreamChatOptions {
  authToken?: string;
  companyId?: string;
  pageContext?: string;
  /** Extra fields merged into the request body (e.g., mode, research_context) */
  extraBody?: Record<string, unknown>;
}

interface UseStreamChatReturn {
  messages: CopilotMessage[];
  isStreaming: boolean;
  conversationId: string | undefined;
  send: (question: string) => Promise<void>;
  newConversation: () => void;
}
```

**Behavior:**
- Manages `messages` array, `isStreaming`, and `conversationId` as local React state (not Zustand)
- `send()` appends a user message, creates an empty assistant message, then iterates over `streamChat()` events updating the assistant message incrementally
- Handles all event types: `init`, `tool_start`, `tool_end`, `delta`, `citation`, `done`, `error`
- `newConversation()` resets messages and conversationId

**File:** `apps/web/src/hooks/use-stream-chat.ts`

### 1.2 `ChatMessageList` Component

Renders a scrollable list of `CopilotMessage[]` with auto-scroll.

**Props:**

```typescript
interface ChatMessageListProps {
  messages: CopilotMessage[];
  isStreaming: boolean;
  /** Whether to show tool call chips (default: true) */
  showToolCalls?: boolean;
  /** Whether to show citation pills (default: true) */
  showCitations?: boolean;
  /** Optional empty state content */
  emptyState?: React.ReactNode;
}
```

**Behavior:**
- Maps messages to existing `ChatMessage` component (reused, not rewritten)
- Auto-scrolls to bottom on new content (using `useEffect` + `scrollIntoView`)
- Shows `emptyState` when messages array is empty
- Shows typing indicator (animated dots) when `isStreaming` and last message has no content yet

**File:** `apps/web/src/components/chat/chat-message-list.tsx`

### 1.3 `ChatInputBar` Component

Clean wrapper around existing `ChatInput`.

**Props:**

```typescript
interface ChatInputBarProps {
  onSend: (message: string) => void;
  disabled?: boolean;
  placeholder?: string;
  /** Optional context chip above the input */
  contextLabel?: string;
  onClearContext?: () => void;
}
```

**File:** `apps/web/src/components/chat/chat-input-bar.tsx`

### 1.4 `useResearchPoller` Hook

Polls a research task for status updates.

**Interface:**

```typescript
interface UseResearchPollerOptions {
  taskId: string | null;
  intervalMs?: number; // default: 3000
  enabled?: boolean;   // default: true
}

interface ResearchPollResult {
  status: string;           // "pending" | "discovering" | "fetching" | ... | "completed" | "failed"
  sourcesCompleted: string[]; // e.g., ["website_scraper", "search_engine"]
  sourcesTotal: string[];     // all sources that were dispatched
  synthesizedResult: object | null;
  isComplete: boolean;
}
```

**Behavior:**
- Polls `GET /research/{taskId}` (proxied via Next.js API route) at `intervalMs`
- Parses `source_results` to derive `sourcesCompleted` (status === "succeeded")
- Stops polling when `status` is "completed" or "failed"
- Returns `isComplete` for easy conditional checks

**File:** `apps/web/src/hooks/use-research-poller.ts`

**New API route:** `apps/web/src/app/api/research/[taskId]/route.ts` — proxies to backend `GET /research/{taskId}`

### 1.5 Refactor CopilotPanel

The existing `CopilotPanel` is refactored to compose from the shared primitives:
- Replace inline message rendering with `ChatMessageList`
- Replace inline input with `ChatInputBar`
- Keep the Zustand store (`copilot-store.ts`) — the panel still uses it for global state (open/close, pending questions, page context)
- The `handleSend` logic moves into `useStreamChat`, but the panel wraps it to also interact with the store (setting `isStreaming`, etc.)

This is a refactor of existing behavior — no functional changes to the copilot panel.

---

## 2. Onboarding Interview Page

### Route: `/onboarding/interview`

**File:** `apps/web/src/app/(main)/onboarding/interview/page.tsx`

### 2.1 Page States

The page has 3 sequential states managed by local React state:

**State 1: Initial Form**
- Centered card (max-width `md`) with 4 fields:
  - Your name (text input, required)
  - Your role (text input, required)
  - Company name (text input, required)
  - Company website (text input, optional)
- "Start" button submits:
  - Calls `POST /api/research` with `{subject_type: "company", subject_data: {name, website}, mode: "quick"}`
  - Saves the returned `task_id` to local state
  - Transitions to State 2

**State 2: Interview Chat**
- Full-page chat layout, centered, max-width `2xl`
- **Source progress bar** above the chat (see 2.2)
- `useStreamChat` with `extraBody: { mode: "interview" }` — this tells the backend to use the interview system prompt
- `useResearchPoller` with the `task_id` from State 1
- When `useResearchPoller` detects new source completions, the next `send()` call includes `research_context` in the extra body so the backend can enrich the system prompt
- Chat starts automatically with an empty send (or a greeting message from the backend)
- When the LLM's response contains a wrap-up summary (detected by the presence of a final "Here's what I've set up for you..." style message), a "Finish setup" button appears below the last message

**State 3: Completion**
- Clicking "Finish setup" calls `POST /api/onboarding/interview/complete` with `{conversation_id}`
- Shows a centered spinner with "Setting things up..." text
- On success, redirects to `/brief`

### 2.2 Source Progress Bar

A thin horizontal bar above the chat area showing research source status.

**Component:** `apps/web/src/components/research/source-progress-bar.tsx`

```typescript
interface SourceProgressBarProps {
  sourcesCompleted: string[];
  sourcesTotal: string[];
  isComplete: boolean;
}
```

**Rendering:**
- Each source shown as a small label with icon:
  - Completed: `✓ Website` (green check, muted text)
  - In progress: `⋯ Search` (animated dots, normal text)
- When all sources complete, the bar collapses with a brief "Research complete" message that fades out after 2 seconds
- Compact design — should not distract from the chat

### 2.3 Backend API Routes (Next.js proxy)

- `POST /api/research` — proxy to `POST /research` (already exists if the research routes are mounted)
- `POST /api/onboarding/interview/complete` — proxy to `POST /onboarding/interview/complete` (new backend endpoint from Phase 2 spec)

---

## 3. Standalone `/research` Page

### Route: `/research`

**File:** `apps/web/src/app/(main)/research/page.tsx`

### 3.1 Layout

Split view with a collapsible controls bar at the top:

```
┌─────────────────────────────────────────────────────────┐
│  Research Controls (collapsible)                         │
│  [Quick] [Standard ▾] [Deep]   ☑Web ☑Search ☑DNS ☐News │
├──────────────────────────────────┬────────────────────────┤
│  Research Results (60%)          │  Chat (40%)            │
│                                  │                        │
│  ResearchResultCard[]            │  ChatMessageList       │
│  ResearchProgressCard[]          │  ChatInputBar          │
│                                  │                        │
│                                  │  Suggested prompts     │
│                                  │  in empty state        │
└──────────────────────────────────┴────────────────────────┘
```

**Responsive behavior:** On narrow screens (< 1024px), the layout stacks vertically — results on top, chat below. The chat panel gets a minimum height of 400px.

### 3.2 Research Controls

**Component:** `apps/web/src/components/research/research-controls.tsx`

```typescript
interface ResearchControlsProps {
  depth: "quick" | "standard" | "deep";
  onDepthChange: (depth: string) => void;
  enabledSources: string[];
  onSourcesChange: (sources: string[]) => void;
  availableSources: SourceDescriptor[];
  isCollapsed: boolean;
  onToggleCollapse: () => void;
}

interface SourceDescriptor {
  name: string;
  label: string;
  tier: string;
  timeout_seconds: number;
}
```

**Behavior:**
- Fetches available sources from `GET /api/research/sources` on mount (via React Query, cached)
- Depth preset buttons: selecting one auto-checks sources at that tier and below
- Source checkboxes: user can override individually after selecting a preset
- Chevron button to collapse/expand
- State lifted to page component, passed down as props

**New API route:** `apps/web/src/app/api/research/sources/route.ts` — proxies to `GET /research/sources`

### 3.3 Results Panel

**Component:** `apps/web/src/components/research/research-results-panel.tsx`

**Data source:** React Query fetching recent research artifacts from the artifacts API, filtered to research-related types (`company_profile`, `person_profile`, market analysis). Refetches when `research_complete` SSE event arrives or when `useResearchPoller` detects completion.

**Sub-components:**

**`ResearchResultCard`** — renders a completed research result:

```typescript
interface ResearchResultCardProps {
  subjectName: string;
  subjectType: "company" | "person" | "market";
  blocks: Array<{ block_id: string; heading: string; body: string; order: number }>;
  summary: string | null;
  taskId: string;
  artifactId: string | null;
  onWatch?: () => void;   // injects "Watch {name}" into chat
  onDeeper?: () => void;  // injects "Deep dive on {name}" into chat
}
```

- Renders blocks as collapsible sections (heading + markdown body)
- Type badge: colored pill showing "Company" / "Person" / "Market"
- Action buttons in card footer: "Watch" and "Go deeper"
- Card actions work by calling a callback that sets a pending message in the chat panel

**`ResearchProgressCard`** — renders an in-flight research task:

```typescript
interface ResearchProgressCardProps {
  subjectName: string;
  taskId: string;
  sourcesCompleted: string[];
  sourcesTotal: string[];
}
```

- Same card frame as `ResearchResultCard` but with a `SourceProgressBar` inside
- Auto-transitions to `ResearchResultCard` when polling detects completion (React Query refetch)

### 3.4 Chat Panel

**Component:** Inline within the page, not a separate component file — just `useStreamChat` + `ChatMessageList` + `ChatInputBar` composed together.

- `useStreamChat` with `pageContext: JSON.stringify({ scope: "research", depth, enabled_sources: enabledSources })`
- When a card action fires (Watch / Go deeper), the callback calls `send()` with the appropriate message
- Suggested prompts in empty state: "Research a company", "Analyze a market", "Look up a person"
- Independent from the CopilotPanel Zustand store — the side panel copilot and the research page chat are separate conversations

### 3.5 SSE Research Complete Handling

When the chat stream yields a `research_complete` event (new event type):
- The page invalidates the React Query cache key for research artifacts
- Any `ResearchProgressCard` matching the completed `task_id` refetches and transitions to a result card

---

## 4. Supporting Changes

### 4.1 Onboarding Landing Page

**File:** `apps/web/src/app/(main)/onboarding/page.tsx` (modify existing)

- Add a prominent card at the top: "Let us learn about you" with description text and "Start interview" button → navigates to `/onboarding/interview`
- Add a divider: "Or set up manually"
- Existing 3-step wizard cards remain below
- Step 2 label: "Competitors" → "Companies to Watch"

### 4.2 Notification Polling

**File:** `apps/web/src/components/notifications-bell.tsx` (modify existing)

- Add `setInterval` (30 seconds) to re-fetch notification count
- When count increases, badge updates immediately
- Clean up interval on unmount

### 4.3 New API Route Proxies

All backend calls go through Next.js API routes (existing pattern):

- `GET /api/research/sources` → `GET /research/sources`
- `GET /api/research/[taskId]` → `GET /research/{taskId}`
- `POST /api/onboarding/interview/complete` → `POST /onboarding/interview/complete`

These follow the exact pattern of the existing `apps/web/src/app/api/chat/route.ts` proxy.

---

## File Structure Summary

### New files

```
apps/web/src/
├── hooks/
│   ├── use-stream-chat.ts              # Shared streaming chat hook
│   └── use-research-poller.ts          # Research task polling hook
├── components/
│   ├── chat/
│   │   ├── chat-message-list.tsx       # Reusable message list
│   │   └── chat-input-bar.tsx          # Reusable chat input
│   └── research/
│       ├── source-progress-bar.tsx     # Source completion progress
│       ├── research-controls.tsx       # Depth + source toggles
│       ├── research-results-panel.tsx  # Results card list
│       ├── research-result-card.tsx    # Single result card
│       └── research-progress-card.tsx  # In-flight research card
├── app/
│   ├── (main)/
│   │   ├── onboarding/
│   │   │   └── interview/
│   │   │       └── page.tsx            # Interview page
│   │   └── research/
│   │       └── page.tsx                # Research page
│   └── api/
│       ├── research/
│       │   ├── sources/
│       │   │   └── route.ts            # Proxy GET /research/sources
│       │   └── [taskId]/
│       │       └── route.ts            # Proxy GET /research/{taskId}
│       └── onboarding/
│           └── interview/
│               └── complete/
│                   └── route.ts        # Proxy POST /onboarding/interview/complete
```

### Modified files

```
apps/web/src/
├── components/
│   ├── copilot/
│   │   └── copilot-panel.tsx           # Refactor to use shared primitives
│   └── notifications-bell.tsx          # Add polling interval
└── app/
    └── (main)/
        └── onboarding/
            └── page.tsx                # Add dual-path entry
```

---

## Dependency Graph

```
Layer 1: Shared Primitives (no dependencies)
  ├── useStreamChat hook
  ├── useResearchPoller hook
  ├── ChatMessageList component
  └── ChatInputBar component

Layer 2: Research Components (depends on Layer 1)
  ├── SourceProgressBar
  ├── ResearchControls
  ├── ResearchResultCard
  └── ResearchProgressCard

Layer 3: CopilotPanel Refactor (depends on Layer 1)
  └── Refactor to compose from shared primitives

Layer 4: Pages (depends on Layers 1-2)
  ├── Onboarding interview page
  └── /research page

Layer 5: Supporting Changes (independent)
  ├── Onboarding landing page update
  ├── Notification polling
  └── API route proxies
```

---

## Out of Scope

- Mobile-responsive onboarding interview (desktop-first for now)
- Research history search/filtering on the `/research` page
- Sharing research results between users
- Copilot panel showing research_complete events (only the `/research` page handles them)
- Backend changes for interview mode on copilot stream and `POST /onboarding/interview/complete` endpoint — covered in Phase 2 backend spec but not yet implemented. The frontend plan will include stub API routes that return mock responses until the backend is ready. The interview page can be built and tested against stubs.
