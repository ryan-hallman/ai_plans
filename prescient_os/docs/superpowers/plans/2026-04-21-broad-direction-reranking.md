# Broad Direction Reranking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Improve retrieval quality for broad, unscoped company/product-direction queries over the private PrescientOS corpus by adding a narrow, query-text-gated reranker that prefers curated memos and mildly prefers newer direction-setting memos.

**Architecture:** Keep OpenSearch as the candidate generator and add a post-search reranker only for a narrow class of broad direction queries. Load lightweight private-corpus document priors from the snapshot manifest plus memo frontmatter, then use those priors to reorder candidates for broad direction queries without changing the indexed document schema.

**Tech Stack:** Python backend, Pydantic models, private snapshot YAML/markdown parsing, OpenSearch candidate retrieval, pytest unit/integration tests, beads issue `prescient_os-2tt`.

---

## File structure

- Create: `apps/api/src/prescient_benchmark/corpus/private_doc_priors.py`
  - Load lightweight retrieval priors from a private snapshot.
  - Provide `source_kind` and optional memo `as_of` for each `doc_id`.
- Create: `apps/api/src/prescient_benchmark/retrieval/broad_direction_reranking.py`
  - Detect broad direction queries.
  - Compute reranked ordering over `SearchResult` candidates using authority + mild memo recency.
- Modify: `apps/api/src/prescient_benchmark/eval/orchestrator.py`
  - Load doc priors once per baseline run.
  - Apply broad-direction reranking before retrieval records are written.
- Modify: `tests/unit/test_private_doc_priors.py`
  - New focused tests for source-kind and memo-date loading.
- Modify: `tests/unit/test_broad_direction_reranking.py`
  - New focused tests for query gating and reranking behavior.
- Modify: `tests/unit/test_eval_orchestrator.py`
  - Add `q_private_006` baseline regression over the private runner.
  - Preserve non-regression checks for `q_private_002` / `q_private_004`.
- Modify: `tests/integration/test_eval_harness_cli.py`
  - Add CLI-level regression proving broad-direction reranking improves `q_private_006` without disturbing existing mixed-source behavior.

### Task 1: Add private snapshot document priors

**Files:**
- Create: `apps/api/src/prescient_benchmark/corpus/private_doc_priors.py`
- Create: `tests/unit/test_private_doc_priors.py`

- [ ] **Step 1: Write the failing doc-priors tests**

```python
from datetime import date
from pathlib import Path

import pytest

from prescient_benchmark.corpus.private_doc_priors import load_private_doc_priors


def test_load_private_doc_priors_returns_source_kind_for_sessions_and_memos(tmp_path: Path) -> None:
    snapshot_root = tmp_path / "snapshot"
    (snapshot_root / "normalized" / "sessions").mkdir(parents=True)
    (snapshot_root / "memos").mkdir(parents=True)
    (snapshot_root / "manifest.yaml").write_text(
        """
        snapshot_id: snap
        exported_at: 2026-04-18T20:00:00Z
        source_workspace: ws
        sessions:
          - session_id: raw-session-1
            source_provider: anthropic
            source_app: claude
            raw_filename: raw/anthropic/raw-session-1.jsonl
            normalized_filename: normalized/sessions/raw-session-1.yaml
            content_hash: abc
        memos:
          - memo_id: ke-first-pivot-2026-04-16
            filename: memos/ke-first-pivot-2026-04-16.md
        """,
        encoding="utf-8",
    )
    (snapshot_root / "normalized" / "sessions" / "raw-session-1.yaml").write_text(
        """
        session_id: raw-session-1
        source_provider: anthropic
        source_app: claude
        exported_at: 2026-04-18T20:00:00Z
        raw_filename: raw-session-1.jsonl
        content_hash: abc
        provider_metadata: {}
        messages:
          - index: 1
            role: user
            content_text: raw text
        """,
        encoding="utf-8",
    )
    (snapshot_root / "memos" / "ke-first-pivot-2026-04-16.md").write_text(
        """---
        memo_id: ke-first-pivot-2026-04-16
        title: KE First Pivot
        as_of: 2026-04-16
        topic_bucket: Product Strategy
        source_refs: []
        ---

        Body.
        """,
        encoding="utf-8",
    )

    priors = load_private_doc_priors(snapshot_root)

    assert priors["raw-session-1"].source_kind == "raw_session"
    assert priors["raw-session-1"].memo_as_of is None
    assert priors["ke-first-pivot-2026-04-16"].source_kind == "memo"
    assert priors["ke-first-pivot-2026-04-16"].memo_as_of == date(2026, 4, 16)


def test_load_private_doc_priors_rejects_missing_memo_file(tmp_path: Path) -> None:
    snapshot_root = tmp_path / "snapshot"
    snapshot_root.mkdir()
    (snapshot_root / "manifest.yaml").write_text(
        """
        snapshot_id: snap
        exported_at: 2026-04-18T20:00:00Z
        source_workspace: ws
        sessions: []
        memos:
          - memo_id: missing-memo
            filename: memos/missing-memo.md
        """,
        encoding="utf-8",
    )

    with pytest.raises(ValueError, match="private corpus memo file does not exist"):
        load_private_doc_priors(snapshot_root)
```

