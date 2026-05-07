# Diagnostic Reasoning Mode First Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the first diagnostic reasoning-mode slice so symptom-style workshop questions can return bounded, corpus-grounded troubleshooting answers with explicit evidence and inference labels.

**Architecture:** Add a generic reasoning-mode contract under `prescient_benchmark.knowledge`, then let the workshop API select `vehicle_repair_v1 + diagnostic` when the question shape warrants it. The workshop service remains the application boundary that maps retrieved source units into locators/citations and validates labeled claims before returning an answer.

**Tech Stack:** FastAPI, Pydantic v2 models, existing workshop manual store/search, OpenAI Responses structured output, existing pytest unit/integration suites.

---

## Why This Sequence

The first PR should establish the contract before optimizing retrieval. That keeps the domain-neutral shape stable for later business use cases and prevents diagnostic behavior from being hidden inside `VehicleRepairIntent`.

The classifier is rule-first because exact torque/procedure/catalog queries already work and should not start paying LLM latency or ambiguity cost. An LLM classifier can be added later only for boundary cases.

Claim validation is deterministic in this slice because prompt compliance is not sufficient for a knowledge engine. The system must be able to reject an evidence label that lacks source-unit and locator backing.

The diagnostic retrieval loop is intentionally shallow in the first PR. It should broaden candidate evidence enough to support a useful troubleshooting answer, but it should not become an expert-system rewrite or ingest forum/community data yet.

## Files

- Create: `apps/api/src/prescient_benchmark/knowledge/reasoning_modes.py`
- Modify: `apps/api/src/prescient_benchmark/knowledge/models.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/providers.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/service.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/events.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Modify: `tests/unit/test_reasoning_modes.py`
- Modify: `tests/unit/test_workshop_providers.py`
- Modify: `tests/unit/test_workshop_service.py`
- Modify: `tests/integration/test_workshop_api.py`

## Task 1: Reasoning Mode Domain Models And Classifier

**Files:**
- Create: `apps/api/src/prescient_benchmark/knowledge/reasoning_modes.py`
- Test: `tests/unit/test_reasoning_modes.py`

- [ ] **Step 1: Write failing model/classifier tests**

```python
from pydantic import ValidationError
import pytest

from prescient_benchmark.knowledge.reasoning_modes import (
    AnswerClaim,
    ClaimLabel,
    ClaimValidationContext,
    ReasoningMode,
    classify_reasoning_mode,
    validate_answer_claims,
)


def test_classifies_symptom_condition_question_as_diagnostic() -> None:
    decision = classify_reasoning_mode(
        "When I load the suspension in a right turn, I get a lot of vibration. What could be wrong?"
    )

    assert decision.primary_mode is ReasoningMode.DIAGNOSTIC
    assert decision.confidence >= 0.75
    assert "symptom described" in decision.reasons
    assert "asks what could be wrong" in decision.reasons


def test_direct_lookup_wins_for_torque_questions() -> None:
    decision = classify_reasoning_mode("What are the torque specs for the engine mounts?")

    assert decision.primary_mode is ReasoningMode.DIRECT_LOOKUP
    assert decision.confidence >= 0.75


def test_manual_evidence_claim_requires_source_and_locator_ids() -> None:
    claim = AnswerClaim(
        claim_id="claim-1",
        text="The manual-backed inspection is cited.",
        label=ClaimLabel.MANUAL_EVIDENCE,
        source_unit_ids=["unit-1"],
        locator_ids=[],
        confidence="high",
    )

    with pytest.raises(ValueError, match="manual_evidence requires source_unit_ids and locator_ids"):
        validate_answer_claims(
            [claim],
            ClaimValidationContext(
                mode=ReasoningMode.DIAGNOSTIC,
                available_source_unit_ids={"unit-1"},
                available_locator_ids={"loc-1"},
                artifact_claim_ids=set(),
            ),
        )


