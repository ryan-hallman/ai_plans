# Mixed-Source Retrieval Selection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a benchmark-only mixed-source retrieval selector that can lift `q_private_002` and `q_private_004` by preserving the exact required raw-session doc in the emitted bundle without weakening claim coverage.

**Architecture:** Keep OpenSearch query generation unchanged. Add a diagnostic-first helper that finds the exact required raw doc rank for `memo_plus_raw` questions, then apply a source-balanced bundle selector only inside the benchmark harness for those questions. Preserve both original `retrieval_rank` and emitted `bundle_rank`, and report both diagnostic-rank and emitted-bundle rank metrics in retrieval summaries.

**Tech Stack:** FastAPI/Python backend, Typer CLI, Pydantic models, OpenSearch search results, pytest unit/integration tests, beads issue `prescient_os-viw`.

---

## File structure

- Create: `apps/api/src/prescient_benchmark/retrieval/mixed_source_selection.py`
  - Diagnostic helpers for exact required raw-doc rank detection.
  - Bundle-selection helpers that preserve at least one memo and one raw candidate when both exist.
- Modify: `apps/api/src/prescient_benchmark/eval/models.py`
  - Add `bundle_rank` to retrieved evidence and add emitted-bundle rank metrics to retrieval score models.
- Modify: `apps/api/src/prescient_benchmark/eval/retrieval_scoring.py`
  - Compute both original-rank and bundle-rank metrics from the same bundle.
- Modify: `apps/api/src/prescient_benchmark/eval/orchestrator.py`
  - Load evidence keys during baseline retrieval, run the top-100 diagnostic for `memo_plus_raw` questions, derive `candidate_k`, apply the selector, and emit the new retriever provenance.
- Modify: `apps/api/src/prescient_benchmark/eval/summary.py`
  - Report both diagnostic-rank and emitted-bundle rank metrics so selector lift is visible.
- Modify: `tests/unit/test_eval_models.py`
  - Validate `bundle_rank` shape and rank consistency rules.
- Create: `tests/unit/test_mixed_source_selection.py`
  - Pin the diagnostic helper, candidate-depth derivation, and source-balanced selector behavior.
- Modify: `tests/unit/test_retrieval_scoring.py`
  - Cover emitted-bundle rank metrics and exact required raw-doc semantics.
- Modify: `tests/unit/test_eval_orchestrator.py`
  - Exercise baseline-run orchestration for diagnostic failure, selector success, and no-op behavior on non-target questions.
- Modify: `tests/unit/test_eval_summary.py`
  - Lock the new summary fields and bundle-rank-visible reporting.
- Modify: `tests/integration/test_eval_harness_cli.py`
  - Verify CLI baseline output still works and selector-enabled runs surface the new retrieval summary fields.

### Task 1: Add diagnostic-first mixed-source helpers

**Files:**
- Create: `apps/api/src/prescient_benchmark/retrieval/mixed_source_selection.py`
- Create: `tests/unit/test_mixed_source_selection.py`

- [ ] **Step 1: Write the failing diagnostic tests**

```python
from prescient_benchmark.retrieval.mixed_source_selection import (
    derive_candidate_depth,
    first_rank_for_doc,
    is_memo_plus_raw_question,
)
from prescient_benchmark.retrieval.models import SearchResult


def test_first_rank_for_doc_returns_1_based_rank() -> None:
    candidates = [
        SearchResult(chunk_id="memo:0", doc_id="memo", title="memo", text="memo", score=9.0),
        SearchResult(chunk_id="raw:0", doc_id="raw-required", title="raw", text="raw", score=8.0),
    ]

    assert first_rank_for_doc(candidates, "raw-required") == 2


def test_derive_candidate_depth_uses_smallest_rank_that_covers_all_required_raw_docs() -> None:
    required_ranks = {"q_private_002": 19, "q_private_004": 27}

    assert derive_candidate_depth(required_ranks, minimum_top_k=20, diagnostic_top_n=100) == 27


def test_derive_candidate_depth_rejects_missing_required_raw_doc() -> None:
    with pytest.raises(ValueError, match="required raw doc not found in top-100 candidates"):
        derive_candidate_depth({"q_private_002": 12, "q_private_004": None}, minimum_top_k=20, diagnostic_top_n=100)


def test_is_memo_plus_raw_question_requires_both_labels() -> None:
    assert is_memo_plus_raw_question(source_mix="mixed", resolution_rule="memo_plus_raw") is True
    assert is_memo_plus_raw_question(source_mix="memo_led", resolution_rule="memo_plus_raw") is False
```

