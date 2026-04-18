# 2026-04-16 Peloton RAG Benchmark POC — Code Review

## Context

Code review of the Peloton RAG benchmark POC as built on branch `poc/peloton-rag-benchmark` in the code repo.

- Spec: `docs/superpowers/specs/2026-04-16-peloton-rag-benchmark-poc-design.md`
- Plan: `docs/superpowers/plans/2026-04-16-peloton-rag-benchmark-poc.md`
- Paired review: `docs/superpowers/findings/2026-04-16-ke-first-foundation-spec-review.md` (prior critique of the KE-first foundation spec and the Peloton POC spec that replaced it)

## Bottom Line

**The code faithfully executes the implementation plan, but the plan builds scaffolding — not the benchmark.** Every module named in the plan exists, every test passes, TDD discipline is visible in the commit trail — yet almost every "serious" commitment from the spec is a placeholder. The spec's own warning is the relevant verdict:

> *"A weak RAG implementation is not an acceptable baseline."*

What is in the repo today is the weakest possible version of RAG (BM25 + string-concatenation "answer"), with no corpus loaded, no LLM calls wired, no comparator harness. Running the benchmark today would compare an empty index against a missing filesystem against two hosted services with no runbook.

## Plan-vs-Code Conformance: Strong

| Plan task | Delivered |
|---|---|
| 1. Bootstrap FastAPI + Docker + health | Yes |
| 2. Corpus manifest + public import plan | Yes (hardened path validation, stricter pydantic than plan asked for) |
| 3. Synthetic doc generation | Yes (prompt builder only — no LLM call) |
| 4. Parsing/chunking/indexing | Yes (lexical only) |
| 5. Synthesized answers with citations | Yes (template-string "answer", no LLM call) |
| 6. Eval questions + export + scorecard | Yes (3 questions, CSV header only) |
| 7. CLI filesystem baseline | Yes (toy Python substring match) |
| 8. End-to-end smoke | Yes |

## Spec-vs-Code Conformance: Weak

The spec lists six core technical commitments for the local system. The code delivers shells for all six and substance for none.

| Spec commitment | What's implemented | Verdict |
|---|---|---|
| "Strong parsing" | `re.sub(r"\s+", " ", ...)` whitespace normalization. `pymupdf` / `bs4` in deps but unused. | Stub. |
| "Semantic chunking" | Split on `\n\n` + word-count cap. No embedding-similarity or layout-aware boundaries. | Mislabeled — this is paragraph chunking. |
| "Hybrid retrieval" | OpenSearch `multi_match` (lexical only). `text-embedding-3-large` is configured but never called. No dense vectors anywhere. | Not hybrid. Lexical only. |
| "Reranking" | `sorted(results, key=score, reverse=True)` — OpenSearch already returns sorted results. No cross-encoder, no rerank model. | No-op. |
| "Synthesized answers with strict citations" | `f"Based on {title}, {text}"` with one citation to the top chunk. No LLM call; `ANSWER_MODEL` is configured but never used. | Template string. Citation is degenerate — the "answer" is the chunk. |
| "Reproducible corpus" | 3 hardcoded URL stubs, no fetcher. No LLM-generated synthetic docs exist. No intake for anonymized long-form docs. `corpus/generated/` is gitignored and empty. | Corpus does not exist. |

From the spec's "Deliverables" list:

- **Reproducible Peloton corpus** — none of the three sources (public, synthetic, anonymized long-form) have files or an ingest path.
- **Local retrieval system over that corpus** — exists but has nothing to retrieve.
- **Documented question set** — 3 questions, no ideal answers, no category balance plan, no scoring rubric.
- **Captured answers from the 4 systems** — no capture workflow for any system. `export_question_packet()` writes `# q1\n\n{prompt}\n` — that is not a baseline-runbook packet.
- **Human-scored comparison record** — `scorecard.csv` is a CSV header. Zero scoring logic.
- **CLI filesystem agent** — `cli.py` returns the first file where any question word appears as a substring. Not an agent. Does not use `rg`, `find`, or a model. The spec called for adaptive filesystem search; this is a 15-line string match.

## What the Code Gets Right

Worth naming, because the discipline is visible:

- **Package layout is clean**, matches the plan, no operator-first contamination from the archived branch.
- **Strict pydantic models** (`extra="forbid"`, `StrictStr`) across corpus, synthetic, and citation models — good rigor with test coverage of rejection cases.
- **Path hardening on scenario load** (`normalize_scenario_path`) prevents traversal, non-file, unreadable manifests. This is genuinely better than the plan required.
- **Tests go beyond happy-path** — chunking tests cover oversized paragraphs and word-count boundaries; citation tests cover extras rejection; scenario tests cover scalar coercion.
- **TDD trail is clear** in commits, and commits are small and focused.
- **Config split** between `config.py` and the test `conftest.py` OpenSearch client is reasonable.
- **Citations are typed** (commit `7331aa9`) — nicely done.

## Specific Issues Worth Flagging

### Correctness / likely to break

