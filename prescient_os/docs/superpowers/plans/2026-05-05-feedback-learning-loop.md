# Feedback Learning Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert wrong-answer feedback into structured, reviewable learning signals without automatically activating new retrieval behavior.

**Architecture:** Add a small knowledge-layer feedback triage module that classifies answer feedback and builds review-only proposals. The FastAPI route stays thin: persist the raw feedback, call the triage module, optionally persist a proposed terminology mapping, and return the triage/proposal payload. Proposed mappings remain `proposed` until a human or future review agent approves them.

**Tech Stack:** FastAPI, Pydantic, pytest, existing JSON-backed `KnowledgeStore`, existing terminology mapping/workbench models.

---

### Task 1: Feedback Triage Domain Service

**Files:**
- Create: `apps/api/src/prescient_benchmark/knowledge/feedback_learning.py`
- Test: `tests/unit/test_feedback_learning.py`

- [ ] **Step 1: Write failing tests**

```python
from prescient_benchmark.knowledge.feedback_learning import triage_answer_feedback
from prescient_benchmark.knowledge.models import AnswerFeedback


def test_wrong_terminology_feedback_proposes_review_only_mapping() -> None:
    result = triage_answer_feedback(
        AnswerFeedback(
            answer_id="answer-1",
            rating="wrong",
            notes="reason: terminology\nquestion: What are the LCA torque specs?",
            scope_id="scope-ferrari-360-modena",
            question="What are the LCA torque specs?",
            corrected_unit_ids=["unit-source-ferrari-360-wsm-p250"],
        ),
        active_layer_ids=["product:ferrari_360:v1", "domain:vehicle_repair:v1"],
        retrieval_profile_id="vehicle_repair_v1",
        source_ids=["source-ferrari-360-wsm"],
    )

    assert result.triage_reason == "terminology_gap"
    assert result.proposed_mapping is not None
    assert result.proposed_mapping.status == "proposed"
    assert result.proposed_mapping.scope_id == "scope-ferrari-360-modena"
    assert result.proposed_mapping.originating_answer_id == "answer-1"


def test_wrong_retrieval_feedback_proposes_eval_seed_not_mapping() -> None:
    result = triage_answer_feedback(
        AnswerFeedback(
            answer_id="answer-1",
            rating="wrong",
            notes="reason: retrieval_miss\nquestion: Where are the grounds?",
            scope_id="scope-ferrari-360-modena",
            question="Where are the grounds?",
            corrected_unit_ids=["unit-source-ferrari-360-wsm-p488"],
        ),
        active_layer_ids=["product:ferrari_360:v1"],
        retrieval_profile_id="vehicle_repair_v1",
        source_ids=["source-ferrari-360-wsm"],
    )

    assert result.triage_reason == "retrieval_miss"
    assert result.proposed_mapping is None
    assert result.eval_seed is not None
    assert result.eval_seed.expected_evidence_unit_ids == ["unit-source-ferrari-360-wsm-p488"]


def test_engineering_feedback_does_not_create_learning_proposal() -> None:
    result = triage_answer_feedback(
        AnswerFeedback(
            answer_id="answer-1",
            rating="wrong",
            notes="reason: claim_value\nThe page crashed with a runtime exception.",
        ),
        active_layer_ids=["product:ferrari_360:v1"],
        retrieval_profile_id="vehicle_repair_v1",
        source_ids=["source-ferrari-360-wsm"],
    )

    assert result.triage_reason == "engineering_ticket"
    assert result.proposed_mapping is None
    assert result.eval_seed is None
```

- [ ] **Step 2: Run unit tests to verify RED**

Run: `uv run python -m pytest tests/unit/test_feedback_learning.py -q`

Expected: fails because `prescient_benchmark.knowledge.feedback_learning` does not exist.

- [ ] **Step 3: Implement the minimal triage module**

Create Pydantic response/proposal models and a deterministic `triage_answer_feedback` function. Keep heuristics explicit and conservative: terminology feedback proposes a `TerminologyMapping` with `status=proposed`; retrieval/source/value misses produce a lightweight `WorkbenchEvalSeed`; engineering wording produces no learning proposal.

- [ ] **Step 4: Run unit tests to verify GREEN**

Run: `uv run python -m pytest tests/unit/test_feedback_learning.py -q`

Expected: all tests pass.

### Task 2: Feedback API Integration

**Files:**
- Modify: `apps/api/src/prescient_benchmark/knowledge/models.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing route tests**

Add tests showing `/knowledge/feedback` persists raw feedback, returns `triage_reason`, persists a proposed terminology mapping for terminology feedback, and does not persist a mapping for engineering-ticket feedback.

- [ ] **Step 2: Run route tests to verify RED**

Run: `uv run python -m pytest tests/integration/test_workshop_api.py -q -k "feedback_route"`

Expected: new assertions fail because the route only returns `{"recorded": True}` today.

- [ ] **Step 3: Wire route to triage service**

Extend `AnswerFeedback` with optional context fields (`scope_id`, `question`, `answer_text`, `provenance_unit_ids`, `applied_mappings`, `expected_correction`). In `/knowledge/feedback`, resolve scope when present, call `triage_answer_feedback`, persist `proposed_mapping` with `store.upsert_terminology_mapping`, and return the triage payload. Keep existing clients compatible by preserving `recorded: true`.

- [ ] **Step 4: Run route tests to verify GREEN**

Run: `uv run python -m pytest tests/integration/test_workshop_api.py -q -k "feedback_route"`

Expected: all selected tests pass.

### Task 3: UI Context Enrichment

**Files:**
- Modify: `apps/web/app/page.tsx`
- Modify: `apps/web/app/sessionTypes.ts`

- [ ] **Step 1: Update feedback payload**

Send `scope_id`, `question`, `answer_text`, `provenance_unit_ids`, `applied_mappings`, and `expected_correction` where available so backend triage has enough context to build useful proposals.

- [ ] **Step 2: Run frontend quality gate**

Run: `npm run lint` from `apps/web`.

Expected: lint passes.

### Task 4: Verification and Closeout

**Files:**
- Beads metadata
- Git commit

- [ ] **Step 1: Run targeted backend tests**

Run: `uv run python -m pytest tests/unit/test_feedback_learning.py tests/integration/test_workshop_api.py -q -k "feedback_learning or feedback_route"`

Expected: selected backend tests pass.

- [ ] **Step 2: Check git status**

Run: `git status --short --branch --untracked-files=all`

Expected: only intentional code, test, plan, and Beads files changed.

- [ ] **Step 3: Close Beads issue and commit**

Run: `bd close prescient_os-44y --reason="Implemented feedback triage and review-only learning proposals"`

Commit and push code from `prescient_os`; commit and push docs from `../ai_plans` if the plan file is tracked there.
