# 2026-04-28 Workshop Manual Dogfood Design

## Goal

Build the first interaction-first dogfood slice of Prescient OS over workshop manuals. The immediate user need is to ask real Ferrari 360 repair questions while working in the shop and receive conservative, page-cited answers that can be inspected quickly.

The product goal is not an automotive vertical. The product goal is to pressure-test the KE-first architecture with a demanding, real corpus where grounding, scope, provenance, and citation quality are non-negotiable.

## Background

The active repository direction is an API-first knowledge engine that privileges correctness, provenance, reviewability, and stable contracts. Workshop manuals are a useful validation corpus because they force the system to handle:

- source scope, such as vehicle, model year, variant, and manual family
- long sections where tables at the beginning govern later procedural steps
- page-level citations and figures that users must be able to inspect
- ambiguous adjacent variants, such as 360 Modena, 360 Challenge Stradale, and 360 Challenge
- proprietary vocabulary and aliases that generic retrieval often misses

The user is actively working on a Ferrari 360, so the first slice should become usable before a complete evaluation suite exists. Real shop questions should then feed the evaluation corpus.

## Relationship To Prior Vehicle-Manuals Idea

This spec supersedes one guardrail in `docs/superpowers/ideas/2026-04-22-vehicle-manuals-validation-corpus.md`.

That idea doc argued for validation through the retrieval scorer only and explicitly rejected Ferrari-specific UI. That was the right caution when the manuals were only a validation corpus. The implementation direction changed because the user identified a stronger iteration loop: the system will improve fastest if it can be used during real shop work, then convert real interactions into evaluation cases.

The decision is:

- Build a dogfood UI and MCP access path because daily use is the fastest way to discover retrieval failures.
- Keep those access paths as thin probes over shared KE services, not as an automotive product surface.
- Continue to treat the scorer and evidence key as the validation authority once enough real questions have been captured.
- Preserve the prior guardrails against car-only artifact types, car-only extractors, and shop-workflow automation.

This is not a pivot from KE-first to automotive. It is a decision to use automotive repair as a high-pressure, personally useful corpus while keeping the underlying source, scope, evidence, retrieval, and citation model horizontal.

## Implementation Placement And Reuse

Implementation should extend the current `prescient_benchmark` code path before introducing new infrastructure.

Existing modules to reuse or evolve:

- `prescient_benchmark/retrieval/index.py` for OpenSearch indexing, index identity, and reuse behavior.
- `prescient_benchmark/retrieval/search.py` for lexical search entry points.
- `prescient_benchmark/retrieval/chunk.py` and `parse.py` as the initial text normalization and chunking baseline, while adding page-aware units for PDFs.
- `prescient_benchmark/retrieval/answer.py` as the nearest existing answer/citation contract, to be replaced or extended by the generic answer contract below.
- `prescient_benchmark/eval/orchestrator.py`, retrieval records, and evidence keys for scoring promoted dogfood interactions.
- Existing FastAPI route patterns under `prescient_benchmark/api/` for the web/MCP backend service boundary.

New work should be placed as a small workshop-manual dogfood slice inside the benchmark package unless the implementation plan identifies a concrete reason to split it. The planned shape is:

- generic source/evidence models under a KE-oriented module
- PDF/manual ingestion adapters under a source-specific module
- web and MCP adapters that call the same application service
- eval promotion utilities that write into the existing eval formats

The implementation plan must state the "why" for each major sequencing and infrastructure choice. In particular, it must explain when we are reusing benchmark infrastructure, when we are extending it, and when we are deliberately deferring the future production architecture.

## Product Scope

V1 is an interaction-first manual assistant for the 360 family.

Included:

- Seed one active vehicle profile for the user's Ferrari 360 Modena.
- Ingest 360-family manuals from `~/Projects/workshop_manuals`.
- Include the base 360 Modena workshop manual and related 360 Challenge, Challenge Stradale, and gearbox manuals.
- Provide a basic web chat probe for repair questions.
- Provide an MCP probe so Hermes can query the same backend when the user is away from the web UI.
- Return conservative answers with exact manual citations.
- Let the user open a cited source page directly from each citation.
- Capture failed, ambiguous, or manually corrected answers as candidate eval cases.