def test_general_domain_knowledge_is_disallowed_for_direct_lookup() -> None:
    claim = AnswerClaim(
        claim_id="claim-1",
        text="General repair intuition.",
        label=ClaimLabel.GENERAL_DOMAIN_KNOWLEDGE,
        confidence="low",
    )

    with pytest.raises(ValueError, match="general_domain_knowledge is not allowed in direct_lookup"):
        validate_answer_claims(
            [claim],
            ClaimValidationContext(
                mode=ReasoningMode.DIRECT_LOOKUP,
                available_source_unit_ids=set(),
                available_locator_ids=set(),
                artifact_claim_ids=set(),
            ),
        )
```

- [ ] **Step 2: Run tests to verify RED**

Run: `uv run python -m pytest tests/unit/test_reasoning_modes.py -q`

Expected: FAIL because `prescient_benchmark.knowledge.reasoning_modes` does not exist.

- [ ] **Step 3: Implement minimal models, classifier, and validator**

```python
from enum import StrEnum

from pydantic import BaseModel, ConfigDict, Field, field_validator


class ReasoningMode(StrEnum):
    AUTO = "auto"
    DIRECT_LOOKUP = "direct_lookup"
    PROCEDURE = "procedure"
    CATALOG_LOOKUP = "catalog_lookup"
    DIAGNOSTIC = "diagnostic"
    MECHANISM_EXPLANATION = "mechanism_explanation"
    HYPOTHETICAL = "hypothetical"
    WORK_PLANNING = "work_planning"


class ClaimLabel(StrEnum):
    MANUAL_EVIDENCE = "manual_evidence"
    ARTIFACT_EVIDENCE = "artifact_evidence"
    INFERENCE = "inference"
    GENERAL_DOMAIN_KNOWLEDGE = "general_domain_knowledge"
    NEEDS_USER_OBSERVATION = "needs_user_observation"
    SUGGESTED_CHECK = "suggested_check"


class ClaimConfidence(StrEnum):
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"


class ReasoningModeDecision(BaseModel):
    model_config = ConfigDict(extra="forbid")

    primary_mode: ReasoningMode
    secondary_modes: list[ReasoningMode] = Field(default_factory=list)
    confidence: float = Field(ge=0.0, le=1.0)
    reasons: list[str] = Field(default_factory=list)
    clarification_question: str | None = None


class AnswerClaim(BaseModel):
    model_config = ConfigDict(extra="forbid")

    claim_id: str = Field(min_length=1)
    text: str = Field(min_length=1)
    label: ClaimLabel
    source_unit_ids: list[str] = Field(default_factory=list)
    locator_ids: list[str] = Field(default_factory=list)
    artifact_claim_ids: list[str] = Field(default_factory=list)
    depends_on_claim_ids: list[str] = Field(default_factory=list)
    confidence: ClaimConfidence
    safety_relevant: bool = False

    @field_validator("locator_ids")
    @classmethod
    def locator_ids_parallel_source_units(cls, value: list[str], info):
        source_unit_ids = info.data.get("source_unit_ids", [])
        if value and len(value) != len(source_unit_ids):
            raise ValueError("locator_ids must be parallel with source_unit_ids")
        return value


class ClaimValidationContext(BaseModel):
    model_config = ConfigDict(extra="forbid", arbitrary_types_allowed=True)

    mode: ReasoningMode
    available_source_unit_ids: set[str]
    available_locator_ids: set[str]
    artifact_claim_ids: set[str]


