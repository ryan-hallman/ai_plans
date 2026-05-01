# Artifact-First Knowledge Extraction Design

## Goal

Define the next Prescient OS knowledge-engine direction after the workshop manual dogfood slice: raw sources should be compiled into maintained, entity-backed artifacts, and broad or specific questions should answer from those artifacts first, with raw sources used as provenance and fallback.

This design resolves two related questions:

1. Broad ambiguous questions like "What is Prescient OS being built to do?" should not be answered primarily by noisy raw retrieval when an approved current-state artifact exists.
2. Workshop manuals should not remain a page-retrieval-only system. Manuals are a raw source layer that should feed extracted observations and maintained vehicle-repair artifacts.

The workshop-manual slice remains valuable because it produced citations, evals, terminology mappings, and failure data. The next step is to make that slice more Prescient-native by compiling manual evidence into durable component/spec artifacts.

## Why This Matters

The current workshop assistant is still close to RAG:

```text
PDF pages -> page units -> search -> LLM answer with page citations
```

That is useful for dogfooding, but it forces the engine to rediscover the same facts on every question. The lower-control-arm torque failure is a good example: the system should not need to find the correct table page from scratch every time a user asks about a known component.

The active KE-first design already calls for durable entity-backed primary artifacts that are updated over time and used as the default retrieval surface. This design applies that principle to workshop manuals and generalizes it for future sources such as chats, emails, forum posts, service bulletins, books, policies, and business documents.

The intended shape is:

```text
raw sources
  -> extracted observations
  -> entity/component/procedure/spec artifacts
  -> trust and validation lifecycle
  -> artifact-first answers with provenance
```

This is the structured Prescient version of the "LLM wiki" pattern: the system should maintain a compact, current, inspectable knowledge surface instead of repeatedly asking a model to reinterpret a pile of raw material. Raw sources still matter, but their role is evidence, auditability, and re-extraction. The default answer surface should be the maintained artifact because that is where scope, contradictions, corrections, and current state can be represented explicitly.

## Non-Goals

This design does not:

- replace raw source retention; raw sources remain the evidence layer
- require perfect extraction before artifacts can be useful
- require every artifact to be human-approved before it can answer
- build a full multi-user abuse-resistance system in the first slice
- choose a permanent external ticketing system
- solve every artifact type across every domain in one implementation plan

The first implementation should stay narrow enough to validate the model.

## Considered Alternatives

### Keep raw RAG as the primary answer path

This is the smallest change, but it preserves the failure mode we are trying to remove. Every answer depends on finding and interpreting the right source units at question time, so terminology gaps, section-level context, and repeated facts keep causing repeated failures. Raw retrieval should remain available, but it should be the fallback and evidence layer rather than the primary knowledge surface.

### Require human approval before any extracted artifact can answer

This maximizes caution, but it makes humans the bottleneck and slows dogfooding. The better first slice is progressive trust: extracted artifacts can answer with visible trust labels, and normal usage becomes the validation loop. High-risk domains can still require approval before use, but vehicle-repair dogfooding should not block all utility on manual review.

### Treat authoritative sources as automatically canonical

Workshop manuals, service bulletins, and approved policies deserve higher default source authority, but authority is not the same as correctness. Extraction can be wrong, source scope can be misapplied, and even official documents can contain mistakes or superseded guidance. Prescient should preserve source authority while still requiring claim-level provenance, validation, disputes, and correction.

### Build only vehicle-specific artifacts

Vehicle-specific artifacts are necessary for answering Ferrari 360 questions, but many mappings and claims generalize across platforms. The artifact model should support both scoped vehicle artifacts and reusable domain concepts such as component aliases, procedure types, and spec claim patterns. This avoids trapping the workshop slice inside one make/model while still allowing vehicle-specific differences.

## Core Principle

Prescient should distinguish four independent trust signals:

1. **Source authority**: how authoritative the originating source is.
2. **Extraction confidence**: how confident the system is that it parsed a claim correctly.
3. **Artifact trust**: how validated or disputed the maintained artifact/claim is.
4. **Validator authority**: how much weight to give a user's feedback in a specific domain/scope.

These signals must not collapse into one opaque confidence score. A claim from an authoritative workshop manual can still be extracted incorrectly. A forum claim can be useful but should not carry the same default trust as a service bulletin. A user can be authoritative for one domain and untrusted in another.

## Source Authority

Every ingested source should receive a source authority profile. This profile affects initial claim confidence and answer presentation, but it does not make any extracted claim automatically true.

Initial authority classes:

| Authority | Examples | Default Use |
| --- | --- | --- |
| `authoritative_reference` | official workshop manuals, service bulletins, approved policies, official specs | high initial trust, still disputable |
| `expert_reference` | published maintenance books, known expert guides, vendor docs | useful but may need corroboration |
| `community_report` | forum posts, owner reports, public comments | low initial trust, useful for leads and corroboration |
| `personal_note` | user notes, shop observations, measurements | scoped to the author unless promoted |
| `unknown` | unclassified imports | low initial trust until classified |

Authority assignment should be hybrid:

- inferred by source type/origin at ingestion
- overridable by the user at source or source-set level
- suggestible by the system when contradictions or provenance patterns warrant review
- never silently promoted across source classes

Contradictions should create claim-level disputes or supersession candidates. They should not automatically demote an entire source.

## Observation Layer

Observations are extracted claims, specs, events, or facts with provenance. They are not yet canonical artifacts.

Each observation should include:

- observation id
- source id and source authority
- source unit/page/message ids
- extracted claim text
- structured value where applicable
- claim type, such as `torque_spec`, `procedure_step`, `applicability`, `cross_reference`
- entity/component/procedure candidates
- applicability scope, such as make/model/year/variant
- extraction method and confidence
- contradiction links
- validation state

For workshop manuals, first useful observation types are:

- torque specs
- fluid/spec values
- component relationships
- procedure references
- section/table applicability
- cross-references such as "see B 5.02"
- variant applicability

For business contexts, analogous observations include:

- decisions
- policy rules
- obligations
- risks
- KPI values
- strategic claims
- timeline events

## Artifact Layer

Artifacts are maintained current-state records assembled from observations.

V1 workshop artifact targets:

- `component_spec_artifact`
- `procedure_artifact`
- `vehicle_record`

The first implementation should focus on `component_spec_artifact`, especially torque/spec claims for Ferrari 360 suspension and adjacent service areas.

Example artifact:

```yaml
artifact_id: artifact-ferrari-360-front-lower-control-arm-spec
artifact_type: component_spec_artifact
entity_ref: component.front_lower_control_arm
scope:
  domain: vehicle_repair
  make: Ferrari
  model: 360 Modena
claims:
  - claim_id: lower_arm_to_chassis_torque
    claim_type: torque_spec
    value: 55
    unit: Nm
    trust_state: pending_validation
    source_authority: authoritative_reference
    extraction_confidence: 0.82
    provenance:
      - source_id: source-ferrari-360-wsm
        unit_id: unit-source-ferrari-360-wsm-p250
      - source_id: source-ferrari-360-wsm
        unit_id: unit-source-ferrari-360-wsm-p266
aliases:
  - lower control arm
  - lower arm
  - LCA
  - wishbone
related_procedures:
  - front_suspension_removal
```

Artifacts may include pending claims. They do not need to be perfect before they become useful, but answer surfaces must expose trust state clearly.

## Artifact Trust Lifecycle

Artifacts and individual claims should support a progressive trust lifecycle:

```text
extracted -> pending_validation -> user_validated -> disputed -> corrected/superseded
```

Definitions:

- `extracted`: created by an extraction process, not yet used as a preferred answer surface unless no better artifact exists
- `pending_validation`: usable in answers with a visible trust label
- `user_validated`: validated by a user with sufficient validator authority for the scope
- `disputed`: challenged by feedback or conflicting evidence
- `corrected`: replaced with a revised claim or artifact version
- `superseded`: no longer current because a newer or more authoritative artifact/claim replaced it

The lifecycle should apply at claim level first. Whole-artifact trust can be derived from claim states, but a single disputed claim should not necessarily invalidate the whole artifact.

## Validator Authority

User feedback must be weighted. This is necessary to prevent accidental or intentional poisoning as the system moves beyond a sole-user dogfood setup.

Initial validator authority levels:

| Level | Meaning |
| --- | --- |
| `system_owner` | authoritative for configured domains/scopes; Ryan is user #1 in dogfood |
| `trusted_expert` | manually granted expert authority for a domain/scope |
| `validated_user` | earned through accepted feedback history |
| `standard_user` | default user feedback creates proposals/disputes |
| `low_trust` | feedback requires review and cannot mutate canonical artifacts |

Authority must be scoped. A user can be authoritative for `vehicle_repair:ferrari_360` without being authoritative for legal docs, finance policy, or another vehicle platform.

Feedback from a high-authority validator may promote or correct claims directly within their scope. Feedback from lower-authority validators should create proposed corrections, disputes, or review tasks.

Promotion and demotion of validator authority should be policy-driven:

- accepted corrections can increase authority within a narrow scope
- rejected or abusive feedback can reduce authority
- broad authority should be manually granted, not inferred from a few interactions

## Answer Behavior

Question answering should be artifact-first:

1. Use an approved or user-validated primary artifact when available.
2. Use pending artifacts when no validated artifact exists, but label the answer as extracted/not validated.
3. Use clustered observations when no artifact exists.
4. Fall back to raw source retrieval when extraction/artifacts do not cover the question.
5. Ask for clarification when entity, vehicle, variant, year, or scope materially changes the answer.

Example answer for a workshop question:

```text
The current extracted spec for the Ferrari 360 Modena lower control arm to chassis is 55 Nm.

Trust: extracted from an authoritative reference, pending user validation.
Sources: Ferrari 360 Workshop Manual p250 and front suspension removal p266.

Note: the evidence appears in the suspension torque table and the front suspension removal section. Confirm whether you mean the front or rear lower arm if that distinction matters.
```

Available user actions:

- looks right
- wrong
- partially right
- wrong scope
- needs more context
- show source pages

For broad questions such as "What is Prescient OS being built to do?", the system should default to the approved primary artifact for `company_context` or product direction, then optionally show active workstreams as supporting context. It should not let raw historical docs overrule a maintained current-state artifact without surfacing a contradiction.

Broad-query policy:

- If one approved primary artifact clearly matches the query scope, answer from that artifact first.
- If several artifacts could answer and they do not conflict, synthesize from the highest-trust artifacts and show the scopes used.
- If artifacts materially conflict, surface the disagreement and ask for clarification or review rather than averaging them together.
- If only raw sources match, answer from raw retrieval with a lower trust label and create an opportunity to compile an artifact.
- If a user validates or corrects the answer, feed that event back into the artifact lifecycle instead of treating it as ephemeral chat feedback.

## Feedback Triage Loop

Normal usage should become the validation workflow. The user should be able to mark an answer or claim wrong and explain why in natural language.

Example feedback:

```text
Wrong. That torque is for the upper arm, not the lower arm. The table applies to F3, not F1.
```

A triage agent should classify feedback into one or more outcomes:

- `claim_correction`: value, component, procedure, applicability, or alias needs adjustment
- `evidence_correction`: wrong page cited, missing section context, or table applies more broadly
- `scope_correction`: wrong make/model/year/variant/entity
- `source_contradiction`: sources disagree and need dispute tracking
- `extraction_rule_gap`: parser missed a table, header, section relationship, or unit structure
- `retrieval_rule_gap`: right artifact/source exists but query expansion or ranking failed
- `manual_review_needed`: evidence is conflicting or too low confidence
- `engineering_ticket`: system/tooling behavior requires code or pipeline changes