- `docker-compose.yml` and `apps/api/src/prescient_benchmark/config.py` both set `ANSWER_MODEL: gpt-5.4-mini`. **That is not a real model ID.** When the LLM path is wired, this will 404 immediately. Use a real model ID (e.g., `gpt-4o-mini`, `gpt-4.1-mini`, or whichever is available).
- `docker-compose.yml` sets `OPENSEARCH_INITIAL_ADMIN_PASSWORD` alongside `plugins.security.disabled: "true"`. The password is ignored when security is disabled. Harmless but misleading.
- `pyproject.toml` puts `pytest` and `pytest-asyncio` in both `dependencies` and the `dev` extras. Redundant.

### Dead code / cruft

- `retrieval/search.py` — `rerank()` is called on an already-sorted OpenSearch result list. Currently a no-op. Either replace with a real reranker or remove to avoid implying rerank happens.
- `retrieval/search.py` — `hybrid_search()` re-indexes documents on every call. Used only by the integration test. Either rename (it is not hybrid, and in production it should not re-index per query) or split into `ensure_indexed` + `search`.
- `tests/integration/test_benchmark_smoke.py` — the `if hasattr(app.state, "fixture_documents"): delattr(...)` block has no producer anywhere in the code. Stale from a refactor. Remove.
- `tests/integration/test_benchmark_smoke.py` — creates a second `TestClient(app)` right after constructing the first one. Use one.
- `eval/models.py` defines `EvalQuestion` but nothing ever loads `eval/questions/peloton_v1.yaml` into it. The model and the YAML are disconnected.
- `.gitignore` lists `apps/web/node_modules` and `apps/web/.next` even though the plan explicitly forbids `apps/web`. Dead entries.

### Design smell vs. spec

- `CorpusDocumentPlan.source_url: StrictStr` — should be `HttpUrl` since it is meant to be a fetchable URL.
- `Scenario.sections: dict[str, dict[str, str]]` is loose on purpose, but a typed `SectionConfig` model (with `description: str` and room for future fields) would match the strictness elsewhere.
- Synthetic-doc prompts are a generic template per doc type with only a seed/company substitution. If ever called, expect thin/repetitive output. Real seeding needs per-category context (time period, initiative names, KPIs, org structure).
- `answer.py` picks evidence by word-overlap with the question — essentially a second, weaker retrieval pass over the top OpenSearch hits. Once a real LLM synthesis step exists, this overlap re-rank is noise. Drop it; keep top-k from the retriever; let the LLM synthesize from the evidence pack.

## The One Structural Risk

The code is **scaffolding-complete but functionality-empty**, and every missing piece is a "next PR" story. That is the shape where it is easy to declare victory prematurely — "the benchmark service is running, tests pass, the plan is done" — and then discover that the hard 80% (real parser, dense retrieval, reranker, LLM synthesis, corpus ingest, comparator runbooks, ideal answers, scoring harness) has not started.

**This is the same failure mode the pivot was meant to escape, at a different altitude.** Last time it was shallow features across many bounded contexts; this time it is shallow implementations across every layer of a benchmark stack. The fix is the same: pick the thing that actually tests the thesis (retrieval quality on real documents against real baselines) and make *that* deep before anything else gets polished.

## Recommended Next Moves, Ordered

1. **Pick and wire a real LLM answer path.** Read top-k from OpenSearch, pass to a real model (pick a real model ID), constrain output to cite chunk_ids from the evidence pack, add a grounding-verification step. Highest-value single move — the "answer" today is not an answer.
2. **Add dense retrieval.** OpenSearch kNN with a `text-embedding-3-large` vector field. Fuse BM25 + kNN via RRF or weighted score. Then "hybrid_search" is actually hybrid.
3. **Add a real reranker.** Cross-encoder (bge-reranker-v2-m3) or Cohere rerank. Without rerank, the top-k handed to the LLM is noisy.
4. **Ingest real documents.** Fetch 10-K from EDGAR. Download Q2 FY25 materials. Parse with pymupdf (tables + layout retained). Drop in 1–2 long-form real docs (anonymized). Freeze to `corpus/generated/peloton_v1/` with manifest hash.
5. **Generate the synthetic corpus.** Real seed scenarios per doc type, 3–5 docs each, LLM-driven, committed as artifacts with seed/model/hash metadata.
6. **Grow eval to 20–30 questions with pre-written ideal answers and a blinded side-by-side rubric.** Commit the rubric and the decision rule *before* any runs.
7. **Write the ChatGPT / NotebookLM runbooks.** Exact model, date, file-upload procedure, one-shot vs. multi-turn policy. Capture answers to `eval/results/{system}/{question_id}.md` with metadata.
8. **Replace the toy CLI baseline with a real agent harness.** Pinned model, pinned tools (`rg`/`find`/read), pinned turn and token budget.
9. **Write the scorecard driver.** Load questions → run each system → persist answers → present blinded pairs for scoring → aggregate.

Only at step 9 does the repo actually have a benchmark.
