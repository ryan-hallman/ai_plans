# Review: Model-Assisted Eval Harness Design (2026-04-17)

## Overall

Solid bones. File-based-first is right for a POC, the retrieval-vs-synthesis score split is valuable, and the iteration metadata (`change_type`, `change_note`) is unusually thoughtful. Main weaknesses: grader independence isn't addressed, and several comparison-integrity guarantees are implicit rather than enforced.

## Strengths

- Separating retrieval support from synthesis quality instead of one opaque score.
- Scores-vs-failure-tags framing ("how good" vs. "what to change") is the right decomposition.
- Human-as-final-pass without blocking iteration loop.
- Capability detection for Gemini instead of a hard dependency.
- Layout cleanly separates `answers/` from `grades/`, supporting re-grading without re-answering.

## Gaps and Risks

### 1. Grader/answerer independence (§ Grader Adapters)

If Claude answers and Claude grades, that's a self-preference bias. Spec should require `answer_model ≠ grader_id`, or at minimum record and flag it. Currently not addressed.

### 2. Calibration baseline (§ Success Criteria)

Before the grader fleet is trusted to guide iteration, it needs a correlation check against human scores on a held-out calibration set. Add a criterion such as "graders reach ≥X agreement with human scores on N calibration questions" — otherwise a score delta could be a system change or grader drift, and you can't tell which.

### 3. Prompt/corpus change = re-grade policy (§ Run Tracking)

`grader_prompt_version` and `corpus_version` are versioned but the spec doesn't state that changing either invalidates prior comparisons and requires re-running. The `grades/` separation supports re-grading without re-answering — make that explicit.

### 4. `system_id` vs `iteration_id` for A/B (§ Run Tracking)

Only one `system_id: local_rag` is shown. Clarify whether competing retrieval strategies are different `system_id`s (true A/B) or different `iteration_id`s (timeline progression). Summary semantics differ for each.

### 5. Grader failure handling (§ Grader Output Schema)

What happens when a grader returns invalid JSON, times out, or crashes? Spec says "validate against schema" but no retry/skip/mark-failed policy. Without this, a flaky grader silently poisons a run.

### 6. Grader-side telemetry (§ Grader Output Schema)

`latency_ms` is tracked on answers but not on grader outputs. Track grader latency and token cost — you'll want it when deciding which graders to keep.

### 7. Closed failure taxonomy (§ Failure Taxonomy)

Add `other` with required free-text, or the harness will miss novel failure modes and silently force them into ill-fitting bins.

### 8. Disagreement resolution (§ Summary Generation)

"Largest disagreement" is surfaced but no resolution rule. Cheap default: disagreement above threshold automatically sets `needs_human_attention=true`.

### 9. Ordinal scores averaged as interval (§ Grader Output Schema)

Averaging 1–5 Likerts is conventional but technically lossy. Report median + distribution alongside mean, or acknowledge the limitation.

### 10. Grader prompt missing from implementation order (§ Implementation Order)

The grader prompt (`grader_v1.md`) is the crown jewel but isn't a numbered step. Insert "draft and pilot grader prompt on 5 questions" between steps 2 and 3, or list it as a prerequisite.

## Minor

- Schemas are JSON, records are YAML — fine, but state the rationale once so a reviewer doesn't re-litigate it.
- Gemini "capability probe" is underspecified — one sentence on what it does (trivial prompt → valid JSON?).
- Prompt injection risk from corpus/answer content flowing into grader context isn't in Risks. Low risk for public Peloton docs, but worth a line.

## Recommendation

Address items 1–4 before implementation; they determine whether the harness can actually distinguish system changes from grader noise. Items 5–10 can be tightened in the plan phase.
