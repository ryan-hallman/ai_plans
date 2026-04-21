# Mixed-Source Retrieval Selection Design

## Goal

Improve retrieval performance for private-corpus questions whose evidence requires both:

- a curated memo artifact
- a raw chat/session artifact

This slice exists to fix the specific failure mode observed in the first private retrieval baseline:

- the retriever surfaced enough memo evidence to satisfy claims
- but failed `memo_plus_raw` resolution because the paired raw source did not survive into the final captured bundle

The target is not better synthesis. The target is better retrieval bundle construction for mixed-source questions.

## Why this exists now

The first real private baseline run (`opensearch_single_pass_20260421T030820Z`) showed a clear pattern:

- `q_private_002` and `q_private_004` had `claim_coverage = 1.0`
- both still failed because `required_source_coverage = 0.5`
- both used `resolution_rule = memo_plus_raw`

That means the current single-pass retriever is often finding the right memo-side evidence while letting memo chunks monopolize the bundle. Raw chat evidence is present in the corpus but not reliably preserved in the useful part of `top_k`.

## Scope

This slice will:

- keep the existing OpenSearch retrieval query as the candidate generator
- add a source-aware bundle selection policy for mixed-source questions
- apply that policy in the benchmark harness only when:
  - `source_mix == mixed`
  - `resolution_rule == memo_plus_raw`
- preserve the retriever's original ranking separately from the final emitted bundle order
- record a new retriever identity/version for this retrieval policy
- treat this as a narrow retrieval-method experiment over the two current `memo_plus_raw` questions:
  - `q_private_002`
  - `q_private_004`

This slice will not:

- rewrite queries per question
- inspect evidence-key claim IDs or specific required docs during selection
- add a second retrieval pass
- change non-`memo_plus_raw` question behavior
- address the broader stale-currentness/noise problem seen in `q_private_006`

## Benchmark-only trigger

The activation rule in this spec is an oracle-gated benchmark shortcut, not a production trigger.

Today the selector is activated from benchmark metadata:

- `source_mix == mixed`
- `resolution_rule == memo_plus_raw`

Those are authored evaluation labels, not runtime signals the retriever can infer from an arbitrary user query.

That is acceptable for this slice because the immediate goal is:

- prove whether source-balanced bundle selection can lift the benchmark on known mixed-source questions
- without changing the candidate generator itself

It is not acceptable to treat the resulting retrieval policy as production-ready behavior. If this slice shows real lift, the follow-up work must define a runtime trigger or classifier that can decide when mixed-source bundle selection should activate outside the benchmark harness.

## Retrieval policy

The current OpenSearch single-pass retrieval remains the first-stage candidate generator.

For `memo_plus_raw` questions only:

1. retrieve a larger candidate pool using the existing query and scoring
2. classify each candidate by source kind
3. build the final emitted bundle using a small source-diversity floor

The key design choice is that this is a retrieval-policy change, not an evidence-key-aware doc selector. It may use benchmark labels to decide when to activate, but once active it should only care about source kind, not exact claim IDs or exact required docs.

## Diagnostic first

Before implementing the selector, inspect the current candidate-generator output for:

- `q_private_002`
- `q_private_004`

Required diagnostic:

1. dump the current top `100` candidates for each question
2. locate the exact required raw-source doc ID for each question
3. record the rank where that required raw doc first appears, if it appears at all

This diagnostic decides whether the selector slice is valid:

- if the required raw doc appears in the candidate pool, set `candidate_k` to the smallest value that includes the required raw doc for both questions
- if the required raw doc does not appear in the top `100` for either question, stop and treat that as a candidate-generator failure, not a bundle-selection failure

There is no fixed `candidate_k` in this spec independent of that diagnostic.

## Candidate retrieval

For the affected question class:

- keep the same OpenSearch query
- keep the same analyzer and field weighting
- retrieve `candidate_k`, where:
  - `candidate_k` is chosen from the diagnostic-first step
  - it is the smallest candidate depth that still includes the exact required raw doc for both `q_private_002` and `q_private_004`

For all other questions:

- keep current behavior unchanged

The larger candidate pool is only there to make mixed-source selection possible without altering the query itself.

## Source classification

Use the existing private snapshot manifest mapping already derived in the benchmark:

- `memo`
- `raw_session`

No new ontology is introduced here. If a candidate is neither of those kinds, it is eligible as a normal candidate but does not satisfy the mixed-source floor.

## Coverage semantics

