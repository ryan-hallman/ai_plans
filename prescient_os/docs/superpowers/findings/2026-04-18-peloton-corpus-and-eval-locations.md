# Peloton Corpus and Eval Asset Locations

## Corpus

Corpus root:
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public`

Manifest:
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/manifest.yaml`

Raw files:
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/raw/10-k-latest.htm`
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/raw/10-q-latest.htm`
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/raw/10-q-prior.htm`
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/raw/earnings-release-latest.bin`
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/raw/shareholder-letter-latest.bin`

Normalized files:
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/normalized/10-k-latest.yaml`
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/normalized/10-q-latest.yaml`
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/normalized/10-q-prior.yaml`
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/normalized/earnings-release-latest.yaml`
- `/home/rhallman/Projects/prescient_os/corpus/generated/peloton_v1/public/normalized/shareholder-letter-latest.yaml`

## Eval Assets

Eval root:
- `/home/rhallman/Projects/prescient_os/eval`

Primary public question set:
- `/home/rhallman/Projects/prescient_os/eval/questions/peloton_public_v1.yaml`

Older/alternate question file also present:
- `/home/rhallman/Projects/prescient_os/eval/questions/peloton_v1.yaml`

Grader prompt:
- `/home/rhallman/Projects/prescient_os/eval/prompts/grader_v1.md`

Grader schema:
- `/home/rhallman/Projects/prescient_os/eval/schemas/grader_output_v1.json`

## Current Baseline Run

Run root:
- `/home/rhallman/Projects/prescient_os/eval/runs/local_rag_20260418T174548Z`

Run metadata:
- `/home/rhallman/Projects/prescient_os/eval/runs/local_rag_20260418T174548Z/meta.yaml`

Run summary:
- `/home/rhallman/Projects/prescient_os/eval/runs/local_rag_20260418T174548Z/summary.yaml`

## Important Note

There is not yet a separate canonical expected-answer key on disk.

Current grading guidance lives inside the question-set YAML through:
- `prompt`
- `expected_docs`
- optional `scoring_notes`

If stricter grading is needed later, the next logical addition is an explicit answer-key layer per question, for example:
- `expected_answer_outline`
- `required_facts`
- `disallowed_misses`
