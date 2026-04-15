# Persistent Copilot Panel Design

## Goal

Elevate the copilot from a buried company-specific chat page into a persistent, context-aware panel available on every page of the app. The copilot always grounds on the user's home organization and layers in page-specific context as the user navigates.

## Core Concepts

### Persistent Slide-Over Panel

A 440px right-side panel embedded in the main app layout (`(main)/layout.tsx`). It survives route changes, maintains a single continuous conversation, and shrinks the main content area when open (not overlay).

- **Toggle:** Floating button in bottom-right corner + `Cmd/Ctrl+J` keyboard shortcut.
- **Open:** Panel slides in, content area shrinks. Stays open across navigation.
- **Close:** Panel collapses, content area expands. Conversation preserved — reopening picks up where you left off.

### Dual-Scope Grounding

Every user has a **home organization** (`primary_org_id` on the user record). For PE operators, this is their fund.

1. **Home org (always active):** The copilot always retrieves from the home org's knowledge base — strategy docs, IC memos, focus priorities, meeting notes. This is the copilot's primary identity: "You are assisting a member of {home_org_name}."
2. **Focus company (layered in):** When the user is viewing a portfolio company page, that company's data (KPIs, filings, transcripts) is layered in alongside the home org's data. Both knowledge stores are active simultaneously.

This means a PE operator looking at Peloton's KPIs can ask "how does this relate to our fund's Q2 thesis?" — because the copilot has both contexts.

### Page Context System

Pages register their context with the copilot via a lightweight hook. The copilot uses this to adjust system prompts and suggested questions.

```typescript
interface PageContext {
  pageType: "brief" | "triage" | "actions" | "board-prep" | "knowledge" | "strategy" | "company-detail" | "company-kpis";
  companySlug?: string;
  selectedItemId?: string;
  selectedItemSummary?: string;
}
```

Integration is one line per page:

```typescript
useCopilotContext({ pageType: "actions" });
useCopilotContext({ pageType: "company-kpis", companySlug: "peloton" });
```

**Context levels:**

- **Route-level (all pages):** The copilot knows which page you're on and adjusts suggested prompts and system instructions. "I see you're on the Actions inbox."
- **Selection-level (opt-in):** Pages that offer "Ask about this" on specific items set `selectedItemId` and `selectedItemSummary`. The panel shows a context chip above the input, and the summary is included in the system prompt.

### Single Continuous Conversation

The conversation persists as the user navigates between pages. Asking about Peloton's KPIs, then navigating to Actions, you can say "create an action item for that Peloton issue we just discussed."

- Conversation starts on first message. `conversationId` comes from the backend `init` SSE event.
- Navigation updates `pageContext` but does not reset the conversation.
- "New conversation" button clears messages and starts fresh.
- The existing `/companies/[slug]/chat` page is removed — the panel replaces it entirely.

## Panel Layout

Top to bottom:

1. **Header bar** — "Copilot" title, page context indicator (e.g., "Viewing Peloton KPIs"), "New conversation" button, close button.
2. **Message thread** — Reuses existing `ChatMessage` component. Scrollable, most recent at bottom.
3. **Suggested prompts** — 2-3 contextual suggestions based on current `pageType`. Shown when conversation is empty or after a natural pause. Static mapping per page type, no AI generation.
4. **Input area** — Reuses existing `ChatInput` component, pinned to bottom.
5. **Citations** — When a user clicks a citation marker, source detail expands inline (accordion-style) within the panel. No second side panel.

## Backend Changes

### Stream Endpoint

The stream request adds optional fields:

```python
class StreamRequest(BaseModel):
    question: str
    conversation_id: str | None = None
    company_slug: str | None = None          # becomes optional (was required)
    focus_company_slug: str | None = None     # portfolio company being viewed
    page_context: str | None = None           # e.g., "actions", "board-prep"
    selected_item_summary: str | None = None  # optional item-level context
```

### Scope Resolution

1. **Home org (always):** Resolve user's `primary_org_id`. The planner's tool calls always include the home org's knowledge store.
2. **Focus company (when present):** If `focus_company_slug` is provided, layer in that company's data. `ConversationScope.visible_company_ids` is populated with both home org's company ID and focus company's ID.

