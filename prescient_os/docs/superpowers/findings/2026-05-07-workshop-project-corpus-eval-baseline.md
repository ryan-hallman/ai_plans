# Workshop Project-Corpus Eval Baseline

## Summary

Ran the workshop manual eval baseline after making project-local
`corpus/workshop_manuals` the canonical generated corpus.

Command:

```bash
make eval-workshop-baseline
```

Generated scorecard:

```text
eval/runs/workshop_eval_20260507T044324Z/workshop_scorecard.yaml
```

`eval/runs/` is gitignored, so this finding records the durable baseline summary.

## Results

- Questions: 46
- Scope accuracy: 1.000
- Citation coverage: 0.587
- Citation accuracy: 0.059
- Failed questions: 19

Category results:

| Category | Questions | Coverage | Accuracy | Failed |
| --- | ---: | ---: | ---: | ---: |
| catalog | 1 | 0.000 | 0.000 | 1 |
| cross_reference | 1 | 0.000 | 0.000 | 1 |
| interval | 1 | 1.000 | 0.100 | 0 |
| procedure | 22 | 0.545 | 0.055 | 10 |
| spec | 5 | 0.800 | 0.080 | 1 |
| torque | 16 | 0.625 | 0.062 | 6 |

## Notable Pattern

Many failed Ferrari 360 WSM questions returned the same top candidates,
especially `unit-source-ferrari-360-wsm-p591`,
`unit-source-ferrari-360-wsm-p592`, and early source pages, instead of the
required units. This suggests the next investigation should classify failures
before changing eval expectations.

Possible causes:

- evidence keys shifted after replacing the old incomplete English PDF with the
  complete multilingual PDF
- retrieval ranking is over-weighting generic/high-frequency pages
- extracted text from some pages is sparse or noisy
- query expansion is not reaching the relevant section/table terms

## Follow-Up

Tracked in Beads as `prescient_os-zw2`.
