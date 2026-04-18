# 2026-04-18 Retrieval Thesis

## Purpose of this document

This is a thesis, not a spec or a roadmap. It captures why we are building the retrieval/memory layer the way we are, what we believe is wrong with the default RAG-first design, and what we believe a better approach looks like. It should be revised — not preserved — when our thinking evolves.

Expected lifespan: living document. Promote to `findings/` once validated in practice, fork concrete pieces into `specs/` when we build them.

## The problem we are solving

Prescient OS is a horizontal knowledge and memory engine for business and research work over heterogeneous documents (legal filings, business reports, company docs, research material). It is not:

- a code assistant
- a vertical tool tuned to one industry
- a classical enterprise search product
- a Q&A bot over a fixed corpus

We are optimizing for the quality of investigative, evidence-backed answers across arbitrary document collections, not for speed-to-market or buzzword parity. Users return for answers they can trust and cite, not for snappy but shallow responses.

## Why we are skeptical of RAG-first design

RAG is treated as the default retrieval architecture, but the defaults encode assumptions that no longer match 2026 reality and have never matched the problem shape we care about.

### 1. Embeddings smear structure that users already encode in queries

Real knowledge queries carry scope: a company, a time range, a case, a document type, a party, a statute. Users naturally say "what are the rules around X for case Y" or "how did Peloton explain margin changes in Q3 2024." Classical RAG converts the query to a vector and asks cosine similarity to recover that scope. Structured indexing preserves it deterministically.

### 2. Retrieve-once vs. investigate

Top-k retrieval is a single-shot algorithm. Real investigation is iterative — each finding informs the next query. Agents that can run N rounds of hypothesis-refined search beat any one-shot retriever on non-trivial questions. This is architectural, not a tuning problem.

### 3. Chunking artifacts

Chunk boundaries do not respect semantic units. Long-document reasoning wants pieces that straddle chunks. With long-context models, reading whole documents sidesteps the category of chunk-boundary failures entirely.

### 4. Retrieval trained to answer instead of surface evidence

RAG pipelines optimize for the final answer, which couples retrieval quality to synthesis quality and makes failure modes opaque. Separating "what evidence exists" from "how we synthesized it" is a stronger design — the eval harness we are building reflects this.

### 5. RAG was designed for a context-window world that no longer exists

In a 4k–32k token world, aggressive pre-filtering was necessary. In a 200k–1M token world with cheap agentic iteration, the economics inverted. Most "RAG-first" architectures in 2026 are optimizing a 2023 constraint.

### 6. LLM-driven query expansion replaces most "fuzzy similarity" needs

Agentic CLIs don't grep the user's literal query. They generate synonyms, structural hypotheses, citation patterns, defined terms, and dialectal variants — then search all of them and see which hit. This is strictly more powerful than embedding similarity for most fuzzy queries: it is explicit, auditable, iterative, and produces exact matches.

## Empirical signals

These are the experiences that shaped the thesis. Honest about what they do and do not show.

### Coding agents over large code corpora

Agentic coding tools with `rg` / `grep` / `sed` routinely outperform RAG bolted onto the same codebases. Code has strong structure (symbols, paths, imports, type signatures) and lexical precision matters — exactly the conditions where embeddings lose the most information.

### Million-page legal corpus experiment

An agent was given ~1M pages of legal text across ~100 cases and answered real questions correctly. The queries respected a natural scope ("what are the rules around X for case Y"), which collapsed the effective search space to one case before any content search.

**What this demonstrates:** when scope is respected, agents with lexical and structural primitives can navigate enormous corpora without vector infrastructure.

**What this does not demonstrate:** that agentic retrieval solves truly cross-corpus synthesis queries at scale, or queries where no scope is inferable. Those are real limitations we must design for, not wave away.

## Theory of a better approach

The positive claim. This is what we believe the engine should look like.

### Scope extraction is a first-class primitive

Before any content search, the system parses or extracts scope from the query (entity, date range, doc type, project, jurisdiction). This deterministically narrows the corpus. Most real queries carry scope; the ones that don't are a minority we handle separately.

### The spine is structured indexing + agent primitives, not a RAG pipeline

The engine is a toolbox the agent can compose:

- exact/lexical search (BM25, regex)
- metadata filter (scope-based)
- structural walk (sections, outlines, tables)
- citation/reference graph traversal
- entity lookup
- long-context read of selected documents
- LLM-driven query expansion as a meta-primitive

The agent chooses which primitives to invoke, in what order, for each query. There is no single "retrieval pipeline" — there is an agent making retrieval decisions.

### Retrieval narrows; long-context read answers

Retrieval's job is to pick the right document(s), not to pick the right 500-token windows. Once narrowed to a shortlist, the agent reads whole documents (or large sections) with long-context models. This removes a class of chunk-boundary hallucinations.

### Catalog at ingest, lazy summaries on first touch, deep read on the shortlist

Three tiers of summarization effort, all domain-agnostic:

