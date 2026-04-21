# Private Retrieval Baseline Runner Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a repeatable retrieval-only private-corpus baseline runner that captures top-k retrieval records, scores them with the deterministic scorer, writes retrieval summaries, and exposes the same orchestration path via CLI and MCP.

**Architecture:** Keep the new baseline flow inside the existing `eval/runs/<run_id>/` tree. Introduce a shared internal `RetrievalBundle` projection so both retrieval-only records and existing answer-backed records can feed the same scorer. Reuse the current OpenSearch single-pass retriever, but pin its provenance and index reuse to `corpus_version + index_settings_hash` rather than `run_id`.

**Tech Stack:** Python 3.12, Typer CLI, Pydantic models, OpenSearch, existing private snapshot loader, repo-local stdio MCP server.

**Parent Issue:** `prescient_os-cuh`

---

## File map

### Existing files to modify

- `apps/api/src/prescient_benchmark/eval/models.py`
  - add retrieval-only record model(s)
  - add shared internal retrieval bundle shape
- `apps/api/src/prescient_benchmark/eval/paths.py`
  - add `retrieval_path(question_id)` accessor
- `apps/api/src/prescient_benchmark/eval/orchestrator.py`
  - add retrieval-only baseline orchestration
  - refactor scorer entrypoint to consume shared bundles
- `apps/api/src/prescient_benchmark/eval/retrieval_scoring.py`
  - replace direct `AnswerRecord` dependency with `RetrievalBundle`
- `apps/api/src/prescient_benchmark/eval/summary.py`
  - keep retrieval summary working from retrieval score files when no answers/grades exist
- `apps/api/src/prescient_benchmark/retrieval/search.py`
  - add explicit `top_k` support
  - add index settings hash helper / reusable index naming helper if needed
- `apps/api/src/prescient_benchmark/cli.py`
  - add `run-private-retrieval-baseline`
  - add `--json` output mode
- `apps/api/src/prescient_benchmark/config.py`
  - add any minimal MCP config only if implementation needs it
- `pyproject.toml`
  - add MCP dependency only if the chosen stdio server implementation requires one

### New files to create

- `apps/api/src/prescient_benchmark/eval/retrieval_records.py`
  - narrow helpers for building retrieval records and projecting them to `RetrievalBundle`
- `apps/api/src/prescient_benchmark/mcp/server.py`
  - repo-local stdio MCP server with one tool
- `tests/unit/test_retrieval_records.py`
  - retrieval-only record and bundle projection tests
- `tests/unit/test_mcp_server.py`
  - tool schema / result shape tests

### Existing tests to modify

- `tests/unit/test_eval_models.py`
- `tests/unit/test_eval_orchestrator.py`
- `tests/unit/test_eval_summary.py`
- `tests/integration/test_eval_harness_cli.py`

---

### Task 1: Add retrieval-only record contracts and shared `RetrievalBundle`

**Files:**
- Create: `apps/api/src/prescient_benchmark/eval/retrieval_records.py`
- Modify: `apps/api/src/prescient_benchmark/eval/models.py`
- Modify: `apps/api/src/prescient_benchmark/eval/paths.py`
- Test: `tests/unit/test_retrieval_records.py`
- Test: `tests/unit/test_eval_models.py`

- [ ] **Step 1: Write failing model tests for retrieval-only records and bundle projection**

Add tests covering:
- retrieval record accepts:
  - `retriever_id`
  - `retriever_version`
  - `analyzer`
  - `index_settings_hash`
  - `index_name`
  - `top_k`
  - `query`
  - `latency_ms`
  - `retrieved_evidence`
- locator format is `chunk:<ordinal>`
- projection from retrieval record to `RetrievalBundle`
- projection from `AnswerRecord` to `RetrievalBundle`
- `EvalRunPaths.retrieval_path(question_id)` resolves under `retrievals/`

Example assertions:

