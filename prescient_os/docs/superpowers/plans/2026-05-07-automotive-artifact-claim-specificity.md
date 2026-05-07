# Automotive Artifact Claim Specificity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make automotive artifact-first answers safer than raw retrieval by preserving qualifier-specific service specs, rejecting noisy OCR artifacts, and clarifying or listing multiple matching claims instead of flattening them into one value.

**Architecture:** Keep artifacts as structured, reviewable knowledge and keep BM25 retrieval as fallback/context. Add sparse claim qualifiers to artifact models, harden workshop extraction/assembly against noisy component labels, and move artifact answer selection from "best single claim" to qualifier-aware match outcomes. Do not make artifact-first the default until comparison evals show it improves answer quality.

**Tech Stack:** FastAPI, Pydantic v2, psycopg/Postgres JSONB projections, existing workshop JSON store, existing OpenSearch/BM25 fallback path, pytest.

---

## Governing Spec

This plan implements the "Automotive Component-Spec Claim Specificity" section added to:

- `docs/superpowers/specs/2026-05-01-artifact-first-knowledge-extraction-design.md`

## File Structure

- Modify `apps/api/src/prescient_benchmark/artifacts/models.py`: add `SpecQualifiers` and optional `spec_qualifiers` fields on observations and artifact claims.
- Modify `apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py`: extract sparse qualifiers for deterministic workshop torque observations and reject noisy labels.
- Modify `apps/api/src/prescient_benchmark/artifacts/workshop_assembly.py`: carry qualifiers into claims, use qualifier-aware claim ids/artifact grouping, and avoid one-value-per-component flattening.
- Modify `apps/api/src/prescient_benchmark/artifacts/postgres.py`: make artifact scoring token-boundary and qualifier-aware enough to stop one-letter OCR artifacts from winning broad queries.
- Modify `apps/api/src/prescient_benchmark/artifacts/workshop_answer.py`: return single-claim answers only for specific matches; return multi-claim or clarification answers for broad component queries with multiple applicable claims.
- Modify `tests/unit/test_artifact_models.py`: cover qualifier serialization and backward-compatible defaults.
- Modify `tests/unit/test_workshop_artifact_extraction.py`: cover qualifier extraction and noisy-label rejection.
- Modify `tests/unit/test_artifact_postgres.py`: cover token-boundary scoring.
- Modify `tests/unit/test_workshop_artifact_answer.py`: cover single-claim, multi-claim, and clarification behavior.
- Modify `tests/integration/test_workshop_api.py`: cover `/knowledge/ask/sync` artifact mode for broad lower-control-arm query and specific lower-arm-to-chassis query.

## Task 1: Add Structured Claim Qualifiers

**Files:**
- Modify: `apps/api/src/prescient_benchmark/artifacts/models.py`
- Test: `tests/unit/test_artifact_models.py`

- [ ] **Step 1: Write failing model tests**

Append these tests to `tests/unit/test_artifact_models.py`:

```python
from prescient_benchmark.artifacts.models import (
    ArtifactClaim,
    ArtifactTrustState,
    Observation,
    ObservationClaimType,
    SourceAuthority,
    SpecQualifiers,
)


def test_artifact_claim_serializes_sparse_spec_qualifiers() -> None:
    claim = ArtifactClaim(
        claim_id="rear_lower_arm_to_chassis_nut_torque",
        claim_type=ObservationClaimType.TORQUE_SPEC,
        trust_state=ArtifactTrustState.EXTRACTED,
        value=60,
        unit="Nm",
        claim_text="Nut fastening lower arm to chassis 60 Nm",
        component_terms=["lower control arm"],
        fastener_terms=["nut"],
        spec_qualifiers=SpecQualifiers(
            component="lower arm",
            fastener="nut",
            joint_location="chassis",
            position="rear lower arm",
            procedure_context="rear suspension reattachment",
            operation="tightening",
        ),
        provenance_observation_ids=["obs-1"],
        provenance_unit_ids=["unit-p554"],
        source_authority=SourceAuthority.AUTHORITATIVE_REFERENCE,
        extraction_confidence=0.9,
    )

    payload = claim.model_dump(mode="json")

    assert payload["spec_qualifiers"] == {
        "component": "lower arm",
        "fastener": "nut",
        "joint_location": "chassis",
        "position": "rear lower arm",
        "procedure_context": "rear suspension reattachment",
        "operation": "tightening",
    }


def test_observation_spec_qualifiers_default_to_none_for_existing_payloads() -> None:
    observation = Observation(
        observation_id="obs-1",
        extraction_run_id="run-1",
        source_id="source-ferrari-360-wsm",
        source_authority=SourceAuthority.AUTHORITATIVE_REFERENCE,
        unit_ids=["unit-p554"],
        claim_type=ObservationClaimType.TORQUE_SPEC,
        claim_text="Nut fastening lower arm to 60 Nm",
        value=60,
        unit="Nm",
        component_terms=["lower control arm"],
        fastener_terms=[],
        extraction_method="deterministic_torque_table_v1",
        extraction_confidence=0.85,
    )

    assert observation.spec_qualifiers is None
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_artifact_models.py::test_artifact_claim_serializes_sparse_spec_qualifiers tests/unit/test_artifact_models.py::test_observation_spec_qualifiers_default_to_none_for_existing_payloads -q
```

