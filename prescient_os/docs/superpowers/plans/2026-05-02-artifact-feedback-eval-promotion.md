# Artifact Feedback Eval Promotion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Promote reproducible artifact feedback failures into durable eval-case drafts without routing them through engineering tickets.

**Architecture:** Add a narrow artifact eval-promotion application module that converts `ArtifactFeedbackTriageEvent` regression seeds into `ArtifactEvalCaseDraft` records when the feedback contains enough reproduction data. Persist drafts through a separate `EvalCaseSink` port backed by Postgres, keeping eval promotion independent from `IssueSink` and from the artifact trust-state mutation transaction. The feedback route may return the draft it attempted to record, but the source of truth for retryability remains the already-persisted triage event and regression seed.

**Tech Stack:** FastAPI, Pydantic v2, existing artifact triage models, Postgres via `psycopg`, pytest.

---

## Why This Slice

Negative feedback currently mutates claim trust and records a triage event with a regression seed. That captures the failure, but it does not yet turn the failure into a repeatable test. Eval promotion is the next knowledge-engine quality loop: every reproducible wrong answer should become something the system can score against later.

This must not use Beads directly because most artifact failures are knowledge-quality failures, not engineering tickets. `IssueSink` remains only for `engineering_ticket` outcomes; `EvalCaseSink` is the separate boundary for regression coverage.

Eval promotion should be best-effort after feedback persistence. If the eval sink is unavailable, the feedback route should still record the validation and triage event; the triage event contains enough structured data for a later backfill job to promote the eval case.

## Scope

In scope:

- artifact feedback eval-case draft models
- deterministic promotion rules from `ArtifactFeedbackTriageEvent`
- `EvalCaseSink` protocol and disabled adapter
- Postgres table and idempotent schema creation for eval-case drafts
- feedback route wiring that records an eval draft when feedback is reproducible
- focused unit and route tests

Out of scope:

- frontend UI for eval-case review
- LLM-based triage or eval generation
- automatic artifact correction
- raw retrieval vs artifact-first comparison runner
- Beads/IssueSink integration

## Promotion Rules

Create an eval-case draft only when all of these are present:

- `triage_event.regression_seed.question`
- `triage_event.regression_seed.artifact_id`
- `triage_event.regression_seed.claim_id`
- at least one of:
  - `expected_correction`
  - `answer_text`
  - `provenance_unit_ids`

Promote these categories:

- `claim_correction`
- `scope_correction`
- `extraction_rule_gap`
- `retrieval_rule_gap`

Do not auto-promote these categories in v1:

- `manual_review_needed`: not reproducible enough by default
- `engineering_ticket`: belongs to `IssueSink`; a later adapter may also create an eval case if the issue has a reproducible answer failure

## Task 1: Artifact Eval Draft Models And Promotion Rules

**Files:**
- Create: `apps/api/src/prescient_benchmark/artifacts/eval_cases.py`
- Test: `tests/unit/test_artifact_eval_cases.py`

- [ ] **Step 1: Write failing model and promotion tests**

Add `tests/unit/test_artifact_eval_cases.py`:

