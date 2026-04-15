# Brainstorming Mode for the Prescient Copilot

## Context

The Prescient copilot today operates in a single mode: retrieve-then-cite Q&A. Users ask a question, the planner gathers evidence via tools, and returns a grounded answer with `[[n]]` citations. This works well for data retrieval but poorly for exploratory thinking — when a user asks "how should we approach this company's pricing restructure?", dumping a wall of text with citations isn't helpful.

The brainstorming skill in Claude Code demonstrates a better pattern: structured ideation through one-question-at-a-time pacing, multiple-choice options, 2-3 approaches with trade-offs, and incremental validation. We want to bring this conversational pattern to platform users as a **brainstorming mode** — a separate planner with its own loop, system prompt, and grounding contract.

**What we're building:** A full brainstorming mode with explicit toggle, auto-detection, hybrid grounding, structured phase progression, and end-of-session artifact/summary generation. Scoped to single-company conversations.

---

## Step 1 — Domain Layer Additions

### 1a. New enums in `domain/enums.py`

Add `CopilotMode` and `BrainstormPhase`:

```python
class CopilotMode(StrEnum):
    STANDARD = "standard"
    BRAINSTORM = "brainstorm"

class BrainstormPhase(StrEnum):
    UNDERSTAND = "understand"    # Gathering context via tools
    CLARIFY = "clarify"          # Asking one question at a time
    EXPLORE = "explore"          # Presenting 2-3 approaches with trade-offs
    RECOMMEND = "recommend"      # Final recommendation
    OUTPUT = "output"            # Producing artifact or summary
```

Add `PARTIAL` to `GroundingStatus` for hybrid grounding:

```python
class GroundingStatus(StrEnum):
    PENDING = "pending"
    GROUNDED = "grounded"
    PARTIAL = "partial"       # Factual claims grounded, analytical content ungrounded (brainstorm-acceptable)
    UNGROUNDED = "ungrounded"
    FAILED = "failed"
```

### 1b. New entity: `domain/entities/brainstorm_turn.py`

The return type for one brainstorm planner invocation. Deliberately separate from `Answer` — different contracts.

```python
class BrainstormOption:
    label: str          # "A", "B", "C"
    title: str          # Short option title
    description: str    # Rationale / trade-off description

class BrainstormTurn:
    text: str
    phase: BrainstormPhase
    options: tuple[BrainstormOption, ...]  # Empty if no options this turn
    citations: tuple[Citation, ...]
    tool_calls: tuple[ToolCall, ...]
    tool_results: tuple[ToolResult, ...]
    grounding_status: GroundingStatus
    iterations: int
    suggests_artifact: bool  # True when LLM signals OUTPUT phase is ready
    created_at: datetime
```

### 1c. Hybrid grounding in `domain/grounding.py`

New function `validate_brainstorm_grounding()`:
- Reuses `_is_factual()` to detect data claims (dollar amounts, percentages, trend words)
- Adds `_is_analytical_framing()` to detect brainstorm-safe patterns: "One approach...", "You could...", "Consider...", "The trade-off...", "Option A/B/C...", questions, conditionals
- A sentence needs citation only if `_is_factual(s) and not _is_analytical_framing(s)`
- Returns `GROUNDED` when all data claims cited, `PARTIAL` when analytical content is uncited (acceptable), `UNGROUNDED` when hard data claims lack citations
- **No regrounding retry loop** — response is returned as-is with its status

### 1d. Auto-detection classifier: `domain/classification.py`

New file. Pure domain logic, no infrastructure deps.

`classify_question_intent(question: str) -> tuple[CopilotMode, float]`

Heuristic pattern matching:
- **Brainstorm signals** (high confidence): "how should we", "what options", "help me think through", "brainstorm", "explore", "what would you recommend", "trade-offs", "strategy for", "approaches to"
- **Standard signals** (high confidence): "what was", "show me", "how much", "when did", "list the", "compare the X and Y numbers"
- Returns mode + confidence score. If confidence < threshold, the API includes a `suggested_mode` hint in the response for the frontend to prompt the user.

