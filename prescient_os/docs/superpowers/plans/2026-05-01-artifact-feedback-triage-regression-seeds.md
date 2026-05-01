# Artifact Feedback Triage And Regression Seeds Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn negative artifact claim feedback into structured triage output and regression seeds.

**Architecture:** Keep triage deterministic and application-layer for v1. The feedback route will still perform trust-state mutation through `ArtifactRepository.validate_claim`, then use a feedback triage service to create an optional `ArtifactFeedbackTriageEvent` for negative feedback. Postgres persists triage events through the repository protocol; no domain/application code depends on Beads or any ticketing system.

**Tech Stack:** FastAPI, Pydantic v2, existing artifact repository protocol, Postgres adapter, pytest.

---

## Boundary Rules

- Domain models remain framework-free and persistence-free.
- Triage classification lives in `artifacts/feedback.py`, not in API routes.
- Persistence uses `ArtifactRepository`; Postgres SQL stays in `artifacts/postgres.py`.
- V1 triage is deterministic from feedback action and note keywords.
- Do not implement LLM triage, IssueSink, eval-case storage integration, automatic corrections, or frontend changes in this slice.

## Decision Rationale

- Keep triage deterministic for v1 because the first improvement loop needs predictable behavior that is easy to test, debug, and review before we add an agent that can reinterpret user feedback.
- Put classification in the artifact feedback application module because feedback triage is product behavior, not transport behavior; the API should adapt requests and responses, not own business rules.
- Persist triage through `ArtifactRepository` because artifact trust lifecycle data is mutable knowledge-engine state, and keeping it behind a port preserves the future DDD refactor path.
- Store a regression seed payload with the triage event because disputed answers should immediately become testable examples, even before we build a full eval-case promotion workflow.
- Avoid Beads, IssueSink, and eval-store coupling in this slice because those are operational integrations; the knowledge engine should emit structured intent first, then adapters can decide how to route it.
- Do not auto-correct claims from negative feedback because user feedback identifies a problem, but canonical knowledge still needs evidence-backed repair or validation before publication.
- Keep `looks_right` out of triage because positive validation already has a trust-state path; triage is specifically for turning negative feedback into repair work.

## Task 1: Triage Models And Deterministic Classifier

**Files:**
- Modify: `apps/api/src/prescient_benchmark/artifacts/models.py`
- Modify: `apps/api/src/prescient_benchmark/artifacts/feedback.py`
- Test: `tests/unit/test_artifact_feedback.py`

- [ ] **Step 1: Write failing triage tests**

Add tests that assert:

- `looks_right` feedback returns no triage event.
- `wrong_scope` feedback maps to `scope_correction`.
- `wrong` feedback with notes containing `wrong value` maps to `claim_correction`.
- `wrong` feedback with notes containing `table` or `parser` maps to `extraction_rule_gap`.
- generated triage events include a regression seed with question, answer text, artifact id, claim id, feedback action, notes, category, and expected correction when supplied.

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run python -m pytest tests/unit/test_artifact_feedback.py -q`

Expected: fail for missing triage models/classifier.

- [ ] **Step 3: Implement models and classifier**

Add:

- `ArtifactFeedbackTriageCategory`
- `ArtifactRegressionSeed`
- `ArtifactFeedbackTriageEvent`
- `build_feedback_triage_event(...) -> ArtifactFeedbackTriageEvent | None`

Use deterministic keyword rules only.

- [ ] **Step 4: Verify triage tests pass**

Run: `uv run python -m pytest tests/unit/test_artifact_feedback.py -q`

Expected: pass.

## Task 2: Repository Persistence And Feedback Route

**Files:**
- Modify: `apps/api/src/prescient_benchmark/artifacts/repository.py`
- Modify: `apps/api/src/prescient_benchmark/artifacts/postgres.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/unit/test_artifact_repository.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing persistence and route tests**

Add tests that assert:

- Repository protocol/null adapter exposes `record_feedback_triage(...)`.
- Disabled repository mutation raises unavailable instead of false success.
- Feedback route accepts optional `question`, `answer_text`, `expected_correction`.
- Negative feedback returns a `triage_event`.
- `looks_right` feedback returns no `triage_event`.

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run python -m pytest tests/unit/test_artifact_repository.py tests/integration/test_workshop_api.py -q`

Expected: fail for missing repository method/route triage response.

- [ ] **Step 3: Implement persistence and route wiring**

Add repository method:

```python
record_feedback_triage(self, triage_event: ArtifactFeedbackTriageEvent) -> None
```

Postgres should create `artifact_feedback_triage_events` with:

- `triage_event_id`
- `artifact_id`
- `claim_id`
- `category`
- `payload JSONB`
- `created_at`

Update feedback route to build and persist triage only when the classifier returns an event.

- [ ] **Step 4: Verify persistence and route tests pass**

Run: `uv run python -m pytest tests/unit/test_artifact_repository.py tests/integration/test_workshop_api.py -q`

Expected: pass.

## Task 3: Verification And Commit

**Files:**
- All files above.

- [ ] **Step 1: Run focused tests**

Run:

```bash
uv run python -m pytest \
  tests/unit/test_artifact_feedback.py \
  tests/unit/test_artifact_repository.py \
  tests/integration/test_workshop_api.py \
  -q
```

Expected: pass.

- [ ] **Step 2: Run broader backend slice**

Run: `uv run python -m pytest tests/unit tests/integration/test_workshop_api.py tests/integration/test_workshop_manuals_cli.py -q`

Expected: pass.

- [ ] **Step 3: Commit and push**

Commit the plan, code, tests, and Beads update. Push docs and main repo.