```python
def test_retrieval_record_projects_to_bundle() -> None:
    record = RetrievalRecord.model_validate(
        {
            "question_id": "q_private_006",
            "run_id": "run_1",
            "system_id": "opensearch",
            "iteration_id": "opensearch_v1",
            "question_set_version": "prescient_private_v1",
            "corpus_version": "prescient-os-chats-2026-04-18-with-retrieval-only-turn",
            "retriever_id": "opensearch_single_pass",
            "retriever_version": "opensearch_single_pass_v1",
            "analyzer": "standard",
            "index_settings_hash": "abc123",
            "index_name": "prescient-private-abc123",
            "top_k": 20,
            "query": "What is Prescient OS being built to do?",
            "latency_ms": 42,
            "retrieved_evidence": [
                {
                    "doc_id": "ke-first-pivot-2026-04-16",
                    "chunk_id": "ke-first-pivot-2026-04-16:0",
                    "locator": "chunk:0",
                    "text": "The product direction became KE-first.",
                    "retrieval_rank": 1,
                }
            ],
            "created_at": "2026-04-20T12:00:00Z",
        }
    )

    bundle = retrieval_record_to_bundle(record)

    assert bundle.question_id == "q_private_006"
    assert bundle.retrieved_evidence[0].locator == "chunk:0"
```

- [ ] **Step 2: Run the targeted tests and confirm failure**

Run:

```bash
uv run python -m pytest tests/unit/test_retrieval_records.py tests/unit/test_eval_models.py -q
```

Expected: failures for missing `RetrievalRecord`, `RetrievalBundle`, or `retrieval_path`.

- [ ] **Step 3: Implement the minimal record and bundle contracts**

Add:
- `RetrievalBundle`
- `RetrievalRecord`
- `retrieval_record_to_bundle(record)`
- `answer_record_to_bundle(answer)`
- `EvalRunPaths.retrieval_path(question_id)`

Use focused helpers in `retrieval_records.py` instead of expanding `orchestrator.py`.

- [ ] **Step 4: Re-run the targeted tests**

Run:

```bash
uv run python -m pytest tests/unit/test_retrieval_records.py tests/unit/test_eval_models.py -q
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/eval/models.py \
        apps/api/src/prescient_benchmark/eval/retrieval_records.py \
        apps/api/src/prescient_benchmark/eval/paths.py \
        tests/unit/test_retrieval_records.py \
        tests/unit/test_eval_models.py
git commit -m "feat: add retrieval baseline record contracts"
```

---

### Task 2: Refactor the scorer to consume `RetrievalBundle`

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/retrieval_scoring.py`
- Modify: `apps/api/src/prescient_benchmark/eval/orchestrator.py`
- Test: `tests/unit/test_retrieval_scoring.py`
- Test: `tests/unit/test_eval_orchestrator.py`

- [ ] **Step 1: Write failing tests for shared scorer input**

Add tests proving:
- `score_retrieval_bundle(...)` accepts a `RetrievalBundle`
- answer-backed scoring still works through `answer_record_to_bundle(...)`
- retrieval-only scoring works through `retrieval_record_to_bundle(...)`

Example test shape:

```python
def test_score_retrieval_bundle_accepts_retrieval_bundle(evidence_key_question) -> None:
    bundle = RetrievalBundle.model_validate(
        {
            "question_id": "q_private_006",
            "retrieved_evidence": [
                {
                    "doc_id": "ke-first-pivot-2026-04-16",
                    "chunk_id": "ke-first-pivot-2026-04-16:0",
                    "locator": "chunk:0",
                    "text": "The product direction became KE-first.",
                    "retrieval_rank": 1,
                }
            ],
        }
    )

    score = score_retrieval_bundle(
        bundle=bundle,
        evidence_key=evidence_key_question,
        question_set_version="prescient_private_v1",
        source_kind_by_doc_id={"ke-first-pivot-2026-04-16": "memo"},
        scorer_version="retrieval_scorer_v1",
        metric_schema_version="retrieval_score_metrics_v1",
    )

    assert score.question_id == "q_private_006"
```

- [ ] **Step 2: Run the scorer-targeted tests and confirm failure**

Run:

```bash
uv run python -m pytest tests/unit/test_retrieval_scoring.py tests/unit/test_eval_orchestrator.py -q
```

Expected: failures because the scorer still expects `AnswerRecord`.

- [ ] **Step 3: Refactor the scorer and its callers**

Make `score_retrieval_bundle(...)` consume only `RetrievalBundle`, then update current callers to project records into that shared shape before scoring.

- [ ] **Step 4: Re-run the scorer-targeted tests**

Run:

```bash
uv run python -m pytest tests/unit/test_retrieval_scoring.py tests/unit/test_eval_orchestrator.py -q
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/eval/retrieval_scoring.py \
        apps/api/src/prescient_benchmark/eval/orchestrator.py \
        tests/unit/test_retrieval_scoring.py \
        tests/unit/test_eval_orchestrator.py
