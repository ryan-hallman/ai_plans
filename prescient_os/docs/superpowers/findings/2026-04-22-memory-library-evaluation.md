# Memory Library Evaluation for Prescient OS

**Date:** 2026-04-22
**Scope:** 7 open-source memory libraries downloaded to `../vendor/` within the past hour, evaluated against Prescient OS's retrieval thesis and recall/memory design.
**Libraries:** mem0, honcho, byterover-cli, OpenViking, hindsight, RetainDB, nuggets

---

## Bottom line

**Don't adopt any of the seven wholesale.** All are either RAG-first vector pipelines, fixed-step memory libraries, or personal-agent caches — none match the agent+primitives thesis, and the closest fits (hindsight, OpenViking, RetainDB) would cost more in fork work than saved in library code. The "data-science competency" concern is real but these libraries don't resolve it — the retrieval quality work *is* the product.

---

## Why none of them fit

Three structural mismatches appear across all seven:

1. **Fixed pipeline, not composable primitives.** Every one — mem0, honcho, hindsight, RetainDB, byterover, OpenViking, nuggets — exposes `retrieve()` or `recall()` as a monolithic function. The Prescient OS thesis requires the agent to *choose* between BM25, scope-filter, structural walk, entity lookup, long-context read. None expose those as separate, swappable tools.
2. **Vector-first or extract-to-flat-facts.** mem0, OpenViking, and nuggets default to semantic retrieval (vectors or HRR); honcho, hindsight, RetainDB do hybrid but bake the RRF in. None default to "BM25 + LLM expansion + KG alias" the way the recall spec does.
3. **No trust tiers.** Zero of the seven model "verified > approved > draft > conversation > inferred". RetainDB has a confidence float; hindsight has immutable `fact_type`; the rest have nothing. The "automation proposes, user governs" design requires this and would have to be bolted on.

License posture adds friction: only mem0 (Apache 2.0), hindsight (MIT), and nuggets (MIT) have clean licenses for a proprietary product. honcho and OpenViking are AGPL; RetainDB is BSL 1.1; byterover-cli is Elastic License with SaaS tie-in.

---

## Ranked matrix

| Library | Score | Substrate | License | Shape | Verdict |
|---|---|---|---|---|---|
| **hindsight** | 21/30 | BM25 + vector + RRF + cross-encoder rerank | MIT | Agent memory consolidation | Steal ideas |
| **OpenViking** | 16.5/30 | Vectors + LLM intent rewriting | AGPL-3 | Agent-native context DB w/ L0/L1/L2 | Steal ideas |
| **byterover-cli** | 16/30 | BM25 (MiniSearch) + symbol tree | Elastic 2.0 | CLI + daemon (not a library) | Steal ideas |
| **mem0** | 14.5/30 | Vector + BM25 + entity | Apache 2.0 | Personalized chat memory | Avoid |
| **RetainDB** | 14/30 | Postgres + pgvector hybrid | BSL 1.1 / Apache | Typed memory w/ version chains | Steal ideas |
| **honcho** | ~10/30 | pgvector + PG FTS + RRF | AGPL-3 | Peer-centric agent memory | Steal ideas |
| **nuggets** | 4/30 | Holographic Reduced Representation | MIT | Personal agent fact cache (512-fact cap) | Avoid |

---

## Per-library summary

### mem0 — Avoid (14.5/30)
Flat K-V memory extracted from conversations via LLM, stored as opaque strings, retrieved via hybrid semantic+BM25+entity-boost. Strong hybrid tuning and entity linking, mature (Apache 2.0, YC-backed, ~3.2k stars). **Dealbreaker:** no document model, no provenance, no structured scoping by type/jurisdiction, no composable retrieval primitives. Built for personalized chat memory ("user likes pizza"), not knowledge engines over regulated corpora.

### honcho — Steal ideas (~10/30)
Peer-centric agent memory with three-agent pipeline (deriver/dialectic/dreamer) over pgvector + PG FTS + RRF. AGPL-3 copyleft. Source IDs tracked internally but not surfaced; fixed pipeline cannot be decomposed. **Strength:** RRF fusion of semantic + FTS is elegant. **Dealbreaker:** Fixed agent pipeline + AGPL + no composable primitives.

### byterover-cli — Steal ideas (16/30)
CLI + daemon (not a library) with BM25 search via MiniSearch over a markdown context tree, with clever symbol-tree score propagation and multi-tier response strategy (cache → direct → prefetch → LLM). Elastic License 2.0 with SaaS tie. **Strength:** Multi-tier response strategy reduces LLM cost. **Dealbreaker:** Monolithic `QueryExecutor`, path-prefix-only scope, designed for REPL not programmatic composition.

### OpenViking — Steal ideas (16.5/30)
Agent-native context database with filesystem paradigm (`viking://`) and three-layer document model (L0 abstract, L1 overview, L2 leaf). Vector-first with LLM intent rewriting, AGPL-3. Excellent document model and session compression. **Strength:** L0/L1/L2 hierarchy + async session compression with memory extraction — directly maps to the "catalog at ingest, lazy summary, deep read on shortlist" tier model. **Dealbreaker:** No BM25/keyword path; would require forking retrieval to make keyword primary.