- [ ] **Step 2: Run the doc-priors tests to verify they fail**

Run:

```bash
uv run python -m pytest tests/unit/test_private_doc_priors.py -q
```

Expected: FAIL with `ModuleNotFoundError` for `private_doc_priors`.

- [ ] **Step 3: Write the minimal doc-priors loader**

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import date
from pathlib import Path

from prescient_benchmark.corpus.private_chat_loader import _load_manual_memo, _load_manifest
from prescient_benchmark.corpus.private_chat_storage import normalize_private_corpus_root


@dataclass(frozen=True)
class PrivateDocPrior:
    source_kind: str
    memo_as_of: date | None


def load_private_doc_priors(snapshot_root: Path) -> dict[str, PrivateDocPrior]:
    resolved_root = normalize_private_corpus_root(snapshot_root)
    manifest = _load_manifest(resolved_root / "manifest.yaml")

    priors: dict[str, PrivateDocPrior] = {}
    for session_ref in manifest.sessions:
        priors[session_ref.session_id] = PrivateDocPrior(
            source_kind="raw_session",
            memo_as_of=None,
        )

    for memo_ref in manifest.memos:
        memo_path = resolved_root / memo_ref.filename
        if not memo_path.exists():
            raise ValueError(f"private corpus memo file does not exist: {memo_ref.filename}")
        memo = _load_manual_memo(memo_path)
        priors[memo.frontmatter.memo_id] = PrivateDocPrior(
            source_kind="memo",
            memo_as_of=memo.frontmatter.as_of,
        )

    return priors
```

- [ ] **Step 4: Run the doc-priors tests to verify they pass**

Run:

```bash
uv run python -m pytest tests/unit/test_private_doc_priors.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit the doc-priors loader**

```bash
git add apps/api/src/prescient_benchmark/corpus/private_doc_priors.py tests/unit/test_private_doc_priors.py
git commit -m "feat: load private retrieval document priors"
```

### Task 2: Add broad-direction query detection and reranking

**Files:**
- Create: `apps/api/src/prescient_benchmark/retrieval/broad_direction_reranking.py`
- Create: `tests/unit/test_broad_direction_reranking.py`
- Modify: `apps/api/src/prescient_benchmark/retrieval/models.py`

- [ ] **Step 1: Write the failing reranker tests**