def classify_reasoning_mode(question: str) -> ReasoningModeDecision:
    normalized = f" {question.casefold()} "
    if _has_any(normalized, (" torque ", " torques ", " tightening torque ", " spec ", " specs ", " nm ")):
        return ReasoningModeDecision(
            primary_mode=ReasoningMode.DIRECT_LOOKUP,
            confidence=0.9,
            reasons=["exact value/specification requested"],
        )
    if _has_any(normalized, (" how do i remove ", " how do i install ", " how do i check ", " procedure ")):
        return ReasoningModeDecision(
            primary_mode=ReasoningMode.PROCEDURE,
            confidence=0.86,
            reasons=["procedure requested"],
        )
    if _has_any(normalized, (" where are ", " where is ", " list all ", " all the ")):
        return ReasoningModeDecision(
            primary_mode=ReasoningMode.CATALOG_LOOKUP,
            confidence=0.82,
            reasons=["catalog/list lookup requested"],
        )
    symptom_cues = (" vibration", " vibrates", " noise", " clunk", " shake", " misfire", " leak", " loose")
    condition_cues = (" when ", " while ", " under ", " load", " turn", " braking", " accelerating")
    diagnostic_cues = (" what could be wrong", " what might be wrong", " diagnose", " cause", " causing")
    reasons = []
    if _has_any(normalized, symptom_cues):
        reasons.append("symptom described")
    if _has_any(normalized, condition_cues):
        reasons.append("operating condition described")
    if _has_any(normalized, diagnostic_cues):
        reasons.append("asks what could be wrong")
    if len(reasons) >= 2:
        return ReasoningModeDecision(
            primary_mode=ReasoningMode.DIAGNOSTIC,
            confidence=0.82 if len(reasons) == 3 else 0.76,
            reasons=reasons,
        )
    return ReasoningModeDecision(
        primary_mode=ReasoningMode.PROCEDURE,
        confidence=0.62,
        reasons=["default workshop behavior"],
    )


def validate_answer_claims(
    claims: list[AnswerClaim],
    context: ClaimValidationContext,
) -> list[AnswerClaim]:
    claim_ids = {claim.claim_id for claim in claims}
    for claim in claims:
        if claim.label is ClaimLabel.MANUAL_EVIDENCE:
            if not claim.source_unit_ids or not claim.locator_ids:
                raise ValueError("manual_evidence requires source_unit_ids and locator_ids")
        if claim.label is ClaimLabel.ARTIFACT_EVIDENCE and not claim.artifact_claim_ids:
            raise ValueError("artifact_evidence requires artifact_claim_ids")
        if claim.label is ClaimLabel.INFERENCE and not (
            claim.depends_on_claim_ids or claim.source_unit_ids or claim.artifact_claim_ids
        ):
            raise ValueError("inference requires evidence inputs")
        if (
            claim.label is ClaimLabel.GENERAL_DOMAIN_KNOWLEDGE
            and context.mode is ReasoningMode.DIRECT_LOOKUP
        ):
            raise ValueError("general_domain_knowledge is not allowed in direct_lookup")
        unknown_units = set(claim.source_unit_ids) - context.available_source_unit_ids
        if unknown_units:
            raise ValueError(f"claim references unavailable source_unit_ids: {sorted(unknown_units)}")
        unknown_locators = set(claim.locator_ids) - context.available_locator_ids
        if unknown_locators:
            raise ValueError(f"claim references unavailable locator_ids: {sorted(unknown_locators)}")
        unknown_dependencies = set(claim.depends_on_claim_ids) - claim_ids
        if unknown_dependencies:
            raise ValueError(f"claim references unavailable depends_on_claim_ids: {sorted(unknown_dependencies)}")
    return claims


def _has_any(text: str, cues: tuple[str, ...]) -> bool:
    return any(cue in text for cue in cues)
```

- [ ] **Step 4: Run tests to verify GREEN**

Run: `uv run python -m pytest tests/unit/test_reasoning_modes.py -q`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/knowledge/reasoning_modes.py tests/unit/test_reasoning_modes.py
git commit -m "feat: add reasoning mode contract"
```

## Task 2: API Contract Exposes Reasoning Mode Decisions

**Files:**
- Modify: `apps/api/src/prescient_benchmark/knowledge/models.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing API tests**

Add tests near the existing `/knowledge/ask/sync` route tests:

```python
def test_ask_knowledge_sync_exposes_reasoning_mode_decision(client, monkeypatch):
    _seed_single_page_store(monkeypatch, text="Front suspension inspection information. Ball joint and arm inspection.")

    response = client.post(
        "/knowledge/ask/sync",
        json={
            "question": "When I load the suspension in a right turn, I get vibration. What could be wrong?",
            "scope_id": "scope-ferrari-360-modena",
        },
    )

    payload = response.json()
    assert response.status_code == 200
    assert payload["reasoning_mode_decision"]["primary_mode"] == "diagnostic"
    assert payload["reasoning_mode_decision"]["confidence"] >= 0.75
