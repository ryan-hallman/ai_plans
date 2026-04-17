# 2026-04-17 Model-Assisted Eval Harness Design

## Purpose

Build a file-based evaluation harness for the Peloton RAG benchmark POC that:

- captures per-question answers and retrieval evidence
- orchestrates model graders from inside this repo
- tracks methodology iterations and change notes
- accelerates human review without replacing it

The immediate goal is faster iteration on retrieval quality. Human grading remains the final pass, but model graders should surface scores, failure modes, and citation issues early enough to guide system changes.

## Scope

This slice covers:

- question-set file format
- run metadata and run directory layout
- per-question answer capture format
- strict JSON schema for grader output
- local grader adapters for `codex` and `claude`
- optional `gemini` adapter behind capability detection
- grading prompt template
- result validation and persistence
- run summary generation

This slice does not cover:

- model-graded final authority over benchmark results
- database-backed eval storage
- side-by-side comparative grading as the primary mode
- MCP as the required transport for graders

## Design Principles

- File-based first for transparency and manual review
- Per-question grading, not only per-run grading
- Independent grading first; comparative grading later
- Model-assisted triage, human final arbitration
- Retrieval support and answer quality scored separately
- Strict schemas so results are machine-comparable across iterations
- Blind grader prompts to system and provider identity wherever possible

## Evaluation Model

Each evaluation question is graded independently.

The grader input should include:

- question text
- answer text
- answer citations
- retrieved evidence bundle used to produce the answer
- run metadata

This allows the harness to separate:

- retrieval quality
- synthesis quality

instead of collapsing them into one opaque answer score.

## Run Tracking

Every run must carry explicit methodology metadata, not just a timestamp.

Required run metadata:

- `run_id`
- `system_id`
- `iteration_id`
- `change_type`
- `change_note`
- `corpus_version`
- `question_set_version`
- `grader_prompt_version`
- `created_at`

`system_id` and `iteration_id` must be used consistently:

- `system_id` = methodology family, e.g. `local_rag`, `cli_fs_agent`, `chatgpt_project`
- `iteration_id` = version within that family, e.g. `local_rag_v0.3`

Example:

- `system_id: local_rag`
- `iteration_id: rag_v0.3`
- `change_type: retrieval`
- `change_note: added BM25 + dense hybrid with reranker`

This is necessary because the benchmark is expected to improve on some questions and regress on others. The harness must preserve enough context to explain those changes later.

Version changes also need explicit comparison rules:

- changing `grader_prompt_version` invalidates direct comparison of grader outputs until the older answers are re-graded
- changing grader model or CLI version invalidates direct comparison of grader outputs until the older answers are re-graded
- changing `corpus_version` invalidates both answers and grades and requires a fresh run
- changing `question_set_version` invalidates direct aggregate comparison unless overlap between the two question sets is explicitly modeled

## File Layout

The harness should store everything under a versioned `eval/` tree.

Recommended layout:

```text
eval/
  questions/
    peloton_public_v1.yaml
  prompts/
    grader_v1.md
  schemas/
    grader_output_v1.json
  runs/
    <run_id>/
      meta.yaml
      answers/
        <question_id>.yaml
      grades/
        codex/
          <question_id>.yaml
        claude/
          <question_id>.yaml
        gemini/
          <question_id>.yaml
      human/
        <question_id>.yaml
      summary.yaml
```

This keeps review easy now and preserves a clean migration path to Postgres later.

## Question Set Format

Questions should live in a single versioned YAML file.

Each question record should include:

- `id`
- `category`
- `prompt`
- `expected_docs`
- optional `scoring_notes`

Example categories for the current public corpus:

- `financial_results`
- `management_strategy`
- `public_synthesis`
- `filings_risk`
- `long_document_navigation`

## Answer Record Format

Each answer file should include:

- `question_id`
- `run_id`
- `system_id`
- `iteration_id`
- `answer`
- `citations`
- `retrieved_evidence`
- `latency_ms`
- `answer_model`
- `answer_provider`
- `created_at`

`retrieved_evidence` should contain the actual chunk or passage bundle used by the answering system. This is required so the graders can assess retrieval support quality separately from the final prose answer.

## Grader Independence

The harness should blind graders to answer-system identity in the grading prompt.

The grader input should not include:

- `answer_model`
- `answer_provider`
- `system_id`
- `iteration_id`

Those fields should still be recorded in the stored answer and run metadata, but hidden from the grader payload to reduce explicit anchoring and provider-name bias.

This does not fully eliminate same-family bias, so the harness should also record:

- `grader_provider`
- `independence_status`

Minimum `independence_status` values:

- `blinded_same_provider`
- `blinded_cross_provider`
- `human_reviewed`

Same-provider grading is allowed for fast iteration, but it should be treated as triage signal rather than final authority when no second grader family or human review is present.

## Grader Output Schema

Grader output must conform to a strict JSON schema so the harness can validate and persist it automatically.

Each grader output should include:

