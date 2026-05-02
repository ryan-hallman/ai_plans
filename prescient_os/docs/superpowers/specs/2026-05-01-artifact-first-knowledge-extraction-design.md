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
- build `procedure_artifact`, `vehicle_record`, or broad `company_context` artifacts in v1

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

### Keep artifacts in file-backed JSON for dogfood

File-backed JSON was acceptable for immutable workshop ingest outputs because those files were produced by batch jobs and read by retrieval. Artifacts are different: they mutate, carry event history, point to many sources, get disputed, and need current-state projections. JSON files would make concurrency, versioning, and provenance queries harder immediately. V1 should commit to Postgres for observations, artifacts, artifact claims, validation events, and current projections.

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

Observation `claim_type` and artifact-claim `claim_type` share the same closed type space. V1 starts with `torque_spec`, `procedure_step`, `applicability`, and `cross_reference`; later domains may add values, but observation and artifact projections should not drift into separate enums.

Applicability should be structured even when v1 stores it in a flexible JSON field:

```yaml
applicability:
  domain: vehicle_repair
  make: Ferrari
  model: 360 Modena
  variant: null      # null means applies to all variants in the scoped family
  year_range: [1999, 2005]
```

Missing applicability fields should lower extraction confidence and may require clarification before answer use.

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

## V1 Extraction Technique

Extraction is the load-bearing primitive, so the v1 slice should be deliberately narrow and measurable.

For Ferrari 360 torque/spec extraction, use a deterministic-first pipeline:

1. Read already-ingested manual units and page locators.
2. Extract candidate table rows and nearby section/table headers from PDF text and structured page metadata.
3. Parse numeric values, units, component terms, fastener terms, and applicability terms with explicit rules.
4. Use the terminology profile to normalize aliases such as `LCA`, `wishbone`, `lower arm`, and `lower control arm`.
5. Use an LLM only as a bounded classifier/normalizer when deterministic parsing produces candidates, not as the sole source of a value.
6. Emit observations with provenance to exact source units/pages and extraction-run metadata.

Bounded LLM use means the model chooses among already-extracted candidates or normalizes scoped text, not invents a claim. Examples: disambiguating `lower arm` between front and rear when section context is unclear, choosing the correct unit when OCR strips nearby unit labels, or resolving an abbreviation against the active terminology profile.

Initial extraction confidence should be explainable rather than opaque. It should be derived from visible factors:

- numeric value and unit were parsed cleanly
- candidate came from a table or labeled spec section
- nearby headers support the component/procedure scope
- terminology normalization has a known mapping
- multiple source units agree on value and scope
- contradictory values exist for the same candidate claim
- required applicability fields are missing

`extraction_confidence` is a single 0.0-1.0 score for sorting, thresholds, and answer policy. It must be accompanied by structured `confidence_factors` that list the visible factors used to compute the score so a user or eval can inspect why a claim was considered strong or weak.

When the same candidate fact appears with different values or scopes, the extractor should emit separate observations and mark them as conflicting candidates. The artifact assembler should not average or silently pick a value. It should either choose the highest-supported claim with a visible dispute note or create a disputed draft claim for validation.

Extraction must be re-runnable. Each run should have an extraction-run id, parser/profile version, input source snapshot, and output observation ids so improved OCR, better table extraction, new terminology mappings, or new manual editions can produce new observations without overwriting the old ones.

The input source snapshot is the immutable source-unit view the extractor read: source ids, content fingerprints, source-unit ids, source-unit text fingerprints, locator/page metadata fingerprints, and active parser/profile versions. It is not a copy of every raw PDF byte; it is the versioned reference that lets a later run explain exactly which indexed source state produced an observation.

## Artifact Layer

Artifacts are maintained current-state records assembled from observations.

Near-term workshop artifact types:

- `component_spec_artifact`
- `procedure_artifact`
- `vehicle_record`

The first implementation should only build `component_spec_artifact`, especially torque/spec claims for Ferrari 360 suspension and adjacent service areas. Procedure and vehicle-record artifacts are Phase 2 because procedure extraction is substantially harder than extracting structured torque/spec values.

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
    trust_state: extracted
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

