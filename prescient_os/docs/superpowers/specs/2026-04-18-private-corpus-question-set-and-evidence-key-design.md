# Private Corpus Question Set And Evidence Key Design

## Goal

Define the first retrieval-benchmark assets for the private PrescientOS corpus:

- a private question set
- a matching evidence-key schema
- a first authored batch of benchmark questions that can be scored against
  explicit corpus truth rather than broad synthesis quality

This slice is about benchmark assets and validation shape, not retrieval scoring
implementation yet.

## Why This Slice Exists

The benchmark has already shifted in two important ways:

- PrescientOS, not Peloton, is now the primary benchmark company and corpus
- retrieval-only evaluation is the primary posture, with synthesis scoring
  parked

That means the next bottleneck is no longer corpus import. It is benchmark
definition:

- what questions should the system answer over the private corpus?
- what evidence counts as sufficient?
- how do we encode truth conditions for point-in-time conflicts and superseded
  beliefs?

The answer is a small private question set paired with explicit evidence keys.

## Benchmark Scope

The first private benchmark should be built against:

- snapshot: `prescient-os-chats-2026-04-18-with-retrieval-only-turn`

The eventual first set should contain roughly `10-15` questions, but the
initial implementation should only author the first `5`.

Those questions should focus on:

- pivots
- superseded claims
- point-in-time conflicts
- scope disambiguation
- “what was believed then vs later?”

The set should deliberately require both:

- curated chronology memos
- raw chat sessions

This is important because the real retrieval problem is not “find the summary.”
It is “retrieve across summaries plus the messy underlying company history.”

## Question Set Shape

The first private set should be deliberately mixed.

Recommended composition:

- `3-4` memo-led questions
  - the memo should get the system most of the way there
  - raw chat can corroborate or add detail
- `4-5` mixed-source questions
  - correct retrieval should need both a chronology memo and one or more raw
    sessions
- `2-3` raw-chat-led questions
  - the memo is absent, incomplete, or too compressed

The first authored batch of `5` questions should cover:

- one memo-led chronology question
- two mixed-source pivot questions
- one raw-chat-led question
- one conflict or superseded-belief question

Recommended question types:

- `current_state_at_time`
- `change_over_time`
- `why_pivoted`
- `scope_disambiguation`
- `superseded_belief`

The first set should stay biased toward chronology and product-direction
questions, not engineering trivia.

## Evidence Key Schema

Each question needs an evidence-and-interpretation key rather than a prose gold
answer.

Recommended per-question evidence-key fields:

- `question_id`
- `question_type`
- `prompt`
- `corpus_version`
- `required_docs`
- `required_sources`
  - memo IDs and/or session IDs
- `required_claims`
- `acceptable_evidence`
  - source reference plus locator or message range
- `resolution_rule`
  - `latest_wins`
  - `must_compare_periods`
  - `must_surface_conflict`
  - `memo_plus_raw`
  - `raw_only`
- `minimum_sufficient_set`
- optional `notes_on_conflicts`

This schema should support deterministic scoring later on:

- source recall
- claim recall
- rank of first required evidence
- whether the minimum sufficient evidence set was retrieved

It is intentionally not a prose answer key.

## First-Pass Implementation Target

Keep the first implementation pass narrow and manual.

Implement:

- a new private question-set asset
- a matching private evidence-key asset
- schema and validation support in the benchmark codebase
- only the first `5` authored questions and evidence keys

Do not implement in this slice:

- retrieval scoring logic
- answer generation changes
- benchmark-run orchestration changes
- broader corpus or memo authoring work

The immediate purpose is to create the first honest benchmark substrate for the
private corpus, not to finish the whole retrieval evaluation stack.