```

- [ ] **Step 2: Run tests to verify RED**

Run: `uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_sync_exposes_reasoning_mode_decision -q`

Expected: FAIL because the response has no `reasoning_mode_decision`.

- [ ] **Step 3: Add the response/request fields and mode selection**

In `KnowledgeAnswer`, add:

```python
from prescient_benchmark.knowledge.reasoning_modes import AnswerClaim, ReasoningModeDecision

reasoning_mode_decision: ReasoningModeDecision | None = None
claims: list[AnswerClaim] = Field(default_factory=list)
```

In `AskRouteRequest`, add:

```python
from prescient_benchmark.knowledge.reasoning_modes import ReasoningMode, classify_reasoning_mode

reasoning_mode: ReasoningMode = ReasoningMode.AUTO
```

In `routes_knowledge.py`, add `_reasoning_mode_decision_for_request(request)` that returns the explicit mode with confidence `1.0` when the request is not `auto`, otherwise calls `classify_reasoning_mode(request.question)`.

Pass the decision into `AskKnowledgeService.answer_from_candidates(...)` and `stream_answer_from_candidates(...)`.

- [ ] **Step 4: Run focused tests**

Run: `uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_sync_exposes_reasoning_mode_decision -q`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/knowledge/models.py apps/api/src/prescient_benchmark/api/routes_knowledge.py tests/integration/test_workshop_api.py
git commit -m "feat: expose reasoning mode decisions"
```

## Task 3: Diagnostic Provider Contract And Claim Validation

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/providers.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/service.py`
- Test: `tests/unit/test_workshop_providers.py`
- Test: `tests/unit/test_workshop_service.py`

- [ ] **Step 1: Write failing provider/service tests**

Add provider parsing tests that verify structured diagnostic JSON is accepted only when claims are well-formed:

```python
def test_diagnostic_answer_parses_structured_claims() -> None:
    answer = providers._diagnostic_answer_from_json_text(
        json.dumps(
            {
                "answer": "Inspect the left front suspension first.",
                "support_state": "supported",
                "cited_unit_indices": [0],
                "claims": [
                    {
                        "claim_id": "manual-1",
                        "text": "The manual has a suspension inspection procedure.",
                        "label": "manual_evidence",
                        "source_unit_indices": [0],
                        "depends_on_claim_ids": [],
                        "confidence": "high",
                        "safety_relevant": False,
                    },
                    {
                        "claim_id": "infer-1",
                        "text": "Right turns can make left-side components more suspect.",
                        "label": "inference",
                        "source_unit_indices": [],
                        "depends_on_claim_ids": ["manual-1"],
                        "confidence": "medium",
                        "safety_relevant": False,
                    },
                ],
            }
        )
    )

    assert answer.claims[1].label == "inference"
```

Add a service test:

```python
def test_service_rejects_diagnostic_manual_claim_without_citation(tmp_path: Path) -> None:
    store = _store_with_one_supported_page(tmp_path)
    service = AskKnowledgeService(
        store=store,
        scope=default_360_scope(),
        llm_provider=_DiagnosticProviderWithInvalidManualClaim(),
    )

    answer = service.answer_from_candidates(
        question="When turning right I get vibration. What could be wrong?",
        candidate_unit_ids=["unit-source-ferrari-360-wsm-p1"],
        reasoning_mode_decision=ReasoningModeDecision(
            primary_mode=ReasoningMode.DIAGNOSTIC,
            confidence=0.9,
            reasons=["symptom described", "asks what could be wrong"],
        ),
    )

    assert answer.status is AnswerStatus.INSUFFICIENT_EVIDENCE
    assert "unsupported diagnostic claim label" in answer.text