Expected: FAIL because `SpecQualifiers` and `spec_qualifiers` do not exist.

- [ ] **Step 3: Implement model changes**

In `apps/api/src/prescient_benchmark/artifacts/models.py`, add this model above `Observation`:

```python
class SpecQualifiers(BaseModel):
    model_config = ConfigDict(extra="forbid")

    component: str | None = None
    fastener: str | None = None
    joint_location: str | None = None
    position: str | None = None
    procedure_context: str | None = None
    operation: str | None = None
```

Add this field to both `Observation` and `ArtifactClaim`:

```python
    spec_qualifiers: SpecQualifiers | None = None
```

Place the field after `fastener_terms` in both models so related service-spec metadata stays grouped.

- [ ] **Step 4: Run tests to verify pass**

Run:

```bash
uv run python -m pytest tests/unit/test_artifact_models.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/artifacts/models.py tests/unit/test_artifact_models.py
git commit -m "feat(artifacts): add spec qualifiers to claims"
```

## Task 2: Reject Noisy Workshop Torque Labels

**Files:**
- Modify: `apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py`
- Test: `tests/unit/test_workshop_artifact_extraction.py`

- [ ] **Step 1: Write failing extraction tests**

Append these tests to `tests/unit/test_workshop_artifact_extraction.py`:

```python
def test_extract_torque_observations_rejects_one_letter_ocr_label() -> None:
    unit = SourceUnitRecord(
        unit_id="unit-source-ferrari-360-wsm-p610",
        source_id="source-ferrari-360-wsm",
        unit_kind="pdf_page",
        ordinal=610,
        heading="Pedal board tightening torques",
        text="I 20 Nm",
    )

    observations = extract_torque_observations(
        extraction_run_id="run-1",
        source=_source(),
        units=[unit],
        scope=default_360_scope(),
    )

    assert observations == []


def test_extract_torque_observations_rejects_bare_unit_label() -> None:
    unit = SourceUnitRecord(
        unit_id="unit-source-ferrari-360-wsm-p200",
        source_id="source-ferrari-360-wsm",
        unit_kind="pdf_page",
        ordinal=200,
        heading="Engine tightening torques",
        text="Nm 65 Nm",
    )

    observations = extract_torque_observations(
        extraction_run_id="run-1",
        source=_source(),
        units=[unit],
        scope=default_360_scope(),
    )

    assert observations == []
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_artifact_extraction.py::test_extract_torque_observations_rejects_one_letter_ocr_label tests/unit/test_workshop_artifact_extraction.py::test_extract_torque_observations_rejects_bare_unit_label -q
```

Expected: FAIL because current extraction emits observations for noisy labels.

- [ ] **Step 3: Add label validation helpers**

In `apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py`, add these constants below `_LOWER_CONTROL_ARM_ALIASES`:

```python
_NOISY_LABELS = {
    "-",
    "a",
    "de",
    "di",
    "f",
    "i",
    "l",
    "mn",
    "nm",
    "o",
    "of",
    "re",
    "the",
}

_GENERIC_INSTRUCTION_LABELS = (
    "tighten the",
    "tightening the",
    "screw in the screws",
    "accostare le viti",
    "serrare la vite",
)
```

Add this helper near `_component_terms`:

```python
def _is_valid_torque_label(label: str) -> bool:
    normalized = label.strip().lower()
    normalized = re.sub(r"[^a-z0-9 ]+", " ", normalized)
    normalized = " ".join(normalized.split())
    if len(normalized) < 3:
        return False
    if normalized in _NOISY_LABELS:
        return False
    if normalized.isdigit():
        return False
    if any(normalized.startswith(prefix) for prefix in _GENERIC_INSTRUCTION_LABELS):
        return False
    if normalized in {"n m", "nm"}:
        return False
    return True
```

