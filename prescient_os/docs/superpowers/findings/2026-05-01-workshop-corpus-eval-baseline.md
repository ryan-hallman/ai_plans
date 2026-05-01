# Workshop Corpus Eval Baseline

Date: 2026-05-01

## Context

This finding records the first scored workshop-manual retrieval baseline for the 360 dogfood corpus. The purpose is to make retrieval quality measurable, so subsequent query expansion, reranking, agent, and cross-reference work can be judged by grounded citation lift instead of anecdotal answers.

The run used the local generated workshop corpus at `/home/rhallman/.cache/prescient/workshop_manuals` and the Docker OpenSearch index `prescient-workshop-v1`. The committed eval asset is `eval/questions/workshop_manuals_v1.yaml`; the generated manual corpus and rendered page images remain local artifacts.

## Command

```bash
PYTHONPATH=apps/api/src uv run python -m prescient_benchmark.cli run-workshop-eval-baseline \
  --question-set-path eval/questions/workshop_manuals_v1.yaml \
  --data-root /home/rhallman/.cache/prescient/workshop_manuals \
  --output-root eval/runs \
  --top-k 10
```

Output:

```text
eval/runs/workshop_eval_20260501T054713Z/workshop_scorecard.yaml
questions: 45
scope_accuracy: 1.000
citation_coverage: 0.644
citation_accuracy: 0.064
failed_questions: 16
```

## Scorecard

- Run id: `workshop_eval_20260501T054713Z`
- Question set: `workshop_manuals_v1`
- Questions: 45
- Scope accuracy: 1.000
- Average citation coverage: 0.644
- Average citation accuracy: 0.064
- Mean retrieval latency: 30.8 ms
- Failed questions: 16

## Interpretation

Scope resolution is working for the seeded Modena, Challenge Stradale, and Challenge-family scopes in this baseline. The weak result is citation quality: many queries retrieve plausible pages from the right source set, but the exact expected unit often is not in the top 10, and the candidate set contains many non-required pages. This matches the lower-control-arm failure mode that motivated the workbench: terminology and section-level context are not enough with simple lexical top-k retrieval.

The next backend work should treat the eval as a regression gate for query expansion and reranking. The highest-value metric to move first is citation coverage on torque/procedure questions, because those are the questions where a single missed page can produce unsafe or unusable answers.

## Follow-Up

- Use this scorecard as the pre-agent baseline for future retrieval and agent-layer comparisons.
- Add per-category summary fields to the scorecard if week-over-week review needs faster diagnosis.
- Preserve failed-question IDs so terminology/reranking changes can be checked against the same cases.
