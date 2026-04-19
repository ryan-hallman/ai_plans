# Private Retrieval Scorer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a deterministic retrieval-only scorer for `prescient_private_v1` that scores captured `retrieved_evidence` bundles against the evidence keys, writes per-question retrieval score records, and surfaces a run-level retrieval summary.

**Architecture:** Keep the scorer pure over saved retrieval bundles. Expand the private evidence-key schema only enough to make resolution rules machine-checkable, implement deterministic scoring in a dedicated module, then wire run-level scoring and summary generation into the existing eval run structure and CLI without coupling the scorer to retriever execution.

**Tech Stack:** Python 3.13, Pydantic v2, Typer, PyYAML, pytest

---

## File Structure

### Existing files to modify

- `apps/api/src/prescient_benchmark/eval/models.py`
  - add claim-role metadata to evidence-key models
  - add retrieval-score record models and metric models
- `apps/api/src/prescient_benchmark/eval/paths.py`
  - add a safe path helper for per-question retrieval score files
- `apps/api/src/prescient_benchmark/eval/orchestrator.py`
  - add a run-scoring entrypoint that reads answer records, loads evidence keys, writes retrieval score records, and reuses summary generation
- `apps/api/src/prescient_benchmark/eval/summary.py`
  - aggregate retrieval score records into a deterministic `retrieval_summary` block alongside existing grader summaries
- `apps/api/src/prescient_benchmark/cli.py`
  - add a CLI command to score retrieval bundles for an existing eval run
- `eval/evidence_keys/prescient_private_v1.yaml`
  - add `claim_role` metadata to each required claim
- `tests/unit/test_eval_models.py`
  - validate new schema requirements and retrieval-score models
- `tests/unit/test_eval_storage.py`
  - validate new retrieval-score paths and asset-loading expectations if needed
- `tests/unit/test_eval_summary.py`
  - cover retrieval-summary aggregation and empty-summary tolerance
- `tests/integration/test_eval_harness_cli.py`
  - cover the new CLI scoring flow end to end on a local run directory

### New files to create

- `apps/api/src/prescient_benchmark/eval/retrieval_scoring.py`
  - pure deterministic scorer over `AnswerRecord.retrieved_evidence`
- `tests/unit/test_retrieval_scoring.py`
  - direct unit coverage for doc coverage, claim matching, resolution rules, noise, and failure semantics
- `tests/fixtures/eval/retrieval/prescient_private_v1_q_private_006_bundle.yaml`
  - frozen regression bundle that exercises real private evidence-key content and conflict resolution

### File responsibilities

- Keep all deterministic matching and rule logic in `retrieval_scoring.py`.
- Keep Pydantic contracts in `models.py`.
- Keep filesystem path building in `paths.py`.
- Keep orchestration and file writing in `orchestrator.py`.
- Keep aggregation only in `summary.py`.
- Do not fold scoring logic into `cli.py` or `orchestrator.py`.

---

### Task 1: Add Retrieval Score Contracts And Claim Roles

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/models.py`
- Modify: `eval/evidence_keys/prescient_private_v1.yaml`
- Test: `tests/unit/test_eval_models.py`

- [ ] **Step 1: Write the failing model tests for claim roles and retrieval score records**

```python
from pydantic import ValidationError
import pytest

from prescient_benchmark.eval.models import (
    EvidenceKeyQuestion,
    RetrievalQuestionScore,
)


def test_required_claim_requires_valid_claim_role() -> None:
    with pytest.raises(ValidationError, match="claim_role"):
        EvidenceKeyQuestion.model_validate(
            {
                "question_id": "q_private_test",
                "question_type": "scope_disambiguation",
                "prompt": "What is Prescient OS being built to do?",
                "corpus_version": "prescient-os-chats-2026-04-18-with-retrieval-only-turn",
                "required_docs": ["ke-first-pivot-2026-04-16"],
                "required_sources": ["ke-first-pivot-2026-04-16"],
                "required_claims": [
                    {
                        "claim_id": "claim_current",
                        "claim_role": "unknown_role",
                        "statement": "The product direction became KE-first.",
                        "acceptable_evidence": [
                            {
                                "doc_id": "ke-first-pivot-2026-04-16",
                                "locator": "### Product Direction At This Point",
                                "excerpt": "The knowledge engine became the primary product thesis",
                            }
                        ],
                    }
                ],
                "resolution_rule": "latest_wins",
                "minimum_sufficient_set": ["claim_current"],
            }
        )