- [ ] **Step 4: Use the validator before creating observations**

In `extract_torque_observations`, after:

```python
            label = " ".join(match.group("label").split())
```

add:

```python
            if not _is_valid_torque_label(label):
                continue
```

- [ ] **Step 5: Run tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_artifact_extraction.py -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py tests/unit/test_workshop_artifact_extraction.py
git commit -m "fix(artifacts): reject noisy workshop torque labels"
```

## Task 3: Extract and Assemble Qualifier-Aware Claims

**Files:**
- Modify: `apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py`
- Modify: `apps/api/src/prescient_benchmark/artifacts/workshop_assembly.py`
- Test: `tests/unit/test_workshop_artifact_extraction.py`

- [ ] **Step 1: Write failing qualifier tests**

Append these tests to `tests/unit/test_workshop_artifact_extraction.py`:

```python
def test_extract_torque_observation_sets_lower_arm_chassis_qualifiers() -> None:
    unit = SourceUnitRecord(
        unit_id="unit-source-ferrari-360-wsm-p554",
        source_id="source-ferrari-360-wsm",
        unit_kind="pdf_page",
        ordinal=554,
        heading="Rear suspension tightening torques",
        text="Nut fastening lower arm to chassis 60 Nm",
    )

    observations = extract_torque_observations(
        extraction_run_id="run-1",
        source=_source(),
        units=[unit],
        scope=default_360_scope(),
    )

    assert len(observations) == 1
    qualifiers = observations[0].spec_qualifiers
    assert qualifiers is not None
    assert qualifiers.component == "lower arm"
    assert qualifiers.fastener == "nut"
    assert qualifiers.joint_location == "chassis"
    assert qualifiers.position == "rear lower arm"
    assert qualifiers.procedure_context == "rear suspension"
    assert qualifiers.operation == "tightening"


def test_assemble_component_spec_artifact_preserves_qualifier_specific_claims() -> None:
    units = [
        SourceUnitRecord(
            unit_id="unit-source-ferrari-360-wsm-p518",
            source_id="source-ferrari-360-wsm",
            unit_kind="pdf_page",
            ordinal=518,
            heading="Front suspension tightening torques",
            text="Screws fastening lower arm to stub axle and hub-holder 85 Nm",
        ),
        SourceUnitRecord(
            unit_id="unit-source-ferrari-360-wsm-p554",
            source_id="source-ferrari-360-wsm",
            unit_kind="pdf_page",
            ordinal=554,
            heading="Rear suspension tightening torques",
            text="Nut fastening lower arm to chassis 60 Nm",
        ),
    ]
    observations = extract_torque_observations(
        extraction_run_id="run-1",
        source=_source(),
        units=units,
        scope=default_360_scope(),
    )

    artifacts = assemble_component_spec_artifacts(observations)

    lower_arm_artifacts = [
        artifact for artifact in artifacts if artifact.entity_ref == "component.lower_control_arm"
    ]
    assert len(lower_arm_artifacts) == 1
    claims = sorted(
        lower_arm_artifacts[0].claims,
        key=lambda claim: claim.spec_qualifiers.joint_location or "",
    )
    assert [claim.value for claim in claims] == [60, 85]
    assert {claim.spec_qualifiers.joint_location for claim in claims} == {
        "chassis",
        "stub axle and hub-holder",
    }
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_artifact_extraction.py::test_extract_torque_observation_sets_lower_arm_chassis_qualifiers tests/unit/test_workshop_artifact_extraction.py::test_assemble_component_spec_artifact_preserves_qualifier_specific_claims -q
```

Expected: FAIL because qualifiers are not extracted and assembly splits claims by value.

- [ ] **Step 3: Add qualifier extraction**

In `apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py`, import `SpecQualifiers`:

```python
from prescient_benchmark.artifacts.models import (
    ArtifactTrustState,
    Observation,
    ObservationClaimType,
    SourceAuthority,
    SpecQualifiers,
)
```

Add this helper:

```python
def _spec_qualifiers(label: str, unit: SourceUnitRecord) -> SpecQualifiers | None:
    lowered = label.lower()
    heading = (unit.heading or "").lower()
    component = "lower arm" if "lower arm" in lowered else None
    fastener = None
    if "nut" in lowered or "nuts" in lowered:
        fastener = "nut"
    elif "screw" in lowered or "screws" in lowered:
        fastener = "screw"

    joint_location = None
    if "chassis" in lowered:
        joint_location = "chassis"
    elif "stub axle" in lowered and "hub-holder" in lowered:
        joint_location = "stub axle and hub-holder"

    position = None
    if "rear" in heading and component == "lower arm":
        position = "rear lower arm"
    elif "front" in heading and component == "lower arm":
        position = "front lower arm"

    procedure_context = None
    if "rear suspension" in heading:
        procedure_context = "rear suspension"
    elif "front suspension" in heading:
        procedure_context = "front suspension"

    qualifiers = SpecQualifiers(
        component=component,
        fastener=fastener,
        joint_location=joint_location,
        position=position,
        procedure_context=procedure_context,
        operation="tightening",
    )
    if not any(value is not None for value in qualifiers.model_dump().values()):
        return None
    return qualifiers