- [ ] **Step 2: Run the new unit tests to verify they fail**

Run:

```bash
uv run python -m pytest tests/unit/test_mixed_source_selection.py -q
```

Expected: FAIL with `ModuleNotFoundError` or missing symbol errors for `mixed_source_selection`.

- [ ] **Step 3: Write the minimal diagnostic helper module**

```python
from __future__ import annotations

from collections.abc import Iterable

from prescient_benchmark.retrieval.models import SearchResult


def is_memo_plus_raw_question(*, source_mix: str | None, resolution_rule: str | None) -> bool:
    return source_mix == "mixed" and resolution_rule == "memo_plus_raw"


def required_raw_doc_id_for_question(
    *,
    required_sources: list[str],
    source_kind_by_doc_id: dict[str, str],
) -> str:
    raw_doc_ids = [
        doc_id for doc_id in required_sources if source_kind_by_doc_id.get(doc_id) == "raw_session"
    ]
    if len(raw_doc_ids) != 1:
        raise ValueError("memo_plus_raw questions must declare exactly one required raw-session doc")
    return raw_doc_ids[0]


def first_rank_for_doc(candidates: Iterable[SearchResult], required_doc_id: str) -> int | None:
    for index, candidate in enumerate(candidates, start=1):
        if candidate.doc_id == required_doc_id:
            return index
    return None


def derive_candidate_depth(
    required_ranks: dict[str, int | None],
    *,
    minimum_top_k: int,
    diagnostic_top_n: int,
) -> int:
    missing = [question_id for question_id, rank in required_ranks.items() if rank is None]
    if missing:
        raise ValueError(
            "required raw doc not found in top-100 candidates: " + ", ".join(sorted(missing))
        )

    return max(minimum_top_k, max(rank for rank in required_ranks.values() if rank is not None))
```

- [ ] **Step 4: Run the diagnostic helper tests to verify they pass**

Run:

```bash
uv run python -m pytest tests/unit/test_mixed_source_selection.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit the diagnostic-first helper**

```bash
git add apps/api/src/prescient_benchmark/retrieval/mixed_source_selection.py tests/unit/test_mixed_source_selection.py
git commit -m "feat: add mixed-source retrieval diagnostics"
```

### Task 2: Add source-balanced bundle emission and model support

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/models.py`
- Modify: `apps/api/src/prescient_benchmark/eval/retrieval_records.py`
- Modify: `tests/unit/test_eval_models.py`
- Modify: `tests/unit/test_retrieval_records.py`
- Modify: `apps/api/src/prescient_benchmark/retrieval/mixed_source_selection.py`
- Modify: `tests/unit/test_mixed_source_selection.py`

- [ ] **Step 1: Write failing tests for `bundle_rank` and selector output**