```python
from prescient_benchmark.artifacts.eval_cases import (
    ArtifactEvalCaseDraft,
    build_artifact_eval_case_draft,
)
from prescient_benchmark.artifacts.models import (
    ArtifactClaimFeedbackAction,
    ArtifactFeedbackTriageCategory,
    ArtifactFeedbackTriageEvent,
    ArtifactRegressionSeed,
)


def _triage_event(
    *,
    category: ArtifactFeedbackTriageCategory = ArtifactFeedbackTriageCategory.CLAIM_CORRECTION,
    question: str | None = "What is the lower control arm torque?",
    answer_text: str | None = "The answer shown was 55 Nm.",
    expected_correction: str | None = "Use 60 Nm for the rear lower arm.",
    provenance_unit_ids: list[str] | None = None,
) -> ArtifactFeedbackTriageEvent:
    return ArtifactFeedbackTriageEvent(
        triage_event_id="triage-1",
        artifact_id="artifact-1",
        claim_id="claim-1",
        category=category,
        feedback_action=ArtifactClaimFeedbackAction.WRONG,
        notes="wrong value",
        regression_seed=ArtifactRegressionSeed(
            question=question,
            answer_text=answer_text,
            artifact_id="artifact-1",
            claim_id="claim-1",
            feedback_action=ArtifactClaimFeedbackAction.WRONG,
            notes="wrong value",
            category=category,
            expected_correction=expected_correction,
            provenance_unit_ids=provenance_unit_ids or ["unit-p250"],
        ),
    )


def test_build_artifact_eval_case_draft_from_reproducible_triage_event() -> None:
    draft = build_artifact_eval_case_draft(
        triage_event=_triage_event(),
        scope_id="scope-ferrari-360-modena",
        retrieval_profile_id="vehicle_repair_v1",
        active_layer_ids=["domain:vehicle_repair:v1", "product:ferrari_360:v1"],
    )

    assert isinstance(draft, ArtifactEvalCaseDraft)
    assert draft.promoted_from == "artifact_feedback_triage"
    assert draft.triage_event_id == "triage-1"
    assert draft.question == "What is the lower control arm torque?"
    assert draft.expected_behavior == "Use 60 Nm for the rear lower arm."
    assert draft.artifact_id == "artifact-1"
    assert draft.claim_id == "claim-1"
    assert draft.required_evidence_unit_ids == ["unit-p250"]
    assert draft.failure_category == "claim_correction"
    assert draft.scope_id == "scope-ferrari-360-modena"
    assert draft.retrieval_profile_id == "vehicle_repair_v1"
    assert draft.active_layer_ids == ["domain:vehicle_repair:v1", "product:ferrari_360:v1"]


def test_eval_case_draft_uses_answer_text_when_expected_correction_is_missing() -> None:
    draft = build_artifact_eval_case_draft(
        triage_event=_triage_event(expected_correction=None),
        scope_id="scope-ferrari-360-modena",
        retrieval_profile_id="vehicle_repair_v1",
        active_layer_ids=[],
    )

    assert draft is not None
    assert draft.expected_behavior == "Correct the answer shown: The answer shown was 55 Nm."


def test_eval_case_draft_skips_non_reproducible_feedback() -> None:
    draft = build_artifact_eval_case_draft(
        triage_event=_triage_event(
            question=None,
            answer_text=None,
            expected_correction=None,
            provenance_unit_ids=[],
        ),
        scope_id="scope-ferrari-360-modena",
        retrieval_profile_id="vehicle_repair_v1",
        active_layer_ids=[],
    )

    assert draft is None


def test_eval_case_draft_skips_engineering_ticket_category() -> None:
    draft = build_artifact_eval_case_draft(
        triage_event=_triage_event(
            category=ArtifactFeedbackTriageCategory.ENGINEERING_TICKET
        ),
        scope_id="scope-ferrari-360-modena",
        retrieval_profile_id="vehicle_repair_v1",
        active_layer_ids=[],
    )

    assert draft is None
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_artifact_eval_cases.py -q
```

Expected: fail because `prescient_benchmark.artifacts.eval_cases` does not exist.

- [ ] **Step 3: Implement models and builder**

Create `apps/api/src/prescient_benchmark/artifacts/eval_cases.py`:

```python
from datetime import UTC, datetime
from typing import Literal
from uuid import uuid4

from pydantic import BaseModel, ConfigDict, Field

from prescient_benchmark.artifacts.models import (
    ArtifactFeedbackTriageCategory,
    ArtifactFeedbackTriageEvent,
)


_PROMOTABLE_CATEGORIES = {
    ArtifactFeedbackTriageCategory.CLAIM_CORRECTION,
    ArtifactFeedbackTriageCategory.SCOPE_CORRECTION,
    ArtifactFeedbackTriageCategory.EXTRACTION_RULE_GAP,
    ArtifactFeedbackTriageCategory.RETRIEVAL_RULE_GAP,
}


class ArtifactEvalCaseDraft(BaseModel):
    model_config = ConfigDict(extra="forbid")

    eval_case_id: str
    promoted_from: Literal["artifact_feedback_triage"] = "artifact_feedback_triage"
    triage_event_id: str
    question: str
    expected_behavior: str
    scope_id: str
    retrieval_profile_id: str
    active_layer_ids: list[str] = Field(default_factory=list)
    artifact_id: str
    claim_id: str
    required_evidence_unit_ids: list[str] = Field(default_factory=list)
    failure_category: ArtifactFeedbackTriageCategory
    answer_text: str | None = None
    feedback_notes: str | None = None
    created_at: datetime


def build_artifact_eval_case_draft(
    *,
    triage_event: ArtifactFeedbackTriageEvent,
    scope_id: str,
    retrieval_profile_id: str,
    active_layer_ids: list[str],
    eval_case_id: str | None = None,
) -> ArtifactEvalCaseDraft | None:
    seed = triage_event.regression_seed
    if triage_event.category not in _PROMOTABLE_CATEGORIES:
        return None
    if not seed.question:
        return None
    if not (seed.expected_correction or seed.answer_text or seed.provenance_unit_ids):
        return None

    expected_behavior = (
        seed.expected_correction
        if seed.expected_correction
        else f"Correct the answer shown: {seed.answer_text}"
    )

    return ArtifactEvalCaseDraft(
        eval_case_id=eval_case_id or f"artifact-feedback-{uuid4()}",
        triage_event_id=triage_event.triage_event_id,
        question=seed.question,
        expected_behavior=expected_behavior,
        scope_id=scope_id,
        retrieval_profile_id=retrieval_profile_id,
        active_layer_ids=active_layer_ids,
        artifact_id=seed.artifact_id,
        claim_id=seed.claim_id,
        required_evidence_unit_ids=seed.provenance_unit_ids,
        failure_category=triage_event.category,
        answer_text=seed.answer_text,
        feedback_notes=seed.notes,
        created_at=datetime.now(UTC),
    )
```

