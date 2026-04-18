# Eval Feedback: Drop Synthesis Grading, Measure Retrieval Directly

## Recommendation

Stop grading the current RAG baseline on synthesis dimensions. Restructure the eval to measure retrieval quality directly against a ground-truth evidence key. Keep the synthesis rubric in the schema for later, but remove it from primary scoring until a real synthesizer exists.

## Why this is the right move

### 1. The current eval mixes two independent layers and can't tell them apart

Every retrieval approach we want to test (current RAG, full-text search, hybrid, agent-with-primitives) varies the *retrieval* layer. The *synthesis* layer, when we add one, should be held constant across comparisons. A benchmark that bundles them produces scores that confound "which retrieval strategy is better?" with "which synthesizer is better?" — and you can't separate the two after the fact.

Right now the bundling is even worse: the baseline has no synthesizer at all. The "answer" is a retrieved chunk with a source label prepended. The graders are being asked to score `usefulness`, `completeness`, `specificity`, and `citation_quality` on output that is literally raw retrieval. The rubric is measuring a property the system does not have, and the aggregate is a fiction.

### 2. The retrieval thesis explicitly separates these layers

Our architecture calls for retrieval that narrows, followed by long-context reading that answers. If that's the bet, the benchmark has to be able to evaluate each layer in isolation. Bundling them in the eval contradicts the architectural position we just committed to. You cannot benchmark a separable system with a fused metric.

### 3. The current grader stack is already evidence of the bundling cost

The codex grader's scores contradict its own rationales on multiple questions. Even Claude's more coherent scores are constrained to guess at synthesis quality in output that has no synthesis. The grader variance we're seeing is partly a bug in codex, but partly a symptom of asking LLM graders to assess dimensions that don't apply to the artifact. Fixing codex does not fix the category error.

### 4. Intrinsic retrieval metrics are strictly better than LLM grading for this layer

For retrieval specifically, the right metrics are deterministic and computable:

- **Fact recall** — of the facts a correct answer would require, what fraction appear in any retrieved passage?
- **Document recall** — did retrieval surface all `expected_docs`?
- **Passage precision** — what fraction of retrieved passages are actually relevant?
- **Rank** — where in the retrieval order did the required evidence appear?

These have no grader variance, cost nothing per run, are trivially reproducible, and are directly optimizable. You can tune retrieval parameters against a numerical target without paying for a grading API call on every iteration. LLM graders cannot match any of those properties.

### 5. An evidence key is answer-agnostic and retrieval-method-agnostic

Per-question, author a list of passages in the corpus that contain the facts a correct answer would need. Version it. Validate it. Then:

- It works for any synthesizer you bolt on later.
- It works for BM25, dense, hybrid, agentic — anything that surfaces passages.
- It works at passage or document granularity depending on the question.
- It's auditable: you can point at exactly which required passages were found or missed.

The grader-variance problem disappears. The codex brokenness becomes irrelevant to primary scoring. Cross-approach comparison becomes trivial.

### 6. This is the right moment to do it, not later

The eval infrastructure has to exist to harden anything. Deferring the evidence-key work to "when we compare approaches" means either building the benchmark twice or retrofitting comparison logic onto scores that were never designed to support it. Doing it now means the v0 RAG baseline is measured on the axis we actually care about — and every subsequent retrieval change gets a deterministic, trustworthy delta.

## What changes in practice

**Rename the artifact.** `answers/` → `retrievals/`. Answer records → retrieval records. This removes the framing that invites synthesis grading.

**Primary scoring becomes intrinsic retrieval metrics.** Fact recall, document recall, passage precision, rank. Computed deterministically against the evidence key. No LLM calls on the critical path.

**LLM grader kept, narrowly.** One question per retrieval: "is this retrieval bundle sufficient to answer the question?" Binary or 1–5. Used for calibrating the evidence key and sanity-checking edge cases, not for primary scoring. Optional per run.

**Dropped from primary scoring (schema retained for future synthesizer runs):**

- `usefulness`, `specificity`, `completeness`, `citation_quality` — synthesis dimensions
- `hallucination_flag` — not meaningful without synthesis

**Kept:** `retrieval_relevance`, `retrieval_sufficiency` — but now these are redundant with the intrinsic metrics. Either retire them or keep as a human-readable narrative layer.

## What this enables

- Deterministic, cheap, reproducible retrieval benchmarks.
- Clean cross-approach comparison: same questions, same evidence key, different retrievers, comparable scores.
- Retrieval changes become tunable against a numerical target — you can run hundreds of parameter sweeps without grading cost.
- Synthesis evaluation, when added later, runs on top of a known-good retrieval layer rather than being tangled with it.

## What this costs

- Roughly half a day to author the evidence key for the current 10 questions, with LLM assistance surfacing candidate passages that a human confirms. The corpus-coverage pass already identified most of the needed passages.
- Ongoing: when questions change, the affected keys update. Low marginal cost.

## What it leaves open

- End-to-end user-experience quality isn't captured and will need a separate evaluation once a synthesizer exists.
- Questions with diffuse evidence (e.g., q_public_009's "biggest risks" across a 10-K) require the key to encode "any N of these M passages" rather than a strict all-or-nothing match. This is a question-design issue, not an eval-architecture one.
- The evidence key has authorship bias, but it's a fixed, auditable, versioned bias — not a moving grader bias. Disagreements about the key become visible and debatable rather than hidden in grader rationales.

## Related

- Corpus and eval asset locations: `docs/superpowers/findings/2026-04-18-peloton-corpus-and-eval-locations.md`
- Retrieval architecture thesis: `docs/superpowers/ideas/2026-04-18-retrieval-thesis.md`
- Original eval harness design: `docs/superpowers/specs/2026-04-17-model-assisted-eval-harness-design.md`