```python
from prescient_benchmark.eval.models import RetrievedEvidence


def test_retrieval_record_accepts_bundle_rank_when_it_differs_from_retrieval_rank() -> None:
    record = RetrievalRecord.model_validate(
        {
            "question_id": "q_private_002",
            "run_id": "run_1",
            "system_id": "opensearch",
            "iteration_id": "balanced_v1",
            "question_set_version": "prescient_private_v1",
            "corpus_version": "prescient-os-chats-2026-04-18-with-retrieval-only-turn",
            "retriever_id": "opensearch_single_pass_source_balanced",
            "retriever_version": "opensearch_single_pass_source_balanced_v1",
            "analyzer": "standard",
            "index_settings_hash": "abc123",
            "index_name": "index",
            "top_k": 20,
            "query": "Why did Prescient OS pivot?",
            "latency_ms": 15,
            "retrieved_evidence": [
                {
                    "doc_id": "operator-memo",
                    "chunk_id": "operator-memo:0",
                    "locator": "chunk:0",
                    "text": "memo text",
                    "retrieval_rank": 1,
                    "bundle_rank": 1,
                },
                {
                    "doc_id": "raw-required",
                    "chunk_id": "raw-required:0",
                    "locator": "chunk:0",
                    "text": "raw text",
                    "retrieval_rank": 8,
                    "bundle_rank": 2,
                },
            ],
            "created_at": "2026-04-20T12:00:00Z",
        }
    )

    assert record.retrieved_evidence[1].bundle_rank == 2


def test_select_source_balanced_bundle_reserves_highest_ranked_memo_and_raw() -> None:
    selected = select_source_balanced_bundle(
        candidates=[
            RetrievedEvidence.model_validate(
                {
                    "doc_id": "memo-1",
                    "chunk_id": "memo-1:0",
                    "locator": "chunk:0",
                    "text": "memo one",
                    "retrieval_rank": 1,
                }
            ),
            RetrievedEvidence.model_validate(
                {
                    "doc_id": "memo-2",
                    "chunk_id": "memo-2:0",
                    "locator": "chunk:0",
                    "text": "memo two",
                    "retrieval_rank": 2,
                }
            ),
            RetrievedEvidence.model_validate(
                {
                    "doc_id": "raw-required",
                    "chunk_id": "raw-required:0",
                    "locator": "chunk:0",
                    "text": "raw text",
                    "retrieval_rank": 9,
                }
            ),
        ],
        top_k=2,
        source_kind_by_doc_id={"memo-1": "memo", "memo-2": "memo", "raw-required": "raw_session"},
    )

    assert [item.doc_id for item in selected] == ["memo-1", "raw-required"]
    assert [item.bundle_rank for item in selected] == [1, 2]
    assert [item.retrieval_rank for item in selected] == [1, 9]
```

