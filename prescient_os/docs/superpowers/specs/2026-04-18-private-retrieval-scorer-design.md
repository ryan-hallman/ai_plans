# Private Retrieval Scorer Design

## Goal

Define the first deterministic retrieval scorer for the private PrescientOS
benchmark.

This scorer should evaluate whether a captured retrieval bundle contains the
evidence required by the private evidence keys. It should produce structured
retrieval metrics without generating answers, grading prose, or invoking any
LLM-based evaluation.

## Why This Slice Exists

The benchmark already has:

- a private corpus snapshot
- a private question set
- explicit evidence keys

What it does not yet have is a deterministic scoring layer that turns retrieved
evidence into benchmark results.

That gap matters because the benchmark has explicitly shifted away from broad
synthesis grading as the primary score. The current question is no longer “did a
model write a good answer?” It is “did retrieval surface the evidence needed to
answer correctly?”

The scorer should therefore become the first primary evaluation surface for the
private corpus.

## Scope

The first scorer should:

- load a question's evidence key
- evaluate an already-captured `retrieved_evidence` bundle for that question
- emit deterministic retrieval metrics
- write retrieval score records alongside the existing eval run artifacts
- support the current private question set and evidence-key schema

The first scorer should not:

- run retrieval itself
- generate answers
- evaluate prose quality
- invoke model graders
- introduce semantic or fuzzy matching logic

This keeps the scoring layer independent from any one retriever and makes it
easy to replay prior retrieval bundles against improved evidence keys.

## Recommended Architecture

The scorer should be implemented as a pure evaluation layer over captured
retrieval bundles.

Recommended sequence:

1. a retriever produces `retrieved_evidence`
2. the answer record stores that bundle as it does today
3. the scorer reads the question's evidence key and the stored retrieval bundle
4. the scorer writes a deterministic retrieval score record
5. run summary logic aggregates those score records into a retrieval summary

This approach is preferable to embedding scoring inside one retriever because it
separates scoring failures from retrieval failures. If the scorer and retriever
change independently, the benchmark remains interpretable.

## Input Shape

The first scorer should read:

- `question_set_version`
- `corpus_version`
- `question_id`
- `retrieved_evidence[]`
  - `doc_id`
  - `chunk_id`
  - `locator`
  - `text`
  - `retrieval_rank`

This input should come from the existing answer record shape rather than a new
capture format.

The scorer should also load the corresponding evidence key fields:

- `required_docs`
- `required_sources`
- `required_claims`
- `acceptable_evidence`
- claim-level role metadata used by resolution rules
- `minimum_sufficient_set`
- `resolution_rule`
- optional `notes_on_conflicts`

## Output Shape

The scorer should emit one retrieval score record per question with:

- `question_id`
- `run_id`
- `system_id`
- `iteration_id`
- `question_set_version`
- `corpus_version`
- `scorer_version`
- `metric_schema_version`
- `metrics`
  - `required_doc_coverage`
  - `required_source_coverage`
  - `claim_coverage`
  - `minimum_sufficient_set_satisfied`
  - `first_required_evidence_rank`
  - `noise_count`
  - `noise_ratio`
  - `resolution_rule_satisfied`
- `matched_claim_ids`
- `missing_claim_ids`
- `matched_doc_ids`
- `noise_doc_ids`

Recommended write path:

- `eval/runs/<run_id>/retrieval_scores/<question_id>.yaml`

This keeps retrieval scoring parallel to the existing answer and grader
artifacts without replacing them.

## Matching Rules

The first scorer should stay deterministic and conservative.

Recommended rules:

- `required_doc_coverage`
  - matched if any retrieved evidence item has the same `doc_id`

- `required_source_coverage`
  - same logic as required-doc coverage, but against `required_sources`

- `claim_coverage`
  - retrieved evidence should first be grouped by `doc_id` and ordered by
    `retrieval_rank`
  - for each `doc_id`, the scorer should concatenate the retrieved text for that
    document in rank order
  - a claim is matched if any acceptable evidence entry:
    - has the same `doc_id`
    - and the concatenated text for that `doc_id` contains the `excerpt`
      substring exactly