```python
from datetime import date

from prescient_benchmark.corpus.private_doc_priors import PrivateDocPrior
from prescient_benchmark.retrieval.broad_direction_reranking import (
    is_broad_direction_query,
    rerank_broad_direction_results,
)
from prescient_benchmark.retrieval.models import SearchResult


def test_is_broad_direction_query_matches_unscoped_direction_prompt() -> None:
    assert is_broad_direction_query("What is Prescient OS being built to do?") is True


def test_is_broad_direction_query_rejects_project_scoped_prompt() -> None:
    assert is_broad_direction_query("What are we doing on retrieval benchmarking?") is False


def test_rerank_broad_direction_results_prefers_newer_memo_over_older_memo_when_scores_are_close() -> None:
    results = [
        SearchResult(chunk_id="old:0", doc_id="old", title="Old", text="direction", score=10.0),
        SearchResult(chunk_id="new:0", doc_id="new", title="New", text="direction", score=9.9),
    ]
    priors = {
        "old": PrivateDocPrior(source_kind="memo", memo_as_of=date(2026, 4, 12)),
        "new": PrivateDocPrior(source_kind="memo", memo_as_of=date(2026, 4, 16)),
    }

    reranked = rerank_broad_direction_results(
        query="What is Prescient OS being built to do?",
        results=results,
        doc_priors=priors,
    )

    assert [result.doc_id for result in reranked] == ["new", "old"]


def test_rerank_broad_direction_results_prefers_memo_over_raw_when_scores_are_close() -> None:
    results = [
        SearchResult(chunk_id="raw:0", doc_id="raw", title="Raw", text="direction", score=10.0),
        SearchResult(chunk_id="memo:0", doc_id="memo", title="Memo", text="direction", score=9.95),
    ]
    priors = {
        "raw": PrivateDocPrior(source_kind="raw_session", memo_as_of=None),
        "memo": PrivateDocPrior(source_kind="memo", memo_as_of=date(2026, 4, 16)),
    }

    reranked = rerank_broad_direction_results(
        query="What is Prescient OS being built to do?",
        results=results,
        doc_priors=priors,
    )

    assert [result.doc_id for result in reranked] == ["memo", "raw"]


def test_rerank_broad_direction_results_keeps_raw_first_when_relevance_gap_is_large() -> None:
    results = [
        SearchResult(chunk_id="raw:0", doc_id="raw", title="Raw", text="exact answer", score=15.0),
        SearchResult(chunk_id="memo:0", doc_id="memo", title="Memo", text="weak overlap", score=9.0),
    ]
    priors = {
        "raw": PrivateDocPrior(source_kind="raw_session", memo_as_of=None),
        "memo": PrivateDocPrior(source_kind="memo", memo_as_of=date(2026, 4, 16)),
    }

    reranked = rerank_broad_direction_results(
        query="What is Prescient OS being built to do?",
        results=results,
        doc_priors=priors,
    )

    assert [result.doc_id for result in reranked] == ["raw", "memo"]
```

- [ ] **Step 2: Run the reranker tests to verify they fail**

Run:

```bash
uv run python -m pytest tests/unit/test_broad_direction_reranking.py -q
```

Expected: FAIL with `ModuleNotFoundError` for `broad_direction_reranking`.

- [ ] **Step 3: Add reranker support and optional adjusted score field**

```python
# apps/api/src/prescient_benchmark/retrieval/models.py
from pydantic import BaseModel


class SearchResult(BaseModel):
    chunk_id: str
    doc_id: str
    title: str
    text: str
    score: float
    adjusted_score: float | None = None
```

```python
# apps/api/src/prescient_benchmark/retrieval/broad_direction_reranking.py
from __future__ import annotations

from datetime import date
import re

from prescient_benchmark.corpus.private_doc_priors import PrivateDocPrior
from prescient_benchmark.retrieval.models import SearchResult

_DIRECTION_PATTERNS = (
    re.compile(r"^what is prescient os being built to do\??$"),
    re.compile(r"^what are we building\??$"),
    re.compile(r"^what is the product direction\??$"),
    re.compile(r"^what is the company direction\??$"),
    re.compile(r"^what is prescient os\??$"),
    re.compile(r"^what is the current direction\??$"),
)
_EXCLUSION_TERMS = (
    "today",
    "currently",
    "april",
    "2026",
    "this week",
    "latest",
    "retrieval",
    "benchmark",
    "peloton",
    "question set",
    "evidence key",
    "chat import",
    "snapshot",
    "top-up",
    "ke-first pivot",
    "operator-first",
    "april 12",
    "april 16",
    "what's going on with",
    "status",
    "progress",
    "working on",
    "specific to",
)
_MEMO_AUTHORITY_BONUS = 0.75
_MAX_RECENCY_BONUS = 0.35
_RECENCY_DENOMINATOR_DAYS = 30


def is_broad_direction_query(query: str) -> bool:
    normalized = " ".join(query.lower().strip().split())
    if any(term in normalized for term in _EXCLUSION_TERMS):
        return False
    return any(pattern.fullmatch(normalized) for pattern in _DIRECTION_PATTERNS)


def rerank_broad_direction_results(
    *,
    query: str,
    results: list[SearchResult],
    doc_priors: dict[str, PrivateDocPrior],
) -> list[SearchResult]:
    if not is_broad_direction_query(query):
        return results

    memo_dates = [prior.memo_as_of for prior in doc_priors.values() if prior.memo_as_of is not None]
    newest_memo_date = max(memo_dates) if memo_dates else None

    reranked: list[SearchResult] = []
    for result in results:
        prior = doc_priors.get(result.doc_id)
        adjusted_score = result.score
        if prior is not None and prior.source_kind == "memo":
            adjusted_score += _MEMO_AUTHORITY_BONUS
            adjusted_score += _memo_recency_bonus(prior.memo_as_of, newest_memo_date)
        reranked.append(result.model_copy(update={"adjusted_score": adjusted_score}))

    return sorted(
        reranked,
        key=lambda item: (item.adjusted_score if item.adjusted_score is not None else item.score, item.score),
        reverse=True,
    )


def _memo_recency_bonus(memo_as_of: date | None, newest_memo_date: date | None) -> float:
    if memo_as_of is None or newest_memo_date is None:
        return 0.0
    age_delta = (newest_memo_date - memo_as_of).days
    scaled = max(0.0, 1.0 - (age_delta / _RECENCY_DENOMINATOR_DAYS))
    return scaled * _MAX_RECENCY_BONUS
```