- [ ] **Step 2: Run the targeted tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_models.py tests/unit/test_retrieval_records.py tests/unit/test_mixed_source_selection.py -q
```

Expected: FAIL because `bundle_rank` does not exist and `select_source_balanced_bundle` is missing.

- [ ] **Step 3: Add `bundle_rank` and selector helpers**

```python
class RetrievedEvidence(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    doc_id: StrictStr
    chunk_id: StrictStr
    locator: StrictStr
    text: StrictStr
    retrieval_rank: StrictInt
    bundle_rank: StrictInt | None = Field(default=None, ge=1)
```

```python
def select_source_balanced_bundle(
    *,
    candidates: list[RetrievedEvidence],
    top_k: int,
    source_kind_by_doc_id: dict[str, str],
) -> list[RetrievedEvidence]:
    memo = next((item for item in candidates if source_kind_by_doc_id.get(item.doc_id) == "memo"), None)
    raw = next((item for item in candidates if source_kind_by_doc_id.get(item.doc_id) == "raw_session"), None)

    selected: list[RetrievedEvidence] = []
    for candidate in (memo, raw):
        if candidate is not None and candidate.doc_id not in {item.doc_id for item in selected}:
            selected.append(candidate.model_copy())

    for candidate in candidates:
        if len(selected) >= top_k:
            break
        if candidate.chunk_id in {item.chunk_id for item in selected}:
            continue
        selected.append(candidate.model_copy())

    for index, evidence in enumerate(selected, start=1):
        evidence.bundle_rank = index

    return selected[:top_k]
```

- [ ] **Step 4: Re-run the model and selector tests**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_models.py tests/unit/test_retrieval_records.py tests/unit/test_mixed_source_selection.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit the selector and model changes**

```bash
git add apps/api/src/prescient_benchmark/eval/models.py apps/api/src/prescient_benchmark/eval/retrieval_records.py apps/api/src/prescient_benchmark/retrieval/mixed_source_selection.py tests/unit/test_eval_models.py tests/unit/test_retrieval_records.py tests/unit/test_mixed_source_selection.py
git commit -m "feat: add source-balanced retrieval bundle selection"
```

### Task 3: Integrate diagnostic gating and selector use into the baseline runner

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/orchestrator.py`
- Modify: `tests/unit/test_eval_orchestrator.py`
- Modify: `tests/integration/test_eval_harness_cli.py`

- [ ] **Step 1: Write failing orchestration tests for selector activation and diagnostic failure**

```python
from datetime import UTC, datetime
from pathlib import Path

import pytest
from prescient_benchmark.eval.models import RetrievalRecord
from prescient_benchmark.eval.orchestrator import init_eval_run, run_private_retrieval_baseline
from prescient_benchmark.eval.paths import EvalRunPaths
from prescient_benchmark.eval.storage import read_yaml_model
from prescient_benchmark.retrieval.models import SearchResult


def test_run_private_retrieval_baseline_rejects_target_question_when_required_raw_doc_missing_from_top_100(
    tmp_path: Path,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator.search_index",
        lambda **kwargs: [
            SearchResult(
                chunk_id=f"memo-doc:{index}",
                doc_id="memo-doc",
                title="memo",
                text=f"memo chunk {index}",
                score=float(100 - index),
            )
            for index in range(100)
        ],
    )

    with pytest.raises(ValueError, match="required raw doc not found in top-100 candidates"):
        run_private_retrieval_baseline(run_id=run_meta.run_id, data_root=data_root, top_k=20, opensearch_client=client)


def test_run_private_retrieval_baseline_emits_source_balanced_bundle_for_memo_plus_raw_question(
    tmp_path: Path,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator.search_index",
        lambda **kwargs: [
            SearchResult(chunk_id="memo-doc:0", doc_id="memo-doc", title="memo", text="memo", score=9.0),
            SearchResult(chunk_id="memo-doc:1", doc_id="memo-doc", title="memo", text="memo", score=8.0),
            SearchResult(
                chunk_id="raw-required:0",
                doc_id="raw-required",
                title="raw",
                text="raw",
                score=1.0,
            ),
        ],
    )

    result = run_private_retrieval_baseline(run_id=run_meta.run_id, data_root=data_root, top_k=20, opensearch_client=client)

    record = read_yaml_model(EvalRunPaths.from_run_id(run_meta.run_id).retrieval_path("q_private_002"), RetrievalRecord)
    assert record.retriever_id == "opensearch_single_pass_source_balanced"
    assert [item.doc_id for item in record.retrieved_evidence[:2]] == ["memo-doc", "raw-required"]
    assert record.retrieved_evidence[1].retrieval_rank == 19
    assert record.retrieved_evidence[1].bundle_rank == 2
```

- [ ] **Step 2: Run the orchestration tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_orchestrator.py tests/integration/test_eval_harness_cli.py -q
```

Expected: FAIL because the baseline runner does not yet load evidence keys or rebalance emitted bundles.

- [ ] **Step 3: Update the baseline runner to use the diagnostic and selector**

```python
evidence_key_set = load_evidence_key_set(_evidence_key_path(run_meta.question_set_version))
evidence_key_by_question_id = {item.question_id: item for item in evidence_key_set.questions}

for question in question_set.questions:
    evidence_key = evidence_key_by_question_id[question.id]
    target_question = question.source_mix == "mixed" and evidence_key.resolution_rule == "memo_plus_raw"

    if target_question:
        diagnostic_results = search_index(client=client, index_name=index_name, query=question.prompt, top_k=100)
        required_raw_doc_id = _required_raw_doc_id(evidence_key=evidence_key, source_kind_by_doc_id=source_kind_by_doc_id)
        required_raw_rank = first_rank_for_doc(diagnostic_results, required_raw_doc_id)
        candidate_k = derive_candidate_depth({question.id: required_raw_rank}, minimum_top_k=top_k, diagnostic_top_n=100)
        search_results = search_index(client=client, index_name=index_name, query=question.prompt, top_k=candidate_k)
        retrieved = _retrieved_evidence_from_search_results(search_results)
        retrieved = select_source_balanced_bundle(
            candidates=retrieved,
            top_k=top_k,
            source_kind_by_doc_id=source_kind_by_doc_id,
        )
        retriever_id = "opensearch_single_pass_source_balanced"
        retriever_version = "opensearch_single_pass_source_balanced_v1"
    else:
        search_results = search_index(client=client, index_name=index_name, query=question.prompt, top_k=top_k)
        retrieved = _retrieved_evidence_from_search_results(search_results)
        for index, evidence in enumerate(retrieved, start=1):
            evidence.bundle_rank = index
        retriever_id = _PRIVATE_RETRIEVAL_RETRIEVER_ID
        retriever_version = _PRIVATE_RETRIEVAL_RETRIEVER_VERSION
```

- [ ] **Step 4: Re-run orchestration and CLI coverage**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_orchestrator.py tests/integration/test_eval_harness_cli.py -q
```

Expected: PASS, with selector-enabled runs emitting balanced bundles and failing fast when the exact required raw doc is absent from top-100.

- [ ] **Step 5: Commit the orchestrator integration**

```bash
git add apps/api/src/prescient_benchmark/eval/orchestrator.py tests/unit/test_eval_orchestrator.py tests/integration/test_eval_harness_cli.py
git commit -m "feat: apply mixed-source selection in baseline runner"
```

### Task 4: Score and summarize both diagnostic rank and emitted-bundle rank

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/models.py`
- Modify: `apps/api/src/prescient_benchmark/eval/retrieval_scoring.py`
- Modify: `apps/api/src/prescient_benchmark/eval/summary.py`
- Modify: `tests/unit/test_retrieval_scoring.py`
- Modify: `tests/unit/test_eval_summary.py`

- [ ] **Step 1: Write failing scoring and summary tests**

```python
from prescient_benchmark.eval.models import RetrievalBundle
from prescient_benchmark.eval.retrieval_scoring import score_retrieval_bundle


def test_score_retrieval_bundle_reports_bundle_rank_for_required_doc() -> None:
    score = score_retrieval_bundle(
        bundle=RetrievalBundle.model_validate(
            {
                "question_id": "q_private_002",
                "run_id": "run_1",
                "system_id": "opensearch_single_pass_source_balanced",
                "iteration_id": "balanced_v1",
                "retrieved_evidence": [
                    {
                        "doc_id": "memo-doc",
                        "chunk_id": "memo-doc:0",
                        "locator": "chunk:0",
                        "text": "memo text",
                        "retrieval_rank": 1,
                        "bundle_rank": 1,
                    },
                    {
                        "doc_id": "raw-required",
                        "chunk_id": "raw-required:0",
                        "locator": "chunk:0",
                        "text": "raw text",
                        "retrieval_rank": 19,
                        "bundle_rank": 2,
                    },
                ],
            }
        ),
        evidence_key=EvidenceKeyQuestion.model_validate(
            {
                "question_id": "q_private_002",
                "corpus_version": "prescient-os-chats-2026-04-18-with-retrieval-only-turn",
                "required_docs": ["memo-doc", "raw-required"],
                "required_sources": ["memo-doc", "raw-required"],
                "required_claims": [
                    {
                        "claim_id": "mixed_claim",
                        "claim_role": "supporting",
                        "statement": "memo and raw both surfaced",
                        "acceptable_evidence": [
                            {"doc_id": "memo-doc", "locator": "chunk:0", "excerpt": "memo text"},
                            {"doc_id": "raw-required", "locator": "chunk:0", "excerpt": "raw text"},
                        ],
                    }
                ],
                "resolution_rule": "memo_plus_raw",
                "minimum_sufficient_set": ["mixed_claim"],
            }
        ),
        question_set_version="prescient_private_v1",
        source_kind_by_doc_id={"memo-doc": "memo", "raw-required": "raw_session"},
        scorer_version="retrieval_scorer_v1",
        metric_schema_version="retrieval_score_metrics_v2",
    )

    assert score.metrics.first_required_evidence_rank == 19
    assert score.metrics.first_required_bundle_rank == 2


def test_summarize_run_reports_bundle_rank_mrr() -> None:
    run_root = tmp_path / "eval" / "runs" / "run_001"
    (run_root / "retrieval_scores").mkdir(parents=True)
    (run_root / "retrieval_scores" / "q_private_002.yaml").write_text(
        \"\"\"
question_id: q_private_002
run_id: run_001
system_id: opensearch_single_pass_source_balanced
iteration_id: balanced_v1
question_set_version: prescient_private_v1
corpus_version: prescient-os-chats-2026-04-18-with-retrieval-only-turn
scorer_version: retrieval_scorer_v1
metric_schema_version: retrieval_score_metrics_v2
metrics:
  required_doc_coverage: 1.0
  required_source_coverage: 1.0
  claim_coverage: 1.0
  minimum_sufficient_set_satisfied: true
  first_required_evidence_rank: 19
  first_required_bundle_rank: 2
  noise_count: 0
  noise_ratio: 0.0
  resolution_rule_satisfied: true
  failure: false
matched_claim_ids: [mixed_claim]
missing_claim_ids: []
matched_doc_ids: [memo-doc, raw-required]
noise_doc_ids: []
\"\"\",
        encoding="utf-8",
    )

    summary = summarize_run(run_root=run_root)

    assert summary["retrieval_summary"]["mrr"] == pytest.approx(1 / 19)
    assert summary["retrieval_summary"]["bundle_mrr"] == pytest.approx(1 / 2)
```

- [ ] **Step 2: Run the scoring and summary tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_retrieval_scoring.py tests/unit/test_eval_summary.py -q
```

Expected: FAIL because `first_required_bundle_rank` and `bundle_mrr` do not exist yet.

- [ ] **Step 3: Add emitted-bundle rank metrics**

```python
class RetrievalScoreMetrics(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    required_doc_coverage: float = Field(ge=0, le=1)
    required_source_coverage: float = Field(ge=0, le=1)
    claim_coverage: float = Field(ge=0, le=1)
    minimum_sufficient_set_satisfied: StrictBool
    first_required_evidence_rank: StrictInt | None = Field(default=None, ge=1)
    first_required_bundle_rank: StrictInt | None = Field(default=None, ge=1)
    noise_count: StrictInt = Field(ge=0)
    noise_ratio: float = Field(ge=0, le=1)
    resolution_rule_satisfied: StrictBool
    failure: StrictBool
```

```python
def _first_required_bundle_rank(retrieved_evidence, matched_required_doc_ids: set[str]) -> int | None:
    ranks = [
        evidence.bundle_rank
        for evidence in retrieved_evidence
        if evidence.doc_id in matched_required_doc_ids and evidence.bundle_rank is not None
    ]
    return min(ranks) if ranks else None
```

```python
summary["retrieval_summary"] = {
    "question_count": len(retrieval_scores),
    "hit_rate": sum(
        1 for score in retrieval_scores if score.metrics.first_required_evidence_rank is not None
    ) / len(retrieval_scores),
    "mrr": mean(reciprocal_ranks),
    "bundle_hit_rate": sum(1 for score in retrieval_scores if score.metrics.first_required_bundle_rank is not None) / len(retrieval_scores),
    "bundle_mrr": mean(
        1.0 / score.metrics.first_required_bundle_rank if score.metrics.first_required_bundle_rank is not None else 0.0
        for score in retrieval_scores
    ),
    "average_required_doc_coverage": mean(score.metrics.required_doc_coverage for score in retrieval_scores),
    "average_required_source_coverage": mean(score.metrics.required_source_coverage for score in retrieval_scores),
    "average_claim_coverage": mean(score.metrics.claim_coverage for score in retrieval_scores),
    "minimum_sufficient_set_rate": mean(
        1.0 if score.metrics.minimum_sufficient_set_satisfied else 0.0 for score in retrieval_scores
    ),
    "average_noise_ratio": mean(score.metrics.noise_ratio for score in retrieval_scores),
    "resolution_rule_satisfaction_rate": mean(
        1.0 if score.metrics.resolution_rule_satisfied else 0.0 for score in retrieval_scores
    ),
    "failure_count": sum(1 for score in retrieval_scores if score.metrics.failure),
    "failed_questions": [score.question_id for score in retrieval_scores if score.metrics.failure],
}
```

- [ ] **Step 4: Re-run scoring and summary tests**

Run:

```bash
uv run python -m pytest tests/unit/test_retrieval_scoring.py tests/unit/test_eval_summary.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit scoring and summary visibility**

```bash
git add apps/api/src/prescient_benchmark/eval/models.py apps/api/src/prescient_benchmark/eval/retrieval_scoring.py apps/api/src/prescient_benchmark/eval/summary.py tests/unit/test_retrieval_scoring.py tests/unit/test_eval_summary.py
git commit -m "feat: report emitted bundle rank metrics"
```

### Task 5: Run the targeted regression and close the slice

**Files:**
- Modify: `tests/unit/test_eval_orchestrator.py`
- Modify: `tests/integration/test_eval_harness_cli.py`
- Modify: `tests/unit/test_retrieval_scoring.py`

- [ ] **Step 1: Add the exact pass/fail regression assertions for the two target questions**

```python
def test_mixed_source_selector_makes_q_private_002_pass_exact_required_source_coverage() -> None:
    score = read_yaml_model(run_paths.retrieval_score_path("q_private_002"), RetrievalQuestionScore)
    assert score.metrics.required_source_coverage == 1.0
    assert score.metrics.resolution_rule_satisfied is True
    assert score.metrics.failure is False


def test_mixed_source_selector_makes_q_private_004_pass_exact_required_source_coverage() -> None:
    score = read_yaml_model(run_paths.retrieval_score_path("q_private_004"), RetrievalQuestionScore)
    assert score.metrics.required_source_coverage == 1.0
    assert score.metrics.resolution_rule_satisfied is True
    assert score.metrics.failure is False
```

- [ ] **Step 2: Run the focused regression suite**

Run:

```bash
uv run python -m pytest tests/unit/test_mixed_source_selection.py tests/unit/test_eval_models.py tests/unit/test_retrieval_records.py tests/unit/test_retrieval_scoring.py tests/unit/test_eval_orchestrator.py tests/unit/test_eval_summary.py tests/integration/test_eval_harness_cli.py -q
```

Expected: PASS.

- [ ] **Step 3: Run the full unit and integration suite**

Run:

```bash
uv run python -m pytest tests/unit tests/integration -q
```

Expected: PASS.

- [ ] **Step 4: Run diff hygiene checks**

Run:

```bash
git diff --check
```

Expected: no output.

- [ ] **Step 5: Commit, sync beads, and push**

```bash
bd close prescient_os-viw
git add apps/api/src/prescient_benchmark/eval/models.py apps/api/src/prescient_benchmark/eval/orchestrator.py apps/api/src/prescient_benchmark/eval/retrieval_scoring.py apps/api/src/prescient_benchmark/eval/summary.py apps/api/src/prescient_benchmark/retrieval/mixed_source_selection.py tests/unit/test_mixed_source_selection.py tests/unit/test_eval_models.py tests/unit/test_retrieval_records.py tests/unit/test_retrieval_scoring.py tests/unit/test_eval_orchestrator.py tests/unit/test_eval_summary.py tests/integration/test_eval_harness_cli.py
git commit -m "feat: add mixed-source retrieval selection"
git pull --rebase
bd dolt push
git push
```

## Self-review

- Spec coverage:
  - benchmark-only oracle gating: Task 3
  - diagnostic-first candidate-depth derivation: Task 1 and Task 3
  - exact required raw-doc semantics: Task 1, Task 3, and Task 5
  - bundle-rank-visible scoring: Task 4
  - falsifying outcome for missing required raw doc: Task 3
- Placeholder scan:
  - no `TBD`, `TODO`, or “implement later”
  - every code-changing step includes concrete snippets
- Type consistency:
  - `bundle_rank` is introduced in Task 2 and consumed in Tasks 3-4
  - selector function names match across tests and implementation
  - summary metrics use `first_required_bundle_rank` consistently