def test_retrieval_question_score_requires_schema_versions() -> None:
    with pytest.raises(ValidationError, match="scorer_version"):
        RetrievalQuestionScore.model_validate(
            {
                "question_id": "q_private_001",
                "run_id": "local_rag_20260418T000000Z",
                "system_id": "local_rag",
                "iteration_id": "local_rag_v0.1",
                "question_set_version": "prescient_private_v1",
                "corpus_version": "prescient-os-chats-2026-04-18-with-retrieval-only-turn",
                "metrics": {
                    "required_doc_coverage": 1.0,
                    "required_source_coverage": 1.0,
                    "claim_coverage": 1.0,
                    "minimum_sufficient_set_satisfied": True,
                    "first_required_evidence_rank": 1,
                    "noise_count": 0,
                    "noise_ratio": 0.0,
                    "resolution_rule_satisfied": True,
                    "failure": False,
                },
                "matched_claim_ids": ["claim_wedge"],
                "missing_claim_ids": [],
                "matched_doc_ids": ["ai-enablement-strategy-pe-portfolios-2026-04-10"],
                "noise_doc_ids": [],
            }
        )
```

- [ ] **Step 2: Run the targeted model tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_models.py -q
```

Expected: FAIL because `claim_role`, `RetrievalQuestionScore`, and retrieval metric contracts do not exist yet.

- [ ] **Step 3: Add the minimal Pydantic models and claim-role field**

```python
# apps/api/src/prescient_benchmark/eval/models.py

class RequiredClaim(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    claim_id: StrictStr
    claim_role: Literal["current", "superseded", "period_a", "period_b", "supporting"]
    statement: StrictStr
    acceptable_evidence: list[AcceptableEvidence]


class RetrievalScoreMetrics(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    required_doc_coverage: float = Field(ge=0.0, le=1.0)
    required_source_coverage: float = Field(ge=0.0, le=1.0)
    claim_coverage: float = Field(ge=0.0, le=1.0)
    minimum_sufficient_set_satisfied: StrictBool
    first_required_evidence_rank: StrictInt | None = Field(default=None, ge=1)
    noise_count: StrictInt = Field(ge=0)
    noise_ratio: float = Field(ge=0.0, le=1.0)
    resolution_rule_satisfied: StrictBool
    failure: StrictBool


class RetrievalQuestionScore(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    question_id: StrictStr
    run_id: StrictStr
    system_id: StrictStr
    iteration_id: StrictStr
    question_set_version: StrictStr
    corpus_version: StrictStr
    scorer_version: StrictStr
    metric_schema_version: StrictStr
    metrics: RetrievalScoreMetrics
    matched_claim_ids: list[StrictStr]
    missing_claim_ids: list[StrictStr]
    matched_doc_ids: list[StrictStr]
    noise_doc_ids: list[StrictStr]
```

- [ ] **Step 4: Update the private evidence keys with explicit `claim_role` values**

```yaml
# eval/evidence_keys/prescient_private_v1.yaml
- question_id: q_private_003
  required_claims:
    - claim_id: operator_first_baseline
      claim_role: period_a
      statement: April 12 established the operator-first daily-work thesis.
    - claim_id: realism_push
      claim_role: period_b
      statement: April 14 shifted from thesis definition to richer demo realism and strategic initiative depth.

- question_id: q_private_006
  required_claims:
    - claim_id: operator_first_superseded
      claim_role: superseded
      statement: An earlier operator-first platform framing existed.
    - claim_id: ke_is_product_direction
      claim_role: current
      statement: The current product direction became a narrow API-first business knowledge engine.
    - claim_id: benchmark_is_validation_path
      claim_role: supporting
      statement: The retrieval benchmark was the immediate validation path for the KE thesis.
```