Excluded:

- Vehicle CRUD or a full "garage" management UI.
- A broad automotive workflow product.
- Car-only extraction objects such as a first-class `TorqueSpec`.
- Guessing when the right vehicle, manual, section, or page support is unclear.
- Externalizing the manuals into a hosted third-party knowledge base outside the user's controlled environment.

## Design Principles

1. The shop chat path and the evaluation path must share the same ingestion, catalog, retrieval, answer, and citation contracts.
2. Vehicle concepts should sit at the scope edge, not inside the retrieval spine.
3. Sources must be modeled generically so PDFs, forums, web pages, emails, transcripts, and future business documents can share the same evidence model.
4. Every numeric or procedural claim must cite source evidence. Unsupported claims are omitted or presented as uncertainty.
5. Cloud LLM/OCR providers are acceptable in v1, but provider adapters must be replaceable with local models later.

## Core Concepts

### KnowledgeScope

A bounded scope for answering a question. In v1, the primary scope is the seeded Ferrari 360 Modena profile. Later scopes can be companies, projects, accounts, legal matters, or research collections.

Fields should include:

- stable id
- scope type
- display name
- aliases
- default status
- linked entities
- linked source collections

`KnowledgeScope` owns the generic scope contract. `VehicleProfile` is a linked entity that provides vehicle-specific attributes used by the scope resolver. Query services should depend on `KnowledgeScope` plus resolved source collections, not on `VehicleProfile` directly.

### VehicleProfile

A v1 specialization used to seed the active car.

Fields should include:

- make
- model
- year or year range
- variant
- aliases
- default manual collection
- related manual collections that require explicit query evidence or clarification

No CRUD is required in v1. The profile can be seeded from config or fixture data.

Variant detection should use this profile in two stages:

1. Deterministic alias matching against make, model, variant, year, chassis/engine aliases, and manual-family aliases.
2. LLM-assisted classification only when deterministic matching cannot decide whether a query refers to the default 360 Modena or an adjacent 360 variant.

Low-confidence classification must produce `needs clarification`, not a guessed answer.

### Source

The canonical raw input, regardless of shape.

Examples:

- PDF workshop manual
- forum thread
- forum post collection
- web page
- email thread
- chat thread
- video transcript
- image set

Fields should include:

- source id
- source kind
- title
- origin path or URL
- content fingerprint
- ingestion status
- visibility or retention policy
- provider processing permissions
- applicability metadata

For the first slice, each workshop manual PDF is a `Source`.

`provider processing permissions` should be explicit source-level policy, not an opaque note. V1 should support:

- cloud LLM allowed
- cloud OCR allowed
- cloud embedding allowed
- provider must not retain training data
- provider must not retain inputs beyond service operation
- local-only required

The user's manuals may use cloud processing in v1, but source policy must be modeled so later local-only sources can use the same pipeline safely.

### SourceUnit

An addressable evidence unit inside a source.

Examples:

- PDF page
- extracted figure
- table span
- forum post
- quoted reply
- email message
- transcript segment

Fields should include:

- unit id
- source id
- unit kind
- ordinal or sequence
- extracted text
- OCR confidence when available
- image or rendered artifact reference when available
- locator references

For workshop manuals, pages are the primary source units. Figures and tables may initially be represented as page-bound structure until extraction improves.

### SourceStructure

Inferred structure over source units.

Examples:

- manual section
- procedure
- table range
- figure range
- forum thread hierarchy
- Q&A exchange
- quoted-reply chain

Fields should include:

- structure id
- source id
- structure kind
- heading or label
- unit range
- parent structure id when nested
- extracted aliases
- confidence

### SourceLocator

A stable pointer that opens or identifies evidence.

Examples:

- PDF path plus page number
- rendered page image path
- forum URL plus post anchor
- email message id
- transcript timestamp

Locators are separate from extracted text so citation display can survive re-indexing and parser changes.

### EvidencePassage

The exact evidence used to support an answer.

Fields should include:

- evidence id
- source id
- source unit ids or structure id
- quoted or summarized support text
- locator ids
- support role, such as answer support, scope support, or clarification support

