# Private Retrieval Baseline Runner Design

## Goal

Add a thin retrieval-only baseline runner for the private corpus that:

- executes the current OpenSearch single-pass retriever over the private question set
- captures retrieval results without inventing placeholder answers
- scores those retrieval bundles with the deterministic retrieval scorer
- writes a run summary
- exposes the same execution path through both CLI and MCP
- records enough retriever provenance that the baseline does not silently drift

This slice is about producing the first repeatable private-corpus retrieval baseline. It is not about synthesis, model grading, reranking, or multi-pass retrieval.

## Why this exists now

The repo already has:

- private-corpus snapshots and a loader
- a private question set and evidence keys
- a deterministic retrieval scorer over captured bundles
- run summaries that can aggregate retrieval metrics

What is missing is the thin execution path that actually runs retrieval for a run, records those bundles as first-class artifacts, and drives the scorer over them. That gap blocks the first real benchmark pass.

## Scope

This slice will:

- keep retrieval-only baselines under `eval/runs/<run_id>/`
- add retrieval-only records under `retrievals/<question_id>.yaml`
- add one OpenSearch-backed baseline runner over the private corpus
- add one CLI command to invoke that runner
- add one MCP tool that invokes the same orchestration path
- introduce a shared internal `RetrievalBundle` shape that both answer-backed and retrieval-only records can project into
- feed the deterministic scorer from retrieval records rather than from answer records
- write `retrieval_scores/*.yaml` and `summary.yaml`

This slice will not:

- generate answers
- reuse `answers/*.yaml` with placeholder answer fields
- add multi-pass retrieval
- add reranking changes
- add model-based grading to the baseline flow
- create a second top-level run tree outside `eval/runs/`

## Run format

Retrieval-only baselines stay under the existing eval run tree.

New artifact path:

- `eval/runs/<run_id>/retrievals/<question_id>.yaml`

Each retrieval record should include:

- `question_id`
- `run_id`
- `system_id`
- `iteration_id`
- `question_set_version`
- `corpus_version`
- `retriever_id`
- `retriever_version`
- `analyzer`
- `index_settings_hash`
- `index_name`
- `top_k`
- `query`
- `latency_ms`
- `retrieved_evidence`
  - `doc_id`
  - `chunk_id`
  - `locator`
  - `text`
  - `retrieval_rank`
- `created_at`

This keeps retrieval capture separate from answer generation and preserves a clean path to add real answer generation later.

For this first baseline:

- `retriever_id` should be `opensearch_single_pass`
- `retriever_version` should identify the current retriever implementation, starting with `opensearch_single_pass_v1`
- `analyzer` should be pinned to OpenSearch's default `standard` analyzer unless the retriever implementation changes explicitly
- `index_settings_hash` should capture the effective index settings/mappings used for retrieval

`locator` should be explicit and deterministic for retrieval records:

- for baseline retrieval records, `locator` should be `chunk:<ordinal>`
- the runner should derive that ordinal from the current chunk contract rather than invent section names

The first baseline runner depends on the current `semantic_chunks()` contract:

- `chunk_id = <doc_id>:<ordinal>`
- `ordinal` increases monotonically within a document

If a retrieved chunk does not satisfy that contract, the runner should fail rather than emit an ambiguous retrieval record.

## Retrieval execution model

The first runner should be deliberately thin and deterministic.

Behavior:

1. Read run metadata from `eval/runs/<run_id>/meta.yaml`.
2. Load the question set for `question_set_version`.
3. Resolve the private snapshot from `corpus_version`.
4. Load snapshot documents with `load_private_corpus_documents(...)`.
5. Index those documents into an OpenSearch index keyed by `corpus_version` plus `index_settings_hash`, not by `run_id`.
6. For each question:
   - use `question.prompt` as the retrieval query
   - run the existing OpenSearch single-pass retrieval path using the current `multi_match` query over `title^2` and `text`
   - capture the top `20` results by default
   - persist a retrieval record under `retrievals/<question_id>.yaml`
7. Run deterministic retrieval scoring over those retrieval records.
8. Write `retrieval_scores/<question_id>.yaml`.
9. Write `summary.yaml`.

Fixed defaults for the first baseline:

- `retriever_id: opensearch_single_pass`
- `retriever_version: opensearch_single_pass_v1`
- `analyzer: standard`
- `top_k: 20`

