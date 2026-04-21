# Broad Direction Query Reranking Design

## Goal

Improve retrieval quality for broad, unscoped product-direction queries over the private PrescientOS corpus without adding another benchmark-only selector.

The immediate target is the `q_private_006` failure mode from the first private retrieval baseline, where the retriever over-ranked the older operator-first memo and noisy raw chat sessions instead of surfacing the KE-first and benchmark-path artifacts needed to answer `What is Prescient OS being built to do?`

## Problem

The current private retrieval path is plain OpenSearch `multi_match` over `title^2` and `text`, followed by a trivial local rerank that preserves descending `_score` only. That means the retriever currently lacks:

- authority distinctions between curated memos/artifacts and raw chat sessions
- recency distinctions among curated direction-setting artifacts
- any special handling for broad, unscoped direction queries that should prefer durable company/product framing over incidental chat noise

This failure mode is product-relevant, not just benchmark-relevant. In a real system, broad strategy queries should not be dominated by stale framing or random chat residue.

## Non-goals

This slice does not attempt to solve project-status ambiguity or general multi-project disambiguation. That follow-up is tracked separately in bead `prescient_os-1by`.

This slice also does not:

- add benchmark-label-gated retrieval logic
- add a second retrieval pass
- change the indexed document schema
- add explicit `scope` / `priority` ranking metadata yet
- apply authority/recency weighting to every private-corpus query

## Approaches considered

### 1. Broad-query reranker on top of existing retrieval

Keep OpenSearch as the candidate generator, then apply a narrow reranker only for broad product-direction queries.

Pros:
- smallest change
- product-relevant behavior rather than eval-only behavior
- no reindexing required
- easy to validate directly against `q_private_006`

Cons:
- adds a query heuristic
- authority/recency priors live outside the search engine for now

### 2. Index-time metadata with OpenSearch-native ranking

Add source-kind and memo recency metadata to indexed chunks, then encode the priors into the OpenSearch query.

Pros:
- cleaner long-term retrieval architecture
- keeps ranking logic inside the search layer

Cons:
- broader schema/indexing work
- slower path to learning whether the ranking priors even help

### 3. Two-pass retrieval for currentness/scope-disambiguation

Retrieve broad candidates first, then run a second targeted retrieval for current-direction and superseded-framing evidence.

Pros:
- likely powerful for hard ambiguity cases

Cons:
- more moving parts
- harder to attribute gains
- unnecessary before the simple reranker is tried

## Recommendation

Use approach 1.

Add a narrow, query-text-gated reranker for broad company/product-direction queries. Keep OpenSearch candidate generation unchanged and treat the reranker as a lightweight post-search ordering step.

## Architecture

### Retrieval flow

1. Run the existing `search_index(...)` candidate retrieval exactly as today.
2. Detect whether the normalized query is a broad product-direction query.
3. If it is not, return the normal ranking unchanged.
4. If it is, rerank the candidate list using lightweight document priors:
   - `source_kind`: `memo` vs `raw_session`
   - `memo_as_of`: parsed from memo frontmatter
5. Emit the reranked list as the final retrieval order.

OpenSearch remains the candidate generator. The new logic changes only the final ordering for a narrow query class.

## Broad-query heuristic

### Positive triggers

The reranker should trigger only for a small, auditable set of broad direction-query patterns such as:

- `what is prescient os being built to do`
- `what are we building`
- `what is the product direction`
- `what is the company direction`
- `what is prescient os`
- `what is the current direction`

### Exclusions

Do not trigger if the query contains a clear scoping signal, including:

- explicit date/time words like `today`, `currently`, `april`, `2026`, `this week`, `latest`
- initiative/project terms like `retrieval`, `benchmark`, `peloton`, `question set`, `evidence key`, `chat import`, `snapshot`, `top-up`
- pivot/event terms like `ke-first pivot`, `operator-first`, `april 12`, `april 16`
- project-status phrasing like `what's going on with`, `status`, `progress`, `working on`, `specific to`

The heuristic should be intentionally narrow. Queries outside this class should stay on the normal retriever.

## Ranking policy

The reranker should use the original OpenSearch `_score` as the base signal, then add small explicit priors.

### Priority order

1. lexical relevance
2. authority
3. recency

### Authority

For this first pass:

- `memo` gets a positive authority bonus
- `raw_session` gets no authority bonus

This means curated artifacts are preferred over raw chat sessions when lexical scores are close.

### Recency

For this first pass:

- apply recency only to `memo`
- newer memo `as_of` dates receive a small positive bonus
- do not add recency bonuses to raw chat sessions yet

This means a recent direction-setting memo can outrank an older direction-setting memo when both are similarly relevant, but a recent raw chat should not outrank a strong memo solely because it is newer.

### Intended effect

For broad direction queries, this should lift:

- `ke-first-pivot-2026-04-16`
- `retrieval-benchmark-narrowing-2026-04-16`

above the older:

- `prescient-os-operator-first-redesign-2026-04-12`

while pushing down noisy chat sessions that are only weakly lexically related.

The bonuses must stay modest. They are for tie-breaking and close-call steering, not for bulldozing obviously irrelevant documents to the top.

## Data bridge

Do not change the indexed document schema in this slice.

Instead, load a lightweight prior map from the private snapshot:

- `doc_id -> source_kind`
- `doc_id -> memo_as_of | None`

This can be built from the snapshot manifest plus memo frontmatter alongside the existing private-corpus load path.

## Testing strategy

### Heuristic tests

Add unit tests that prove:

- broad direction queries trigger the reranker
- scoped pivot/project/time queries do not trigger it

### Reranker tests

Add unit tests that prove:

- a newer memo beats an older memo when lexical scores are close
- a memo beats a raw session when lexical scores are close
- a raw session still wins if it is much more lexically relevant

### Baseline regression

Add baseline/orchestration coverage showing that `q_private_006` improves materially relative to the first baseline.

Expected improvements:

- `required_doc_coverage` improves above the original `0.2`
- `claim_coverage` improves above the original `0.333...`
- `noise_ratio` decreases below the original `0.9`
- ideally `failure` becomes `false`

### Non-regression

Add or preserve checks that:

- `q_private_002` still passes
- `q_private_004` still passes

## Future extensions

If this reranker helps but remains too heuristic-bound, the next likely evolutions are:

- move authority/recency signals into indexed metadata and OpenSearch-native ranking
- add `scope` and `priority` metadata for curated artifacts
- handle ambiguous multi-project intent, including clarifying-question behavior, under bead `prescient_os-1by`