### hindsight — Steal ideas (21/30 — highest)
Post-conversation memory consolidation with biomimetic 3-tier memory (world/experience/observation) on Postgres + pgvector. 4-way parallel retrieval (semantic, BM25, graph, temporal) fused via RRF + cross-encoder rerank. MIT, 10k stars, Fortune 500 deployments claimed. **Strength:** Fact provenance is genuinely first-class — `source_fact_ids`, `document_id`, `chunk_id`, `source_facts` dict all surfaced. **Dealbreaker:** Fixed `retain/recall/reflect` pipeline, all 4 retrievers always run in parallel, no trust-tier model, designed for agent memory not document corpus.

### RetainDB — Steal ideas (14/30)
Self-hostable memory layer with strongly structured multi-layer model (Document → Chunk → Memory + Entity graph) and 20+ pre-built source connectors. Memories are typed, scoped, confidence-scored, with version chains (`supersededBy`) and temporal bounds. BSL 1.1 cloud-restricted + Apache SDK. **Strength:** Versioned memory schema with relationship graphs and temporal validity — the most mature memory data model in the set. **Dealbreaker:** Monolithic retriever pipeline; agents cannot compose BM25 + entity lookup + scope filter as independent tool calls. Pre-v1 repo, no community.

### nuggets — Avoid (4/30)
Holographic Reduced Representation (HRR) in-memory cache — facts stored as superposed complex-valued vectors, recalled via algebraic unbinding. MIT, single author, <1 month old. **Strength:** Sub-millisecond recall with zero dependencies. **Dealbreaker:** 512-fact hard cap per nugget, flat K-V, no document structure, no provenance. Solves a different problem (personal agent preference cache) exceptionally well; solves the Prescient problem not at all.

---

## What's actually worth stealing (concrete)

- **From hindsight:** the source-fact provenance pattern (`source_fact_ids` + `chunks` + `source_facts` dict surfaced to caller). Cleanest citation model of the bunch and directly maps to the "source excerpts with positional references" requirement in the recall spec.
- **From OpenViking:** the L0/L1/L2 document-tier pattern (`.abstract.md` / `.overview.md` / leaf content), and async session compression into typed long-term memories. Matches the "catalog at ingest, lazy summary on first touch, deep read on shortlist" tier model almost exactly.
- **From RetainDB:** the versioned memory schema — typed memories, `supersededBy` chains, `validFrom`/`validUntil` temporal bounds, and typed relationship edges (`updates`/`contradicts`/`supports`/`derives`). Needed for the "conflicting information" UX in the recall spec.
- **From beads/gastown (already validated in prior findings):** push-not-pull session-start injection, human-readable slug keys, all-writes-on-main Dolt discipline.

None of these require adoption — they're design patterns to port.

---

## Honest counter-argument on the data-science concern

The worry about data-science competency is legitimate but the libraries don't resolve it:

1. **The hard part is eval, not implementation.** Prescient OS already has a private-retrieval scorer, an eval harness spec, and a question set with an evidence key. Battle-tested BM25 ships for free in OpenSearch, Postgres FTS, or Tantivy — you don't write the ranker, you tune field boosts against your eval. That's a product judgment skill, not a data-science one.
2. **Vector retrieval done well is a narrower skill than vector retrieval.** The thesis deliberately scopes vectors to exemplar/dedup/unfamiliar-dialect. Those are discrete, testable problems — not "do RAG correctly over arbitrary docs."
3. **Adopting a library shifts the risk without reducing it.** If mem0's hybrid scorer underperforms on your corpus, you're debugging someone else's pipeline through a library boundary. The prior deep-dives on beads and gastown already show the team's strongest move is orchestration + primitives, not ML tuning.

If the competency gap still feels real, the higher-leverage hire is a **retrieval eval engineer** (extends the existing harness with more corpora, failure-mode analysis, golden sets) — not a vector DB.

---

## Suggested next step

Rather than a wholesale swap, a single focused experiment:

Port hindsight's `source_facts`-dict provenance surface and RetainDB's versioned memory schema into the existing retrieval scorer's output, run the private eval, and see whether provenance + version-chain work changes the score. One-week test, real signal on what "memory maturity" is worth *in your measurement*, all primitives intact.

---

## Evaluation rubric (shared across all 7 agents)

Each library was scored 0–3 on:
- Thesis alignment (agent+primitives vs RAG-first)
- Search substrate quality (BM25/FTS with ranking)
- Scope/metadata filtering (entity, date, type)
- Document model depth (vs flat K-V)
- Provenance/citation first-class
- Trust tiers / confidence modeling
- Conversation ↔ artifact separation
- Extensibility (swap-in primitives)
- Scale ceiling for 10k+ docs
- License + maturity

Source docs referenced for the thesis:
- `docs/superpowers/ideas/2026-04-18-retrieval-thesis.md`
- `docs/superpowers/specs/2026-04-12-recall-and-memory-design.md`
- `docs/superpowers/findings/2026-04-19-beads-memory-deep-dive.md`
- `docs/superpowers/findings/2026-04-19-gastown-deep-dive.md`