```

Pass it into `Observation(...)`:

```python
                    spec_qualifiers=_spec_qualifiers(label, unit),
```

- [ ] **Step 4: Assemble one component artifact with multiple qualified claims**

In `apps/api/src/prescient_benchmark/artifacts/workshop_assembly.py`, change:

```python
        key = (scope_id, component, fastener, str(observation.value))
```

to:

```python
        key = (scope_id, component, fastener)
```

Add `spec_qualifiers=observation.spec_qualifiers` to `ArtifactClaim(...)`.

Change `claim_id` construction to include qualifiers:

```python
            claim_id=_claim_id(component, fastener, observation),
```

Add this helper:

```python
def _claim_id(component: str, fastener: str, observation: Observation) -> str:
    qualifiers = observation.spec_qualifiers
    parts = [component.removeprefix("component.")]
    if qualifiers is not None:
        for value in (
            qualifiers.position,
            qualifiers.joint_location,
            qualifiers.fastener,
            qualifiers.operation,
        ):
            if value:
                parts.append(value)
    parts.append(fastener)
    parts.append("torque")
    return _slug("_".join(parts))
```

- [ ] **Step 5: Run tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_artifact_extraction.py -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py apps/api/src/prescient_benchmark/artifacts/workshop_assembly.py tests/unit/test_workshop_artifact_extraction.py
git commit -m "feat(artifacts): preserve automotive spec qualifiers"
```

## Task 4: Make Artifact Scoring Token-Boundary and Qualifier-Aware

**Files:**
- Modify: `apps/api/src/prescient_benchmark/artifacts/postgres.py`
- Test: `tests/unit/test_artifact_postgres.py`

- [ ] **Step 1: Write failing scoring tests**

Append these tests to `tests/unit/test_artifact_postgres.py`:

Add `SpecQualifiers` to the model imports at the top of the file:

```python
from prescient_benchmark.artifacts.models import (
    Artifact,
    ArtifactClaim,
    ArtifactClaimFeedbackAction,
    ArtifactFeedbackTriageCategory,
    ArtifactFeedbackTriageEvent,
    ArtifactRegressionSeed,
    ArtifactTrustState,
    ArtifactType,
    ObservationClaimType,
    SourceAuthority,
    SpecQualifiers,
    ValidationEvent,
)
```

```python
def test_score_artifact_ignores_one_letter_substring_matches() -> None:
    artifact = Artifact(
        artifact_id="artifact-noisy-i",
        artifact_type=ArtifactType.COMPONENT_SPEC,
        entity_ref="component.i",
        scope={"scope_id": "scope-1"},
        claims=[
            ArtifactClaim(
                claim_id="i_to_fastener_torque",
                claim_type=ObservationClaimType.TORQUE_SPEC,
                trust_state=ArtifactTrustState.EXTRACTED,
                value=20,
                unit="Nm",
                claim_text="I 20 Nm",
                component_terms=["i"],
                fastener_terms=[],
                provenance_observation_ids=["obs-i"],
                provenance_unit_ids=["unit-i"],
                source_authority=SourceAuthority.AUTHORITATIVE_REFERENCE,
                extraction_confidence=0.7,
            )
        ],
    )

    assert postgres.score_artifact_for_question(artifact, "torque lower control arm") == 0


def test_score_artifact_rewards_qualified_lower_arm_match() -> None:
    artifact = _artifact()
    artifact.claims[0].claim_text = "Nut fastening lower arm to chassis 60 Nm"
    artifact.claims[0].spec_qualifiers = SpecQualifiers(
        component="lower arm",
        fastener="nut",
        joint_location="chassis",
        position="rear lower arm",
        operation="tightening",
    )

    assert postgres.score_artifact_for_question(
        artifact,
        "rear lower arm nut to chassis torque",
    ) > postgres.score_artifact_for_question(
        artifact,
        "torque",
    )
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_artifact_postgres.py::test_score_artifact_ignores_one_letter_substring_matches tests/unit/test_artifact_postgres.py::test_score_artifact_rewards_qualified_lower_arm_match -q
```