- [ ] **Step 4: Run the reranker tests to verify they pass**

Run:

```bash
uv run python -m pytest tests/unit/test_broad_direction_reranking.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit the reranker**

```bash
git add apps/api/src/prescient_benchmark/retrieval/models.py apps/api/src/prescient_benchmark/retrieval/broad_direction_reranking.py tests/unit/test_broad_direction_reranking.py
git commit -m "feat: add broad direction query reranking"
```

### Task 3: Integrate broad-direction reranking into the private baseline runner

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/orchestrator.py`
- Modify: `tests/unit/test_eval_orchestrator.py`

- [ ] **Step 1: Write the failing baseline-runner regression test for `q_private_006`**

```python
def test_run_private_retrieval_baseline_improves_q_private_006_with_broad_direction_reranking(
    tmp_path: Path,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    monkeypatch.chdir(tmp_path)
    fixed_now = datetime(2026, 4, 18, 12, 0, 0, tzinfo=UTC)
    monkeypatch.setattr("prescient_benchmark.eval.orchestrator._now_utc", lambda: fixed_now)

    run_meta = init_eval_run(
        system_id="local_rag",
        iteration_id="local_rag_v0.1",
        change_type="baseline",
        change_note="q_private_006 reranking regression",
        corpus_version="prescient-os-chats-2026-04-18-with-retrieval-only-turn",
        question_set_version="prescient_private_v1",
        grader_prompt_version="grader_v1",
    )

    question = SimpleNamespace(
        id="q_private_006",
        prompt="What is Prescient OS being built to do?",
        source_mix="mixed",
        expected_docs=[
            "prescient-os-operator-first-redesign-2026-04-12",
            "ke-first-pivot-2026-04-16",
            "retrieval-benchmark-narrowing-2026-04-16",
        ],
    )
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator.load_question_set",
        lambda _path: SimpleNamespace(questions=[question]),
    )
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator.load_evidence_key_set",
        lambda _path: load_evidence_key_set(Path("eval/evidence_keys/prescient_private_v1.yaml")),
    )
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator._private_corpus_snapshot_root",
        lambda corpus_version, data_root=None: Path("/home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-retrieval-only-turn"),
    )
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator.load_private_corpus_documents",
        lambda _snapshot_root: [
            InMemoryDocument(doc_id="prescient-os-operator-first-redesign-2026-04-12", title="Operator First", text="first touchpoint each morning"),
            InMemoryDocument(doc_id="ke-first-pivot-2026-04-16", title="KE First", text="knowledge engine became the primary product thesis"),
            InMemoryDocument(doc_id="retrieval-benchmark-narrowing-2026-04-16", title="Benchmark Narrowing", text="retrieval-first, not curation-first"),
            InMemoryDocument(doc_id="random-chat", title="Random Chat", text="What is today's date?"),
        ],
    )
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator._source_kind_by_doc_id_from_private_snapshot",
        lambda _snapshot_root: {
            "prescient-os-operator-first-redesign-2026-04-12": "memo",
            "ke-first-pivot-2026-04-16": "memo",
            "retrieval-benchmark-narrowing-2026-04-16": "memo",
            "random-chat": "raw_session",
        },
    )
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator.load_private_doc_priors",
        lambda _snapshot_root: {
            "prescient-os-operator-first-redesign-2026-04-12": PrivateDocPrior(source_kind="memo", memo_as_of=date(2026, 4, 12)),
            "ke-first-pivot-2026-04-16": PrivateDocPrior(source_kind="memo", memo_as_of=date(2026, 4, 16)),
            "retrieval-benchmark-narrowing-2026-04-16": PrivateDocPrior(source_kind="memo", memo_as_of=date(2026, 4, 16)),
            "random-chat": PrivateDocPrior(source_kind="raw_session", memo_as_of=None),
        },
    )
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator.ensure_private_retrieval_index",
        lambda **_kwargs: None,
    )
    monkeypatch.setattr(
        "prescient_benchmark.eval.orchestrator.search_index",
        lambda **_kwargs: [
            SearchResult(chunk_id="prescient-os-operator-first-redesign-2026-04-12:0", doc_id="prescient-os-operator-first-redesign-2026-04-12", title="Operator First", text="first touchpoint each morning", score=10.0),
            SearchResult(chunk_id="random-chat:0", doc_id="random-chat", title="Random Chat", text="What is today's date?", score=9.8),
            SearchResult(chunk_id="ke-first-pivot-2026-04-16:0", doc_id="ke-first-pivot-2026-04-16", title="KE First", text="knowledge engine became the primary product thesis", score=9.7),
            SearchResult(chunk_id="retrieval-benchmark-narrowing-2026-04-16:0", doc_id="retrieval-benchmark-narrowing-2026-04-16", title="Benchmark Narrowing", text="retrieval-first, not curation-first", score=9.6),
        ],
    )

    run_private_retrieval_baseline(
        run_id=run_meta.run_id,
        data_root=tmp_path / "prescient_os_data",
        top_k=4,
        opensearch_client=object(),
    )

    score = read_yaml_model(
        EvalRunPaths.from_run_id(run_meta.run_id).retrieval_score_path("q_private_006"),
        RetrievalQuestionScore,
    )

    assert score.metrics.required_doc_coverage > 0.2
    assert score.metrics.claim_coverage > 0.3333333333333333
    assert score.metrics.noise_ratio < 0.9
```

