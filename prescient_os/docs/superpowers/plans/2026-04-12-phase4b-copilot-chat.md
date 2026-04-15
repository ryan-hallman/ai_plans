# Phase 4b: Copilot Chat & SSE Streaming

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the copilot chat experience — SSE streaming from the planner loop through FastAPI → Next.js BFF → browser, with inline citation rendering, tool call visibility, multi-turn conversation, and a provenance panel.

**Architecture:** Add a streaming `send_stream()` method to the Anthropic adapter. Wrap the planner loop in an async generator that yields typed events (tool_start, tool_end, delta, citation, done). FastAPI SSE endpoint streams these events. Next.js API route proxies the stream. React chat UI consumes events and renders incrementally.

**Tech Stack:** Anthropic SDK streaming (`client.messages.stream()`), FastAPI `StreamingResponse`, Next.js Route Handlers, React state + `EventSource`/`fetch` streaming, Tailwind/shadcn.

**Spec:** `docs/superpowers/specs/2026-04-12-phase4-demo-design.md` — Section 3

**Depends on:** Phase 4a (seed data must exist for meaningful copilot answers)

---

## File Structure

### New Files (API)

```
apps/api/src/prescient/intelligence/
├── application/
│   └── planner_stream.py           # Async generator wrapper around planner
└── api/
    └── copilot_stream.py           # POST /intelligence/copilot/stream SSE endpoint
```

### New Files (Frontend)

```
apps/web/src/
├── app/
│   ├── api/chat/
│   │   └── route.ts                # BFF: proxy SSE from FastAPI to browser
│   └── (main)/chat/
│       └── page.tsx                # Chat UI page
├── components/
│   ├── chat-message.tsx            # Single message (user or assistant)
│   ├── chat-input.tsx              # Input bar with send button
│   ├── citation-pill.tsx           # Inline [[n]] citation pill
│   ├── tool-call-chips.tsx         # Tool call status chips
│   └── source-panel.tsx            # Right panel: source viewer on citation click
└── lib/
    └── chat-stream.ts              # SSE stream consumer utility
```

### Modified Files

```
apps/api/src/prescient/intelligence/
├── infrastructure/
│   └── anthropic_adapter.py        # Add send_stream() method
├── domain/
│   └── ports.py                    # Add streaming protocol method
└── main.py                         # Register copilot_stream router
```

---

## Task 1: Streaming Anthropic Adapter

**Files:**
- Modify: `apps/api/src/prescient/intelligence/infrastructure/anthropic_adapter.py`
- Modify: `apps/api/src/prescient/intelligence/domain/ports.py`

- [ ] **Step 1: Add send_stream() to LLMProviderPort**

Add an async generator method to the port protocol:

```python
# In ports.py, add to LLMProviderPort:
async def send_stream(
    self,
    system: str,
    messages: list[dict[str, Any]],
    tools: list[dict[str, Any]],
    model: str,
) -> AsyncIterator[dict[str, Any]]:
    """Stream LLM response. Yields dicts with type: 'text_delta' | 'tool_use' | 'message_stop'."""
    ...
```

- [ ] **Step 2: Implement send_stream() in AnthropicAdapter**

Use the Anthropic SDK's streaming API:

```python
async def send_stream(
    self,
    system: str,
    messages: list[dict[str, Any]],
    tools: list[dict[str, Any]],
    model: str,
) -> AsyncIterator[dict[str, Any]]:
    """Stream response from Anthropic. Yields event dicts."""
    kwargs: dict[str, Any] = {
        "model": model,
        "max_tokens": 4096,
        "system": [{"type": "text", "text": system, "cache_control": {"type": "ephemeral"}}],
        "messages": messages,
    }
    if tools:
        kwargs["tools"] = tools

    async with self._client.messages.stream(**kwargs) as stream:
        async for event in stream:
            # Anthropic SDK yields typed events. Convert to simple dicts.
            if event.type == "content_block_start":
                block = event.content_block
                if block.type == "tool_use":
                    yield {"type": "tool_use_start", "id": block.id, "name": block.name}
                elif block.type == "text":
                    yield {"type": "text_start"}
            elif event.type == "content_block_delta":
                delta = event.delta
                if delta.type == "text_delta":
                    yield {"type": "text_delta", "text": delta.text}
                elif delta.type == "input_json_delta":
                    yield {"type": "tool_input_delta", "partial_json": delta.partial_json}
            elif event.type == "content_block_stop":
                yield {"type": "content_block_stop"}
            elif event.type == "message_stop":
                # Get the final message for tool_use blocks
                final = stream.get_final_message()
                yield {
                    "type": "message_complete",
                    "content": [_block_to_dict(b) for b in final.content],
                    "stop_reason": final.stop_reason,
                    "usage": {"input_tokens": final.usage.input_tokens, "output_tokens": final.usage.output_tokens},
                }
```