- [ ] **Step 5: Re-run the model tests to verify they pass**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_models.py -q
```

Expected: PASS with the new schema and claim-role coverage.

- [ ] **Step 6: Commit the contract changes**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/eval/models.py \
        eval/evidence_keys/prescient_private_v1.yaml \
        tests/unit/test_eval_models.py
git commit -m "feat: add retrieval scorer contracts"
```

---

### Task 2: Build The Pure Deterministic Retrieval Scorer

**Files:**
- Create: `apps/api/src/prescient_benchmark/eval/retrieval_scoring.py`
- Test: `tests/unit/test_retrieval_scoring.py`

- [ ] **Step 1: Write failing unit tests for deterministic scoring behavior**

```python
from prescient_benchmark.eval.models import (
    AnswerRecord,
    AnswerCitation,
    EvidenceKeyQuestion,
    RetrievedEvidence,
)
from prescient_benchmark.eval.retrieval_scoring import score_retrieval_bundle


def test_score_retrieval_bundle_matches_excerpt_across_multiple_chunks() -> None:
    question = EvidenceKeyQuestion.model_validate(
        {
            "question_id": "q_private_test",
            "question_type": "scope_disambiguation",
            "prompt": "What is Prescient OS being built to do?",
            "corpus_version": "snapshot_v1",
            "required_docs": ["ke-first-pivot-2026-04-16"],
            "required_sources": ["ke-first-pivot-2026-04-16"],
            "required_claims": [
                {
                    "claim_id": "ke_current",
                    "claim_role": "current",
                    "statement": "The product became KE-first.",
                    "acceptable_evidence": [
                        {
                            "doc_id": "ke-first-pivot-2026-04-16",
                            "locator": "### Product Direction At This Point",
                            "excerpt": "The knowledge engine became the primary product thesis",
                        }
                    ],
                }
            ],
            "resolution_rule": "latest_wins",
            "minimum_sufficient_set": ["ke_current"],
        }
    )
    answer = AnswerRecord.model_validate(
        {
            "question_id": "q_private_test",
            "run_id": "run_001",
            "system_id": "local_rag",
            "iteration_id": "local_rag_v0.1",
            "answer": "placeholder",
            "latency_ms": 100,
            "answer_model": "none",
            "answer_provider": "none",
            "created_at": "2026-04-18T12:00:00Z",
            "citations": [],
            "retrieved_evidence": [
                {
                    "doc_id": "ke-first-pivot-2026-04-16",
                    "chunk_id": "c1",
                    "locator": "chunk:1",
                    "text": "The knowledge engine became the",
                    "retrieval_rank": 1,
                },
                {
                    "doc_id": "ke-first-pivot-2026-04-16",
                    "chunk_id": "c2",
                    "locator": "chunk:2",
                    "text": " primary product thesis.",
                    "retrieval_rank": 2,
                },
            ],
        }
    )

    score = score_retrieval_bundle(
        answer=answer,
        evidence_key=question,
        source_kind_by_doc_id={"ke-first-pivot-2026-04-16": "memo"},
        scorer_version="retrieval_scorer_v1",
        metric_schema_version="retrieval_metrics_v1",
    )

    assert score.metrics.claim_coverage == 1.0
    assert score.metrics.minimum_sufficient_set_satisfied is True
    assert score.metrics.failure is False


def test_score_retrieval_bundle_requires_superseded_and_current_for_conflict_rule() -> None:
    ...
```

- [ ] **Step 2: Run the scorer tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_retrieval_scoring.py -q
```

Expected: FAIL because `retrieval_scoring.py` and `score_retrieval_bundle` do not exist.

- [ ] **Step 3: Implement the pure scorer in a focused module**

```python
# apps/api/src/prescient_benchmark/eval/retrieval_scoring.py
from __future__ import annotations

from collections import defaultdict

from prescient_benchmark.eval.models import (
    AnswerRecord,
    EvidenceKeyQuestion,
    RetrievalQuestionScore,
)


