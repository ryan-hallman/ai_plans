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
- apply that policy only when:
  - `source_mix == mixed`
  - `resolution_rule == memo_plus_raw`
- preserve the retriever's original ranking separately from the final emitted bundle order
- record a new retriever identity/version for this retrieval policy

This slice will not:

- rewrite queries per question
- inspect evidence-key claim IDs or specific required docs during selection
- add a second retrieval pass
- change non-`memo_plus_raw` question behavior
- address the broader stale-currentness/noise problem seen in `q_private_006`

## Retrieval policy

The current OpenSearch single-pass retrieval remains the first-stage candidate generator.

For `memo_plus_raw` questions only:

1. retrieve a larger candidate pool using the existing query and scoring
2. classify each candidate by source kind
3. build the final emitted bundle using a small source-diversity floor

The key design choice is that this is a retrieval-policy change, not a reranker that knows about exact evidence keys. It should only care about source kind, not ground-truth labels.

## Candidate retrieval

For the affected question class:

- keep the same OpenSearch query
- keep the same analyzer and field weighting
- retrieve `candidate_k = 40`

For all other questions:

- keep current behavior unchanged

The larger candidate pool is only there to make mixed-source selection possible without altering the query itself.

## Source classification

Use the existing private snapshot manifest mapping already derived in the benchmark:

- `memo`
- `raw_session`

No new ontology is introduced here. If a candidate is neither of those kinds, it is eligible as a normal candidate but does not satisfy the mixed-source floor.

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

The scorer should continue to use `retrieval_rank` for rank-sensitive metrics unless explicitly changed in a later spec.

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

by making it much more likely that at least one relevant raw-session artifact survives into the final `top_k` bundle alongside memo evidence.

If a relevant raw session is nowhere in the top `40`, the question should still fail. That is a useful signal: the candidate generator itself is still insufficient.

## Testing

Add or update tests for:

- a `memo_plus_raw` fixture where memo chunks dominate lexical ranking but a relevant raw-session candidate exists in the wider candidate pool
- final bundle preserves:
  - at least one `memo`
  - at least one `raw_session`
- original `retrieval_rank` is preserved even when emitted `bundle_rank` changes
- non-`memo_plus_raw` questions remain unchanged
- failure case where no raw-session candidate exists in the wider pool, and the scorer still fails honestly
- run-level regression proving `q_private_002`-style and `q_private_004`-style mixed-source questions can pass with the new selection policy

## Success criteria

This slice is successful if:

- `q_private_002` and `q_private_004` stop failing because the raw-side source is preserved in the final bundle
- currently passing questions do not regress
- provenance clearly distinguishes this policy from the plain single-pass retriever

## Non-goals and follow-up

Explicitly deferred:

- evidence-key-aware doc forcing
- query rewriting per question
- second-pass retrieval
- broader noise reduction for stale or irrelevant chats
- currentness-sensitive improvements for `q_private_006`

Those belong to later retrieval-methodology slices once this smaller mixed-source failure mode is isolated and measured.