```

- [ ] **Step 2: Run tests to verify RED**

Run: `uv run python -m pytest tests/unit/test_workshop_providers.py::test_diagnostic_answer_parses_structured_claims tests/unit/test_workshop_service.py::test_service_rejects_diagnostic_manual_claim_without_citation -q`

Expected: FAIL because diagnostic provider parsing and service validation do not exist.

- [ ] **Step 3: Implement diagnostic provider payloads**

Add dataclasses `LlmDiagnosticClaim` and `LlmDiagnosticAnswer` to `providers.py`. Add `_diagnostic_answer_json_schema_format()`, `_diagnostic_openai_input_messages(...)`, and `_diagnostic_answer_from_json_text(...)`.

Extend `OpenAIWorkshopLlmProvider`, `HttpJsonWorkshopLlmProvider`, and `TestOnlyDeterministicLlmProvider` with:

```python
def answer_diagnostic_from_evidence(self, *, question: str, evidence_text: str) -> LlmDiagnosticAnswer:
    ...
```

The OpenAI diagnostic system prompt must require:

- ranked hypotheses only, not definitive diagnosis
- `manual_evidence` only for direct cited facts
- `inference` only when it depends on manual evidence
- `needs_user_observation` for narrowing observations
- `suggested_check` for practical inspection steps
- no unsupported torque/procedure invention

- [ ] **Step 4: Implement service mapping and validation**

In `AskKnowledgeService.answer_from_candidates`, branch when `reasoning_mode_decision.primary_mode == ReasoningMode.DIAGNOSTIC`:

1. Call `llm_provider.answer_diagnostic_from_evidence(...)`.
2. Map `source_unit_indices` from the provider payload to actual `source_unit_ids` and `locator_ids`.
3. Build `AnswerClaim` values.
4. Call `validate_answer_claims(...)`.
5. If validation fails, return `AnswerStatus.INSUFFICIENT_EVIDENCE` with candidate citations and the trace item `reject_unsupported_claim_labels`.

- [ ] **Step 5: Run focused tests**

Run: `uv run python -m pytest tests/unit/test_workshop_providers.py tests/unit/test_workshop_service.py -q`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/providers.py apps/api/src/prescient_benchmark/workshop_manuals/service.py tests/unit/test_workshop_providers.py tests/unit/test_workshop_service.py
git commit -m "feat: validate diagnostic answer claims"
```

## Task 4: Diagnostic Retrieval Expansion

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/retrieval_profile.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/unit/test_workshop_retrieval_profile.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing retrieval tests**

```python
def test_diagnostic_query_terms_expand_symptom_to_related_vehicle_systems() -> None:
    terms = built_in_vehicle_repair_reasoning_terms(
        question="Vibration while loading the suspension in a right turn",
        reasoning_mode=ReasoningMode.DIAGNOSTIC,
    )

    assert "wheel bearing" in terms
    assert "ball joint" in terms
    assert "control arm" in terms
    assert "steering joint" in terms
    assert "wheel alignment data" in terms
```

- [ ] **Step 2: Run test to verify RED**

Run: `uv run python -m pytest tests/unit/test_workshop_retrieval_profile.py::test_diagnostic_query_terms_expand_symptom_to_related_vehicle_systems -q`

Expected: FAIL because `built_in_vehicle_repair_reasoning_terms` does not exist.

- [ ] **Step 3: Implement narrow diagnostic expansion**

Add `built_in_vehicle_repair_reasoning_terms(question, reasoning_mode)` that returns no terms outside diagnostic mode and adds suspension/steering/wheel/hub/alignment terms for vibration questions.

In `_gather_candidate_unit_ids`, include these terms in the single-pass search query when the decision is diagnostic. Use `top_k=50`, rerank with procedure intent for now, and context expansion `max_units=30` for diagnostic.

- [ ] **Step 4: Run focused tests**

Run: `uv run python -m pytest tests/unit/test_workshop_retrieval_profile.py tests/integration/test_workshop_api.py::test_ask_knowledge_sync_exposes_reasoning_mode_decision -q`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/retrieval_profile.py apps/api/src/prescient_benchmark/api/routes_knowledge.py tests/unit/test_workshop_retrieval_profile.py tests/integration/test_workshop_api.py
git commit -m "feat: broaden diagnostic workshop retrieval"
```

## Task 5: Streaming Event And Web Compatibility

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/events.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/service.py`
- Modify: `apps/web/app/sessionTypes.ts`
- Modify: `apps/web/app/page.tsx`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing SSE test**