Artifacts may include extracted, not-yet-validated claims. They do not need to be perfect before they become useful, but answer surfaces must expose trust state clearly.

Terminology workbench mappings and artifact aliases should be one system, not two. The workbench should create scoped alias claims, and the artifact assembler should project those aliases into relevant artifacts. Artifact-local aliases may still be displayed for convenience, but the durable source of truth is the scoped terminology/alias claim so `vehicle_repair_v1`, `ferrari_v1`, `ferrari_360_v1`, and source-specific mappings do not drift from artifact records.

A terminology mapping can exist before a matching artifact exists. In that case it remains in the terminology layer and is projected into future artifacts when an assembler later creates an artifact whose scope/entity matches the mapping's applicability layers.

`artifact.scope` should correspond to the active retrieval-profile layer set, even if v1 stores a compact flat object for convenience. For Ferrari 360 dogfooding, the artifact scope should be derivable from layers such as `domain:vehicle_repair:v1`, `org:ferrari:v1`, and `product:ferrari_360:v1`; future business/legal domains should not need a separate scoping system.

## Storage And Versioning

V1 should use Postgres for mutable artifact data.

Required storage concepts:

- observations, keyed by extraction run and source provenance
- artifacts, keyed by artifact id/type/scope
- artifact claims, keyed by artifact and claim id
- artifact aliases, projected from scoped terminology mappings
- validation events, keyed by user/validator profile and claim/artifact
- artifact events, stored append-only
- current artifact projections for fast answer-time lookup

`ArtifactVersion` should be minimal but explicit. V1 does not need a complex branching version model; it needs an append-only event log plus a current projection:

```text
artifact_event -> rebuild current artifact projection
```

Examples of artifact events:

- `artifact_created`
- `claim_added`
- `claim_validated`
- `claim_disputed`
- `claim_corrected`
- `claim_superseded`
- `alias_added`
- `alias_removed`

Every current claim should point back to the events and observations that produced it. This lets evals pin expected behavior to a known artifact version and lets a user inspect what the system used to believe before a correction.

## Artifact Trust Lifecycle

Artifacts and individual claims should support a progressive trust lifecycle. V1 should implement only three answer-relevant states:

```text
extracted -> validated -> disputed
```

Definitions:

- `extracted`: created by an extraction process and usable in answers with a visible not-validated trust label when no validated claim exists
- `validated`: validated by a user with sufficient validator authority for the scope
- `disputed`: challenged by feedback or conflicting evidence

`corrected` and `superseded` are event types in v1, not separate current trust states. A correction creates a new current claim version and records the prior claim in history. Phase 2 may add richer states such as `pending_validation`, `needs_review`, or `superseded_current_source`, but those should wait until real usage demands them.

The lifecycle should apply at claim level first. Whole-artifact trust can be derived from claim states, but a single disputed claim should not necessarily invalidate the whole artifact.

## Validator Authority

User feedback must be weighted. This is necessary to prevent accidental or intentional poisoning as the system moves beyond a sole-user dogfood setup.

The long-term validator authority model has multiple levels:

| Level | Meaning |
| --- | --- |
| `system_owner` | authoritative for configured domains/scopes; Ryan is user #1 in dogfood |
| `trusted_expert` | manually granted expert authority for a domain/scope |
| `validated_user` | earned through accepted feedback history |
| `standard_user` | default user feedback creates proposals/disputes |
| `low_trust` | feedback requires review and cannot mutate canonical artifacts |

V1 should implement only `system_owner` and explicit "not implemented" guards for the other levels. Ryan is the configured `system_owner` for the dogfood vehicle-repair scope. This keeps the schema compatible with the future trust model without paying the complexity cost of unused multi-user authority.

Authority must be scoped. A user can be authoritative for `vehicle_repair:ferrari_360` without being authoritative for legal docs, finance policy, or another vehicle platform.

Feedback from a high-authority validator may promote or correct claims directly within their scope. Feedback from lower-authority validators should create proposed corrections, disputes, or review tasks.

