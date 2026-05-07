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

Reasoning modes are **agent policies and answer contracts**, not static prompt
templates. A mode defines:

- how the question is classified
- which retrieval primitives the agent may use
- how many retrieval/reasoning loops are allowed
- which claim labels may appear
- what must be verified before the answer is returned
- when the system should ask the user for more observations instead of
  continuing to infer

The implementation may use templates for presentation, but a template is not the
mode. This keeps the design aligned with the retrieval thesis: agent plus
retrieval primitives over a fixed RAG-first pipeline.

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

`direct_lookup`, `procedure`, and `catalog_lookup` describe behavior that
already exists in the workshop slice. The new design does not reimplement those
from scratch. It names the existing behavior so new reasoning modes can compose
with it and share one answer contract.

Only `diagnostic` is validated by current workshop dogfooding demand. The
mechanism, hypothetical, and planning modes are planned extensions. They should
not be fully implemented until real user questions or eval cases justify them.
The diagnostic-first slice is based on current workshop usage patterns such as
vibration under cornering load, "what could be wrong?" symptom prompts, and
manual-backed troubleshooting questions that need inspection guidance rather
than a single quoted procedure.

## Mode Detection Policy

Auto-detection is part of the product behavior, not a UI convenience. V1 should
use a conservative classifier with this shape:

```yaml
reasoning_mode_decision:
  primary_mode: diagnostic
  secondary_modes: [procedure]
  confidence: 0.82
  reasons:
    - symptom described
    - operating condition described
    - asks what could be wrong
  clarification_question: null
```

Classification order:

1. **Rule-first cues for high-confidence cases.** Exact torque/spec/value
   questions should map to `direct_lookup`; "how do I remove/install/check"
   should map to `procedure`; "where/list all" should map to `catalog_lookup`;
   symptom + operating condition + "what could be wrong" should map to
   `diagnostic`.
2. **LLM classifier for boundary cases.** Use an LLM only when rule cues are
   weak or multiple modes are plausible. The LLM must return a structured
   decision, confidence, and reasons.
3. **Multi-mode when appropriate.** Questions such as "How do I check wheel
   bearings, and what would cause vibration if they are bad?" should produce
   procedure and diagnostic segments rather than forcing a single winner.
4. **Clarify instead of guessing.** If confidence is low or the requested mode
   changes the safety posture, the system should ask a concise clarifying
   question before giving a diagnostic or hypothetical answer.

Initial confidence thresholds:

- `confidence >= 0.75`: proceed with the selected mode.
- `0.60 <= confidence < 0.75`: proceed only when the answer can be segmented
  into clearly labeled modes; otherwise ask for confirmation.
- `confidence < 0.60`: ask a clarifying question before retrieving broadly or
  producing diagnostic/hypothetical reasoning.

These thresholds are starting defaults. They should be revisited after the
first diagnostic eval runs.

Mode boundary defaults:

- `direct_lookup` wins when the user asks for an exact source-backed value.
- `procedure` wins when the user asks how to perform an operation.
- `diagnostic` wins when the user describes a symptom or failure condition.
- `mechanism_explanation` wins when the user asks how/why something works,
  without asking for a repair recommendation.
- `hypothetical` wins when the user asks about a counterfactual or consequence.
- `work_planning` wins when the user asks what to combine, inspect, or sequence.

The UI should show the detected mode and allow override, but the classifier must
be good enough that most users can leave it on auto.

If retrieval evidence conflicts with the classifier decision, evidence should
trigger reclassification or segmentation. For example, a diagnostic-classified
question may retrieve an exact source-backed procedure; the answer should expose
the direct procedure evidence and then separately label any diagnostic
inference. Conversely, if a direct-looking question retrieves no direct evidence
but does retrieve relevant inspection context, the system should not silently
turn it into a diagnostic answer; it should ask whether the user wants
troubleshooting guidance.

## Claim Labels

Answers in non-direct modes must classify claims. Labels are not free-form prose;
they must be attached to structured answer claims or answer sections.

V1 labels:

