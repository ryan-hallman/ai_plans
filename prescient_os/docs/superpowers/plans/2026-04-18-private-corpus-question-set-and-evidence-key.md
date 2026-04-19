# Private Corpus Question Set And Evidence Key Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add validated private-corpus benchmark assets by extending the eval schema for private questions and evidence keys, then author the first five PrescientOS benchmark questions against the current private snapshot.

**Architecture:** Keep this slice asset-first. Extend `prescient_benchmark.eval` with typed question and evidence-key models plus storage helpers, then add a new private question-set YAML and matching evidence-key YAML under `eval/`. Do not change retrieval scoring, answer generation, or run orchestration yet.

**Tech Stack:** Python 3.13, Pydantic v2, PyYAML, pytest, repository YAML assets under `eval/`

---

## File Structure

### Product repo: `/home/rhallman/Projects/prescient_os`

- Modify: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/eval/models.py`
  - extend `EvalQuestion`; add typed evidence-key models and sets
- Modify: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/eval/storage.py`
  - add evidence-key loading and writing helpers
- Create: `/home/rhallman/Projects/prescient_os/eval/questions/prescient_private_v1.yaml`
  - first five private benchmark questions
- Create: `/home/rhallman/Projects/prescient_os/eval/evidence_keys/prescient_private_v1.yaml`
  - matching evidence keys for those five questions
- Modify: `/home/rhallman/Projects/prescient_os/tests/unit/test_eval_models.py`
  - validate richer question metadata and evidence-key models
- Modify: `/home/rhallman/Projects/prescient_os/tests/unit/test_eval_storage.py`
  - validate loading of the new private question set and evidence-key asset

This slice intentionally does **not** modify:

- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/eval/orchestrator.py`
- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/eval/summary.py`
- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/retrieval/*`

### Task 1: Extend eval models for private questions and evidence keys

**Files:**
- Modify: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/eval/models.py`
- Modify: `/home/rhallman/Projects/prescient_os/tests/unit/test_eval_models.py`

- [ ] **Step 1: Write the failing model tests**

```python
import pytest
from pydantic import ValidationError

from prescient_benchmark.eval.models import (
    EvidenceKeyQuestion,
    EvidenceKeySet,
    EvalQuestion,
)


def test_eval_question_accepts_private_question_metadata() -> None:
    question = EvalQuestion.model_validate(
        {
            "id": "q_private_001",
            "category": "chronology",
            "question_type": "current_state_at_time",
            "source_mix": "memo_led",
            "prompt": "What operating wedge came out of the April 10 meeting?",
            "expected_docs": ["ai-enablement-strategy-pe-portfolios-2026-04-10"],
            "scoring_notes": "Answer should cite the origin summary memo.",
        }
    )

    assert question.question_type == "current_state_at_time"
    assert question.source_mix == "memo_led"


def test_eval_question_rejects_unknown_source_mix() -> None:
    with pytest.raises(ValidationError, match="source_mix"):
        EvalQuestion.model_validate(
            {
                "id": "q_private_001",
                "category": "chronology",
                "question_type": "current_state_at_time",
                "source_mix": "memo_only_forever",
                "prompt": "bad",
                "expected_docs": ["doc_1"],
            }
        )


def test_evidence_key_question_requires_minimum_claim_ids_to_exist() -> None:
    with pytest.raises(ValidationError, match="minimum_sufficient_set"):
        EvidenceKeyQuestion.model_validate(
            {
                "question_id": "q_private_001",
                "question_type": "current_state_at_time",
                "prompt": "What operating wedge came out of the April 10 meeting?",
                "corpus_version": "prescient-os-chats-2026-04-18-with-retrieval-only-turn",
                "required_docs": ["ai-enablement-strategy-pe-portfolios-2026-04-10"],
                "required_sources": ["ai-enablement-strategy-pe-portfolios-2026-04-10"],
                "required_claims": [
                    {
                        "claim_id": "claim_wedge",
                        "statement": "Board-deck/reporting pain became the initial wedge.",
                        "acceptable_evidence": [
                            {
                                "doc_id": "ai-enablement-strategy-pe-portfolios-2026-04-10",
                                "locator": "Specific Ideas",
                                "excerpt": "The end of the quarterly board deck",
                            }
                        ],
                    }
                ],
                "resolution_rule": "latest_wins",
                "minimum_sufficient_set": ["claim_missing"],
            }
        )


def test_evidence_key_set_accepts_private_question_entry() -> None:
    evidence_keys = EvidenceKeySet.model_validate(
        {
            "question_set_version": "prescient_private_v1",
            "questions": [
                {
                    "question_id": "q_private_001",
                    "question_type": "current_state_at_time",
                    "prompt": "What operating wedge came out of the April 10 meeting?",
                    "corpus_version": "prescient-os-chats-2026-04-18-with-retrieval-only-turn",
                    "required_docs": ["ai-enablement-strategy-pe-portfolios-2026-04-10"],
                    "required_sources": ["ai-enablement-strategy-pe-portfolios-2026-04-10"],
                    "required_claims": [
                        {
                            "claim_id": "claim_wedge",
                            "statement": "Board-deck/reporting pain became the initial wedge.",
                            "acceptable_evidence": [
                                {
                                    "doc_id": "ai-enablement-strategy-pe-portfolios-2026-04-10",
                                    "locator": "Specific Ideas",
                                    "excerpt": "The end of the quarterly board deck",
                                }
                            ],
                        }
                    ],
                    "resolution_rule": "latest_wins",
                    "minimum_sufficient_set": ["claim_wedge"],
                }
            ],
        }
    )

    assert evidence_keys.questions[0].required_claims[0].claim_id == "claim_wedge"
```

- [ ] **Step 2: Run the model tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_models.py -q
```

Expected: FAIL with import or validation errors because `question_type`,
`source_mix`, and the evidence-key models do not exist yet.

- [ ] **Step 3: Add the minimal model implementation**

```python
from typing import Literal

from pydantic import ValidationInfo, field_validator


class EvalQuestion(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    id: StrictStr
    category: StrictStr
    question_type: Literal[
        "current_state_at_time",
        "change_over_time",
        "why_pivoted",
        "scope_disambiguation",
        "superseded_belief",
    ]
    source_mix: Literal["memo_led", "mixed", "raw_chat_led"]
    prompt: StrictStr
    expected_docs: list[StrictStr]
    scoring_notes: StrictStr | None = None


class AcceptableEvidence(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    doc_id: StrictStr
    locator: StrictStr
    excerpt: StrictStr


class RequiredClaim(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    claim_id: StrictStr
    statement: StrictStr
    acceptable_evidence: tuple[AcceptableEvidence, ...]


class EvidenceKeyQuestion(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    question_id: StrictStr
    question_type: Literal[
        "current_state_at_time",
        "change_over_time",
        "why_pivoted",
        "scope_disambiguation",
        "superseded_belief",
    ]
    prompt: StrictStr
    corpus_version: StrictStr
    required_docs: list[StrictStr]
    required_sources: list[StrictStr]
    required_claims: tuple[RequiredClaim, ...]
    resolution_rule: Literal[
        "latest_wins",
        "must_compare_periods",
        "must_surface_conflict",
        "memo_plus_raw",
        "raw_only",
    ]
    minimum_sufficient_set: list[StrictStr]
    notes_on_conflicts: StrictStr | None = None

    @field_validator("minimum_sufficient_set")
    @classmethod
    def minimum_sufficient_set_must_reference_claim_ids(cls, value: list[str], info: ValidationInfo) -> list[str]:
        claim_ids = {claim.claim_id for claim in info.data.get("required_claims", ())}
        missing = [claim_id for claim_id in value if claim_id not in claim_ids]
        if missing:
            raise ValueError(f"minimum_sufficient_set must reference declared claim_ids: {', '.join(missing)}")
        return value


class EvidenceKeySet(BaseModel):
    model_config = ConfigDict(extra="forbid")

    question_set_version: StrictStr
    questions: tuple[EvidenceKeyQuestion, ...]
```

- [ ] **Step 4: Run the model tests to verify they pass**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_models.py -q
```

Expected: PASS

- [ ] **Step 5: Commit the model changes**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/eval/models.py tests/unit/test_eval_models.py
git commit -m "feat: add private eval question and evidence key models"
```

### Task 2: Add evidence-key storage support

**Files:**
- Modify: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/eval/storage.py`
- Modify: `/home/rhallman/Projects/prescient_os/tests/unit/test_eval_storage.py`

- [ ] **Step 1: Write the failing storage tests**

```python
from prescient_benchmark.eval.storage import load_evidence_key_set


def test_load_evidence_key_set_reads_private_asset() -> None:
    asset_path = Path(__file__).resolve().parents[2] / "eval/evidence_keys/prescient_private_v1.yaml"

    evidence_keys = load_evidence_key_set(asset_path)

    assert evidence_keys.question_set_version == "prescient_private_v1"
    assert len(evidence_keys.questions) == 5
    assert evidence_keys.questions[0].question_id == "q_private_001"


def test_load_evidence_key_set_rejects_extra_top_level_keys(tmp_path: Path) -> None:
    path = tmp_path / "evidence_keys.yaml"
    path.write_text(
        yaml.safe_dump(
            {
                "question_set_version": "prescient_private_v1",
                "questions": [],
                "unexpected": "extra",
            },
            sort_keys=False,
        ),
        encoding="utf-8",
    )

    with pytest.raises(ValidationError, match="Extra inputs are not permitted"):
        load_evidence_key_set(path)
```

- [ ] **Step 2: Run the storage tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_storage.py -q
```

Expected: FAIL because `EvidenceKeySet` and `load_evidence_key_set()` are not
yet available and the private asset does not exist yet.

- [ ] **Step 3: Add the minimal storage implementation**

```python
from prescient_benchmark.eval.models import EvidenceKeySet, EvalQuestion, GraderOutput


def load_evidence_key_set(path: Path) -> EvidenceKeySet:
    payload = yaml.safe_load(path.read_text(encoding="utf-8"))
    return EvidenceKeySet.model_validate(payload)
```

```python
from prescient_benchmark.eval.storage import (
    EvidenceKeySet,
    QuestionSet,
    load_evidence_key_set,
    load_question_set,
    read_yaml_model,
    write_grader_schema,
    write_yaml_model,
)
```

- [ ] **Step 4: Run the storage tests again**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_storage.py -q
```

Expected: still FAIL because the `prescient_private_v1` asset files do not yet
exist, but import-time errors should be gone.

- [ ] **Step 5: Commit the storage changes**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/eval/storage.py tests/unit/test_eval_storage.py
git commit -m "feat: add evidence key storage support"
```

### Task 3: Author the first five private benchmark questions

**Files:**
- Create: `/home/rhallman/Projects/prescient_os/eval/questions/prescient_private_v1.yaml`
- Modify: `/home/rhallman/Projects/prescient_os/tests/unit/test_eval_storage.py`

- [ ] **Step 1: Create the private question-set asset**

```yaml
question_set_version: prescient_private_v1
questions:
  - id: q_private_001
    category: chronology
    question_type: current_state_at_time
    source_mix: memo_led
    prompt: What operating wedge came out of the April 10 TCW and BCG meeting, and how was board-deck pain positioned?
    expected_docs: [ai-enablement-strategy-pe-portfolios-2026-04-10]
    scoring_notes: Answer should identify reporting and communication pain as the wedge and connect it to board-deck preparation.
  - id: q_private_002
    category: pivot_reasoning
    question_type: why_pivoted
    source_mix: mixed
    prompt: Why did Prescient OS redesign around an operator-first product on April 12, and what trust constraint came with that redesign?
    expected_docs: [prescient-os-operator-first-redesign-2026-04-12, 5d270129-0515-458d-b9cc-a2515ffab359]
    scoring_notes: Answer should explain both the managing-up pain and the anti-surveillance trust principle.
  - id: q_private_003
    category: chronology
    question_type: change_over_time
    source_mix: mixed
    prompt: How did the product evolve from the April 12 operator-first redesign to the April 14 demo-enrichment phase?
    expected_docs: [prescient-os-operator-first-redesign-2026-04-12, operator-demo-enrichment-2026-04-14, d7029891-96fe-4835-b71c-1972f9dad8df]
    scoring_notes: Answer should show the shift from product thesis to richer operating realism and strategic-initiative depth.
  - id: q_private_004
    category: pivot_reasoning
    question_type: superseded_belief
    source_mix: mixed
    prompt: Why was the operator-first workflow-heavy build replaced by a KE-first pivot on April 16?
    expected_docs: [ke-first-pivot-2026-04-16, 761e7568-136e-4f4f-9b49-a3c78f9889ad]
    scoring_notes: Answer should make clear that workflow breadth was deprioritized because the core knowledge-engine risk was unresolved.
  - id: q_private_005
    category: evaluation_method
    question_type: scope_disambiguation
    source_mix: raw_chat_led
    prompt: When the benchmark turned retrieval-only on April 18, how was ground truth defined?
    expected_docs: [019d9319-2511-7281-b136-977181ef84b1, retrieval-only-evaluation-turn-2026-04-18]
    scoring_notes: Answer should define ground truth as authored question truth over a fixed corpus version, not automatically discovered truth.
```

- [ ] **Step 2: Extend the storage test to assert the new question asset loads**

```python
def test_load_question_set_reads_private_question_set_asset() -> None:
    asset_path = Path(__file__).resolve().parents[2] / "eval/questions/prescient_private_v1.yaml"

    question_set = load_question_set(asset_path)

    assert question_set.question_set_version == "prescient_private_v1"
    assert len(question_set.questions) == 5
    assert question_set.questions[0].id == "q_private_001"
    assert question_set.questions[0].source_mix == "memo_led"
    assert question_set.questions[-1].question_type == "scope_disambiguation"
```

- [ ] **Step 3: Run the storage tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_storage.py -q
```

Expected: FAIL because the evidence-key asset is still missing.

- [ ] **Step 4: Commit the private question-set asset**

```bash
cd /home/rhallman/Projects/prescient_os
git add eval/questions/prescient_private_v1.yaml tests/unit/test_eval_storage.py
git commit -m "data: add first private benchmark question set"
```

### Task 4: Author the first five private evidence keys

**Files:**
- Create: `/home/rhallman/Projects/prescient_os/eval/evidence_keys/prescient_private_v1.yaml`
- Modify: `/home/rhallman/Projects/prescient_os/tests/unit/test_eval_storage.py`

- [ ] **Step 1: Create the private evidence-key asset**

```yaml
question_set_version: prescient_private_v1
questions:
  - question_id: q_private_001
    question_type: current_state_at_time
    prompt: What operating wedge came out of the April 10 TCW and BCG meeting, and how was board-deck pain positioned?
    corpus_version: prescient-os-chats-2026-04-18-with-retrieval-only-turn
    required_docs: [ai-enablement-strategy-pe-portfolios-2026-04-10]
    required_sources: [ai-enablement-strategy-pe-portfolios-2026-04-10]
    required_claims:
      - claim_id: wedge_reporting_pain
        statement: Reporting and communication pain became the initial wedge.
        acceptable_evidence:
          - doc_id: ai-enablement-strategy-pe-portfolios-2026-04-10
            locator: "### Executive Summary"
            excerpt: "the end of the quarterly board deck"
      - claim_id: board_prep_reframed
        statement: Quarterly board reporting was reframed as obsolete and strategically weak.
        acceptable_evidence:
          - doc_id: ai-enablement-strategy-pe-portfolios-2026-04-10
            locator: "### Specific Ideas"
            excerpt: "Why the quarterly board deck is dead"
    resolution_rule: latest_wins
    minimum_sufficient_set: [wedge_reporting_pain, board_prep_reframed]
  - question_id: q_private_002
    question_type: why_pivoted
    prompt: Why did Prescient OS redesign around an operator-first product on April 12, and what trust constraint came with that redesign?
    corpus_version: prescient-os-chats-2026-04-18-with-retrieval-only-turn
    required_docs: [prescient-os-operator-first-redesign-2026-04-12, 5d270129-0515-458d-b9cc-a2515ffab359]
    required_sources: [prescient-os-operator-first-redesign-2026-04-12, 5d270129-0515-458d-b9cc-a2515ffab359]
    required_claims:
      - claim_id: managing_up_pain
        statement: The redesign was driven by operator time lost to managing up and board prep.
        acceptable_evidence:
          - doc_id: prescient-os-operator-first-redesign-2026-04-12
            locator: "### Why This Shift Happened"
            excerpt: "amount of time operators spent managing up"
          - doc_id: 5d270129-0515-458d-b9cc-a2515ffab359
            locator: "messages: user/assistant reasoning around morning touchpoint"
            excerpt: "board prep was a multi-week quarterly lift"
      - claim_id: anti_surveillance_trust
        statement: The system could not feel like PE surveillance software.
        acceptable_evidence:
          - doc_id: prescient-os-operator-first-redesign-2026-04-12
            locator: "### Trust Principle"
            excerpt: "could not feel like PE surveillance software"
    resolution_rule: memo_plus_raw
    minimum_sufficient_set: [managing_up_pain, anti_surveillance_trust]
  - question_id: q_private_003
    question_type: change_over_time
    prompt: How did the product evolve from the April 12 operator-first redesign to the April 14 demo-enrichment phase?
    corpus_version: prescient-os-chats-2026-04-18-with-retrieval-only-turn
    required_docs: [prescient-os-operator-first-redesign-2026-04-12, operator-demo-enrichment-2026-04-14, d7029891-96fe-4835-b71c-1972f9dad8df]
    required_sources: [prescient-os-operator-first-redesign-2026-04-12, operator-demo-enrichment-2026-04-14, d7029891-96fe-4835-b71c-1972f9dad8df]
    required_claims:
      - claim_id: operator_first_baseline
        statement: April 12 established the operator-first daily-work thesis.
        acceptable_evidence:
          - doc_id: prescient-os-operator-first-redesign-2026-04-12
            locator: "### Product Direction At This Point"
            excerpt: "first touchpoint each morning"
      - claim_id: realism_push
        statement: April 14 shifted from thesis definition to richer demo realism and strategic initiative depth.
        acceptable_evidence:
          - doc_id: operator-demo-enrichment-2026-04-14
            locator: "### What Got Added To Make It Real"
            excerpt: "Project Pulse and Peloton IQ Next"
          - doc_id: d7029891-96fe-4835-b71c-1972f9dad8df
            locator: "messages around making the seed data richer"
            excerpt: "richer and more fullsome"
    resolution_rule: must_compare_periods
    minimum_sufficient_set: [operator_first_baseline, realism_push]
  - question_id: q_private_004
    question_type: superseded_belief
    prompt: Why was the operator-first workflow-heavy build replaced by a KE-first pivot on April 16?
    corpus_version: prescient-os-chats-2026-04-18-with-retrieval-only-turn
    required_docs: [ke-first-pivot-2026-04-16, 761e7568-136e-4f4f-9b49-a3c78f9889ad]
    required_sources: [ke-first-pivot-2026-04-16, 761e7568-136e-4f4f-9b49-a3c78f9889ad]
    required_claims:
      - claim_id: shallow_surface_area
        statement: Too much shallow workflow surface area was accumulating.
        acceptable_evidence:
          - doc_id: ke-first-pivot-2026-04-16
            locator: "### Why This Shift Happened"
            excerpt: "building too many features that were shallow and wide"
      - claim_id: unresolved_ke_risk
        statement: The unresolved ingestion and retrieval problem became the higher-leverage risk.
        acceptable_evidence:
          - doc_id: ke-first-pivot-2026-04-16
            locator: "### Why This Shift Happened"
            excerpt: "knowledge curation, storage, update semantics, provenance, and retrieval"
          - doc_id: 761e7568-136e-4f4f-9b49-a3c78f9889ad
            locator: "messages around greenfield reset"
            excerpt: "core engine risk was still unresolved"
    resolution_rule: memo_plus_raw
    minimum_sufficient_set: [shallow_surface_area, unresolved_ke_risk]
  - question_id: q_private_005
    question_type: scope_disambiguation
    prompt: When the benchmark turned retrieval-only on April 18, how was ground truth defined?
    corpus_version: prescient-os-chats-2026-04-18-with-retrieval-only-turn
    required_docs: [019d9319-2511-7281-b136-977181ef84b1, retrieval-only-evaluation-turn-2026-04-18]
    required_sources: [019d9319-2511-7281-b136-977181ef84b1]
    required_claims:
      - claim_id: authored_question_truth
        statement: Ground truth was defined as human-authored question truth over a fixed corpus version.
        acceptable_evidence:
          - doc_id: 019d9319-2511-7281-b136-977181ef84b1
            locator: "messages:1727-1728"
            excerpt: "human-defined correct interpretation of a fixed corpus version"
      - claim_id: explicit_resolution_rule
        statement: Each question needed an explicit interpretation and conflict-resolution rule.
        acceptable_evidence:
          - doc_id: 019d9319-2511-7281-b136-977181ef84b1
            locator: "messages:1728"
            excerpt: "latest authoritative source wins"
    resolution_rule: raw_only
    minimum_sufficient_set: [authored_question_truth, explicit_resolution_rule]
```

- [ ] **Step 2: Add the storage test for the evidence-key asset**

```python
def test_load_evidence_key_set_reads_private_evidence_key_asset() -> None:
    asset_path = Path(__file__).resolve().parents[2] / "eval/evidence_keys/prescient_private_v1.yaml"

    evidence_keys = load_evidence_key_set(asset_path)

    assert evidence_keys.question_set_version == "prescient_private_v1"
    assert len(evidence_keys.questions) == 5
    assert evidence_keys.questions[0].question_id == "q_private_001"
    assert evidence_keys.questions[-1].resolution_rule == "raw_only"
```

- [ ] **Step 3: Run the model and storage tests together**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_models.py tests/unit/test_eval_storage.py -q
```

Expected: PASS

- [ ] **Step 4: Commit the evidence-key asset**

```bash
cd /home/rhallman/Projects/prescient_os
git add eval/evidence_keys/prescient_private_v1.yaml tests/unit/test_eval_storage.py
git commit -m "data: add first private benchmark evidence keys"
```

### Task 5: Run the focused verification pass

**Files:**
- Modify: `/home/rhallman/Projects/prescient_os/tests/unit/test_eval_models.py`
- Modify: `/home/rhallman/Projects/prescient_os/tests/unit/test_eval_storage.py`

- [ ] **Step 1: Run the focused eval test suite**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_models.py tests/unit/test_eval_storage.py -q
```

Expected: PASS

- [ ] **Step 2: Run the broader unit and integration suite**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit tests/integration -q
```

Expected: PASS with no regressions outside the eval asset layer

- [ ] **Step 3: Check for whitespace and merge-artifact issues**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
git diff --check
```

Expected: no output

- [ ] **Step 4: Commit the final verification-state cleanup if needed**

```bash
cd /home/rhallman/Projects/prescient_os
git status --short
```

Expected: clean working tree or only the intended files already committed in prior tasks