### Answer

The response returned to a UI or MCP caller.

Fields should include:

- answer text
- answer status: answered, needs clarification, insufficient evidence, or error
- cited evidence
- clarification question when needed
- retrieval trace summary
- confidence posture

## Ingestion Design

The ingestion pipeline should scan the configured 360-family folders under `~/Projects/workshop_manuals`.

Initial source set:

- `Ferrari 360 Modena/ferrari_360_wsm_english.pdf`
- `Ferrari 360 Challenge Stradale/206990-Ferrari_360_Challenge_Stradale_1999-2005.pdf`
- `Ferrari 360 Challenge/1_Chall_mo_GB.pdf`
- `Ferrari 360 Challenge/0_REGTECN_Chall_I-GB.pdf`
- `Ferrari 360 Challenge/360chall_DAS_GB.pdf`
- `Ferrari 360 Challenge/index.pdf`
- `Ferrari 360 Challenge/RevCambio_GB.pdf`
- `Ferrari 360 Challenge/cat_ric.pdf`

The first ingestion pass should:

1. Register each PDF as a `Source`.
2. Extract page text into `SourceUnit` records.
3. Render or cache page images for citation viewing.
4. Store PDF page locators for direct open-by-page behavior.
5. Infer coarse `SourceStructure` from PDF outlines, visible headings, page labels, and section markers.
6. Record extraction quality metrics such as empty pages, OCR confidence if available, and render failures.
7. Build a searchable index over source title, unit text, headings, aliases, and applicability metadata.

The pipeline should be resumable and fingerprint-based. Re-running ingestion should skip unchanged sources and replace derived units only when the source fingerprint or extraction settings change.

V1 should not bypass the current retrieval stack. It should adapt PDF pages and inferred structures into the existing indexing path, then evolve that path where page-level evidence requires richer fields than today's `InMemoryDocument` and chunk-only locator model.

### Section Governance

Section governance is the headline ingestion problem. Torque tables often appear at the beginning of a section while the relevant bolt appears later, so page-only retrieval can return a number without its governed object or procedure.

V1 section inference should use this order:

1. PDF outline/bookmark entries when present.
2. Page labels, visible headings, and section-number patterns.
3. Heading-range heuristics that assign each page to the nearest active section until the next peer heading.
4. LLM-assisted section repair only for high-value failures discovered during dogfood use.

Each `SourceStructure` should record its inference method and confidence. Low-confidence structure can still support candidate-section display, but the answer synthesizer should treat it as weaker support than explicit outline or heading evidence.

Known v1 limitation: some manuals will have imperfect section ranges. The conservative answer policy is the mitigation: when the system cannot establish that a table and procedure belong to the same section, it should show candidate pages or ask for clarification instead of asserting the torque value.

## Retrieval And Answering Flow

The retrieval design should be agent-with-primitives, not a fixed RAG pipeline. V1 can execute a conservative default strategy, but the application boundary should expose retrieval primitives that can later be selected iteratively.

Required primitives:

- `resolve_scope`: map the question and optional supplied scope to a `KnowledgeScope`.
- `expand_aliases`: derive deterministic and LLM-assisted aliases for vehicle, system, part, procedure, and manual vocabulary.
- `search_units`: search scoped `SourceUnit` records with lexical search and query expansion.
- `walk_structure`: move from a hit page to its enclosing section, procedure, table span, figure span, or neighboring pages.
- `read_evidence_span`: read a selected page or section span with the LLM.
- `enforce_citations`: verify that each material claim is backed by a cited `EvidencePassage`.
- `ask_clarification`: return a structured clarification when scope or support is ambiguous.
- `record_trace`: persist the primitive calls, selected evidence, and answer status for eval promotion.

The default v1 strategy should:

1. Resolve scope against the seeded 360 Modena profile.
2. Run alias expansion for the question and scope.
3. Search within the resolved source set.
4. Walk from promising hits to enclosing structures and neighboring pages.
5. Read selected evidence spans.
6. Synthesize only from selected evidence.
7. Enforce citation support before returning an answer.
8. Record the trace and feedback hooks.