Expected: FAIL because current scoring uses substring matching and ignores qualifiers.

- [ ] **Step 3: Add token-boundary helpers**

In `apps/api/src/prescient_benchmark/artifacts/postgres.py`, import `re`:

```python
import re
```

Add these helpers above `score_artifact_for_question`:

```python
def _contains_term(text: str, term: str) -> bool:
    normalized_term = " ".join(term.lower().split())
    if len(normalized_term) < 3:
        return False
    return re.search(rf"\b{re.escape(normalized_term)}\b", text) is not None


def _qualifier_values(claim: ArtifactClaim) -> list[str]:
    if claim.spec_qualifiers is None:
        return []
    return [
        value
        for value in claim.spec_qualifiers.model_dump().values()
        if isinstance(value, str) and value.strip()
    ]
```

Update the existing import from models to include `ArtifactClaim`.

- [ ] **Step 4: Replace substring scoring**

Replace `score_artifact_for_question` with:

```python
def score_artifact_for_question(artifact: Artifact, lowered_question: str) -> int:
    question = " ".join(lowered_question.lower().split())
    score = 0
    for alias in artifact.aliases:
        if _contains_term(question, alias):
            score += 3
    for claim in artifact.claims:
        claim_score = 0
        for term in [*claim.component_terms, *claim.fastener_terms]:
            if _contains_term(question, term):
                claim_score += 2
        for qualifier in _qualifier_values(claim):
            if _contains_term(question, qualifier):
                claim_score += 2
        if "torque" in question and claim.claim_type.value == "torque_spec" and claim_score > 0:
            claim_score += 2
        score = max(score, claim_score)
    return score
```

- [ ] **Step 5: Run tests**

Run:

```bash
uv run python -m pytest tests/unit/test_artifact_postgres.py -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/artifacts/postgres.py tests/unit/test_artifact_postgres.py
git commit -m "fix(artifacts): harden artifact query scoring"
```

## Task 5: Return Multi-Claim or Clarification Answers

**Files:**
- Modify: `apps/api/src/prescient_benchmark/artifacts/workshop_answer.py`
- Test: `tests/unit/test_workshop_artifact_answer.py`

- [ ] **Step 1: Write failing answer tests**

Append these tests to `tests/unit/test_workshop_artifact_answer.py`:

```python
def test_artifact_answer_lists_multiple_lower_arm_claims_for_broad_query() -> None:
    artifact = _artifact()
    artifact.claims = [
        ArtifactClaim(
            claim_id="lower_arm_stub_axle_hub_holder_screw_torque",
            claim_type=ObservationClaimType.TORQUE_SPEC,
            trust_state=ArtifactTrustState.EXTRACTED,
            value=85,
            unit="Nm",
            claim_text="Screws fastening lower arm to stub axle and hub-holder 85 Nm",
            component_terms=["lower control arm"],
            fastener_terms=["screw"],
            spec_qualifiers=SpecQualifiers(
                component="lower arm",
                fastener="screw",
                joint_location="stub axle and hub-holder",
                position="front lower arm",
                operation="tightening",
            ),
            provenance_observation_ids=["obs-1"],
            provenance_unit_ids=["unit-source-ferrari-360-wsm-p250"],
            source_authority=SourceAuthority.AUTHORITATIVE_REFERENCE,
            extraction_confidence=0.9,
        ),
        ArtifactClaim(
            claim_id="rear_lower_arm_chassis_nut_torque",
            claim_type=ObservationClaimType.TORQUE_SPEC,
            trust_state=ArtifactTrustState.EXTRACTED,
            value=60,
            unit="Nm",
            claim_text="Nut fastening lower arm to chassis 60 Nm",
            component_terms=["lower control arm"],
            fastener_terms=["nut"],
            spec_qualifiers=SpecQualifiers(
                component="lower arm",
                fastener="nut",
                joint_location="chassis",
                position="rear lower arm",
                operation="tightening",
            ),
            provenance_observation_ids=["obs-2"],
            provenance_unit_ids=["unit-source-ferrari-360-wsm-p250"],
            source_authority=SourceAuthority.AUTHORITATIVE_REFERENCE,
            extraction_confidence=0.9,
        ),
    ]
    service = ArtifactAnswerService(
        artifact_repository=_ArtifactRepository(artifact),
        knowledge_store=_KnowledgeStore(),
        scope=default_360_scope(),
    )

    answer = service.answer_if_available(question="Torque lower control arm")

    assert answer is not None
    assert answer.status is AnswerStatus.NEEDS_CLARIFICATION
    assert "stub axle and hub-holder" in answer.text
    assert "chassis" in answer.text
    assert answer.artifact_claim_ref is None
    assert "artifact_multiple_claims" in answer.retrieval_trace


def test_artifact_answer_uses_specific_qualified_claim() -> None:
    artifact = _artifact()
    artifact.claims[0].claim_text = "Nut fastening lower arm to chassis 60 Nm"
    artifact.claims[0].value = 60
    artifact.claims[0].fastener_terms = ["nut"]
    artifact.claims[0].spec_qualifiers = SpecQualifiers(
        component="lower arm",
        fastener="nut",
        joint_location="chassis",
        position="rear lower arm",
        operation="tightening",
    )
    service = ArtifactAnswerService(
        artifact_repository=_ArtifactRepository(artifact),
        knowledge_store=_KnowledgeStore(),
        scope=default_360_scope(),
    )

    answer = service.answer_if_available(
        question="What is the rear lower arm nut to chassis torque?"
    )

    assert answer is not None
    assert answer.status is AnswerStatus.ANSWERED
    assert "60 Nm" in answer.text
    assert answer.artifact_claim_ref is not None
    assert answer.artifact_claim_ref.claim_id == "lower_control_arm_to_chassis_nuts_torque"
```

