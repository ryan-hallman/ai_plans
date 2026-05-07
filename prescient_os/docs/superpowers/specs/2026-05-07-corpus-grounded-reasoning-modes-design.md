# Corpus-Grounded Reasoning Modes Design

## Goal

Define how Prescient should answer questions that are not directly answerable by
quoting or summarizing one source passage, but can still be answered usefully by
reasoning over corpus-backed evidence.

The motivating vehicle-repair example is:

> When I am driving and I load up the suspension taking a right turn, I have a
> lot of vibrations. What could be wrong?

The workshop manual does not directly diagnose that symptom. It can still ground
an answer by providing suspension, steering, wheel-bearing, alignment, tire,
mount, and inspection procedures. The system should support that kind of answer
without pretending the manual directly states a cause.

## Why This Matters

The current workshop assistant mostly behaves like direct evidence retrieval:

```text
question -> retrieve pages -> answer if evidence directly supports answer
```

That is correct for torque specs and procedures, but too narrow for a knowledge
engine. Users also ask diagnostic, hypothetical, mechanism, and planning
questions. These require inference, but the inference should be constrained,
auditable, and clearly labeled.

The broader Prescient pattern should generalize beyond cars:

- vehicle repair: "What could cause vibration under cornering load?"
- business: "Why might sales be slowing?"
- legal/policy: "What changes if we adopt this clause?"
- research: "What does this result imply for the next experiment?"

The common requirement is not "repair intent." The common requirement is
corpus-grounded reasoning with explicit separation between evidence and
inference.

## Core Decision

Prescient should model a generic **reasoning mode** or **answer mode** above the
domain profile.

The domain profile supplies vocabulary, source authority, retrieval expansions,
and safety rules. The reasoning mode describes the user's question shape and the
answer contract.

```text
domain profile: vehicle_repair_v1
reasoning mode: diagnostic
```

is different from:

```text
domain profile: business_strategy_v1
reasoning mode: diagnostic
```

but both share the same evidence/inference discipline.

## Reasoning Modes

Initial generic modes:

| Mode | User Shape | Answer Contract |
| --- | --- | --- |
| `direct_lookup` | "What is the torque/spec/value for X?" | Answer only from directly supporting evidence. |
| `procedure` | "How do I remove/install/check X?" | Use source-backed steps and cite relevant pages. Light explanation is allowed when labeled. |
| `catalog_lookup` | "Where are all X?" or "List the X references." | Retrieve and synthesize list/table/catalog evidence across source regions. |
| `diagnostic` | "I observe symptom Y under condition Z. What could be wrong?" | Generate ranked hypotheses backed by related evidence, with inference labels and clarifying questions. |
| `mechanism_explanation` | "How does X work?" or "Why does X cause Y?" | Explain mechanisms using source evidence where available, and label general domain knowledge separately. |
| `hypothetical` | "What happens if X?" | Reason through possible consequences, separating source-backed constraints from inferred outcomes. |
| `work_planning` | "What should I inspect/do while X is apart?" | Build a practical plan from procedures, access overlap, specs, and source-backed inspection points. |

These are not domain-specific names. Vehicle repair can specialize retrieval and
presentation inside each mode, but the mode taxonomy should remain reusable.

## Claim Labels

Answers in non-direct modes must classify claims. V1 labels:

- `manual_evidence`: the corpus directly supports the statement
- `artifact_evidence`: a maintained artifact supports the statement
- `inference`: the system inferred the statement from evidence and domain reasoning
- `general_knowledge`: useful domain knowledge not directly sourced from the active corpus
- `needs_user_observation`: the next step requires user-provided observation
- `suggested_check`: a practical action or inspection step
- `safety_relevant`: a warning or stop condition

Direct lookup answers should mostly contain `manual_evidence` or
`artifact_evidence`. Diagnostic, mechanism, hypothetical, and planning answers
will usually contain a mixture.

## Evidence Discipline

The system may say:

> This symptom is consistent with a loaded-side wheel bearing, control-arm
> bushing, tire/wheel, alignment, or steering component issue.

It must not say:

> The workshop manual says the cause is a wheel bearing.

unless the manual directly states that diagnostic conclusion.

The answer must make source support explicit:

1. **Manual-backed facts:** cited procedures, specs, diagrams, component
   relationships, inspection steps, or tolerances.
2. **Reasoned hypotheses:** labeled inference from symptom, condition, and
   source-backed system knowledge.
3. **Clarifying observations:** what the user should observe next to narrow the
   branch.
4. **Safety boundaries:** conditions where the system should recommend stopping
   work or avoiding use until inspection.

## Vehicle Repair Diagnostic Example

For:

> vibration while loading suspension in a right turn

the answer should reason as follows:

- A right turn tends to increase load on the left-side suspension and wheel
  assembly. This is an inference, not a manual quote.
- Retrieve manual-backed evidence for suspension arms, ball joints, shock
  mounting, wheel bearing checks, steering joints, alignment/ride height data,
  and related torque/procedure pages.
- Ask whether the vibration is felt through the steering wheel, seat/chassis,
  brake pedal, throttle load, or only at specific speeds.
- Suggest an inspection order that prioritizes safety and likely loaded-side
  components.

The answer should include page citations for the manual-backed inspection and
spec material. It should label the loaded-side hypothesis as inference.

## Relationship To Artifacts

Artifact-first remains the long-term default for stable knowledge. Reasoning
modes do not replace artifacts; they define how the system uses artifacts and
raw evidence when the user asks for reasoning.

For example:

- A torque artifact can answer `direct_lookup`.
- A procedure artifact can support `work_planning`.
- A diagnostic artifact may eventually encode confirmed symptom-cause patterns.
- Raw manual pages remain evidence and fallback when artifacts are missing.

User feedback can promote reasoning outcomes into future artifacts only after
validation. A user saying "it was the left wheel bearing" should become a
candidate observation or issue, not an automatically canonical diagnostic rule.

## UI Implications

The UI should not require the user to pick a mode before every question. V1
should auto-detect the likely mode and show the selected mode visibly.

The UI should allow override when detection is wrong:

- Direct
- Procedure
- Troubleshoot
- Explain
- What-if
- Plan work

For diagnostic/troubleshooting mode, the UI should present answer sections that
make evidence and inference easy to scan.

## API Implications

The API should separate:

- `domain_profile_id`, such as `vehicle_repair_v1`
- `reasoning_mode`, such as `diagnostic`
- existing lower-level retrieval intent details, such as torque or catalog
  expansion policies

This avoids hard-coding "repair intent" as the top-level abstraction and keeps
the answer contract portable to other domains.

## Non-Goals

This design does not:

- implement a full diagnostic expert system
- guarantee causal diagnosis from symptoms alone
- allow unsupported repair advice to masquerade as manual evidence
- ingest forums or community reports yet
- build a full multi-user feedback authority system
- replace direct evidence answers for exact spec/procedure questions

## Implementation Direction

The first implementation should be narrow:

1. Add mode classification for the workshop ask route.
2. Add `diagnostic` as the first non-direct reasoning mode.
3. Retrieve broader related evidence across vehicle systems.
4. Use a diagnostic answer template requiring labeled evidence, inference,
   clarifying questions, and safety notes.
5. Add one or two eval cases for symptom-style questions.

Mechanism, hypothetical, and planning modes should use the same model later,
but they do not need to ship in the first slice.
