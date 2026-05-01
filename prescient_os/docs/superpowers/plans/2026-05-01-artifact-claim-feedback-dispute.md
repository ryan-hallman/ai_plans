# Artifact Claim Feedback And Dispute Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the system owner validate or dispute an artifact claim from normal answer feedback.

**Architecture:** Keep artifact trust transitions behind the `ArtifactRepository` protocol. Add a small application service that maps feedback actions to `ValidationEvent` trust states, then expose it through a backend route. Postgres remains the only SQL adapter; routes remain wiring only.

**Tech Stack:** FastAPI, Pydantic v2, existing artifact domain models, existing Postgres artifact repository, pytest.

---

## Boundary Rules

- Domain models stay persistence-free and framework-free.
- Feedback-to-trust mapping lives in an artifact application module, not in the API route.
- Postgres continues to receive `ValidationEvent` objects through the repository protocol.
- The route is system-owner-only for v1; do not add multi-user authority infrastructure.
- Do not implement correction triage, eval-case creation, or IssueSink in this slice.

## Task 1: Claim Feedback Mapping Service

**Files:**
- Create: `apps/api/src/prescient_benchmark/artifacts/feedback.py`
- Modify: `apps/api/src/prescient_benchmark/artifacts/models.py`
- Test: `tests/unit/test_artifact_feedback.py`

- [ ] **Step 1: Write failing feedback mapping tests**

Create tests asserting:

```python
looks_right -> ArtifactTrustState.VALIDATED
wrong -> ArtifactTrustState.DISPUTED
wrong_scope -> ArtifactTrustState.DISPUTED
```

Also assert unsupported actions fail validation.

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run python -m pytest tests/unit/test_artifact_feedback.py -q`

Expected: import failure.

- [ ] **Step 3: Implement feedback mapping**

Add `ArtifactClaimFeedbackAction` enum and `build_system_owner_validation_event(...)`.

- [ ] **Step 4: Verify feedback tests pass**

Run: `uv run python -m pytest tests/unit/test_artifact_feedback.py -q`

Expected: pass.

## Task 2: Feedback Route And Answer Claim Identity

**Files:**
- Modify: `apps/api/src/prescient_benchmark/artifacts/workshop_answer.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/unit/test_workshop_artifact_answer.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing route tests**

Add API tests for:

- `looks_right` returns a validation event with `trust_state == "validated"`
- `wrong_scope` returns a validation event with `trust_state == "disputed"`
- artifact-first answers include `artifact_id` and `claim_id` in retrieval diagnostics so clients can send feedback without guessing IDs

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run python -m pytest tests/unit/test_workshop_artifact_answer.py tests/integration/test_workshop_api.py -q`

Expected: fail for missing feedback route/diagnostics.

- [ ] **Step 3: Implement route and diagnostics**

Add `POST /knowledge/artifacts/{artifact_id}/claims/{claim_id}/feedback` with request body `{ "action": "...", "notes": "..." }`.

Use the feedback mapping service to build the `ValidationEvent`, call `ArtifactRepository.validate_claim`, and return the event.

Add artifact metadata to `KnowledgeAnswer.retrieval_diagnostics` for artifact-first answers:

```python
{
  "artifact_id": "...",
  "claim_id": "..."
}
```

- [ ] **Step 4: Verify route tests pass**

Run: `uv run python -m pytest tests/unit/test_workshop_artifact_answer.py tests/integration/test_workshop_api.py -q`

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
  tests/unit/test_workshop_artifact_answer.py \
  tests/integration/test_workshop_api.py \
  -q
```

Expected: pass.

- [ ] **Step 2: Run broader backend slice**

Run: `uv run python -m pytest tests/unit tests/integration/test_workshop_api.py tests/integration/test_workshop_manuals_cli.py -q`

Expected: pass.

- [ ] **Step 3: Commit and push**

Commit the plan, code, tests, and Beads update. Push docs and main repo.