Add missing imports at the top of the test:

```python
from prescient_benchmark.artifacts.models import SpecQualifiers
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_artifact_answer.py::test_artifact_answer_lists_multiple_lower_arm_claims_for_broad_query tests/unit/test_workshop_artifact_answer.py::test_artifact_answer_uses_specific_qualified_claim -q
```

Expected: FAIL because current answer selection returns one claim.

- [ ] **Step 3: Add claim match outcome helpers**

In `apps/api/src/prescient_benchmark/artifacts/workshop_answer.py`, add:

```python
def _matching_claims(claims: list[ArtifactClaim], question: str) -> list[ArtifactClaim]:
    lowered_question = question.lower()
    scored = [
        (_score_claim(claim, lowered_question), claim)
        for claim in claims
        if claim.claim_type == "torque_spec"
    ]
    scored = [(score, claim) for score, claim in scored if score > 0]
    scored.sort(key=lambda item: item[0], reverse=True)
    return [claim for _, claim in scored]


def _has_specific_qualifier_match(claim: ArtifactClaim, question: str) -> bool:
    lowered_question = question.lower()
    if claim.spec_qualifiers is None:
        return False
    qualifier_values = [
        value
        for value in claim.spec_qualifiers.model_dump().values()
        if isinstance(value, str) and value
    ]
    return sum(1 for value in qualifier_values if value.lower() in lowered_question) >= 2
```

Replace `_best_claim_for_question` call flow in `answer_if_available` with:

```python
        matching_claims = _matching_claims(artifact.claims, question)
        if not matching_claims:
            return None
        if len(matching_claims) > 1 and not _has_specific_qualifier_match(
            matching_claims[0],
            question,
        ):
            return self._multiple_claims_answer(artifact=artifact, claims=matching_claims[:5])
        claim = matching_claims[0]
```

- [ ] **Step 4: Add multiple-claim answer builder**

Add this method to `ArtifactAnswerService`:

```python
    def _multiple_claims_answer(
        self,
        *,
        artifact,
        claims: list[ArtifactClaim],
    ) -> KnowledgeAnswer | None:
        located_units = []
        for claim in claims:
            for unit_id in claim.provenance_unit_ids:
                locator = self.knowledge_store.locator_for_unit(unit_id)
                if locator is not None and locator not in located_units:
                    located_units.append(locator)
        if not located_units:
            return None
        citations = [
            self._citation_for_locator(
                locator=locator,
                claim=claims[0],
                citation_index=index,
            )
            for index, locator in enumerate(located_units)
        ]
        claim_lines = "\n".join(f"- {_claim_sentence(claim)}" for claim in claims)
        return KnowledgeAnswer(
            answer_id=f"answer-{uuid4()}",
            status=AnswerStatus.NEEDS_CLARIFICATION,
            text=(
                "The artifact store has multiple matching extracted specs. "
                "Choose the exact fixing/location before using one value:\n"
                f"{claim_lines}"
            ),
            scope=self.scope,
            citations=citations,
            locators=located_units,
            clarification_question="Which fixing or location do you mean?",
            retrieval_trace=[
                "resolve_scope",
                "artifact_first",
                "artifact_multiple_claims",
            ],
            retrieval_diagnostics={
                "stale_hit_count": 0,
                "missing_locator_count": 0,
            },
            artifact_claim_ref=None,
        )
```