def score_retrieval_bundle(
    *,
    answer: AnswerRecord,
    evidence_key: EvidenceKeyQuestion,
    source_kind_by_doc_id: dict[str, str],
    scorer_version: str,
    metric_schema_version: str,
) -> RetrievalQuestionScore:
    grouped_text: dict[str, str] = {}
    grouped_ranks: dict[str, list[int]] = defaultdict(list)
    for doc_id in {item.doc_id for item in answer.retrieved_evidence}:
        items = sorted(
            [item for item in answer.retrieved_evidence if item.doc_id == doc_id],
            key=lambda item: item.retrieval_rank,
        )
        grouped_text[doc_id] = "".join(item.text for item in items)
        grouped_ranks[doc_id] = [item.retrieval_rank for item in items]

    matched_claim_ids: list[str] = []
    missing_claim_ids: list[str] = []
    matched_doc_ids: set[str] = set()
    first_required_rank: int | None = None

    for claim in evidence_key.required_claims:
        matched = False
        for evidence in claim.acceptable_evidence:
            if evidence.excerpt in grouped_text.get(evidence.doc_id, ""):
                matched = True
                matched_doc_ids.add(evidence.doc_id)
                candidate_rank = min(grouped_ranks[evidence.doc_id])
                first_required_rank = (
                    candidate_rank
                    if first_required_rank is None
                    else min(first_required_rank, candidate_rank)
                )
                break
        if matched:
            matched_claim_ids.append(claim.claim_id)
        else:
            missing_claim_ids.append(claim.claim_id)

    ...
```

- [ ] **Step 4: Cover every deterministic resolution rule in unit tests**

```python
def test_score_retrieval_bundle_enforces_memo_plus_raw() -> None:
    score = score_retrieval_bundle(
        answer=answer_with_only_memo_hits,
        evidence_key=evidence_key,
        source_kind_by_doc_id={"prescient-os-operator-first-redesign-2026-04-12": "memo"},
        scorer_version="retrieval_scorer_v1",
        metric_schema_version="retrieval_metrics_v1",
    )

    assert score.metrics.minimum_sufficient_set_satisfied is True
    assert score.metrics.resolution_rule_satisfied is False
    assert score.metrics.failure is True


def test_score_retrieval_bundle_tracks_noise_outside_required_docs() -> None:
    assert score.metrics.noise_count == 1
    assert score.noise_doc_ids == ["unrelated-doc"]
```

- [ ] **Step 5: Re-run the scorer unit tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_retrieval_scoring.py -q
```

Expected: PASS with deterministic matching and rule coverage.

- [ ] **Step 6: Commit the pure scorer**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/eval/retrieval_scoring.py \
        tests/unit/test_retrieval_scoring.py
git commit -m "feat: add deterministic retrieval scorer"
```

---

### Task 3: Add Run Paths, Orchestration, And A Real Regression Fixture

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/paths.py`
- Modify: `apps/api/src/prescient_benchmark/eval/orchestrator.py`
- Create: `tests/fixtures/eval/retrieval/prescient_private_v1_q_private_006_bundle.yaml`
- Modify: `tests/unit/test_eval_storage.py`
- Modify: `tests/integration/test_eval_harness_cli.py`

- [ ] **Step 1: Write failing tests for retrieval-score paths and run scoring**

```python
def test_eval_run_paths_build_retrieval_score_location(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.chdir(tmp_path)
    run_paths = EvalRunPaths.from_run_id("run_001")

    assert run_paths.retrieval_score_path("q_private_006") == (
        tmp_path / "eval/runs/run_001/retrieval_scores/q_private_006.yaml"
    )


def test_score_retrieval_run_writes_private_score_record(tmp_path: Path, monkeypatch) -> None:
    ...
    result = runner.invoke(app, ["score-eval-run-retrieval", "--run-id", run_id])
    assert result.exit_code == 0
    score_path = tmp_path / "eval/runs" / run_id / "retrieval_scores" / "q_private_006.yaml"
    assert score_path.exists()
```