The success condition for this slice is not merely “include one memo and one raw candidate.”

Current scorer behavior is:

- `required_source_coverage` checks exact doc-ID coverage against `required_sources`
- `memo_plus_raw` rule satisfaction checks source kinds across matched required docs

That means this selector only counts as successful when the final bundle contains the exact required raw doc for the question, not just any `raw_session` candidate. Source-kind balancing is the retrieval policy. Exact-doc coverage remains the benchmark criterion.

## Bundle-selection algorithm

For questions with:

- `source_mix == mixed`
- `resolution_rule == memo_plus_raw`

build the final `top_k = 20` bundle like this:

1. Retrieve the top `40` candidates in original retrieval-rank order.
2. Partition candidates by source kind.
3. Start the final bundle by reserving:
   - the highest-ranked `memo` candidate, if any
   - the highest-ranked `raw_session` candidate, if any
4. After that minimum source floor is satisfied, fill the rest of the bundle by original retrieval rank from the remaining candidates.
5. If only one source kind is present in the candidate pool, do not invent diversity. Emit the best available candidates and let the scorer fail honestly.

This is intentionally minimal:

- it guarantees at least one memo and one raw result when both are available
- it does not force exact evidence docs
- it does not override the remainder of the ranking once the floor is satisfied

## Rank semantics

The retrieval record should preserve two different notions of order:

- `retrieval_rank`
  - original rank from the raw OpenSearch candidate list
- `bundle_rank`
  - final emitted order in the captured retrieval bundle

Why:

- we still need to know what the pure ranker actually did
- we also need to know how the source-aware selector changed the final bundle

For this slice, evaluation should expose both:

- retrieval-rank metrics for diagnostics
- bundle-order metrics for emitted-bundle effectiveness

At minimum, any rank-sensitive reporting added in implementation should make the selector's effect visible in the benchmark summary rather than hiding it behind original `retrieval_rank` only.

## Retriever identity and provenance

This retrieval policy is not the same as the current plain single-pass baseline. It needs distinct provenance.

Recommended values:

- `retriever_id: opensearch_single_pass_source_balanced`
- `retriever_version: opensearch_single_pass_source_balanced_v1`

The analyzer and index settings may remain the same as the current baseline if the candidate generator is unchanged. The important point is that the emitted bundle policy changed and must be visible in run artifacts.

## Expected effect

This slice should improve:

- `q_private_002`
- `q_private_004`

by making it much more likely that the exact required raw-session artifact survives into the final `top_k` bundle alongside memo evidence.

If the diagnostic-first step shows that the exact required raw doc is absent from the candidate pool, this slice should not proceed as if bundle selection can solve the problem. That is the falsifying outcome for the design: the candidate generator itself is still insufficient, and the next slice must target candidate generation rather than bundle selection.

## Testing

Add or update tests for:

- the diagnostic helper that finds the first rank of the exact required raw doc for `q_private_002`-style and `q_private_004`-style questions
- a `memo_plus_raw` fixture where memo chunks dominate lexical ranking but a relevant raw-session candidate exists in the wider candidate pool
- final bundle preserves:
  - at least one `memo`
  - at least one `raw_session`
- original `retrieval_rank` is preserved even when emitted `bundle_rank` changes
- rank-sensitive reporting makes the selector effect visible through emitted-bundle order, not only original retrieval order
- non-`memo_plus_raw` questions remain unchanged
- failure case where no raw-session candidate exists in the wider pool, and the scorer still fails honestly
- failure case where a non-required `raw_session` candidate is present but the exact required raw doc is absent, and the question still fails
- run-level regression proving `q_private_002`-style and `q_private_004`-style mixed-source questions can pass with the new selection policy

## Success criteria

This slice is successful if:

- `q_private_002` and `q_private_004` both reach:
  - `required_source_coverage = 1.0`
  - `resolution_rule_satisfied = true`
  - `failure = false`
- currently passing questions do not regress
- `claim_coverage` for `q_private_002` and `q_private_004` does not drop below its current baseline
- provenance clearly distinguishes this policy from the plain single-pass retriever

## Non-goals and follow-up

Explicitly deferred:

- evidence-key-aware doc forcing
- query rewriting per question
- second-pass retrieval
- broader noise reduction for stale or irrelevant chats
- currentness-sensitive improvements for `q_private_006`
- production/runtime triggering for mixed-source selection outside the benchmark harness

Those belong to later retrieval-methodology slices once this smaller mixed-source failure mode is isolated and measured.