- [ ] **Step 4: Run tests to verify pass**

Run:

```bash
uv run python -m pytest tests/unit/test_artifact_eval_cases.py -q
```

Expected: pass.

## Task 2: EvalCaseSink Port And Postgres Persistence

**Files:**
- Create: `apps/api/src/prescient_benchmark/artifacts/eval_sink.py`
- Modify: `apps/api/src/prescient_benchmark/artifacts/postgres.py`
- Test: `tests/unit/test_artifact_eval_sink.py`
- Test: `tests/unit/test_artifact_postgres.py`

- [ ] **Step 1: Write failing sink and schema tests**

Create `tests/unit/test_artifact_eval_sink.py`:

```python
from datetime import UTC, datetime

import pytest

from prescient_benchmark.artifacts.eval_cases import ArtifactEvalCaseDraft
from prescient_benchmark.artifacts.eval_sink import DisabledEvalCaseSink
from prescient_benchmark.artifacts.models import ArtifactFeedbackTriageCategory
from prescient_benchmark.artifacts.repository import RepositoryUnavailableError


def _draft() -> ArtifactEvalCaseDraft:
    return ArtifactEvalCaseDraft(
        eval_case_id="eval-1",
        triage_event_id="triage-1",
        question="What is the torque?",
        expected_behavior="Use 60 Nm.",
        scope_id="scope-ferrari-360-modena",
        retrieval_profile_id="vehicle_repair_v1",
        active_layer_ids=["product:ferrari_360:v1"],
        artifact_id="artifact-1",
        claim_id="claim-1",
        required_evidence_unit_ids=["unit-p250"],
        failure_category=ArtifactFeedbackTriageCategory.CLAIM_CORRECTION,
        answer_text="55 Nm",
        feedback_notes="wrong value",
        created_at=datetime(2026, 5, 2, tzinfo=UTC),
    )


def test_disabled_eval_case_sink_rejects_mutation() -> None:
    with pytest.raises(RepositoryUnavailableError, match="eval case sink is unavailable"):
        DisabledEvalCaseSink().record_eval_case_draft(_draft())
```

Update `tests/unit/test_artifact_postgres.py` with a schema assertion:

```python
def test_schema_creates_artifact_eval_case_drafts_table() -> None:
    assert "CREATE TABLE IF NOT EXISTS artifact_eval_case_drafts" in postgres._SCHEMA_SQL
    assert "eval_case_id TEXT PRIMARY KEY" in postgres._SCHEMA_SQL
    assert (
        "triage_event_id TEXT NOT NULL REFERENCES artifact_feedback_triage_events(triage_event_id)"
        in postgres._SCHEMA_SQL
    )
    assert "payload JSONB NOT NULL" in postgres._SCHEMA_SQL
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_artifact_eval_sink.py tests/unit/test_artifact_postgres.py -q
```

Expected: fail because sink and schema do not exist.

- [ ] **Step 3: Implement sink port and Postgres adapter**

Create `apps/api/src/prescient_benchmark/artifacts/eval_sink.py`:

```python
from typing import Protocol

from prescient_benchmark.artifacts.eval_cases import ArtifactEvalCaseDraft
from prescient_benchmark.artifacts.repository import RepositoryUnavailableError


class EvalCaseSink(Protocol):
    def record_eval_case_draft(self, draft: ArtifactEvalCaseDraft) -> None: ...


class DisabledEvalCaseSink:
    def record_eval_case_draft(self, draft: ArtifactEvalCaseDraft) -> None:
        raise RepositoryUnavailableError("eval case sink is unavailable")
```