**Files:**
- `apps/api/src/prescient/intelligence/domain/enums.py`
- `apps/api/src/prescient/intelligence/domain/entities/brainstorm_turn.py`
- `apps/api/src/prescient/intelligence/domain/grounding.py`
- `apps/api/src/prescient/intelligence/domain/classification.py`

---

## Step 2 — Brainstorming System Prompt

### 2a. New prompt file: `prompts/brainstorm.md`

This is the core of brainstorming behavior. The prompt instructs the LLM to:

**Phase model:**
- Begin in `understand` phase: retrieve relevant company data via tools before engaging the user
- Move to `clarify`: ask one focused question at a time, prefer multiple-choice (A/B/C with brief rationale per option)
- Move to `explore`: present 2-3 approaches with explicit trade-offs, lead with recommendation
- Move to `recommend`: synthesize and present final recommendation
- Move to `output`: offer to produce a draft artifact or conversation summary

**Pacing rules:**
- One question per response — never batch multiple questions
- Multiple-choice options formatted as: `**A)** Title — rationale`
- Scale depth to complexity — simple topics get fewer questions

**Hybrid grounding contract:**
- Factual claims about portfolio data (financials, KPIs, filing details) MUST use `[[n]]` citations
- Frameworks, opinions, hypotheticals, options, and recommendations do NOT need citations
- When using data to support an option, cite the data but not the opinion built on it

**Structured output markers:**
- Emit `<!-- phase: clarify -->` at start of response (stripped before showing to user)
- Emit `<!-- artifact_ready -->` when ready to produce output
- Format options with `**A)** Title — description` pattern for parsing

**Tone:**
- Collaborative analyst, not lecturer
- "Let's think through this" not "Here are 10 things to consider"

### 2b. Update `prompts/__init__.py`

Add `load_brainstorm_prompt()` — same pattern as `load_system_prompt()` but loads `brainstorm.md`. Takes the same interpolation variables (company name, slug, tool descriptions).

**Files:**
- `apps/api/src/prescient/intelligence/prompts/brainstorm.md`
- `apps/api/src/prescient/intelligence/prompts/__init__.py`

---

## Step 3 — Brainstorm Planner

### 3a. New file: `application/brainstorm_planner.py`

`run_brainstorm_planner()` — the brainstorming equivalent of `run_planner()`.

**Key differences from `run_planner()`:**

1. **Single-turn oriented.** Runs one conversational turn per invocation (may do multiple tool calls to gather context, but produces one user-facing response). Multi-turn conversation happens at the API level via repeated calls with `conversation_id`.

2. **No grounding retry loop.** Runs hybrid grounding once. If factual claims lack citations, flags them but still returns the response. No regrounding re-prompt cycle.

3. **Phase extraction.** Parses `<!-- phase: X -->` from LLM response, strips it from output text. Fallback: infer phase from conversation length and content if tag is missing.

4. **Option parsing.** Parses `**A)** Title — description` patterns into `BrainstormOption` objects. If parsing fails, returns empty options list — the text still readable.

5. **Higher iteration limit** (15 vs 10) since brainstorming may need more tool calls for context gathering before responding.

**Signature:**
```python
async def run_brainstorm_planner(
    *,
    llm: LLMProviderPort,
    tools: ToolRegistryPort,
    question: str,
    conversation_messages: list[dict[str, Any]],
    primary_company_name: str,
    primary_company_slug: str,
    model: str,
    current_phase: BrainstormPhase | None = None,
) -> BrainstormTurn:
```

**Loop structure:**
```
while True:
    response = llm.send(system=brainstorm_prompt, messages, tools, model)
    
    if tool_use blocks -> execute tools, append results, continue
    
    # Text response = brainstorm turn output
    phase = _extract_phase(response_text)
    clean_text = _strip_metadata_tags(response_text)
    options = _extract_options(clean_text)
    grounding = validate_brainstorm_grounding(clean_text, tool_result_ids, citation_map)
    
    return BrainstormTurn(...)
```