- [ ] **Step 5: Run tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_artifact_answer.py -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/artifacts/workshop_answer.py tests/unit/test_workshop_artifact_answer.py
git commit -m "feat(artifacts): clarify broad automotive spec matches"
```

## Task 6: Add Route-Level Regression Coverage

**Files:**
- Modify: `tests/integration/test_workshop_api.py`

- [ ] **Step 1: Write failing route tests**

Add tests near `test_ask_knowledge_route_prefers_artifact_when_requested`:

```python
def test_ask_knowledge_route_artifact_mode_clarifies_broad_lower_arm_query(
    tmp_path: Path,
    monkeypatch,
) -> None:
    _seed_api_store(tmp_path, monkeypatch)
    repository = _ArtifactRepository()
    artifact = repository.find_best_component_spec_artifact(
        question="lower control arm torque",
        scope_id="scope-ferrari-360-modena",
    )
    artifact.claims.append(
        ArtifactClaim(
            claim_id="lower_arm_to_stub_axle_hub_holder_screw_torque",
            claim_type=ObservationClaimType.TORQUE_SPEC,
            trust_state=ArtifactTrustState.EXTRACTED,
            value=85,
            unit="Nm",
            claim_text="Screws fastening lower arm to stub axle and hub-holder 85 Nm",
            component_terms=["lower control arm"],
            fastener_terms=["screw"],
            spec_qualifiers=SpecQualifiers(
                component="lower arm",
                fastener="screw",
                joint_location="stub axle and hub-holder",
                position="front lower arm",
                operation="tightening",
            ),
            provenance_observation_ids=["obs-2"],
            provenance_unit_ids=["unit-source-ferrari-360-wsm-p1"],
            source_authority=SourceAuthority.AUTHORITATIVE_REFERENCE,
            extraction_confidence=0.9,
        )
    )
    artifact.claims[0].spec_qualifiers = SpecQualifiers(
        component="lower arm",
        fastener="nut",
        joint_location="chassis",
        position="rear lower arm",
        operation="tightening",
    )
    repository.find_best_component_spec_artifact = lambda *, question, scope_id: artifact
    monkeypatch.setattr(routes_knowledge, "_artifact_repository_provider", lambda: repository)

    client = TestClient(app)
    response = client.post(
        "/knowledge/ask/sync",
        json={
            "question": "Torque lower control arm",
            "candidate_unit_ids": [],
            "artifact_mode": "prefer",
        },
    )

    assert response.status_code == 200
    payload = response.json()
    assert payload["status"] == "needs_clarification"
    assert "artifact_multiple_claims" in payload["retrieval_trace"]
    assert payload["artifact_claim_ref"] is None
```

Add `SpecQualifiers` to the artifact model imports in this file.

- [ ] **Step 2: Run test to verify failure**

Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_route_artifact_mode_clarifies_broad_lower_arm_query -q
```

Expected: FAIL until Task 5 is complete.

- [ ] **Step 3: Run route test after Task 5**

Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_route_prefers_artifact_when_requested tests/integration/test_workshop_api.py::test_ask_knowledge_route_artifact_mode_clarifies_broad_lower_arm_query tests/integration/test_workshop_api.py::test_ask_knowledge_route_falls_back_when_preferred_artifact_repository_unavailable -q
```

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add tests/integration/test_workshop_api.py
git commit -m "test(artifacts): cover broad automotive artifact queries"
```

## Task 7: Add Eval Fixture for Artifact vs Retrieval Comparison

**Files:**
- Modify: `eval/questions/workshop_manuals_v1.yaml`
- Test: existing eval command

- [ ] **Step 1: Add lower-arm ambiguity eval case**

Add a new question to `eval/questions/workshop_manuals_v1.yaml` with these required claims:

```yaml
- id: workshop_lower_arm_torque_ambiguous
  category: torque
  prompt: Torque lower control arm
  scope_id: scope-ferrari-360-modena
  required_unit_ids:
    - unit-source-ferrari-360-wsm-p518
    - unit-source-ferrari-360-wsm-p554
  required_source_ids:
    - source-ferrari-360-wsm
  required_claims:
    - claim_id: lower_arm_torque_not_generic
      statement: The answer must not collapse all lower-arm torques to one generic value.
      required_unit_ids:
        - unit-source-ferrari-360-wsm-p518
        - unit-source-ferrari-360-wsm-p554
    - claim_id: lower_arm_torque_location_distinction
      statement: The answer should distinguish lower arm to stub axle and hub-holder from lower arm to chassis fixings.
      required_unit_ids:
        - unit-source-ferrari-360-wsm-p518
        - unit-source-ferrari-360-wsm-p554
```