Promotion and demotion of validator authority should be policy-driven:

- accepted corrections can increase authority within a narrow scope
- rejected or abusive feedback can reduce authority
- broad authority should be manually granted, not inferred from a few interactions

## Answer Behavior

Question answering should be artifact-first:

1. Ask for clarification when entity, vehicle, variant, year, or scope materially changes the answer.
2. Use a validated primary artifact claim when available.
3. Use extracted artifact claims when no validated claim exists, but label the answer as extracted/not validated.
4. If observations exist but no artifact exists, present a draft artifact candidate or fall back to raw retrieval; do not introduce separate clustering behavior in v1.
5. Fall back to raw source retrieval when extraction/artifacts do not cover the question.

Example answer for a workshop question:

```text
The current extracted spec for the Ferrari 360 Modena lower control arm to chassis is 55 Nm.

Trust: extracted from an authoritative reference, not yet user-validated.
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

Broad `company_context` and product-direction artifacts are Phase 2. They use the same artifact/version/event model, but they are living strategic artifacts rather than stable component-spec artifacts. V1 should keep the broad-query policy in the design while implementing the tractable workshop component-spec slice first.

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

Triage outcomes should route to different sinks:

- claim, evidence, scope, source-contradiction, and manual-review outcomes mutate artifact/observation state or create review events
- extraction and retrieval gaps should seed eval cases when they describe a reproducible answer failure
- only `engineering_ticket` should create an `IssueDraft`

## Issue Sink Boundary

Beads is acceptable for local dogfood but must not become product architecture.

Use a generic `IssueSink` port only for engineering tickets:

```text
feedback -> triage_agent -> engineering_ticket -> IssueDraft -> IssueSink
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

No domain logic should depend on Beads directly.

Eval-case creation should use a separate eval-case path rather than being embedded in `IssueDraft`. Claim corrections and scope corrections often need regression coverage even when they do not require a code ticket.

## Eval Case Sink Boundary

`EvalCaseSink` is the application port for turning reproducible knowledge failures into regression coverage. It should accept structured eval-case drafts with:

- failing question
- expected behavior
- scope id and retrieval profile/layer set
- source-of-truth artifact id and claim id when available
- required evidence source ids, unit ids, and page/message ids
- originating answer id, observation ids, validation event ids, or triage event ids
- suspected failure category
- answer text shown to the user
- expected correction or required claim text when known

This keeps eval promotion separate from issue creation. A claim correction may need a regression case without a code ticket, while an engineering failure may need both an eval case and an `IssueDraft`.

## Re-Extraction Policy

Re-extraction is a normal maintenance operation, not a special migration.

Trigger re-extraction when:

- a new manual edition or source version is ingested
- OCR, page rendering, or table extraction improves
- terminology mappings change in a way that affects component/spec normalization
- extraction rules or model prompts change
- a disputed claim indicates the extractor missed section/table context

Re-extraction should create a new extraction run and new observations. The artifact assembler should compare new observations to current claims, preserve prior artifact events, and create validation or dispute events when new evidence changes a claim. It should not overwrite current artifacts silently.

Conflict handling should be explicit:

- If new observations agree with the current claim, add the observation ids to the claim's supporting provenance without changing trust state.
- If new observations disagree and have higher extraction confidence or stronger source authority, create a dispute event for validation; do not auto-supersede the current claim.
- If new observations refine applicability, such as adding a year range or narrowing a variant, create a `claim_corrected` event with refinement payload and preserve the prior broader claim in history.

## Relationship To Retrieval And Agent Layer

The existing page-aware retrieval pipeline remains actively maintained in v1. It has three roles:

- fallback when no artifact covers the question
- evidence display for artifact citations
- benchmark comparator for evaluating whether artifact-first answers are better

Artifact-first should launch behind a flag or request option until evals show a clear answer-quality improvement over raw retrieval. Raw retrieval should stay exercised by tests and evals so it does not atrophy.

