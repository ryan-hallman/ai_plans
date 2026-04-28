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

## Product Scope

V1 is an interaction-first manual assistant for the 360 family.

Included:

- Seed one active vehicle profile for the user's Ferrari 360 Modena.
- Ingest 360-family manuals from `~/Projects/workshop_manuals`.
- Include the base 360 Modena workshop manual and related 360 Challenge, Challenge Stradale, and gearbox manuals.
- Provide a basic web chat for repair questions.
- Provide an MCP server so Hermes can query the same backend when the user is away from the web UI.
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

## Retrieval And Answering Flow

The query path should be:

1. Accept the user question and optional scope.
2. Resolve scope against the seeded 360 Modena profile.
3. Detect whether the question names or implies an adjacent variant.
4. Ask a clarification question when variant or manual scope is ambiguous.
5. Search within the scoped source set using lexical search and query expansion.
6. Expand promising hits to their enclosing section or procedure when structure is available.
7. Read the selected section/page span with the LLM.
8. Synthesize an answer only from selected evidence.
9. Return cited evidence with locators that open exact source pages.
10. Record the query, selected evidence, answer status, and user correction markers for evaluation.

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
- Page viewer that opens the cited PDF page or rendered page image.
- Clarification state when the backend needs the user to choose vehicle, variant, or manual.
- Lightweight feedback controls for "helpful", "wrong", "wrong manual", and "missing citation".

The UI does not need vehicle management, account management, notification flows, or an artifact library in v1.

## MCP Interface

The MCP server should be a thin adapter over the same application service used by the web chat. It must not implement independent retrieval logic.

Required v1 tools:

- `ask_manual_question`: ask a question against the active or supplied scope and return the same answer contract as the web UI.
- `resolve_manual_scope`: inspect how the system resolved vehicle/manual scope for a question.
- `get_citation_page`: return locator metadata for a cited page or source unit so Hermes can show or summarize the evidence.
- `record_manual_feedback`: submit correction or usefulness feedback for later eval conversion.

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

## Persistence

V1 may use simple local persistence as long as the boundaries match the future KE architecture.

Expected stores:

- Postgres for source, unit, structure, locator, answer, and feedback metadata.
- OpenSearch for lexical retrieval over source units and structures.
- Local filesystem cache for rendered page images and derived extraction artifacts.

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

Integration tests should cover a tiny fixture PDF or synthetic page set rather than the copyrighted workshop manuals.

## Contract Stability

The first stable contracts should be:

- `AskManualQuestionRequest`
- `ManualAnswer`
- `ManualCitation`
- `ManualScopeResolution`
- `CitationPageLocator`
- `ManualFeedback`

These names can later become more generic once the implementation proves the shape. The internal model should remain generic enough that the contracts can evolve toward `AskKnowledgeQuestion`, `KnowledgeAnswer`, and `EvidenceCitation` without rewriting retrieval.

## Follow-On Work

After the first usable slice:

1. Add a formal 360-family question set and evidence key from real shop questions.
2. Add better structure extraction for tables, figures, and procedures.
3. Add forum or web-source ingestion using the same `Source` and `SourceUnit` model.
4. Add vehicle profile CRUD only after the seeded profile becomes limiting.
5. Compare the interaction-first agent path against a classical RAG baseline.
6. Generalize naming from manual-specific API contracts to broader KE contracts once the shape stabilizes.

## Success Criteria

V1 is successful when:

- The user can ask real 360 shop questions through the web UI.
- Hermes can ask the same questions through MCP.
- Answers either cite exact manual pages or refuse to answer definitively.
- Citation links open the cited page or rendered page image.
- The system asks for clarification instead of mixing 360 Modena, Challenge Stradale, and Challenge evidence.
- Real shop interactions create reusable eval candidates.
- The implementation remains provider-swappable and does not hard-code car-only retrieval logic into the KE spine.