1. **Ingest (cheap, generic):** per-doc extraction — title, entities, date range, doc type (LLM-classified), section outline, citations/references. This is the catalog.
2. **First query touch (medium, lazy):** generate an outline-style summary when a doc is first read in a session. Cache it. Repeated work in the same scope gets cheaper over time.
3. **Query time (expensive, deliberate):** for synthesis questions, scope filter + catalog → shortlist → sub-agent per unit reads what it needs → synthesis. Parallelizable and bounded.

No pre-computed industry-specific schemas are required. The catalog rubric is generic; the content fills itself in from the document.

### Vectors as one tool, not the spine

This is where we diverge from both RAG-first orthodoxy and naive "no vectors" minimalism. Vectors are *one primitive* the agent can reach for — not the default path, not excluded by principle.

**Where BM25 + LLM expansion suffices** (the majority):

- named entities, dates, citations, statute references, defined terms
- known-vocabulary concept queries where the model can enumerate surface forms
- anything with an explicit scope filter
- structural patterns

**Where vectors earn their keep** (narrower but real):

- **exemplar-based retrieval** — "find docs like this one"; embedding one doc and cosine-searching is cleaner than reverse-engineering it through LLM query generation
- **cross-vocabulary mismatches in unfamiliar dialects** — niche jargon, proprietary terminology, company-internal shorthand the LLM does not know
- **low-signal concept queries** where there is nothing specific to expand from
- **semantic deduplication at ingest** — detecting that two documents cover the same material with different words
- **genuine cross-lingual corpora** (real in multinational business docs)

On standardized benchmarks, hybrid BM25 + dense consistently beats either alone. The research is not ambiguous. The judgment call is when the marginal gain justifies the operational overhead.

**The operational test:** an agent should be able to explain *why* it invoked vector search for a given query. If it can't, BM25 + expansion probably suffices and the call was cargo-culted. Vectors are justified, built with care, and invoked deliberately — not the default retrieval path.

### Design for investigation, not Q&A

Business and research knowledge work is iterative investigation, not single-turn lookup. The product surface should support "here is what I found, here is what I would check next" loops. Citations, provenance, and the ability to drill into specific evidence are first-class.

## What we are betting v1 is

Concrete implications for the initial build. These are claims, subject to revision, not commitments.

- **Ingest:** generic catalog extraction (entities, sections, dates, doc type, citations) across all corpora. No per-industry schemas.
- **Query path:** scope extraction → catalog shortlist → agent reads shortlist with composable primitives → synthesis with citations. No single-shot RAG pipeline.
- **Vector infrastructure:** built and operated, but scoped to specific query shapes (exemplar, dedup, unfamiliar-dialect, low-signal). Not the default. Not exercised on most queries.
- **Synthesis across units:** deliberate deep-read on the shortlist. Parallelized sub-agents per unit where meaningful. Presented to users as premium, explainable latency — not masked as "fast."
- **Evaluation:** benchmarks compare agent-with-primitives against classical RAG, not against nothing. Retrieval support and synthesis quality are scored separately (per the eval harness design).

## Where this could be wrong

Open questions and honest risks. The thesis lives or dies on these.

- **Queries where scope cannot be inferred.** "Tell me something interesting about this company." If the agent-with-primitives approach consistently stalls on low-signal queries, we need either better exploratory primitives or a stronger vector-driven path for this class.
- **Cross-corpus synthesis at scale.** Agent MapReduce works on 30 units; does it still work on 500? At what latency/cost ceiling do users disengage?
- **LLM expansion failure in unfamiliar dialects.** If users consistently work in vocabularies the model doesn't know, expansion stalls. How do we detect this and route to vector/learned retrieval without a human switch?
- **User willingness to wait.** We assume premium users tolerate 30–120s for defensible answers. If they don't, the deep-read model breaks and we need more aggressive caching or different query shapes.
- **Go-to-market framing.** Buyers expect "RAG" as a bingo word. Selling "agent with composable primitives" requires educating. This is a product/positioning cost, not a technical flaw, but it is real.
- **Operational cost of hybrid.** Running vectors deliberately still means running them. The per-query economics of hybrid at scale need validation.

## What would falsify or revise this

Concrete signals that would make us change the thesis, not just tune it.

- Scope extraction fails on more than ~20% of real user queries → we need either stronger exploratory primitives or to rethink scope-first as the spine.
- Users measurably prefer fast-wrong over slow-right in blind tests → the deep-read model is wrong for this product.
- A durable class of valuable queries emerges that only similarity solves (not exemplar, not dedup — a genuinely new class) → vectors earn a larger role, possibly primary for that class.
- Agent-with-primitives consistently underperforms classical RAG on our own eval harness across diverse corpora → the thesis is wrong; the defaults exist for a reason we misjudged.
- Per-query deep-read costs exceed what any reasonable user will pay and caching does not close the gap → we need pre-computed summaries or similarity as a cheaper default.

Until one of these fires, we proceed.