Modify `apps/api/src/prescient_benchmark/artifacts/postgres.py`:

- import `ArtifactEvalCaseDraft`
- add `connect_eval_case_sink(database_url: str | None) -> EvalCaseSink`
- add `record_eval_case_draft(...)` to `PostgresArtifactRepository`
- add table SQL:

```sql
CREATE TABLE IF NOT EXISTS artifact_eval_case_drafts (
    eval_case_id TEXT PRIMARY KEY,
    triage_event_id TEXT NOT NULL REFERENCES artifact_feedback_triage_events(triage_event_id),
    artifact_id TEXT NOT NULL,
    claim_id TEXT NOT NULL,
    failure_category TEXT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Implementation:

```python
def record_eval_case_draft(self, draft: ArtifactEvalCaseDraft) -> None:
    with self._connection() as connection:
        connection.execute(
            """
            INSERT INTO artifact_eval_case_drafts (
                eval_case_id, triage_event_id, artifact_id, claim_id,
                failure_category, payload, created_at
            )
            VALUES (%s, %s, %s, %s, %s, %s::jsonb, %s)
            ON CONFLICT (eval_case_id) DO UPDATE SET
                payload = EXCLUDED.payload
            """,
            (
                draft.eval_case_id,
                draft.triage_event_id,
                draft.artifact_id,
                draft.claim_id,
                draft.failure_category.value,
                _json(draft.model_dump(mode="json")),
                draft.created_at,
            ),
        )
```

- [ ] **Step 4: Run persistence tests**

Run:

```bash
uv run python -m pytest tests/unit/test_artifact_eval_sink.py tests/unit/test_artifact_postgres.py -q
```

Expected: pass.

## Task 3: Feedback Route Eval Promotion

**Files:**
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing route tests**

Confirm the existing `_ArtifactRepository` fake in `tests/integration/test_workshop_api.py` still records `record_claim_feedback(...)` calls; keep that behavior because the feedback route must continue using the atomic artifact mutation path.

Add an eval sink fake:

```python
class _EvalCaseSink:
    def __init__(self) -> None:
        self.drafts = []

    def record_eval_case_draft(self, draft) -> None:
        self.drafts.append(draft)
```

Add a test:

```python
def test_artifact_claim_feedback_route_promotes_reproducible_triage_to_eval_case(
    tmp_path: Path,
    monkeypatch,
) -> None:
    _seed_api_store(tmp_path, monkeypatch)
    repository = _ArtifactRepository()
    eval_sink = _EvalCaseSink()
    monkeypatch.setattr(routes_knowledge, "_artifact_repository_provider", lambda: repository)
    monkeypatch.setattr(routes_knowledge, "_eval_case_sink_provider", lambda: eval_sink)

    client = TestClient(app)
    response = client.post(
        "/knowledge/artifacts/artifact-ferrari-360-lower-control-arm-spec/claims/lower_control_arm_to_chassis_nuts_torque/feedback",
        json={
            "action": "wrong",
            "notes": "wrong value",
            "scope_id": "scope-ferrari-360-modena",
            "question": "What is the lower control arm torque?",
            "answer_text": "The answer shown was 55 Nm.",
            "provenance_unit_ids": ["unit-source-ferrari-360-wsm-p1"],
            "expected_correction": "Use 60 Nm for the rear lower arm.",
        },
    )

    assert response.status_code == 200
    payload = response.json()
    assert payload["eval_case_draft"]["promoted_from"] == "artifact_feedback_triage"
    assert payload["eval_case_draft"]["expected_behavior"] == "Use 60 Nm for the rear lower arm."
    assert payload["eval_case_draft"]["required_evidence_unit_ids"] == [
        "unit-source-ferrari-360-wsm-p1"
    ]
    assert payload["eval_case_draft"]["scope_id"] == "scope-ferrari-360-modena"
    assert payload["eval_case_draft"]["retrieval_profile_id"] == "vehicle_repair_v1"
    assert len(eval_sink.drafts) == 1