git commit -m "refactor: score retrieval from shared bundles"
```

---

### Task 3: Implement the OpenSearch private baseline runner and CLI

**Files:**
- Modify: `apps/api/src/prescient_benchmark/retrieval/search.py`
- Modify: `apps/api/src/prescient_benchmark/eval/orchestrator.py`
- Modify: `apps/api/src/prescient_benchmark/cli.py`
- Test: `tests/integration/test_eval_harness_cli.py`
- Test: `tests/unit/test_eval_orchestrator.py`

- [ ] **Step 1: Write failing tests for the runner orchestration**

Add tests that:
- materialize a small private snapshot
- initialize a run with `question_set_version="prescient_private_v1"`
- execute the new orchestration entrypoint
- assert:
  - `retrievals/<question_id>.yaml` files are written
  - `retrieval_scores/<question_id>.yaml` files are written
  - `summary.yaml` exists
  - `index_name` is keyed by `corpus_version + index_settings_hash`, not `run_id`
  - `top_k` defaults to `20`

Also add a CLI test for:
- `run-private-retrieval-baseline --run-id <id> --json`

- [ ] **Step 2: Run the integration and orchestrator tests to confirm failure**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_orchestrator.py tests/integration/test_eval_harness_cli.py -q
```

Expected: failures for missing runner entrypoint / CLI command.

- [ ] **Step 3: Implement the runner orchestration**

Add a function shaped like:

```python
def run_private_retrieval_baseline(
    *,
    run_id: str,
    data_root: Path | None = None,
    top_k: int = 20,
) -> dict[str, object]:
    ...
```

Required behavior:
- load run metadata and question set
- resolve private snapshot
- load private documents
- compute `index_settings_hash`
- reuse or create the shared index for `corpus_version + index_settings_hash`
- run OpenSearch single-pass retrieval for each question
- persist `RetrievalRecord`s under `retrievals/`
- call retrieval scoring
- write summary
- return a JSON-serializable result

Update `search_index(...)` to accept `top_k`.

- [ ] **Step 4: Implement the CLI command**

Add:

```python
@app.command("run-private-retrieval-baseline")
def run_private_retrieval_baseline_command(
    *,
    run_id: str = typer.Option(...),
    data_root: Path | None = typer.Option(None),
    top_k: int = typer.Option(20),
    json_output: bool = typer.Option(False, "--json"),
) -> None:
    ...
```

The command should echo JSON when `--json` is set and a compact human-readable summary otherwise.

- [ ] **Step 5: Re-run the targeted tests**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_orchestrator.py tests/integration/test_eval_harness_cli.py -q
```

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/retrieval/search.py \
        apps/api/src/prescient_benchmark/eval/orchestrator.py \
        apps/api/src/prescient_benchmark/cli.py \
        tests/unit/test_eval_orchestrator.py \
        tests/integration/test_eval_harness_cli.py
git commit -m "feat: run private retrieval baseline"
```

---

### Task 4: Add repo-local stdio MCP exposure

**Files:**
- Modify: `pyproject.toml`
- Create: `apps/api/src/prescient_benchmark/mcp/server.py`
- Test: `tests/unit/test_mcp_server.py`

- [ ] **Step 1: Write failing MCP tool tests**

Add tests that assert:
- the MCP server exposes one tool named `run_private_retrieval_baseline`
- the tool input schema includes:
  - `run_id`
  - optional `data_root`
  - optional `top_k`
- the tool result includes:
  - `run_id`
  - `retrieval_count`
  - `score_count`
  - `summary_path`
  - `retrieval_summary`
- error responses are structured and JSON-serializable

- [ ] **Step 2: Run the MCP tests and confirm failure**

Run:

```bash
uv run python -m pytest tests/unit/test_mcp_server.py -q
```