- `manual_evidence`: the corpus directly supports the statement
- `artifact_evidence`: a maintained artifact supports the statement
- `inference`: the system inferred the statement from evidence and domain reasoning
- `general_domain_knowledge`: useful domain knowledge not directly sourced from
  the active corpus
- `needs_user_observation`: the next step requires user-provided observation
- `suggested_check`: a practical action or inspection step

Minimum structured representation:

```yaml
claim:
  text: Right turns tend to load the left-side suspension more heavily.
  label: inference
  source_unit_ids: []
  locator_ids: []
  artifact_claim_ids: []
  depends_on_claim_ids: [manual-claim-1, manual-claim-2]
  confidence: medium
  safety_relevant: false
```

`safety_relevant` is a boolean modifier on a claim or check, not a claim label.
It marks stop conditions, warnings, or inspections where unsafe operation or
incorrect action could cause harm.

V1 confidence values:

- `high`: directly supported by cited corpus/artifact evidence or a narrow
  inference from strong evidence.
- `medium`: supported by relevant evidence but requiring domain reasoning or an
  unresolved user observation.
- `low`: plausible but weakly supported; should usually be framed as a question
  to investigate, not as a recommendation.

Label enforcement rules:

- `manual_evidence` requires at least one source unit id and locator id from the
  retrieved corpus.
- `artifact_evidence` requires at least one artifact claim id.
- `inference` requires at least one evidence input. This can be non-empty
  `depends_on_claim_ids`, direct source-unit/locator references, or artifact
  claim references.
- `general_domain_knowledge` must be explicitly marked as not sourced from the
  active corpus and cannot be used as the sole support for a repair conclusion,
  legal conclusion, medical conclusion, or other safety-sensitive action.
- `suggested_check` must either cite a manual-backed procedure/spec or point to
  the inference chain that motivates it.
- `needs_user_observation` is for information the user must provide, such as
  "Does it happen under braking?" It is not the same as `suggested_check`.

Direct lookup answers should mostly contain `manual_evidence` or
`artifact_evidence`. Diagnostic, mechanism, hypothetical, and planning answers
will usually contain a mixture.

`general_domain_knowledge` is disallowed in `direct_lookup`. In `procedure`, it
is limited to short explanatory framing and must not add uncited procedural
steps. In `diagnostic`, it may help rank or explain hypotheses, but it must not
create a definitive cause or safety-sensitive recommendation without corpus or
artifact support.

Before returning an answer, a support verifier should reject or downgrade claims
that violate these rules. In V1, this can be deterministic validation of the
structured output. Later versions may add a separate verifier model, but the
first implementation should not rely on prompt compliance alone.

Verifier behavior on violations:

1. Reject invalid `manual_evidence` and `artifact_evidence` labels when the
   required source-unit/locator or artifact-claim ids are missing.
2. Allow one agent retry to repair labels or remove unsupported claims.
3. If retry still fails, remove the unsupported claim from the answer or move it
   to `general_domain_knowledge` only when the mode allows that label and the
   claim is not safety-sensitive.
4. Never downgrade an unsupported evidence claim into `inference` unless the
   claim has valid evidence inputs through `depends_on_claim_ids`, source
   references, or artifact references.
5. Return partial insufficient evidence when a required answer section cannot be
   repaired without violating label rules.

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

If the support verifier cannot validate a claim label after the retry budget,
the answer should either ask for clarification or return insufficient evidence
for that part of the answer.

It should not silently present an unverifiable claim as evidence-backed.

## Agent Policy And Retrieval Loop

Diagnostic mode should use a bounded reasoning loop:

1. Classify the symptom, operating condition, and affected systems.
2. Retrieve broad evidence for those systems.
3. Form a small set of hypotheses.
4. Re-retrieve targeted evidence to support or weaken each hypothesis.
5. Decide whether to answer, ask for user observations, or refuse to infer.
6. Validate claim labels before returning.

V1 budget:

- maximum 4 retrieval calls per diagnostic answer
- maximum 5 hypotheses before pruning
- maximum 30 source units passed to the final answer step
- target interactive latency: under 15 seconds on the local Docker stack when
  the LLM provider is responsive