The triage agent may propose artifact updates, observation updates, eval cases, or issue drafts. It should not silently overwrite high-trust claims unless validator authority and policy permit it.

## Issue Sink Boundary

Beads is acceptable for local dogfood but must not become product architecture.

Use a generic `IssueSink` port:

```text
feedback -> triage_agent -> IssueDraft -> IssueSink
```

Implementations:

- local dogfood: `BeadsIssueSink`
- hosted/product: Linear, Jira, GitHub Issues, or internal ticket API
- enterprise: customer ticketing integration

The domain should emit an `IssueDraft` with structured fields:

- failing question
- answer shown
- user feedback
- artifact ids
- observation ids
- source ids and page/message ids
- suspected failure category
- reproduction steps
- eval-case seed when relevant

No domain logic should depend on Beads directly.

## Architecture Boundaries

Core domain concepts:

- `Source`
- `SourceAuthorityProfile`
- `Observation`
- `Artifact`
- `ArtifactVersion`
- `ArtifactClaim`
- `ValidationEvent`
- `ValidatorProfile`
- `IssueDraft`

Application ports:

- `SourceIngestor`
- `ObservationExtractor`
- `ArtifactAssembler`
- `ArtifactRepository`
- `ArtifactAnswerService`
- `FeedbackTriageAgent`
- `IssueSink`

Workshop-specific adapters:

- PDF/manual source ingestor
- torque/spec extractor
- component/spec artifact assembler
- vehicle-repair terminology/profile adapter

Future adapters:

- chat/email/forum ingestors
- policy/contract extractors
- business-entity artifact assemblers
- external ticketing issue sinks

## First Implementation Slice

The first implementation should stay narrow:

1. Extract Ferrari 360 suspension and adjacent service torque/spec observations from existing manual units.
2. Assemble pending `component_spec_artifact` records.
3. Add artifact-first answer path for component/spec questions.
4. Show trust labels and source citations.
5. Capture validation feedback from the system owner.
6. Route tooling/code-change feedback to Beads through an `IssueSink` adapter.

This slice is intentionally not a full manual-understanding engine. It proves that artifact-first retrieval improves dogfood answers and produces a better improvement loop than page retrieval alone.

## Testing Strategy

Minimum coverage:

- source authority defaulting and user override behavior
- observation extraction from known Ferrari 360 manual pages
- claim/artifact trust transitions
- validator authority weighting for feedback
- artifact-first answers include trust labels and citations
- raw retrieval fallback still works when no artifact exists
- feedback triage classification for claim correction, scope correction, extraction gap, retrieval gap, and issue draft
- eval-case creation from disputed or corrected answers

The lower-control-arm torque question should become a regression case:

- before artifact extraction, retrieval may miss the correct page
- after artifact extraction, artifact-first answer should surface the compiled spec with page citations and trust state

## Open Questions

These should be resolved during implementation planning, not in this design:

- exact storage schema for observations and artifacts
- whether file-backed JSON remains acceptable for the first dogfood artifact store or whether this is the point to introduce Postgres tables
- exact confidence scoring formula
- exact validator promotion/demotion thresholds
- whether pending artifacts should answer by default in all domains or only in configured dogfood domains

## Success Criteria

The design is successful when:

- workshop torque/spec questions can answer from compiled artifacts rather than raw page search alone
- source authority, extraction confidence, artifact trust, and validator authority are visible and separate
- user feedback can correct or dispute extracted knowledge without poisoning canonical artifacts
- system/tooling failures become structured issues through an adapter boundary
- the pattern generalizes from workshop manuals to business docs, chats, emails, forums, and other heterogeneous sources