```python
def test_ask_knowledge_stream_emits_reasoning_mode_event(client, monkeypatch):
    _seed_single_page_store(monkeypatch, text="Suspension inspection and wheel bearing check information.")

    response = client.post(
        "/knowledge/ask",
        json={
            "question": "When loading the suspension in a right turn I get vibration. What could be wrong?",
            "scope_id": "scope-ferrari-360-modena",
        },
        headers={"Accept": "text/event-stream"},
    )

    events = _decode_sse_events(response.content)
    reasoning_event = next(event for event in events if event["event"] == "reasoning_mode")
    assert reasoning_event["data"]["decision"]["primary_mode"] == "diagnostic"
```

- [ ] **Step 2: Run test to verify RED**

Run: `uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_stream_emits_reasoning_mode_event -q`

Expected: FAIL because no `reasoning_mode` event exists.

- [ ] **Step 3: Add event and frontend fields**

Add `ReasoningModeEvent` with `decision: ReasoningModeDecision` and include it in `ProviderEvent`. Yield it before retrieval diagnostics when a decision is present.

Update the web `Answer` type with optional `reasoning_mode_decision` and `claims`, consume the SSE event, and show a compact mode chip using labels `Direct`, `Procedure`, `Troubleshoot`, `Explain`, `What-if`, and `Plan`.

- [ ] **Step 4: Run focused API and web checks**

Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_stream_emits_reasoning_mode_event -q
npm --prefix apps/web run lint
```

Expected: API test PASS and lint PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/events.py apps/api/src/prescient_benchmark/workshop_manuals/service.py apps/web/app/sessionTypes.ts apps/web/app/page.tsx tests/integration/test_workshop_api.py
git commit -m "feat: stream reasoning mode metadata"
```

## Task 6: Diagnostic Eval Seeds

**Files:**
- Modify: `eval/questions/workshop_manuals_v1.yaml`
- Modify: `tests/unit/test_workshop_eval.py`

- [ ] **Step 1: Write failing eval asset test**

Add a test asserting at least one diagnostic category exists and includes an adversarial symptom question whose expected answer behavior is bounded hypotheses or observation request.

- [ ] **Step 2: Run test to verify RED**

Run: `uv run python -m pytest tests/unit/test_workshop_eval.py::test_workshop_eval_includes_diagnostic_reasoning_seed -q`

Expected: FAIL because no diagnostic eval seed exists.

- [ ] **Step 3: Add a small initial diagnostic seed set**

Add four diagnostic questions for suspension/wheel vibration, steering looseness, brake-adjacent vibration, and an adversarial insufficient-evidence symptom. Mark the evidence requirements as manual-backed inspection/spec pages where known; if evidence keys are not verified in this PR, record only the category and expected answer behavior and create a follow-up Beads issue for full 12+ question coverage.

- [ ] **Step 4: Run eval asset tests**

Run: `uv run python -m pytest tests/unit/test_workshop_eval.py -q`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add eval/questions/workshop_manuals_v1.yaml tests/unit/test_workshop_eval.py
git commit -m "test: seed diagnostic workshop eval cases"
```

## Final Verification

- [ ] Run backend focused suite:

```bash
uv run python -m pytest \
  tests/unit/test_reasoning_modes.py \
  tests/unit/test_workshop_providers.py \
  tests/unit/test_workshop_service.py \
  tests/unit/test_workshop_retrieval_profile.py \
  tests/integration/test_workshop_api.py -q
```

- [ ] Run frontend lint if Task 5 changed UI:

```bash
npm --prefix apps/web run lint
```

- [ ] Run git status in the app repo and docs repo.
- [ ] Close `prescient_os-x9j` only after code, tests, docs, and push are complete.

## Deferred Follow-Ups

- LLM classifier for low-confidence boundary cases.
- Full diagnostic retrieval loop with up to four targeted retrieval calls.
- Full 12+ question diagnostic eval set and 25+ mode-classification examples.
- Forum/community ingestion.
- Feedback-driven diagnostic artifact promotion.