This gives the first usable version a predictable path while preserving the thesis that retrieval is a set of composable primitives rather than a one-shot top-k answer generator.

The v1 answer policy is conservative:

- If no cited evidence supports the answer, return `insufficient evidence`.
- If multiple 360 variants could apply and the evidence does not resolve the user's intended car, return `needs clarification`.
- If candidate evidence exists but does not answer the question directly, present the candidate sections as evidence to inspect rather than producing a definitive answer.
- For torque specs, capacities, safety-critical procedures, and assembly instructions, include citations for each material claim.

## Web UI

The web UI should be a basic Next.js interface that favors shop usability over product polish.

Required v1 views:

- Chat panel with the active vehicle scope visible.
- Answer view with cited passages.
- Citation list with manual title, page number, section heading when known, and support reason.
- Page viewer that opens the rendered page image served by the app.
- Secondary source metadata that records the original PDF path and page number.
- Clarification state when the backend needs the user to choose vehicle, variant, or manual.
- Lightweight feedback controls for "helpful", "wrong", "wrong manual", and "missing citation".

The UI does not need vehicle management, account management, notification flows, or an artifact library in v1.

The UI is not a durable automotive product surface. It is a dogfood probe over the same API contracts that the MCP server and future business UI should use.

## MCP Interface

The MCP server should be a thin adapter over the same application service used by the web chat. It must not implement independent retrieval logic.

Required v1 tools:

- `ask_knowledge_question`: ask a question against the active or supplied scope and return the same answer contract as the web UI.
- `resolve_knowledge_scope`: inspect how the system resolved vehicle/manual scope for a question.
- `get_citation_page`: return locator metadata for a cited page or source unit so Hermes can show or summarize the evidence.
- `record_answer_feedback`: submit correction or usefulness feedback for later eval conversion.

The MCP contract should preserve answer status and citations exactly. A caller should be able to distinguish answered, needs-clarification, and insufficient-evidence responses without parsing prose.

## Provider Boundary

Cloud processing is allowed in v1, including cloud LLM calls over extracted manual text and page images when needed.

The design should still isolate providers behind ports:

- `TextExtractionProvider`
- `PageRenderProvider`
- `OcrProvider`
- `LlmProvider`
- `EmbeddingProvider` if vectors are added

Application services should depend on these ports, not on OpenAI, local model runtimes, OCR libraries, or vendor-specific request types directly. This keeps local model substitution possible later.

Provider calls should record:

- provider name
- model or engine id
- input unit ids
- output fingerprint
- processing timestamp
- extraction or answer quality metadata when available
- source permission policy that authorized the call

## Persistence

V1 may use simple local persistence as long as the boundaries match the future KE architecture.

Target production stores:

- Postgres for source, unit, structure, locator, answer, and feedback metadata.
- OpenSearch for lexical retrieval over source units and structures.
- Local filesystem cache for rendered page images and derived extraction artifacts.

Dogfood v1 may start with file-backed manifests plus OpenSearch if that is the shortest path inside `prescient_benchmark`. If so, repository ports should still be shaped so a later Postgres adapter can replace the file manifest without changing retrieval, UI, MCP, or eval contracts.

The implementation should not require storing original PDFs inside the application repo. Source paths can reference `~/Projects/workshop_manuals`, and derived artifacts should live under an ignored local data/cache directory.

## Evaluation Capture

The dogfood path should produce evaluation material without blocking daily use.

For each question, store:

- question text
- resolved scope
- selected sources and units
- final answer status
- cited evidence ids
- retrieval trace summary
- user feedback
- corrected answer or expected source when the user provides one

The first eval suite should be promoted from real usage. Candidate categories:

- direct torque lookup
- procedure removal or installation
- figure-dependent answer
- section-level context where a table governs later steps
- ambiguous variant question
- honest gap where the manual does not contain the requested fact

Promotion workflow:

1. Capture every dogfood interaction as a candidate record.
2. Mark candidates that have user feedback, corrections, low confidence, or missing citations.
3. Triage marked candidates into eval questions with expected source units and required claims.
4. Write promoted cases into the existing private question set and evidence-key format.
5. Score promoted cases with the existing retrieval scorer.