The existing `send()` method stays unchanged — the non-streaming planner still uses it.

Read the actual Anthropic SDK source or docs to verify the streaming event types. The SDK may use `anthropic.AsyncStream` or `client.messages.stream()` context manager. Check the installed SDK version.

- [ ] **Step 3: Test streaming works**

Write a quick manual test or unit test that calls `send_stream()` with a simple prompt and verifies events are yielded.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/intelligence/infrastructure/anthropic_adapter.py
git add apps/api/src/prescient/intelligence/domain/ports.py
git commit -m "feat: add streaming support to Anthropic adapter"
```

---

## Task 2: Streaming Planner Loop

**Files:**
- Create: `apps/api/src/prescient/intelligence/application/planner_stream.py`

- [ ] **Step 1: Create the streaming planner**

An async generator that wraps the planner loop logic. Instead of accumulating and returning an Answer, it yields events as they happen.

```python
"""Streaming planner loop — yields SSE events as the planner executes.

This is the streaming equivalent of run_planner(). It follows the same
logic: call LLM → if tool_use, execute tools and loop → if text, run
grounding validator → yield final result. The difference is that text
deltas and tool call events are yielded incrementally.
"""

from __future__ import annotations

import json
from collections.abc import AsyncIterator
from dataclasses import dataclass
from typing import Any
from uuid import uuid4

from prescient.intelligence.application.planner import (
    _build_citation_map,
    _extract_citations,
    _format_tool_result,
)
from prescient.intelligence.domain.entities.answer import Answer
from prescient.intelligence.domain.grounding import validate_grounding
from prescient.intelligence.domain.ports import LLMProviderPort, ToolRegistryPort
from prescient.intelligence.prompts import load_system_prompt


@dataclass(frozen=True)
class StreamEvent:
    """A single event in the copilot stream."""
    event: str    # tool_start, tool_end, delta, citation, done, error
    data: dict[str, Any]