This first baseline should be treated as the current OpenSearch single-pass floor, not the ceiling for full-text retrieval quality. Later FTS work can tune analyzers, field weighting, phrase handling, and multi-pass retrieval without redefining what this first run measured.

Rationale for `top_k = 20`:

- preserves enough rank headroom to see whether required evidence exists but ranks too low
- provides a less quantized noise signal than `top_k = 10`
- allows later derived analysis for narrower cuts such as effective `top_5` or `top_10` behavior without rerunning capture

## Scoring integration

The retrieval scorer should read retrieval records directly rather than `answers/*.yaml`.

The scorer integration should not stay at an `AnswerRecord` boundary. This slice should introduce a shared internal `RetrievalBundle` shape and make the scorer consume that shape only.

Both:

- `AnswerRecord`
- retrieval-only records

should project into that common `RetrievalBundle`.

For this slice, the important requirement is that retrieval-only runs do not need fake answer text or citations just to be scored, and the scorer should not need special-case logic for multiple record types.

## CLI

Add one thin command:

- `run-private-retrieval-baseline`

Inputs:

- `run_id`
- optional `--data-root`
- optional `--top-k` defaulting to `20`
- optional `--json`

Behavior:

- execute retrieval capture
- execute deterministic retrieval scoring
- write a summary
- print a compact result such as counts and summary path
- when `--json` is set, return deterministic machine-readable output including summary metadata

This command should use the same orchestration function that MCP will call.

## MCP exposure

There is no existing project MCP server in this repo today. For this slice, add a small repo-local MCP server with one tool that calls the same orchestration function as the CLI command.

Protocol decisions for this slice:

- transport: stdio
- server scope: one repo-local server module for benchmark tooling
- tool count: one tool only in this slice
- request/response shape: JSON-serializable, matching the CLI orchestration result
- error shape: structured error payload with a stable message and failure category

Tool:

- `run_private_retrieval_baseline`

Inputs:

- `run_id`
- optional `data_root`
- optional `top_k` default `20`

Outputs:

- `run_id`
- `retrieval_count`
- `score_count`
- `summary_path`
- parsed `retrieval_summary` if present

This keeps CLI and MCP behavior aligned and avoids a shell-wrapper-only integration.

## Indexed State Sharing

Runs against the same `corpus_version` and the same effective index settings should share indexed state. The index boundary should reflect indexed corpus state, not run identity.

Recommended pattern:

- derive index name from `corpus_version` plus `index_settings_hash`
- reuse that index across runs when those inputs match exactly
- create a new index only when corpus version or effective retrieval settings change

This makes runs cheaper, more honest, and easier to compare. The run record should still persist the concrete `index_name` that was used.

## Error handling

The runner should fail fast when:

- the run metadata cannot be loaded
- the question set is missing
- the private snapshot for the run’s `corpus_version` cannot be resolved
- OpenSearch is unavailable
- indexing fails
- retrieval scoring fails

This first version should use no retry policy for transient OpenSearch failures. Fail fast is the explicit behavior.

Partial retrieval capture should not silently produce a successful run summary:

- partial `retrievals/*.yaml` files may remain on disk if a run fails mid-stream
- `summary.yaml` should only be written after retrieval capture and scoring both complete successfully
- a failed run should be obvious from the absence of a summary and the surfaced error

## Testing

Add or update tests for:

- retrieval-only record model validation
- orchestration over a small fixture-backed private snapshot
- at least one exact top-k doc/rank regression over deterministic fixture data
- CLI invocation of `run-private-retrieval-baseline`
- MCP tool invocation returning paths plus parsed summary metrics
- summary generation from retrieval-only records and retrieval scores
- failure behavior when the snapshot or OpenSearch dependency is unavailable

Use deterministic fixtures and keep the first runner limited to the existing single-pass OpenSearch path.

## Non-goals and follow-up

Explicitly deferred to later slices:

- multi-pass retrieval
- CLI filesystem baseline
- reranking changes
- answer generation over retrieval runs
- model-assisted grading for retrieval runs
- broader MCP benchmark control surface

The next phase after this runner is stable should be the first methodology comparison, most likely single-pass vs multi-pass retrieval over the same private question set and scorer.