Expected: failures for missing MCP server module/tool.

- [ ] **Step 3: Implement the stdio MCP server**

If the implementation uses an MCP SDK, add the minimal dependency in `pyproject.toml`. Keep scope narrow:
- one server module
- one tool
- stdio transport only
- tool delegates to `run_private_retrieval_baseline(...)`

Example result payload shape:

```python
{
    "run_id": run_id,
    "retrieval_count": retrieval_count,
    "score_count": score_count,
    "summary_path": str(summary_path),
    "retrieval_summary": summary.get("retrieval_summary"),
}
```

- [ ] **Step 4: Re-run the MCP tests**

Run:

```bash
uv run python -m pytest tests/unit/test_mcp_server.py -q
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml \
        apps/api/src/prescient_benchmark/mcp/server.py \
        tests/unit/test_mcp_server.py
git commit -m "feat: expose retrieval baseline via mcp"
```

---

### Task 5: Tighten summary behavior and add retrieval-correctness regressions

**Files:**
- Modify: `apps/api/src/prescient_benchmark/eval/summary.py`
- Modify: `tests/unit/test_eval_summary.py`
- Modify: `tests/integration/test_eval_harness_cli.py`

- [ ] **Step 1: Write failing tests for retrieval-only summaries and exact ranking**

Add tests proving:
- a retrieval-only run can produce `summary.yaml` with `retrieval_summary` and without answer-grade requirements
- at least one deterministic fixture-backed run asserts exact top-k `doc_id` order and/or exact first-hit rank
- failed retrieval runs do not write `summary.yaml`

- [ ] **Step 2: Run the summary and integration tests to confirm failure**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_summary.py tests/integration/test_eval_harness_cli.py -q
```

Expected: failures until retrieval-only summary behavior is explicit.

- [ ] **Step 3: Implement the minimal summary fixes**

Keep existing mixed-mode behavior intact while ensuring retrieval-only baseline runs summarize cleanly from `retrieval_scores/*.yaml`.

- [ ] **Step 4: Re-run the targeted tests**

Run:

```bash
uv run python -m pytest tests/unit/test_eval_summary.py tests/integration/test_eval_harness_cli.py -q
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/eval/summary.py \
        tests/unit/test_eval_summary.py \
        tests/integration/test_eval_harness_cli.py
git commit -m "test: lock retrieval baseline summaries"
```

---

### Task 6: End-to-end verification

**Files:**
- No new source files
- Verify all touched files above

- [ ] **Step 1: Run the full focused suite**

Run:

```bash
uv run python -m pytest tests/unit/test_retrieval_records.py \
  tests/unit/test_eval_models.py \
  tests/unit/test_retrieval_scoring.py \
  tests/unit/test_eval_orchestrator.py \
  tests/unit/test_eval_summary.py \
  tests/unit/test_mcp_server.py \
  tests/integration/test_eval_harness_cli.py -q
```

Expected: PASS

- [ ] **Step 2: Run the full repo unit/integration suite**

Run:

```bash
uv run python -m pytest tests/unit tests/integration -q
```

Expected: PASS

- [ ] **Step 3: Run diff hygiene**

Run:

```bash
git diff --check
git status -sb
```

Expected:
- no whitespace or merge-marker issues
- only intended files modified

- [ ] **Step 4: Update beads state**

- close child issues if the work was decomposed
- close `prescient_os-cuh` only after code, tests, and docs all land

- [ ] **Step 5: Final commit if verification changed anything**

```bash
git add -A
git commit -m "chore: finalize private retrieval baseline runner"
```

Only do this if verification required a final code or test adjustment.

---

## Spec coverage check

- Retrieval-only records under `eval/runs/<run_id>/retrievals/`: covered by Task 1 and Task 3.
- Shared `RetrievalBundle` scorer integration: covered by Task 1 and Task 2.
- OpenSearch single-pass runner with `top_k = 20`: covered by Task 3.
- CLI command with `--json`: covered by Task 3.
- Stdio MCP tool: covered by Task 4.
- Retrieval-only summary behavior and failure semantics: covered by Task 5.
- Deterministic ranking regression coverage: covered by Task 5.

No uncovered spec requirement remains.