- [ ] **Step 2: Run the targeted orchestration test to verify it fails**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_orchestrator.py::test_run_private_retrieval_baseline_improves_q_private_006_with_broad_direction_reranking -q
```

Expected: FAIL because the baseline runner does not yet load doc priors or rerank broad direction queries.

- [ ] **Step 3: Integrate doc priors and reranking into the baseline runner**

```python
# apps/api/src/prescient_benchmark/eval/orchestrator.py
from prescient_benchmark.corpus.private_doc_priors import load_private_doc_priors
from prescient_benchmark.retrieval.broad_direction_reranking import rerank_broad_direction_results
```

```python
snapshot_root = _private_corpus_snapshot_root(run_meta.corpus_version, data_root)
documents = load_private_corpus_documents(snapshot_root)
source_kind_by_doc_id = _source_kind_by_doc_id_from_private_snapshot(snapshot_root)
doc_priors = load_private_doc_priors(snapshot_root)
```

```python
search_results = search_index(
    client=client,
    index_name=index_name,
    query=question.prompt,
    top_k=top_k,
)
search_results = rerank_broad_direction_results(
    query=question.prompt,
    results=search_results,
    doc_priors=doc_priors,
)
retrieved_evidence = _retrieved_evidence_from_search_results(search_results)
```

Keep the existing mixed-source selector path intact for `memo_plus_raw` questions. The broad-direction reranker should run only on the plain candidate list and should not replace the mixed-source selection logic.

- [ ] **Step 4: Add non-regression assertions for the existing mixed-source questions**

Add to `tests/unit/test_eval_orchestrator.py` assertions that the existing `q_private_002` and `q_private_004` tests still keep:

```python
assert score.metrics.required_source_coverage == 1.0
assert score.metrics.claim_coverage == 1.0
assert score.metrics.failure is False
```

Use the existing mixed-source test fixtures rather than creating parallel duplicates.

- [ ] **Step 5: Re-run the focused orchestration slice**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_orchestrator.py -k "q_private_006 or q_private_002 or q_private_004" -q
```

Expected: PASS.

- [ ] **Step 6: Commit the baseline-runner integration**

```bash
git add apps/api/src/prescient_benchmark/eval/orchestrator.py tests/unit/test_eval_orchestrator.py
git commit -m "feat: rerank broad direction queries in private baseline"
```

### Task 4: Add CLI-level regression and finish verification