If the budget is exhausted, the answer should say which hypotheses remain
unresolved and what observation would narrow them.

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

For diagnostic/troubleshooting mode, the UI should present answer sections that
make evidence and inference easy to scan.

The override list should be small in the first UI slice. Recommended labels:

- Direct
- Procedure
- Troubleshoot
- Explain
- What-if
- Plan

If the classifier returns multiple modes, the UI should show segmented answer
sections rather than forcing the user to rerun the question.

## API Implications

The API should separate:

- `domain_profile_id`, such as `vehicle_repair_v1`
- `reasoning_mode`, such as `diagnostic`
- existing lower-level retrieval intent details, such as torque or catalog
  expansion policies

This avoids hard-coding "repair intent" as the top-level abstraction and keeps
the answer contract portable to other domains.

The API should expose the mode decision and claim labels in structured form, not
only in rendered answer text. A V1 response should include:

- selected primary mode
- optional secondary modes
- mode confidence and reasons
- labeled answer sections or claims
- citation/source-unit ids attached to evidence labels
- retrieval diagnostics and budget usage
- estimated token usage when available

Existing direct/procedure answer fields can remain for compatibility, but the
new reasoning response shape should be available to the UI and MCP surfaces.
Token-budget controls are not part of the first workshop diagnostic slice, but
the API should preserve enough diagnostics to add them later.

## Evaluation And Success Criteria

Reasoning modes are hallucination-sensitive and require a higher eval bar than
direct lookup.

V1 diagnostic eval should include:

- at least 12 symptom-style questions across suspension, steering, wheel/hub,
  drivetrain, and braking-adjacent symptoms
- at least 4 adversarial questions where the corpus does not support a specific
  cause and the expected behavior is to ask for observation or give only
  bounded hypotheses
- label-correctness checks for `manual_evidence`, `inference`,
  `general_domain_knowledge`, `suggested_check`, and `needs_user_observation`
- citation validation for every `manual_evidence` claim
- a regression metric for unsupported evidence labels

Initial success target before enabling diagnostic mode by default:

- zero `manual_evidence` claims without valid source-unit and locator ids
- zero definitive causal diagnoses when the corpus only supports hypotheses
- mode classification accuracy is tracked but not a hard release gate until the
  eval set reaches at least 25 mode-classification examples
- no safety-relevant answer that omits a stop/inspect boundary when the prompt
  describes severe vibration, braking instability, steering looseness, or wheel
  looseness

Once the mode-classification eval has at least 25 examples, the initial hard
gate should be at least 85% correct primary-mode classification with every
misclassification reviewed for safety impact.

User usefulness can be tracked later, but it is not a substitute for label and
citation correctness.

## Artifact Promotion Scope

Feedback-driven artifact promotion is explicitly deferred from the first
reasoning-mode slice.

For V1, user feedback on a reasoning answer may create:

- raw feedback
- a proposed eval seed
- a proposed observation candidate
- a Beads issue when code or extraction behavior appears wrong

It must not automatically create a canonical diagnostic artifact. Promotion into
artifacts requires a separate validation workflow that defines reviewer
authority, source support, contradiction handling, and audit trail.

## Non-Goals

This design does not:

- implement a full diagnostic expert system
- guarantee causal diagnosis from symptoms alone
- allow unsupported repair advice to masquerade as manual evidence
- allow `general_domain_knowledge` to act as hidden evidence
- ingest forums or community reports yet
- build a full multi-user feedback authority system
- replace direct evidence answers for exact spec/procedure questions

## Implementation Direction

The first implementation should be narrow:

1. Add mode classification for the workshop ask route.
2. Add `diagnostic` as the first non-direct reasoning mode.
3. Add structured claim labels and deterministic label validation.
4. Retrieve broader related evidence across vehicle systems with the diagnostic
   retrieval budget.
5. Use a diagnostic agent policy requiring labeled evidence, inference,
   clarifying questions, and safety notes.
6. Add the V1 diagnostic eval set and label-correctness checks.

Mechanism, hypothetical, and planning modes should use the same model later,
but they do not need to ship in the first slice.