The first validation milestone should inherit the idea doc's thresholds:

- 40-60 promoted questions across torque lookup, procedure, figure, structure, ambiguity, vocabulary, and honest-gap cases.
- Scope extraction identifies or clarifies vehicle/variant in at least 90% of scoped questions.
- Citation coverage is at least 95% for numeric or procedural claims.
- Honest-gap questions return insufficient evidence at least 80% of the time.
- Agent-with-primitives retrieval beats the classical RAG baseline on retrieval-support score.

These thresholds should not block the first usable dogfood UI. They define when the dogfood slice has enough evidence to validate or revise the retrieval thesis.

## Error Handling

Ingestion errors should be source-local. A failed PDF, page render, or OCR pass should not block unrelated manuals.

Query errors should preserve user trust:

- Missing index: report that the manual corpus has not been ingested.
- Failed citation resolution: answer should include evidence ids and mark page viewing unavailable.
- LLM/provider failure: return an error status and keep retrieved candidate evidence when available.
- Ambiguous scope: ask a clarification question rather than guessing.
- Low support: return candidate sections or insufficient evidence rather than a definitive answer.

## Testing Strategy

Tests should focus on domain and application behavior before UI polish.

Required v1 coverage:

- Source and source-unit model validation.
- Fingerprint-based ingestion idempotency.
- PDF page locator generation.
- Scope resolution for seeded 360 Modena aliases.
- Clarification behavior for adjacent 360 variants.
- Conservative answer status when evidence is missing.
- Citation serialization shared by API and MCP responses.
- Feedback capture for future eval conversion.
- Mocked `LlmProvider` behavior for supported, unsupported, ambiguous, and provider-failure answers.
- Retrieval trace capture without relying on live model calls.

Integration tests should cover a tiny fixture PDF or synthetic page set rather than the copyrighted workshop manuals.

## Contract Stability

The first stable contracts should use generic names from day one:

- `AskKnowledgeQuestionRequest`
- `KnowledgeAnswer`
- `EvidenceCitation`
- `KnowledgeScopeResolution`
- `CitationPageLocator`
- `AnswerFeedback`

Vehicle/manual details should appear inside scope and source metadata, not in top-level contract names. This avoids a later rename from manual-specific API names to KE-wide API names.

## Follow-On Work

After the first usable slice:

1. Add a formal 360-family question set and evidence key from real shop questions.
2. Add better structure extraction for tables, figures, and procedures.
3. Add forum or web-source ingestion using the same `Source` and `SourceUnit` model.
4. Add vehicle profile CRUD only after the seeded profile becomes limiting.
5. Compare the interaction-first agent path against a classical RAG baseline.
6. Add a Postgres adapter for source/evidence metadata if dogfood v1 starts with file-backed manifests.

## Implementation Plan Requirements

The implementation plan derived from this spec must clearly explain the "why" behind sequencing and design decisions, not just list tasks.

At minimum, the plan should explain:

- Why the first slice starts with daily dogfood interaction instead of a complete eval suite.
- Why web UI and MCP are thin adapters over one backend service.
- Why the implementation extends `prescient_benchmark` before introducing new production infrastructure.
- Why rendered page images are the primary citation-viewing path.
- Why section governance starts with outlines and heading-range heuristics.
- Why generic source/evidence contracts are used even though the first corpus is automotive.
- Why Postgres can be deferred only if the port boundary remains compatible with the target KE architecture.
- Why each beads issue exists and how it links back to this spec and the implementation plan.

## Success Criteria

V1 is successful when:

- The user can ask real 360 shop questions through the web UI.
- Hermes can ask the same questions through MCP.
- Answers either cite exact manual pages or refuse to answer definitively.
- Citation links open the cited rendered page image.
- The system asks for clarification instead of mixing 360 Modena, Challenge Stradale, and Challenge evidence.
- Real shop interactions create reusable eval candidates.
- The implementation remains provider-swappable and does not hard-code car-only retrieval logic into the KE spine.

The follow-on validation milestone is successful when the promoted question set meets the quantitative thresholds listed in the Evaluation Capture section.