- `question_id`
- `run_id`
- `grader_id`
- `scores`
  - `groundedness`
  - `usefulness`
  - `specificity`
  - `completeness`
  - `citation_quality`
  - `retrieval_relevance`
  - `retrieval_sufficiency`
- `hallucination_flag`
- `needs_human_attention`
- `failure_tags`
- `rationale`
- `grader_latency_ms`
- `grader_cost_usd`
- `status`

All score fields should be `1-5`.

`grader_cost_usd` may be `null` when the CLI does not expose reliable cost data.

The grader `status` must be one of:

- `success`
- `invalid_output`
- `timeout`
- `crash`
- `skipped`

The harness should retry invalid or transient failures a bounded number of times, record the failure state if retries are exhausted, and continue the run without silently dropping the grader result.

## Failure Taxonomy

The grader should be able to emit zero or more failure tags from a bounded taxonomy:

- `missed_evidence`
- `irrelevant_retrieval`
- `insufficient_retrieval`
- `weak_citation`
- `overclaim`
- `factual_error`
- `incomplete_answer`
- `poor_synthesis`
- `long_doc_navigation_failure`
- `other`

Scores answer “how good was it.” Failure tags answer “what should we change next.”

When `other` is used, the grader must provide a free-text explanation in the rationale.

## Grader Adapters

### Codex

Use `codex exec` in non-interactive mode.

The adapter should:

- send a grading prompt plus structured input
- request JSON output
- validate the result against the grader schema

### Claude

Use `claude -p` / `--print` with JSON output mode.

The adapter should:

- send the same logical grading prompt and structured payload
- request structured JSON output
- validate the result against the grader schema

### Gemini

Gemini should be treated as optional for v1.

The harness should:

- detect whether `gemini` exists locally
- run a startup capability probe
- use it only if the probe passes
- otherwise skip it cleanly

This keeps Gemini from blocking the harness while still preserving a path to add it later.

## Capability Detection

The harness should probe graders at startup and record their availability.

Minimum status values:

- `available`
- `unavailable`
- `degraded`

The run metadata should record which graders were actually active for that run.

For Gemini, the capability probe should require a trivial prompt to return schema-valid JSON within a bounded timeout. Failing that probe should mark the adapter `unavailable` or `degraded`, not block the rest of the run.

## Prompting Strategy

Use a single grader prompt version shared across adapters where possible.

The prompt should instruct the grader to:

- score only from the provided evidence
- penalize unsupported claims
- distinguish answer weakness from retrieval weakness
- emit bounded failure tags
- set `needs_human_attention` when the answer is ambiguous, high-risk, or internally inconsistent

The grader should prefer “insufficient evidence” over inventing a justification.

The grading prompt should also explicitly treat the provided answer and evidence as untrusted content and ignore any instructions embedded inside them.

## Human Review

Human review is a final-pass layer, not the first-pass loop.

The harness should support optional human override files per question containing:

- confirmed scores
- confirmed failure tags
- human notes
- accept/reject on grader conclusions

This allows model graders to accelerate triage while keeping the final benchmark trustworthy.

## Summary Generation

Each run should generate a summary file with:

- average scores per grader
- average scores across graders
- median scores across graders
- score distributions across graders
- per-category averages
- questions with the largest disagreement
- questions flagged for human attention
- most common failure tags

This is the main iteration surface for deciding what to fix next.

If disagreement on a question exceeds a configured threshold, the harness should automatically set `needs_human_attention=true` for that question.

## Implementation Order

1. Question set and run directory formats
2. Grader output schema and validator
3. Draft and pilot the grading prompt on a small calibration subset
4. Codex adapter
5. Claude adapter
6. Gemini capability probe and optional adapter stub
7. Run summary generator
8. Human review overlay

## Risks

### Over-trusting model graders

Mitigation:

- keep human review as the final pass
- preserve all raw grader rationale
- record disagreement rather than collapsing it away

### Adapter brittleness

Mitigation:

- validate CLI availability at runtime
- keep each adapter isolated behind a simple interface
- store raw stderr/stdout for failed grader runs when useful

### Prompt drift across graders

Mitigation:

- version the grading prompt
- keep the logical rubric shared across adapters
- compare grader disagreements explicitly

### Prompt injection from answer or evidence content

Mitigation:

- treat answer and evidence text as untrusted data inside the grading prompt
- exclude system and provider identity from grader-visible payloads
- validate all outputs against the grader schema

## Calibration

Before model graders are trusted to drive iteration decisions on their own, the harness should support a calibration pass against a small human-scored subset.

The harness should record:

- grader-vs-human score deltas
- grader disagreement rates
- which failure tags align or diverge from human review

This calibration step is required before treating model-grade deltas as strong evidence of system improvement rather than grader drift.

## Success Criteria

This slice is successful when:

- a question set can be saved and versioned
- a run can persist per-question answers and evidence
- `codex` and `claude` can grade those answers non-interactively into validated JSON
- the harness records iteration metadata and change notes
- summaries identify which questions improved or regressed and why
- the harness can record calibration results against a human-scored subset