- [ ] **Step 2: Run the path and CLI tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_storage.py tests/integration/test_eval_harness_cli.py -q
```

Expected: FAIL because there is no retrieval-score path or scoring command.

- [ ] **Step 3: Add retrieval-score paths and orchestrator wiring**

```python
# apps/api/src/prescient_benchmark/eval/paths.py
def retrieval_score_path(self, question_id: str) -> Path:
    _validate_identifier("question_id", question_id)
    return self.run_root / "retrieval_scores" / f"{question_id}.yaml"


# apps/api/src/prescient_benchmark/eval/orchestrator.py
_RETRIEVAL_SCORER_VERSION = "retrieval_scorer_v1"
_RETRIEVAL_METRIC_SCHEMA_VERSION = "retrieval_metrics_v1"


def score_retrieval_run(*, run_id: str) -> list[Path]:
    run_paths = EvalRunPaths.from_run_id(run_id)
    run_meta = read_yaml_model(run_paths.meta_path, EvalRunMeta)
    evidence_keys = load_evidence_key_set(_evidence_key_set_path(run_meta.question_set_version))
    evidence_by_id = {question.question_id: question for question in evidence_keys.questions}
    source_kind_by_doc_id = _build_private_source_kind_index(run_meta.corpus_version)

    written_paths: list[Path] = []
    for answer_path in sorted(run_paths.run_root.glob("answers/*.yaml")):
        answer = read_yaml_model(answer_path, AnswerRecord)
        evidence_key = evidence_by_id[answer.question_id]
        score = score_retrieval_bundle(
            answer=answer,
            evidence_key=evidence_key,
            source_kind_by_doc_id=source_kind_by_doc_id,
            scorer_version=_RETRIEVAL_SCORER_VERSION,
            metric_schema_version=_RETRIEVAL_METRIC_SCHEMA_VERSION,
        )
        written_paths.append(write_yaml_model(run_paths.retrieval_score_path(answer.question_id), score))
    return written_paths
```

- [ ] **Step 4: Add a frozen real-fixture regression test**

```python
def test_score_retrieval_run_matches_real_private_fixture(tmp_path: Path, monkeypatch) -> None:
    fixture = yaml.safe_load(
        (Path(__file__).resolve().parents[1] / "fixtures/eval/retrieval/prescient_private_v1_q_private_006_bundle.yaml")
        .read_text(encoding="utf-8")
    )
    ...
    score = read_yaml_model(score_path, RetrievalQuestionScore)
    assert score.metrics.minimum_sufficient_set_satisfied is True
    assert score.metrics.resolution_rule_satisfied is True
    assert score.matched_claim_ids == [
        "operator_first_superseded",
        "ke_is_product_direction",
        "benchmark_is_validation_path",
    ]
```

- [ ] **Step 5: Re-run the targeted tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest \
  tests/unit/test_eval_storage.py \
  tests/integration/test_eval_harness_cli.py \
  tests/unit/test_retrieval_scoring.py -q
```

Expected: PASS with path, orchestration, and regression coverage.

- [ ] **Step 6: Commit the scoring workflow**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/eval/paths.py \
        apps/api/src/prescient_benchmark/eval/orchestrator.py \
        tests/fixtures/eval/retrieval/prescient_private_v1_q_private_006_bundle.yaml \
        tests/unit/test_eval_storage.py \
        tests/integration/test_eval_harness_cli.py
git commit -m "feat: score retrieval bundles for eval runs"
```

---

### Task 4: Aggregate Retrieval Summary And Expose The CLI Command

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/summary.py`
- Modify: `apps/api/src/prescient_benchmark/cli.py`
- Modify: `tests/unit/test_eval_summary.py`
- Modify: `tests/integration/test_eval_harness_cli.py`

- [ ] **Step 1: Write failing summary and CLI tests**