async def run_planner_stream(
    *,
    llm: LLMProviderPort,
    tools: ToolRegistryPort,
    question: str,
    conversation_messages: list[dict[str, Any]],
    primary_company_name: str,
    primary_company_slug: str,
    model: str,
    max_iterations: int = 15,
) -> AsyncIterator[StreamEvent]:
    """Streaming planner loop. Yields StreamEvent objects."""

    system_prompt = load_system_prompt(
        company_name=primary_company_name,
        company_slug=primary_company_slug,
        tool_descriptions=tools.describe_tools(),
    )

    messages = list(conversation_messages)
    tool_calls_log: list[dict] = []
    tool_results_log: list[dict] = []
    iteration = 0

    while iteration < max_iterations:
        iteration += 1

        # Accumulate the full response via streaming
        text_chunks: list[str] = []
        tool_use_blocks: list[dict] = []
        current_tool: dict | None = None
        tool_input_json = ""

        async for event in llm.send_stream(
            system=system_prompt,
            messages=messages,
            tools=tools.tool_definitions(),
            model=model,
        ):
            if event["type"] == "tool_use_start":
                current_tool = {"id": event["id"], "name": event["name"]}
                tool_input_json = ""
                yield StreamEvent("tool_start", {"tool": event["name"]})

            elif event["type"] == "tool_input_delta":
                tool_input_json += event["partial_json"]

            elif event["type"] == "content_block_stop" and current_tool is not None:
                # Tool use block complete — parse arguments
                try:
                    args = json.loads(tool_input_json) if tool_input_json else {}
                except json.JSONDecodeError:
                    args = {}
                tool_use_blocks.append({
                    "type": "tool_use",
                    "id": current_tool["id"],
                    "name": current_tool["name"],
                    "input": args,
                })
                current_tool = None

            elif event["type"] == "text_delta":
                text_chunks.append(event["text"])
                yield StreamEvent("delta", {"text": event["text"]})

            elif event["type"] == "message_complete":
                # Use the final message to get accurate tool_use blocks
                # if we didn't capture them via streaming
                final_content = event.get("content", [])
                stop_reason = event.get("stop_reason", "end_turn")
                if not tool_use_blocks:
                    tool_use_blocks = [b for b in final_content if b.get("type") == "tool_use"]

        full_text = "".join(text_chunks)

        # If LLM wants to call tools, execute them and loop
        if tool_use_blocks:
            # Append assistant message with tool_use blocks
            assistant_content = []
            if full_text:
                assistant_content.append({"type": "text", "text": full_text})
            assistant_content.extend(tool_use_blocks)
            messages.append({"role": "assistant", "content": assistant_content})

            # Execute each tool
            tool_result_blocks = []
            for tool_block in tool_use_blocks:
                tool_name = tool_block["name"]
                tool_args = tool_block.get("input", {})
                tool_id = tool_block["id"]

                execution = await tools.execute(tool_name, tool_args)
                tool_calls_log.append({"tool": tool_name, "args": tool_args})
                tool_results_log.append({
                    "tool_use_id": tool_id,
                    "tool": tool_name,
                    "summary": execution.summary,
                    "record_count": len(execution.records),
                })

                yield StreamEvent("tool_end", {
                    "tool": tool_name,
                    "summary": execution.summary,
                    "record_count": len(execution.records),
                })

                result_content = _format_tool_result(execution)
                tool_result_blocks.append({
                    "type": "tool_result",
                    "tool_use_id": tool_id,
                    "content": result_content,
                })

            messages.append({"role": "user", "content": tool_result_blocks})
            continue

        # Text response — run grounding validation
        citation_map = _build_citation_map(tool_results_log)
        citations = _extract_citations(full_text, citation_map)
        grounding = validate_grounding(full_text, set(citation_map.keys()))

        # Yield citations
        for c in citations:
            yield StreamEvent("citation", {
                "marker": c.marker,
                "tool_name": c.tool_name,
                "source_type": c.source_type,
                "excerpt": c.excerpt,
            })

        # Yield done event
        yield StreamEvent("done", {
            "grounding_status": grounding.value,
            "citations": [
                {"marker": c.marker, "tool_name": c.tool_name, "source_type": c.source_type, "excerpt": c.excerpt}
                for c in citations
            ],
            "tool_calls": tool_calls_log,
            "iterations": iteration,
        })
        return

    # Loop exceeded
    yield StreamEvent("error", {"message": "Planner loop exceeded maximum iterations"})
```

This reuses the existing planner's helper functions (`_build_citation_map`, `_extract_citations`, `_format_tool_result`) and grounding validator. Read `planner.py` to verify these function signatures and import them correctly.

**Important:** The existing planner functions may be private (prefixed with `_`). If they can't be imported, extract them to a shared module or duplicate the logic. Check the actual code.

- [ ] **Step 2: Verify imports work**

```python
python -c "from prescient.intelligence.application.planner_stream import run_planner_stream; print('OK')"
```

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/prescient/intelligence/application/planner_stream.py
git commit -m "feat: add streaming planner loop with SSE event generation"
```

---

## Task 3: FastAPI SSE Endpoint

**Files:**
- Create: `apps/api/src/prescient/intelligence/api/copilot_stream.py`
- Modify: `apps/api/src/prescient/main.py`

- [ ] **Step 1: Create the SSE endpoint**