**Files:**
- Modify: `tests/integration/test_eval_harness_cli.py`
- Modify: `tests/unit/test_broad_direction_reranking.py`
- Modify: `tests/unit/test_private_doc_priors.py`

- [ ] **Step 1: Add the failing CLI regression for `q_private_006`**

```python
def test_run_private_retrieval_baseline_command_improves_q_private_006_broad_direction_order(
    tmp_path: Path,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    monkeypatch.chdir(tmp_path)
    fixed_now = datetime(2026, 4, 18, 20, 0, tzinfo=UTC)
    monkeypatch.setattr("prescient_benchmark.eval.orchestrator._now_utc", lambda: fixed_now)

    run_meta = init_eval_run(
        system_id="local_rag",
        iteration_id="local_rag_v0.1",
        change_type="baseline",
        change_note="broad direction reranking command",
        corpus_version="prescient-os-chats-2026-04-18-with-retrieval-only-turn",
        question_set_version="prescient_private_v1",
        grader_prompt_version="grader_v1",
    )

    # Reuse the same monkeypatch pattern as the existing mixed-source CLI baseline test,
    # but set the target question to q_private_006 and return a candidate list where
    # the older operator-first memo and noisy raw chat outrank the KE-first artifacts
    # before reranking.

    runner = CliRunner()
    result = runner.invoke(
        app,
        [
            "run-private-retrieval-baseline",
            "--run-id",
            run_meta.run_id,
            "--data-root",
            str(tmp_path / "prescient_os_data"),
            "--top-k",
            "4",
            "--json",
        ],
    )

    assert result.exit_code == 0

    run_paths = EvalRunPaths.from_run_id(run_meta.run_id)
    score = read_yaml_model(run_paths.retrieval_score_path("q_private_006"), RetrievalQuestionScore)
    assert score.metrics.required_doc_coverage > 0.2
    assert score.metrics.claim_coverage > 0.3333333333333333
    assert score.metrics.noise_ratio < 0.9
```

- [ ] **Step 2: Run the focused CLI regression to verify it fails**

Run:

```bash
uv run python -m pytest tests/integration/test_eval_harness_cli.py::test_run_private_retrieval_baseline_command_improves_q_private_006_broad_direction_order -q
```

Expected: FAIL until the reranking path is wired through the command flow.

- [ ] **Step 3: Re-run the focused mixed retrieval suite**

Run:

```bash
uv run python -m pytest tests/unit/test_private_doc_priors.py tests/unit/test_broad_direction_reranking.py tests/unit/test_eval_orchestrator.py tests/integration/test_eval_harness_cli.py -q
```

Expected: PASS.

- [ ] **Step 4: Run the full unit and integration suite**

Run:

```bash
uv run python -m pytest tests/unit tests/integration -q
```

Expected: PASS.

- [ ] **Step 5: Run diff hygiene checks**

Run:

```bash
git diff --check
```

Expected: no output.

- [ ] **Step 6: Commit, sync bead, and push**

```bash
bd close prescient_os-2tt --reason "Added broad-direction reranking with authority and mild memo recency priors; improved q_private_006 without regressing mixed-source retrieval."
git add apps/api/src/prescient_benchmark/corpus/private_doc_priors.py apps/api/src/prescient_benchmark/retrieval/models.py apps/api/src/prescient_benchmark/retrieval/broad_direction_reranking.py apps/api/src/prescient_benchmark/eval/orchestrator.py tests/unit/test_private_doc_priors.py tests/unit/test_broad_direction_reranking.py tests/unit/test_eval_orchestrator.py tests/integration/test_eval_harness_cli.py
git commit -m "feat: rerank broad product direction queries"
git pull --rebase
bd dolt push
git push
```

## Self-review

- Spec coverage:
  - broad-query heuristic gating: Task 2
  - memo authority + mild memo recency: Task 2
  - no index schema change: Tasks 1-3 keep priors out of OpenSearch mappings
  - q_private_006 material improvement: Tasks 3-4
  - non-regression for q_private_002 / q_private_004: Task 3
- Placeholder scan:
  - no `TBD`, `TODO`, or unresolved implementation placeholders
  - every code-changing step includes concrete code
- Type consistency:
  - `PrivateDocPrior` is introduced in Task 1 and reused consistently in Task 2 and Task 3
  - `SearchResult.adjusted_score` is introduced before reranker usage
  - `rerank_broad_direction_results(...)` naming is consistent across tests and integration
