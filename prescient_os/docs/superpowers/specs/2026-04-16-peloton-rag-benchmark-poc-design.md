# 2026-04-16 Peloton RAG Benchmark POC Design

## Goal

Validate, as narrowly and honestly as possible, whether a business-specific local retrieval system can outperform general-purpose systems on a realistic, large, document-heavy business corpus.

This POC is not trying to prove the full knowledge-engine architecture. It is trying to answer one immediate question:

> Given the same corpus, can a strong local retrieval system produce better grounded answers than ChatGPT, Gemini, and a CLI filesystem-search agent?

## Core Thesis

The immediate thesis is retrieval-first, not curation-first:

> On a controlled corpus centered on one company, a business-oriented local retrieval stack can beat leading general-purpose systems on answer quality for operator questions and dense long-document questions.

This POC exists to validate retrieval quality before heavier entity-state, curation, governance, or workflow architecture is introduced.

## POC Scope

### In Scope

- One anchor company: Peloton
- One controlled corpus combining:
  - real Peloton public materials
  - LLM-generated synthetic internal documents
  - anonymized long-form real-world documents
- One local indexed retrieval system
- Two hosted comparators:
  - ChatGPT with the same files loaded into a project
  - Gemini/NotebookLM with the same files loaded through the strongest practical document-grounding workflow
- One CLI baseline that answers from raw files using filesystem tools only
- Human-scored evaluation

### Out Of Scope

- Multi-company setup
- Entity graph or canonical entity state
- Artifact curation workflows
- Drafts, approvals, timelines, unresolved inboxes
- Visibility and ACL modeling beyond local handling of the corpus
- Packages and workflow surfaces
- Broad platform UX

## Corpus Design

The corpus should be realistic, reproducible, and scalable.

### Anchor Company

Use Peloton as the public-company anchor because it has rich public materials and a sufficiently understandable but non-trivial operating profile.

### Corpus Components

1. **Public company materials**
   - EDGAR filings
   - investor presentations
   - earnings materials
   - other clearly attributable public documents as needed

2. **Synthetic internal documents**
   - LLM-generated from scenario seeds
   - intended to feel like plausible internal company material that generic hosted systems would not know from pretraining

3. **Anonymized long-form real-world documents**
   - operating agreement
   - RCAs
   - insurance policy
   - similar dense documents where generic systems often struggle with clause-level retrieval and cross-section synthesis

### Synthetic Document Categories

Start with:

- executive strategy memo
- initiative or project updates
- meeting notes
- KPI or operating review summaries
- policy, org, or process docs

### Corpus Tooling Requirements

The corpus must be reproducible.

Build tooling for:
- Peloton public-data import/fetch
- seed-driven synthetic internal-doc generation using an LLM
- corpus manifest generation
- controlled scaling of corpus size over time

Generated outputs should be frozen into versioned files so benchmark runs can be repeated.

## Systems Under Test

### 1. Local Indexed Retrieval System

This is the system being tested.

It should:
- ingest the local corpus from files
- parse and chunk documents well
- index them into a strong local retrieval stack
- produce synthesized answers with strict citations

The retrieval stack should be explicit and serious enough that a loss teaches something real. A weak RAG implementation is not an acceptable baseline.

Current design direction:
- OpenSearch-based retrieval
- strong parsing
- semantic chunking
- hybrid retrieval
- reranking
- synthesized answer generation with strict citations

### 2. ChatGPT Baseline

Load the same raw corpus files into a ChatGPT project manually and ask the same question set.

### 3. Gemini Baseline

Load the same raw corpus files through the strongest practical Gemini document-grounding path, likely NotebookLM or equivalent.

### 4. CLI Filesystem Baseline

Test a CLI agent over the raw filesystem corpus.

Rules:
- raw files only
- shell/file tools allowed
  - `rg`
  - `find`
  - direct file reads
- no custom preprocessing or metadata-manifest retrieval layer in v0

This baseline exists because adaptive filesystem search may outperform chunked indexed retrieval on large corpora, and that hypothesis should be tested directly.

## Answer Mode

All systems should be evaluated on synthesized answers with strict citations.

Requirements:
- every substantive claim should be grounded in retrieved evidence
- citations should point to exact or near-exact source locations/chunks where possible
- the preferred failure mode is insufficient evidence, not confident bluffing

This makes the comparison fairer because ChatGPT, Gemini, the local RAG system, and the CLI baseline are all being judged on useful final answers, not on snippet dumping.

## Evaluation Design

### Primary Goal

Measure answer quality first. Speed is secondary.

### Question Categories

Use a mixed eval set including:

1. operator/business questions
2. fact lookup and synthesis questions
3. dense long-document questions

The dense long-document category is especially important because that is a known failure mode for generic systems and a likely area of differentiation.

### Baseline Fairness Rule

Each external system gets the same corpus through its strongest practical document-grounding workflow.

This benchmark is not comparing against naked pretrained model memory. It is comparing grounded systems over the same files.

### Scoring

Use human scoring first.

Primary evaluation dimensions:
- groundedness
- answer quality/usefulness
- specificity
- completeness
- handling of dense long-document detail

Secondary observations:
- latency
- citation quality
- failure mode behavior

### Quality-First Evaluation Rule

Optimize each system for best answer quality, not fixed response-time budget, in the initial POC.

Latency should still be recorded, but it is not the primary decision variable at this stage.

## Retrieval Design Priority

The local system should be retrieval-first.

Do not add ingestion-time curation artifacts, entity-backed records, or heavier KE state machinery in this POC unless the benchmark later shows a concrete retrieval failure that those mechanisms are meant to solve.

The first thing to validate is:

> Can retrieval over a large business corpus work better than the hosted systems and better than raw filesystem search?

## Deliverables

The POC should produce:

- a reproducible Peloton benchmark corpus
- a local retrieval system over that corpus
- a documented question set
- captured answers from:
  - local RAG
  - ChatGPT
  - Gemini
  - CLI filesystem agent
- a human-scored comparison record

## Success Condition

The POC is successful if it gives a clear answer to at least one of these:

- the local indexed retrieval system is meaningfully better than the hosted baselines
- the CLI filesystem agent is meaningfully better than the indexed retrieval system
- the local retrieval stack loses in a specific, diagnosable way that clarifies what the next architecture bet should be

The POC is not successful merely because a lot of infrastructure was built.

## Next Step

Write a narrow implementation plan focused on:

- corpus tooling
- local retrieval pipeline
- eval harness
- side-by-side benchmark workflow

Do not use the broader KE-first foundation spec as the implementation target for this POC.