**Reuse from existing planner:** Tool dispatch pattern (`_format_tool_result`), citation map building (`_build_citation_map`), citation extraction (`_extract_citations`). Consider extracting these into a shared `application/_tool_dispatch.py` module to avoid duplication.

### 3b. New file: `application/brainstorm_artifact.py`

`generate_brainstorm_artifact()` — called when the user wants to produce output from the session.

Makes one LLM call with a specialized prompt: "Based on this brainstorm conversation, produce a structured [type] with the following sections..." Returns an artifact draft value object that the API layer passes to the knowledge context's `create_artifact_draft` use case.

The LLM never publishes — it only creates drafts. Follows the existing pattern.

**Files:**
- `apps/api/src/prescient/intelligence/application/brainstorm_planner.py`
- `apps/api/src/prescient/intelligence/application/brainstorm_artifact.py`
- `apps/api/src/prescient/intelligence/application/_tool_dispatch.py` (extract shared helpers)

---

## Step 4 — API Layer

### 4a. Update `api/copilot.py`

**Request model changes:**
```python
class AskRequest(BaseModel):
    company_slug: str
    question: str
    conversation_id: UUID | None = None
    mode: CopilotMode | None = None         # Explicit mode selection
```

**New response model:**
```python
class BrainstormOptionResponse(BaseModel):
    label: str
    title: str
    description: str

class BrainstormResponse(BaseModel):
    conversation_id: UUID
    answer: str
    mode: str                                # "brainstorm"
    phase: str                               # BrainstormPhase value
    options: list[BrainstormOptionResponse]
    citations: list[CitationResponse]
    grounding_status: str
    tool_call_count: int
    iterations: int
    suggests_artifact: bool
```

Add `mode: str` field to existing `AskResponse` too (value: `"standard"`).

**Route logic update:**
1. Determine mode: explicit `mode` > auto-detect from question
2. If auto-detect confidence is low, set `suggested_mode` on the response
3. Dispatch to `run_planner()` or `run_brainstorm_planner()` accordingly
4. Return `AskResponse` or `BrainstormResponse` (both have `mode` field for frontend discrimination)

**New endpoint:**
```
POST /intelligence/copilot/brainstorm/artifact
```
Takes `conversation_id`, `output_type` ("summary" | "artifact"), optional `title`. Calls `generate_brainstorm_artifact()` then creates artifact draft via knowledge context.

**Files:**
- `apps/api/src/prescient/intelligence/api/copilot.py`

---

## Step 5 — Persistence

### 5a. Update conversation table

Add to `copilot_conversations`:
- `mode: VARCHAR(20) NOT NULL DEFAULT 'standard'` — queryable, indexable
- `brainstorm_metadata: JSONB NULL` — phase history, options count, etc.

### 5b. Update entity

Extend `CopilotConversation` to include `mode`.

### 5c. Alembic migration

Add migration for the new columns. Existing conversations backfill to `standard`.

---

## Step 6 — User Preference (Default Mode)

Optional user preference:
- `preferred_copilot_mode: standard | brainstorm | auto`
- Default: `auto`
- Stored with the user profile or onboarding context

Not required for the first slice if explicit toggle + auto-detect is enough.

---

## Verification

- Standard Q&A still works unchanged
- Brainstorm mode returns structured phases/options without grounding retry loops
- Factual claims in brainstorm mode still require citations
- Analytical / hypothetical language can remain uncited with `PARTIAL` grounding
- Brainstorm conversations persist with mode metadata
- Artifact generation from brainstorm sessions creates drafts only

---

## Key Design Decisions

- Separate planner, not prompt-only branching, to keep the interaction contract clear
- Hybrid grounding instead of fully grounded brainstorming, to preserve usefulness
- One-question-at-a-time pacing, following the proven Claude brainstorming pattern
- No auto-publishing — brainstorm output still becomes draft artifacts only
