# Metadata-Guided Memo Recovery Design

## Goal

Improve retrieval quality for approved broad-direction queries over the private
Prescient OS corpus when authoritative current memos are not recoverable from
the initial full-corpus lexical search alone.

The immediate target is the residual `q_private_006` failure mode after the
broad-direction reranking slice. The system should be able to recover the
current KE-first and retrieval-benchmark memos without pretending they were
already strong lexical hits in the full-corpus search.

## Problem

The follow-up investigation for `prescient_os-1r1` showed that the remaining
failure is not primarily a `top_k=20` cutoff problem.

For the live broad-direction query:

- the stale operator-first memo appears strongly in full-corpus lexical search
- the KE-first and retrieval-benchmark memos do not appear even in the top-500
  chunk hits
- the same user query over a memo-only search space does recover those missing
  memos

This means the current failure mode is a search-space problem before it is a
ranking-window problem. A broader lexical window mainly recovers more raw chat
duplication; it does not reliably recover the authoritative current memos that
matter most.

The design problem is therefore twofold:

1. recover authoritative memos that the full-corpus lexical path misses
2. preserve honest provenance so downstream consumers can tell whether a memo
   was surfaced lexically, recovered by a memo pass, or both

## Non-goals

This slice does not:

- solve ambiguous multi-project intent or clarifying-question behavior; that
  remains tracked in bead `prescient_os-1by`
- introduce a general query-rewrite framework
- solve broad retrieval quality for every query class
- expand the recovery path beyond `memo` documents to all curated artifacts
- require an index-schema migration before behavior is validated
- treat `top_k` inflation as the primary remedy

## Approaches considered

### 1. Larger lexical candidate pool

Increase the full-corpus lexical `top_k` window and rely on reranking to pull
the right memos upward.

Pros:

- smallest local code change
- no new retrieval path

Cons:

- contradicted by the live investigation evidence
- recovers more duplicated raw-session noise before it recovers the missing
  memos
- does not solve the honest-provenance problem

Verdict:

Reject as the primary next slice.

### 2. Metadata-boosted single-pass retrieval

Add enough indexed metadata to let one OpenSearch query prefer memo and
currentness signals directly.

Pros:

- likely the cleanest long-term architecture
- keeps ranking logic inside the search layer

Cons:

- forces schema and indexing work before the desired behavior is validated
- makes it harder to isolate whether the real win came from the memo-restricted
  search space or from score tuning

Verdict:

Good future direction, but not the next learning slice.

### 3. Memo-focused second pass with audited merge

Keep the current full-corpus lexical retrieval, add a second retrieval pass
over a memo-restricted search space for approved broad-direction queries, then
merge the two candidate sets under explicit provenance rules.

Pros:

- directly matches the investigation result
- recovers missing memos without broadening the raw-session search space
- keeps lexical and recovered provenance auditable
- validates the behavior before schema work

Cons:

- more moving parts than a pure rerank
- requires explicit merge semantics to avoid score confusion

Verdict:

Use this approach.

## Recommendation

Implement a narrow metadata-guided memo recovery path for approved
broad-direction queries:

1. run the existing full-corpus lexical retrieval exactly as today
2. if the query matches the approved broad-direction class, run a second search
   over a memo-restricted search space using the same user query
3. order the recovered memo candidates with explicit authority and currentness
   policy
4. merge memo-recovery and lexical candidates under audited provenance rules
5. emit a final candidate list that preserves candidate origin rather than
   collapsing everything into one opaque score

This keeps the fix narrow, product-relevant, and reviewable.

## Architecture

### Retrieval flow

For a query in the approved broad-direction class:

1. execute the current full-corpus lexical retrieval path
2. execute a memo-restricted retrieval pass using the same normalized query
3. normalize both candidate sets into a mergeable record shape
4. annotate each candidate with origin metadata:
   - `lexical`
   - `memo_recovery`
   - `lexical_and_memo_recovery`
5. apply merge ordering rules to produce the final output list

For all other queries:

- do not run memo recovery
- preserve the existing retrieval path

### Precedence with mixed-source selection

This slice does not apply memo recovery to questions whose retrieval contract
already requires mixed-source `memo_plus_raw` evidence.

If a query would otherwise satisfy both:

- the broad-direction memo-recovery gate
- a mixed-source retrieval path that explicitly requires raw-session evidence

then the mixed-source path wins and memo recovery does not run in this slice.
That overlap needs a separate design if product requirements change.

### Query gate

This path should remain intentionally narrow and auditable.

For this slice, the memo-recovery gate must reuse the same approved
broad-direction query-class definition used by the existing broad-direction
reranking path. The implementation must not fork separate query heuristics for
reranking and memo recovery.

The source of truth for that gate must live in one shared auditable
configuration or implementation surface. If the approved examples or exclusions
change, both retrieval stages must inherit the same revision.

Positive examples include broad, unscoped questions such as:

- `What is Prescient OS being built to do?`
- `What are we building?`
- `What is the product direction?`
- `What is the current company direction?`

Exclusions include queries with clear scoping signals such as:

- initiative names like `retrieval`, `benchmark`, `peloton`, `chat import`
- explicit historical anchors like `operator-first`, `April 12`, `April 16`
- status wording like `what's going on with`, `status`, `progress`, `working on`
- explicit time slicing like `today`, `this week`, `latest`

The spec should treat this as an approved broad-direction query class, not as a
general semantic-intent classifier.

### Memo-restricted search space

The second pass should search only `memo` documents. This slice should not
expand to specs, plans, or other curated artifacts yet, even if the future
architecture may want a broader curated-artifact path.

The memo-restricted pass may be implemented through metadata-aware filtering,
separate indexing, or another bounded mechanism. The governing requirement is
behavioral:

- the same user query must be run against a memo-only search space
- raw-session noise must not occupy the memo-recovery window

## Merge and ranking semantics

### Provenance contract

Every final candidate emitted by the recovery path must preserve enough fields
to distinguish:

- whether it appeared in the original lexical retrieval
- whether it appeared in the memo-recovery pass
- what its lexical rank/score were, if present
- what its memo-recovery rank/score were, if present
- why it was lifted, if lifted

Representative fields may include:

- `retrieval_origin`
- `lexical_rank`
- `lexical_score`
- `memo_recovery_rank`
- `memo_recovery_score`
- `recovery_reason`

The exact field names can be chosen later, but the semantics must be explicit.

### Ordering policy

Final ordering should use explicit policy tiers rather than opaque score
blending.

Recommended ordering:

1. authoritative memos present in both lexical and memo-recovery sets
2. a small capped set of authoritative memos recovered only by the memo pass
3. remaining lexical candidates in original relative order, unless another
   already-approved reranker applies

Within the memo subset, use explicit authority/currentness policy:

- `memo` is authoritative relative to `raw_session`
- newer direction-setting memos outrank older memos when relevance is
  otherwise close

Recovered-only memos need additional guardrails:

- a recovered-only memo may be promoted only if it meets a minimum memo-pass
  relevance floor defined for the recovery path
- at most two recovered-only memos may be inserted ahead of the remaining
  lexical candidates in this slice
- lexical candidates that remain after recovered-only insertion must preserve
  their original relative order

This keeps the fix focused on recovering the specific missing current memos
without allowing a memo-only search space to flood the final list.

This allows the final output to surface the right current memos near the top
without lying that they were strong full-corpus lexical hits.

### Negative rules

Do not:

- silently rewrite the user query as the primary recovery mechanism
- relabel memo-recovered documents as if they were lexical top hits
- collapse lexical and memo-recovery scores into a single number with no audit
  trail
- use larger `top_k` as a substitute for the memo-recovery path

## Data contracts

This slice requires retrieval records that can expose candidate origin and
rank/score per origin path.

Because this slice introduces a new merged-output ordering, it must emit an
explicit final-order field rather than overloading the existing lexical-rank
meaning.

Required semantics:

- `lexical_rank` remains the original full-corpus lexical rank when present
- `memo_recovery_rank` remains the memo-pass rank when present
- `final_rank` is the emitted merged ordering used by downstream consumers

This change should carry an explicit retrieval-record schema version bump so
scorers, summaries, and review surfaces cannot silently misread lexical rank as
final merged rank.

The recovery path should also rely on stable document metadata sufficient to:

- identify `memo` documents reliably
- read memo currentness fields such as `as_of`
- preserve canonical `doc_id` identity across both retrieval passes

## Testing strategy

### Recovery regression

Add a regression proving that the broad-direction recovery path improves the
`q_private_006` result materially over the current reranking-only baseline.

Success should require:

- KE-first memo recovered
- retrieval-benchmark memo recovered
- stale operator-first framing no longer dominates the final top results
- provenance output clearly shows whether each surfaced memo was lexical,
  memo-recovered, or both

### Non-regression

Preserve non-regression guarantees for existing mixed-source questions. This
path must not break the `memo_plus_raw` behavior already added for questions
that explicitly need both memo and raw-session evidence.

Add coverage proving that memo recovery does not run for `memo_plus_raw`
questions in this slice.

### Auditability

Tests should prove not only that recall improved, but that the system remains
honest about why recovered memos moved upward.

At minimum, coverage should pin:

- candidate origin labeling
- preserved lexical rank when a document appears in both paths
- explicit memo-recovery labeling when a document appears only in the second
  pass
- explicit `final_rank` semantics and schema-version handling for merged output
- recovered-only insertion cap behavior

## Future extensions

If this path works, the next likely evolutions are:

- move the memo restriction and authority/currentness signals into indexed
  metadata for a cleaner single-pass query
- generalize from memo recovery to curated-artifact recovery if product needs
  justify it
- handle broad-query ambiguity across simultaneous active efforts under
  `prescient_os-1by`