This keeps matching deterministic while reducing accidental misses caused by one
chunk boundary cutting through an excerpt span. It still intentionally assumes
that the retriever surfaced enough contiguous text for the excerpt to exist in
the retrieved bundle.

- `minimum_sufficient_set_satisfied`
  - true only if every claim ID in `minimum_sufficient_set` is matched

- `first_required_evidence_rank`
  - the smallest `retrieval_rank` among any matched required evidence item
  - `null` if nothing required matched

- `noise_count`
  - retrieved evidence items whose `doc_id` is outside `required_docs`

- `noise_ratio`
  - `noise_count / total_retrieved_items`
  - `0.0` if no evidence items are present

- `failure`
  - a question counts as a deterministic retrieval failure if either:
    - `minimum_sufficient_set_satisfied` is false
    - `resolution_rule_satisfied` is false

The scorer should not do:

- approximate substring matching
- markdown normalization
- semantic similarity
- excerpt paraphrase detection

If an excerpt cannot be matched literally, the evidence key should be fixed.

## Resolution Rule Checks

The first scorer should support only deterministic assertions over matched
claims and sources.

To make the rule checks real rather than decorative, the evidence-key schema
should carry explicit claim-level metadata used only by the scorer:

- `claim_role`
  - `current`
  - `superseded`
  - `period_a`
  - `period_b`
  - `supporting`

The scorer should also resolve a deterministic source kind for every matched
`doc_id`:

- `memo`
- `raw_session`

For the private corpus, source kind should be derived from the private snapshot
manifest and memo/session file lists rather than inferred from slug shape.

Recommended first-pass rule behavior:

- `latest_wins`
  - requires the minimum sufficient set

- `must_compare_periods`
  - requires the minimum sufficient set
  - requires matched claims from both `period_a` and `period_b`

- `must_surface_conflict`
  - requires the minimum sufficient set
  - requires matched claims from both `superseded` and `current`

- `memo_plus_raw`
  - requires the minimum sufficient set
  - requires at least one matched memo source and one matched raw session
    source

- `raw_only`
  - requires the minimum sufficient set
  - requires all matched evidence to come from raw session sources, not memos

This logic should be explicit in code and tied to the current private corpus
conventions rather than inferred from prose.

## Summary Integration

Run summary generation should gain a second deterministic block:

- `retrieval_summary`
  - average required-doc coverage
  - average required-source coverage
  - average claim coverage
  - rate of `minimum_sufficient_set_satisfied`
  - `hit_rate`
    - share of questions with any matched required evidence
  - `mrr`
    - reciprocal-rank aggregate over `first_required_evidence_rank`
  - average `noise_ratio`
  - rate of `resolution_rule_satisfied`
  - per-question failure list
  - most-missed claim IDs across the run

This retrieval summary should be kept separate from the existing model-grader
summary so retrieval tuning has a clean optimization surface.

If a run has no retrieval score records yet, summary generation should tolerate
that and omit the retrieval summary block rather than failing the whole run.

## Testing Requirements

The first implementation should be test-driven and include:

- unit tests for each matching rule
- unit tests for each resolution rule
- unit tests for empty or all-noise retrieval bundles
- unit tests for score-record writing and summary aggregation
- at least one integration-style test over the current private evidence keys and
  a synthetic retrieval bundle
- at least one frozen regression test using a known retrieval bundle against the
  real `prescient_private_v1` evidence keys

The scorer should be treated as benchmark infrastructure, so deterministic tests
matter more than automation breadth.

## First-Pass Implementation Target

Keep the first implementation pass narrow.

Implement:

- minimal evidence-key schema expansion for claim-role metadata
- retrieval score record model(s)
- pure scoring function(s) over `retrieved_evidence`
- retrieval-score file writing
- retrieval summary aggregation
- CLI or orchestration hook that scores already-recorded answer bundles

Do not implement yet:

- retriever execution
- end-to-end benchmark automation
- prose answer judging
- broader locator typing or fuzzy matching
- schema expansion beyond the minimum needed for deterministic resolution-rule
  checks

The goal of this slice is to make retrieval quality measurable over the private
corpus with deterministic evidence-key scoring.