```python
"""SSE streaming endpoint for copilot chat.

POST /intelligence/copilot/stream — accepts a question, streams
planner events as Server-Sent Events.
"""

from __future__ import annotations

from typing import Annotated, Any
from uuid import UUID, uuid4

from fastapi import APIRouter, Depends
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field

from prescient.intelligence.api.copilot import (
    _build_scope,
    _get_opensearch,
    _get_tool_registry,
    _resolve_company,
    get_llm_provider,
    CtxDep,
    OpenSearchDep,
    ToolRegistryDep,
    LLMDep,
)
from prescient.companies.api.dependencies import CompanyRepoDep, RelationshipRepoDep
from prescient.config import get_settings
from prescient.intelligence.application.planner_stream import run_planner_stream
from prescient.intelligence.infrastructure.tools import ToolContext
from prescient.shared.errors import NotFoundError, ValidationError
from prescient.shared.types import CompanyId
from prescient.tenants.api.dependencies import EngagementRepoDep

router = APIRouter(prefix="/intelligence/copilot", tags=["copilot-stream"])


class StreamRequest(BaseModel):
    company_slug: str = Field(min_length=1, max_length=128)
    question: str = Field(min_length=1, max_length=4096)
    conversation_id: UUID | None = None


async def _event_generator(request_data, ctx, llm, opensearch, registry,
                           company_repo, relationship_repo, engagement_repo):
    """Generate SSE events from the streaming planner."""
    try:
        company = await _resolve_company(
            company_repo, engagement_repo, request_data.company_slug, ctx.user.fund_id
        )
        scope = await _build_scope(relationship_repo, CompanyId(company.id))
        settings = get_settings()

        tool_context = ToolContext(
            tenant_id=ctx.user.fund_id,
            user_id=ctx.user.user_id,
            scope=scope,
            primary_company_slug=company.slug,
            session=ctx.session,
            opensearch=opensearch,
        )
        bound_tools = registry.bind(tool_context)
        conversation_id = request_data.conversation_id or uuid4()

        messages: list[dict[str, Any]] = [
            {"role": "user", "content": request_data.question},
        ]

        # Yield conversation_id first
        yield f"event: init\ndata: {json.dumps({'conversation_id': str(conversation_id)})}\n\n"

        async for event in run_planner_stream(
            llm=llm,
            tools=bound_tools,
            question=request_data.question,
            conversation_messages=messages,
            primary_company_name=company.name,
            primary_company_slug=company.slug,
            model=settings.anthropic_model,
        ):
            yield f"event: {event.event}\ndata: {json.dumps(event.data)}\n\n"

    except Exception as e:
        yield f"event: error\ndata: {json.dumps({'message': str(e)})}\n\n"


@router.post("/stream")
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
    return StreamingResponse(
        _event_generator(body, ctx, llm, opensearch, registry,
                        company_repo, relationship_repo, engagement_repo),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )
```

Add `import json` at the top. Adjust imports based on what's actually importable from `copilot.py` — some functions/deps may be private. If so, duplicate the minimal dependency resolution logic.

- [ ] **Step 2: Register the router in main.py**

```python
from prescient.intelligence.api.copilot_stream import router as copilot_stream_router
# ...
app.include_router(copilot_stream_router)
```

- [ ] **Step 3: Verify endpoint responds**

```bash
curl -X POST http://localhost:8000/intelligence/copilot/stream \
  -H "Content-Type: application/json" \
  -d '{"company_slug": "peloton", "question": "What is Peloton?"}' \
  --no-buffer
```

Should see SSE events streamed (or an error event if no API key / no seed data).

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/intelligence/api/copilot_stream.py apps/api/src/prescient/main.py
git commit -m "feat: add POST /intelligence/copilot/stream SSE endpoint"
```

---

## Task 4: Next.js BFF Streaming Route

**Files:**
- Create: `apps/web/src/app/api/chat/route.ts`

- [ ] **Step 1: Create the BFF proxy**

```typescript
// POST /api/chat — proxy SSE from FastAPI to the browser.
// Adds auth headers from the client request.

import { NextRequest } from "next/server";

const API_BASE = process.env.PRESCIENT_API_URL ?? "http://api:8000";