- [ ] **Step 2: Run narrow eval schema/test check**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_eval.py tests/integration/test_workshop_api.py::test_ask_knowledge_route_artifact_mode_clarifies_broad_lower_arm_query -q
```

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add eval/questions/workshop_manuals_v1.yaml
git commit -m "test(eval): add lower arm artifact ambiguity case"
```

## Task 8: Full Verification

**Files:**
- No new files.

- [ ] **Step 1: Run targeted unit tests**

Run:

```bash
uv run python -m pytest \
  tests/unit/test_artifact_models.py \
  tests/unit/test_workshop_artifact_extraction.py \
  tests/unit/test_artifact_postgres.py \
  tests/unit/test_workshop_artifact_answer.py \
  -q
```

Expected: PASS.

- [ ] **Step 2: Run route tests**

Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py -q
```

Expected: PASS.

- [ ] **Step 3: Rebuild local artifact extraction for manual verification**

With Docker services running, run:

```bash
docker compose exec -T -e PYTHONPATH=/workspace/apps/api/src api \
  uv run python -m prescient_benchmark.cli artifact-schema-init \
  --database-url postgresql://prescient:prescient@postgres:5432/prescient

docker compose exec -T -e PYTHONPATH=/workspace/apps/api/src api \
  uv run python -m prescient_benchmark.cli extract-workshop-artifacts \
  --data-root /workspace/corpus/workshop_manuals \
  --scope-id scope-ferrari-360-modena \
  --database-url postgresql://prescient:prescient@postgres:5432/prescient
```

Expected: no one-letter current artifacts should be produced for labels such as `I 20 Nm`, and lower-arm torque claims should carry qualifiers.

- [ ] **Step 4: Manually compare API behavior**

Run:

```bash
curl -sS http://localhost:8000/knowledge/ask/sync \
  -H 'Content-Type: application/json' \
  --data-raw '{"question":"Torque lower control arm","candidate_unit_ids":[],"artifact_mode":"prefer"}' \
  | jq '{status,text,retrieval_trace,artifact_claim_ref}'
```

Expected: `status` is `needs_clarification`, `retrieval_trace` includes `artifact_multiple_claims`, and `artifact_claim_ref` is `null`.

Run:

```bash
curl -sS http://localhost:8000/knowledge/ask/sync \
  -H 'Content-Type: application/json' \
  --data-raw '{"question":"What is the rear lower arm nut to chassis torque?","candidate_unit_ids":[],"artifact_mode":"prefer"}' \
  | jq '{status,text,retrieval_trace,artifact_claim_ref}'
```

Expected: `status` is `answered`, text includes `60 Nm`, `retrieval_trace` includes `artifact_claim_answer`, and `artifact_claim_ref` is present.

- [ ] **Step 5: Commit any final fixes**

If verification required small fixes, commit them:

```bash
git add apps/api/src/prescient_benchmark tests eval/questions/workshop_manuals_v1.yaml
git commit -m "fix(artifacts): verify automotive artifact specificity"
```

Run `git status --short` after verification. If it prints no changed files, there is no final verification-fix commit to make.

## Risks And Follow-Ups

- The plan intentionally handles torque/spec artifacts first. Belt frequencies, fluid capacities, and pinouts should reuse the qualifier model but need separate extraction rules.
- The Postgres schema stores JSONB payloads, so adding optional qualifier fields does not require a table migration. If future queries need SQL filtering by qualifier, add generated columns or expression indexes in a later plan.
- Multi-claim answers are a safe intermediate. A later UI plan can render them as selectable structured rows rather than prose.

## Self-Review

- Spec coverage: The plan implements qualifier preservation, noisy label rejection, token-boundary matching, multi-claim/clarification behavior, retrieval fallback preservation, and comparison eval coverage.
- Placeholder scan: No TODO/TBD placeholders remain, and eval YAML uses the existing question-set shape.
- Type consistency: `SpecQualifiers`, `spec_qualifiers`, `ArtifactClaimRef`, `KnowledgeAnswer`, and `AnswerStatus.NEEDS_CLARIFICATION` names match current project patterns or are defined in Task 1.
