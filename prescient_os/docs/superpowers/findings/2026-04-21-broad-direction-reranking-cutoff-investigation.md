# Broad-Direction Reranking Cutoff Investigation

**Date:** 2026-04-21
**Issue:** `prescient_os-1r1`
**Branch / worktree:** `poc/broad-direction-reranking`
**Corpus:** `prescient-os-chats-2026-04-18-with-retrieval-only-turn`

## Question

For `q_private_006`-style broad direction queries, are the missing authoritative
memos merely falling just outside the current `top_k=20` lexical cutoff, or is
the failure mode deeper than that?

## Setup

- Brought up local OpenSearch with `docker compose up -d opensearch`.
- Used the real private snapshot and the live `opensearch_single_pass_v1`
  retrieval path.
- Measured chunk ranks and unique-doc first ranks for:
  - `What is Prescient OS being built to do?`
  - additional approved broad-direction variants
- Compared three option families:
  - wider candidate pool with the current lexical query
  - query-time metadata restriction via a memo-only search space
  - second-pass query rewrite / expansion

## Findings

### 1. This is not primarily a top-20 cutoff problem

For the exact `q_private_006` query over the full corpus:

- `prescient-os-operator-first-redesign-2026-04-12` first appeared at chunk rank `1`
- `ke-first-pivot-2026-04-16` did **not** appear in the top `500` chunk hits
- `retrieval-benchmark-narrowing-2026-04-16` did **not** appear in the top `500` chunk hits
- `019d9319-2511-7281-b136-977181ef84b1` first appeared at chunk rank `209`
- `761e7568-136e-4f4f-9b49-a3c78f9889ad` first appeared at chunk rank `362`

Conclusion: widening the existing lexical candidate pool from `20` to `100` or
even `500` would recover some supporting raw sessions, but it would still not
recover the two missing memos that matter most.

### 2. Chunk-level duplication is compressing the candidate space

The top chunk hits were dominated by repeated chunks from a small number of raw
sessions. For the baseline query, the top `30` chunk hits represented only four
unique docs:

1. `prescient-os-operator-first-redesign-2026-04-12`
2. `1f8e0d71-2b89-4088-bc44-33d13e105eea`
3. `be5b69a1-853f-4681-a87c-062a391451d7`
4. `d6c595ce-ff17-4dbf-8003-a3335034e022`

This means the search is losing breadth before the reranker ever runs. A larger
raw chunk window helps less than it appears because duplicated chat chunks keep
occupying the expanded window.

### 3. The stale operator-first memo wins on direct lexical overlap

The operator-first memo is the only required memo with explicit `Prescient OS`
language in both the title and the body:

- title: `Prescient OS Operator-First Redesign`
- body: repeatedly uses `Prescient OS`, `product direction`, and direct product
  framing

The missing memos instead lead with different vocabulary:

- `ke-first-pivot-2026-04-16`: `knowledge engine`, `API-first`, `pivot`
- `retrieval-benchmark-narrowing-2026-04-16`: `retrieval`, `benchmark`,
  `retrieval-first`

So the current broad query is not just picking the wrong time slice. It is
lexically aligned with the stale memo more than with the current memos.

### 4. Query-time metadata restriction is the most promising recovery path

Running the **same user query** against a memo-only search space changed the
candidate picture materially:

- `prescient-os-operator-first-redesign-2026-04-12` at rank `2`
- `ke-first-pivot-2026-04-16` at rank `4`
- `retrieval-benchmark-narrowing-2026-04-16` at rank `13`

That means the missing memos are retrievable without any query rewrite once the
search space is constrained away from raw-session noise.

This points toward indexing and querying memo metadata for broad-direction
queries, not simply inflating `top_k`.

### 5. Blind second-pass query expansion is brittle

I tested several expanded second-pass queries on the full corpus.

Generic KE-language expansions such as:

- `api-first business knowledge engine`
- `knowledge engine retrieval benchmark api-first`

did **not** reliably recover the KE-first memo.

More tailored expansions such as:

- `knowledge engine became the primary product thesis`
- `retrieval-first not curation-first`

did recover the KE-first and/or benchmark memos, but those rewrites are too
close to known answer text to treat as a robust runtime strategy.

Conclusion: a pure query-rewrite second pass is possible, but it looks brittle
and high-maintenance compared with a metadata-guided pass.

### 6. Candidate recovery alone is not enough with the current reranker weights

Even after memo-only retrieval made the right memos visible, the current
reranker did not materially improve the ordering.

Reasons:

- `_authority_bonus` is still `0.0`
- `_NEWEST_MEMO_BONUS = 0.025` is too small relative to the lexical score gaps

So a future metadata-guided pass would also need a stronger ordering policy than
the current mild-recency reranker.

## Option Comparison

### Wider candidate pool

- Pros: smallest code change
- Cons: does not recover the KE-first or benchmark memos in the live corpus
- Verdict: reject as the next slice

### Query-time metadata

- Pros: same user query can recover the missing memos when raw-session noise is
  removed from the search space
- Cons: current index does not expose enough metadata to do this directly; the
  reranking policy would still need strengthening
- Verdict: strongest next direction

### Second-pass query rewrite

- Pros: can recover missing docs in targeted cases
- Cons: generic rewrites were weak, while successful rewrites looked too
  bespoke and answer-shaped
- Verdict: possible helper, but not a good primary fix

## Recommendation

The next retrieval slice should **not** spend itself on a larger lexical
candidate pool.

The best next move is a bounded, provenance-preserving **metadata-guided memo
pass** for approved broad-direction queries:

1. retrieve or search a memo-focused candidate set using indexed metadata
2. apply a stronger explicit authority/currentness ordering policy
3. merge that memo pass with the original broad lexical results without lying
   about provenance or lexical rank

If `q_private_006` still needs the supporting raw-session evidence after memo
recovery, treat that as a separate merge/selection problem rather than as part
of the same stale-memo fix.

## Suggested Follow-Up

- Write a spec for metadata-guided broad-direction memo recovery.
- Explicitly reject `top_k` inflation as the primary solution.
- Decide whether the metadata pass should be:
  - a memo-only second pass
  - a metadata-boosted single pass with indexed `source_kind` / topic fields
  - a union of lexical broad search plus memo-focused search with audited merge