The agent layer is redefined rather than replaced. For artifact-covered questions, the agent should select scope, retrieve the right artifact, follow artifact cross-references, decide whether clarification is needed, and only then fall back to raw retrieval. For uncovered or multi-source questions, the agent still decomposes the task and uses retrieval primitives. Artifact-first narrows the agent's search space; it does not remove the need for agent orchestration.

## Architecture Boundaries

Core domain concepts:

- `Source`
- `SourceAuthorityProfile`
- `Observation`
- `Artifact`
- `ArtifactVersion`
- `ArtifactClaim`
- `ArtifactEvent`
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
- `EvalCaseSink`

Workshop-specific adapters:

- PDF/manual source ingestor
- deterministic-first torque/spec extractor
- component/spec artifact assembler
- vehicle-repair terminology/profile adapter
- Postgres artifact repository

Future adapters:

- chat/email/forum ingestors
- policy/contract extractors
- business-entity artifact assemblers
- external ticketing issue sinks

## First Implementation Slice

The first implementation should stay narrow. The current dogfood branch has landed the initial backend spine:

- Postgres-backed side store for extraction runs, observations, artifacts, artifact claims, validation events, feedback triage events, and current projections.
- deterministic-first Ferrari 360 torque/spec extraction from existing manual units.
- `component_spec_artifact` assembly.
- artifact-first answer path behind an explicit request option.
- trust labels, source citations, claim identity, system-owner validation, and feedback triage/regression seed capture.

Remaining implementation should be sequenced by dependency:

1. Manually verify extraction quality against a 20-30 question/page sample before expanding artifact coverage.
2. Add eval-case promotion for reproducible claim, scope, extraction, and retrieval failures.
3. Route only `engineering_ticket` triage outcomes through an `IssueSink` adapter; keep eval-case creation on the separate `EvalCaseSink` path.
4. Compare artifact-first answers with raw retrieval answers on the eval set before making artifact-first the default.

Parallelization guidance:

- Extraction-rule improvements and manual verification can proceed together, but artifact coverage should not expand until the verification sample is reviewed.
- Eval-case promotion and `IssueSink` wiring are independent adapter slices once triage events exist.
- Artifact-first defaulting is blocked on the comparison report, not on the existence of extracted artifacts alone.

This slice is intentionally not a full manual-understanding engine. It proves that artifact-first retrieval improves dogfood answers and produces a better improvement loop than page retrieval alone.

## Testing Strategy

Minimum coverage:

- source authority defaulting and user override behavior
- deterministic-first observation extraction from known Ferrari 360 manual pages
- extraction confidence factor reporting
- extraction run replay/re-extraction behavior
- Postgres artifact repository persistence and current projection rebuild
- claim/artifact trust transitions
- v1 `system_owner` validation behavior and guards for unsupported validator levels
- artifact-first answers include trust labels and citations
- raw retrieval fallback still works when no artifact exists
- feedback triage classification for claim correction, scope correction, extraction gap, retrieval gap, eval-case seed, and engineering ticket
- eval-case creation from disputed or corrected answers without requiring an issue draft

The lower-control-arm torque question should become a regression case:

- before artifact extraction, retrieval may miss the correct page
- after artifact extraction, artifact-first answer should surface the compiled spec with page citations and trust state

## Open Questions

These should be resolved during implementation planning, not in this design:

- exact Postgres table shapes and indexes
- exact extraction confidence scoring weights and thresholds after manual calibration
- exact 20-30 question/page sample for manual extraction verification
- exact flag/request shape for comparing artifact-first answers against raw retrieval
- exact validator promotion/demotion thresholds for Phase 2
- whether extracted artifacts should answer by default in all domains or only in configured dogfood domains

## Success Criteria

The design is successful when:

- workshop torque/spec questions can answer from compiled artifacts rather than raw page search alone
- source authority, extraction confidence, artifact trust, and validator authority are visible and separate
- user feedback can correct or dispute extracted knowledge without poisoning canonical artifacts
- system/tooling failures become structured issues through an adapter boundary
- the pattern generalizes from workshop manuals to business docs, chats, emails, forums, and other heterogeneous sources