### System Prompt Augmentation

The planner's system prompt receives a context block:

- "You are assisting a member of {home_org_name}." (always)
- "They are currently viewing {page_context_description}." (when provided)
- "They are focused on: {selected_item_summary}." (when provided)

### Relationship to Existing Strategy Context

The copilot already injects home-org strategy awareness into the system prompt via `_build_strategy_context()` in `copilot.py`. This function queries the user's fund for active/at-risk initiatives and injects a strategy summary block. This is complementary to — not replaced by — the dual-scope grounding change. The strategy context continues to work as-is; the new `page_context` and `focus_company_slug` fields are additive.

### Backward Compatibility

The endpoint with only `question` and `conversation_id` (no new fields) defaults to home org scope. Existing integrations continue to work.

### Note on `company_slug` → UUID Migration

The `focus_company_slug` field uses string slugs to match the current codebase convention. A slug-to-UUID migration is planned as a follow-up. When that happens, `focus_company_slug` becomes `focus_company_id: UUID`. The panel and store will need a minor refactor at that point.

## File Structure

### New Files

| File | Responsibility |
|------|---------------|
| `stores/copilot-store.ts` | Zustand store: panel state, conversation, page context |
| `components/copilot/copilot-panel.tsx` | Slide-over shell: header, message list, input, open/close |
| `components/copilot/copilot-toggle.tsx` | Floating bottom-right button + keyboard shortcut listener |
| `components/copilot/suggested-prompts.tsx` | Contextual prompt chips based on `pageType` |
| `components/copilot/context-chip.tsx` | Selected item indicator above input |
| `hooks/use-copilot-context.ts` | Hook for pages to register context with the store |

### Reused (Moved from Existing Chat Page)

| File | Change |
|------|--------|
| `components/chat-message.tsx` | Reused as-is in copilot panel |
| `components/chat-input.tsx` | Reused as-is in copilot panel |
| `components/source-panel.tsx` | Adapted to render inline/accordion within the panel |

### Modified

| File | Change |
|------|--------|
| `app/(main)/layout.tsx` | Add `<CopilotPanel />` and `<CopilotToggle />`. Restructure to flex layout so main content shrinks when panel is open. The current `max-w-6xl` constraint on `<main>` will need adjustment — when the panel is open, the content area should still be usable (panel sits outside the max-width container). |
| `app/api/chat/route.ts` | Pass through new fields to backend |
| Each page under `app/(main)/` | Add `useCopilotContext()` call (brief, triage, actions, board-prep, knowledge, strategy, company pages) |
| `intelligence/api/copilot_stream.py` | Update `StreamRequest`, resolve home org scope |
| `intelligence/api/copilot.py` | Same changes for non-streaming endpoint |
| `intelligence/application/planner.py` | Accept page context for system prompt augmentation |
| `intelligence/application/planner_stream.py` | Same |

### Removed

| File | Reason |
|------|--------|
| `app/(main)/companies/[slug]/chat/page.tsx` | Replaced by persistent panel |

## Testing

### Frontend Unit Tests

- **Copilot store:** Panel toggle, context updates, conversation reset, message append during streaming.
- **useCopilotContext hook:** Mounting sets context in store, unmounting clears it.
- **CopilotPanel:** Renders messages, sends via stream API, shows suggested prompts when empty, shows context chip when `selectedItemSummary` present.
- **Layout integration:** Panel shrinks main content when open, expands when closed.

### Backend Unit Tests

- **Scope resolution:** Planner receives home org knowledge base with no focus company; receives both when focus company present.
- **System prompt augmentation:** Page context and selected item summary appear in system prompt.
- **Backward compatibility:** Request with only `question` and `conversation_id` defaults to home org scope.

### Manual/Browser Validation

- Open panel, send message, navigate to different page — conversation persists.
- On company page, suggested prompts reference that company.
- Select item on Actions page — context chip appears, copilot references it.
- Citations expand inline within panel.
- `Cmd+K` opens/closes panel.