export async function POST(req: NextRequest) {
  const body = await req.json();

  const upstream = await fetch(`${API_BASE}/intelligence/copilot/stream`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  if (!upstream.ok || !upstream.body) {
    return new Response(
      JSON.stringify({ error: "Upstream API error" }),
      { status: upstream.status, headers: { "Content-Type": "application/json" } }
    );
  }

  // Pipe the SSE stream through to the browser
  return new Response(upstream.body, {
    status: 200,
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive",
    },
  });
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/app/api/chat/route.ts
git commit -m "feat: add Next.js BFF route for chat SSE proxy"
```

---

## Task 5: Chat Stream Consumer Utility

**Files:**
- Create: `apps/web/src/lib/chat-stream.ts`

- [ ] **Step 1: Create the stream consumer**

A utility that connects to the SSE endpoint and yields typed events to React:

```typescript
export type ChatEvent =
  | { type: "init"; conversation_id: string }
  | { type: "tool_start"; tool: string }
  | { type: "tool_end"; tool: string; summary: string; record_count: number }
  | { type: "delta"; text: string }
  | { type: "citation"; marker: number; tool_name: string; source_type: string; excerpt: string }
  | { type: "done"; grounding_status: string; citations: Citation[]; tool_calls: ToolCallLog[]; iterations: number }
  | { type: "error"; message: string };

export type Citation = {
  marker: number;
  tool_name: string;
  source_type: string;
  excerpt: string;
};

export type ToolCallLog = {
  tool: string;
  args: Record<string, unknown>;
};

export async function* streamChat(
  companySlug: string,
  question: string,
  conversationId?: string,
): AsyncGenerator<ChatEvent> {
  const res = await fetch("/api/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      company_slug: companySlug,
      question,
      conversation_id: conversationId,
    }),
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

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/lib/chat-stream.ts
git commit -m "feat: add chat SSE stream consumer utility"
```

---

## Task 6: Chat UI Components

**Files:**
- Create: `apps/web/src/components/chat-message.tsx`
- Create: `apps/web/src/components/chat-input.tsx`
- Create: `apps/web/src/components/citation-pill.tsx`
- Create: `apps/web/src/components/tool-call-chips.tsx`
- Create: `apps/web/src/components/source-panel.tsx`

- [ ] **Step 1: Create citation-pill.tsx**

A small inline pill that renders `[[n]]` citations. Shows source type icon and marker number. Hover shows excerpt tooltip. Click triggers source panel open.

- [ ] **Step 2: Create tool-call-chips.tsx**

Compact horizontal chips showing tool call status: "Searching artifacts..." → "3 found". Animates during streaming, collapses when answer starts.

- [ ] **Step 3: Create chat-message.tsx**

Renders a single message (user or assistant). For assistant messages:
- Parses `[[n]]` markers in text and replaces with CitationPill components
- Shows ToolCallChips above the message if tool calls were made
- Has a "Show reasoning" toggle for the provenance panel

- [ ] **Step 4: Create chat-input.tsx**

Text input with send button. Supports Cmd+Enter to submit. Disabled while streaming.

- [ ] **Step 5: Create source-panel.tsx**

Right-side sliding panel that shows source content when a citation is clicked. Displays:
- Document/artifact title and metadata
- The cited excerpt highlighted
- Source type badge
- Close button

For the demo, this shows the citation's excerpt and metadata. Deep artifact linking (click-through to full artifact view) can be added in Phase 4c.

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/components/chat-message.tsx apps/web/src/components/chat-input.tsx
git add apps/web/src/components/citation-pill.tsx apps/web/src/components/tool-call-chips.tsx
git add apps/web/src/components/source-panel.tsx
git commit -m "feat: add chat UI components — message, input, citations, tools, source panel"
```

---

## Task 7: Chat Page

**Files:**
- Create: `apps/web/src/app/(main)/chat/page.tsx`
- Modify: `apps/web/src/components/nav.tsx` — add Chat link

- [ ] **Step 1: Create the chat page**

Full-width page with:
- Company context selector at top (defaults to Peloton for demo)
- Message thread (scrollable, auto-scrolls on new content)
- Chat input at bottom
- Source panel slides in from right on citation click

State management:
- `messages[]` — array of user and assistant messages
- `isStreaming` — whether a response is currently streaming
- `activeSource` — which citation's source to show in the panel
- `conversationId` — maintained across turns for multi-turn chat

The page uses the `streamChat()` utility to connect to the SSE endpoint. As events arrive:
- `tool_start` / `tool_end` → update tool call chips on the current message
- `delta` → append text to the current assistant message
- `citation` → store citation data for pill rendering
- `done` → finalize the message, store grounding status
- `error` → show error state with retry button

- [ ] **Step 2: Add Chat link to nav**

Add a "Chat" link in the nav component between "Intelligence" and "Actions".

- [ ] **Step 3: Test the full flow**

Navigate to /chat, type a question, verify streaming works end-to-end. If no API key or seed data, verify error handling works gracefully.

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/app/\(main\)/chat/page.tsx apps/web/src/components/nav.tsx
git commit -m "feat: add copilot chat page with streaming, citations, and source panel"
```

---

## Summary

| Task | What it does | Key files |
|------|-------------|-----------|
| 1 | Streaming Anthropic adapter | `anthropic_adapter.py`, `ports.py` |
| 2 | Streaming planner loop | `planner_stream.py` |
| 3 | FastAPI SSE endpoint | `copilot_stream.py`, `main.py` |
| 4 | Next.js BFF proxy | `app/api/chat/route.ts` |
| 5 | Stream consumer utility | `lib/chat-stream.ts` |
| 6 | Chat UI components | 5 component files |
| 7 | Chat page + nav | `chat/page.tsx`, `nav.tsx` |