```

Add a non-promoted engineering-ticket test:

```python
def test_artifact_claim_feedback_route_does_not_eval_promote_engineering_ticket(
    tmp_path: Path,
    monkeypatch,
) -> None:
    _seed_api_store(tmp_path, monkeypatch)
    repository = _ArtifactRepository()
    eval_sink = _EvalCaseSink()
    monkeypatch.setattr(routes_knowledge, "_artifact_repository_provider", lambda: repository)
    monkeypatch.setattr(routes_knowledge, "_eval_case_sink_provider", lambda: eval_sink)

    client = TestClient(app)
    response = client.post(
        "/knowledge/artifacts/artifact-ferrari-360-lower-control-arm-spec/claims/lower_control_arm_to_chassis_nuts_torque/feedback",
        json={
            "action": "wrong",
            "notes": "server bug caused a crash",
            "scope_id": "scope-ferrari-360-modena",
            "question": "What is the lower control arm torque?",
            "answer_text": "The answer shown was 55 Nm.",
        },
    )

    assert response.status_code == 200
    assert response.json()["triage_event"]["category"] == "engineering_ticket"
    assert response.json()["eval_case_draft"] is None
    assert eval_sink.drafts == []
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py -q
```

Expected: fail because route does not provide `_eval_case_sink_provider` or `eval_case_draft`.

- [ ] **Step 3: Implement route wiring**

Modify `apps/api/src/prescient_benchmark/api/routes_knowledge.py`:

- import:

```python
from prescient_benchmark.artifacts.eval_cases import build_artifact_eval_case_draft
from prescient_benchmark.artifacts.postgres import connect_eval_case_sink
```

- add provider:

```python
_eval_case_sink_provider = lambda: connect_eval_case_sink(settings.artifact_database_url)
```

- after `repository.record_claim_feedback(validation_event, triage_event)`, build and record a draft:

```python
eval_case_draft = None
if triage_event is not None:
    scope = resolve_seeded_scope(request.scope_id)
    eval_case_draft = build_artifact_eval_case_draft(
        triage_event=triage_event,
        scope_id=scope.scope_id,
        retrieval_profile_id=VEHICLE_REPAIR_PROFILE.profile_id,
        active_layer_ids=active_vehicle_repair_layer_ids(scope),
    )
    if eval_case_draft is not None:
        try:
            _eval_case_sink_provider().record_eval_case_draft(eval_case_draft)
        except RepositoryUnavailableError:
            eval_case_draft = None
```

Use the real request shape only after adding `scope_id` if the route request does not already carry it. Prefer:

```python
scope_id: str | None = None
```

on `ArtifactClaimFeedbackRouteRequest`, defaulting through `resolve_seeded_scope(None)` when omitted.

Add to response:

```python
"eval_case_draft": None if eval_case_draft is None else eval_case_draft.model_dump(mode="json")
```

Do not fail the feedback request if eval promotion is unavailable. The triage event is already persisted and can be backfilled.

- [ ] **Step 4: Run route tests**

Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py -q
```

Expected: pass.

## Task 4: Verification And Commit

**Files:**
- All files above.

- [ ] **Step 1: Run focused artifact/eval tests**

Run:

```bash
uv run python -m pytest \
  tests/unit/test_artifact_eval_cases.py \
  tests/unit/test_artifact_eval_sink.py \
  tests/unit/test_artifact_postgres.py \
  tests/unit/test_artifact_feedback.py \
  tests/unit/test_artifact_repository.py \
  tests/integration/test_workshop_api.py \
  -q
```

Expected: pass.

- [ ] **Step 2: Run broader backend slice**

Run:

```bash
uv run python -m pytest tests/unit tests/integration/test_workshop_api.py tests/integration/test_workshop_manuals_cli.py -q
```

Expected: pass.

- [ ] **Step 3: Commit and push**

Run:

```bash
git status --short
git add \
  apps/api/src/prescient_benchmark/artifacts/eval_cases.py \
  apps/api/src/prescient_benchmark/artifacts/eval_sink.py \
  apps/api/src/prescient_benchmark/artifacts/postgres.py \
  apps/api/src/prescient_benchmark/api/routes_knowledge.py \
  tests/unit/test_artifact_eval_cases.py \
  tests/unit/test_artifact_eval_sink.py \
  tests/unit/test_artifact_postgres.py \
  tests/integration/test_workshop_api.py \
  .beads/issues.jsonl
git commit -m "Add artifact feedback eval promotion"
git push
```

Expected: branch pushed.

## Review Checklist

- Eval promotion uses `EvalCaseSink`, not `IssueSink`.
- `engineering_ticket` is not promoted to an eval case in v1.
- Feedback route still records validation and triage even when eval sink is unavailable.
- Eval draft records include question, expected behavior, scope, retrieval profile, artifact id, claim id, category, answer shown, feedback notes, and required evidence units.
- No frontend or LLM triage changes are introduced.