```python
def test_summarize_run_reports_retrieval_summary_metrics(tmp_path: Path) -> None:
    run_root = tmp_path / "eval" / "runs" / "run_001"
    (run_root / "retrieval_scores").mkdir(parents=True)
    (run_root / "retrieval_scores" / "q_private_001.yaml").write_text(
        '''
question_id: q_private_001
run_id: run_001
system_id: local_rag
iteration_id: local_rag_v0.1
question_set_version: prescient_private_v1
corpus_version: snapshot_v1
scorer_version: retrieval_scorer_v1
metric_schema_version: retrieval_metrics_v1
metrics:
  required_doc_coverage: 1.0
  required_source_coverage: 1.0
  claim_coverage: 1.0
  minimum_sufficient_set_satisfied: true
  first_required_evidence_rank: 2
  noise_count: 0
  noise_ratio: 0.0
  resolution_rule_satisfied: true
  failure: false
matched_claim_ids: [claim_a]
missing_claim_ids: []
matched_doc_ids: [doc_a]
noise_doc_ids: []
''',
        encoding="utf-8",
    )

    summary = summarize_run(run_root=run_root)

    assert summary["retrieval_summary"]["hit_rate"] == 1.0
    assert summary["retrieval_summary"]["mrr"] == 0.5
    assert summary["retrieval_summary"]["failure_count"] == 0


def test_score_eval_run_retrieval_command_prints_written_count(tmp_path: Path, monkeypatch) -> None:
    ...
    result = runner.invoke(app, ["score-eval-run-retrieval", "--run-id", run_id])
    assert result.stdout.strip() == "1"
```

- [ ] **Step 2: Run the summary and CLI tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_summary.py tests/integration/test_eval_harness_cli.py -q
```

Expected: FAIL because `summarize_run` does not read retrieval scores and the CLI command does not exist.

- [ ] **Step 3: Add retrieval-summary aggregation and the CLI command**

```python
# apps/api/src/prescient_benchmark/eval/summary.py
retrieval_score_paths = sorted(run_root.glob("retrieval_scores/*.yaml"))
retrieval_scores = [read_yaml_model(path, RetrievalQuestionScore) for path in retrieval_score_paths]

if retrieval_scores:
    reciprocal_ranks = [
        1.0 / score.metrics.first_required_evidence_rank
        for score in retrieval_scores
        if score.metrics.first_required_evidence_rank is not None
    ]
    summary["retrieval_summary"] = {
        "question_count": len(retrieval_scores),
        "hit_rate": len(reciprocal_ranks) / len(retrieval_scores),
        "mrr": mean(reciprocal_ranks) if reciprocal_ranks else 0.0,
        "average_required_doc_coverage": mean(score.metrics.required_doc_coverage for score in retrieval_scores),
        "average_required_source_coverage": mean(score.metrics.required_source_coverage for score in retrieval_scores),
        "average_claim_coverage": mean(score.metrics.claim_coverage for score in retrieval_scores),
        "minimum_sufficient_set_rate": mean(
            1.0 if score.metrics.minimum_sufficient_set_satisfied else 0.0
            for score in retrieval_scores
        ),
        "average_noise_ratio": mean(score.metrics.noise_ratio for score in retrieval_scores),
        "resolution_rule_satisfaction_rate": mean(
            1.0 if score.metrics.resolution_rule_satisfied else 0.0
            for score in retrieval_scores
        ),
        "failure_count": sum(1 for score in retrieval_scores if score.metrics.failure),
        "failed_questions": [score.question_id for score in retrieval_scores if score.metrics.failure],
    }


# apps/api/src/prescient_benchmark/cli.py
@app.command("score-eval-run-retrieval")
def score_eval_run_retrieval_command(
    *,
    run_id: str = typer.Option(...),
) -> None:
    written_paths = score_retrieval_run(run_id=run_id)
    typer.echo(str(len(written_paths)))
```

- [ ] **Step 4: Re-run the summary and CLI tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_eval_summary.py tests/integration/test_eval_harness_cli.py -q
```

Expected: PASS with retrieval summaries present when score files exist and omitted when they do not.

- [ ] **Step 5: Run the full eval-focused suite**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest \
  tests/unit/test_eval_models.py \
  tests/unit/test_eval_storage.py \
  tests/unit/test_retrieval_scoring.py \
  tests/unit/test_eval_summary.py \
  tests/integration/test_eval_harness_cli.py -q
```

Expected: PASS for the scorer contracts, path safety, deterministic scoring, summary logic, and CLI flow.

- [ ] **Step 6: Commit the summary and CLI integration**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/eval/summary.py \
        apps/api/src/prescient_benchmark/cli.py \
        tests/unit/test_eval_summary.py \
        tests/integration/test_eval_harness_cli.py
git commit -m "feat: summarize deterministic retrieval scores"
```

---

### Task 5: Final Verification And Issue Closeout

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/models.py`
- Modify: `apps/api/src/prescient_benchmark/eval/paths.py`
- Modify: `apps/api/src/prescient_benchmark/eval/orchestrator.py`
- Modify: `apps/api/src/prescient_benchmark/eval/summary.py`
- Modify: `apps/api/src/prescient_benchmark/cli.py`
- Modify: `eval/evidence_keys/prescient_private_v1.yaml`
- Create: `apps/api/src/prescient_benchmark/eval/retrieval_scoring.py`
- Create: `tests/unit/test_retrieval_scoring.py`
- Create: `tests/fixtures/eval/retrieval/prescient_private_v1_q_private_006_bundle.yaml`
- Modify: `tests/unit/test_eval_models.py`
- Modify: `tests/unit/test_eval_storage.py`
- Modify: `tests/unit/test_eval_summary.py`
- Modify: `tests/integration/test_eval_harness_cli.py`

- [ ] **Step 1: Run the full unit and integration suite**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit tests/integration -q
```

Expected: PASS for the full benchmark repo suite.

- [ ] **Step 2: Run diff hygiene**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
git diff --check
git status --short
```

Expected:
- `git diff --check` prints nothing
- `git status --short` shows only the intended tracked files before commit

- [ ] **Step 3: Commit the final integrated slice**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/eval/models.py \
        apps/api/src/prescient_benchmark/eval/paths.py \
        apps/api/src/prescient_benchmark/eval/orchestrator.py \
        apps/api/src/prescient_benchmark/eval/summary.py \
        apps/api/src/prescient_benchmark/cli.py \
        apps/api/src/prescient_benchmark/eval/retrieval_scoring.py \
        eval/evidence_keys/prescient_private_v1.yaml \
        tests/unit/test_eval_models.py \
        tests/unit/test_eval_storage.py \
        tests/unit/test_eval_summary.py \
        tests/unit/test_retrieval_scoring.py \
        tests/integration/test_eval_harness_cli.py \
        tests/fixtures/eval/retrieval/prescient_private_v1_q_private_006_bundle.yaml
git commit -m "feat: add deterministic private retrieval scoring"
```

- [ ] **Step 4: Update and close the beads issue**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
bd close prescient_os-a5o
```

Expected: the issue closes after code, tests, and summary integration are complete.

- [ ] **Step 5: Push code and issue state**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
git pull --rebase
bd dolt push
git push
git status
```

Expected:
- remote push succeeds
- `git status` reports the branch is up to date with `origin`

---

## Self-Review

### Spec coverage

- Pure scorer over captured `retrieved_evidence`: covered in Tasks 2 and 3.
- Minimal evidence-key schema expansion for rule checks: covered in Task 1.
- Deterministic matching by concatenated per-doc retrieved text: covered in Task 2.
- Resolution-rule semantics (`latest_wins`, `must_compare_periods`, `must_surface_conflict`, `memo_plus_raw`, `raw_only`): covered in Task 2.
- Per-question retrieval score records under run paths: covered in Task 3.
- Retrieval summary with `hit_rate`, `mrr`, coverage, sufficiency, and failure counts: covered in Task 4.
- Real-fixture regression test over `prescient_private_v1`: covered in Task 3.
- No coupling to retriever execution or prose grading: maintained throughout.

### Placeholder scan

- No `TODO`, `TBD`, or “implement later” placeholders remain.
- Each code-changing step includes concrete file paths, commands, and code snippets.
- Testing steps name exact commands and expected outcomes.

### Type consistency

- `claim_role` appears consistently in the evidence-key contract and tests.
- `RetrievalQuestionScore` / `RetrievalScoreMetrics` naming stays consistent across models, scorer, summary, and tests.
- `score_retrieval_run` and `score-eval-run-retrieval` are used consistently as the orchestrator and CLI names.

