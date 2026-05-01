# 2026-05-01 Workshop Terminology Workbench Design

## Goal

Add a human-in-the-loop terminology workbench to the workshop manual dogfood UI so the user can diagnose and improve retrieval failures while asking real repair questions.

The immediate problem is visible in the lower-control-arm torque query:

> What are the torque specs for the nuts holding the lower control arms to the chassis?

The indexed Ferrari 360 Modena workshop manual contains the answer on pages 250 and 252, but the current lexical retrieval path does not rank those pages high enough for the answer step. The manual uses terms such as `lower arm`, `suspension arm`, and `leva inferiore`; the user's natural vocabulary includes `lower control arm`, `LCA`, and `wishbone`.

The workbench should let the user try alternate terminology directly in the UI, compare the resulting candidate pages, answer from the better candidate set, and eventually save useful mappings with human approval.

## Why This Matters

Workshop manuals are a useful stress test for the KE-first retrieval thesis because they combine domain-specific vocabulary, OCR noise, multilingual terms, and safety-critical numeric claims. A generic top-k search path can return pages that mention the operation but omit the governing torque table.

The system should not silently guess. It should expose enough retrieval state that a human can see why an answer failed, try alternate terms, and turn successful terminology experiments into durable, scoped retrieval knowledge.

This is not an automotive-only feature. The same pattern applies later to business knowledge:

- `ARR` and `annual recurring revenue`
- `logo retention` and `customer retention`
- internal project codenames and formal initiative names
- legal abbreviations and clause names

The automotive UI is the first dogfood surface because it gives rapid feedback while the user is working on the car.

## Design Principles

1. Terminology learning is human-reviewed. The system may suggest mappings, but approved mappings must be explicit.
2. Temporary experimentation comes before persistence. The user should be able to try terms without changing global retrieval behavior.
3. Mapping scope must be narrow by default. A term such as `wishbone` can mean different things across domains; the safe default is the current manual scope.
4. Terminology should be reusable across vehicle repair use cases when it is safe, but source-specific quirks should stay source-specific.
5. The normal answer path and the workbench must share retrieval primitives. The workbench should not become a separate search system.
6. Retrieval diagnostics should be visible enough to explain why a candidate moved up or down.

## Chosen Approach

Build the feature in two layers:

1. A retrieval-preview API that accepts a question plus optional extra terms and returns candidate pages with diagnostics. It does not call the answer provider.
2. A UI workbench panel that lets the user edit term chips, rerun retrieval, compare runs, and answer from a selected candidate set.

Durable terminology mappings are a later step on the same surface. The first usable version should optimize for fast manual experimentation rather than dictionary administration.

The retrieval behavior should be organized around a `vehicle_repair_v1` retrieval profile, not a `workshop_manuals_v1` source-type profile. The profile represents how vehicle repair knowledge should be searched and reranked across workshop manuals, parts catalogs, service bulletins, forum posts, personal notes, and future repair sources.

## Decision Rationale

### Human-in-the-loop terminology workbench

The workbench exists because retrieval failures are often visible to a domain user before they are visible to the system. In the lower-control-arm example, the user can recognize that `LCA`, `wishbone`, `suspension arm`, and `lower arm` may refer to the same component. Capturing that judgment in the moment of failure is more reliable than trying to infer every synonym automatically after the fact.

The decision also keeps terminology learning reviewable. A successful experiment can become durable knowledge only after a human sees the candidate pages it recovered and chooses the scope where that mapping is safe.

### Retrieval preview before answer generation

The preview endpoint is separate from answer generation because the user needs to debug retrieval independently of LLM synthesis. If the wrong pages are retrieved, changing the answer prompt or increasing model effort does not fix the root issue.

Preview also reduces cost and noise while experimenting. The user can try terms, inspect candidate pages, and compare rank changes without repeatedly invoking the answer provider.

### Shared retrieval primitives

The workbench must use the same store provider, scope resolver, OpenSearch index, locator validation, and candidate identifiers as `/knowledge/ask`. This prevents a diagnostic-only path that works in the UI but does not improve real answers.

The UI may select or constrain candidates, but the answer service remains responsible for support checking and citations. That preserves the KE-first contract that answers are grounded in cited evidence.

### RetrievalProfile instead of source type

`vehicle_repair_v1` is a retrieval profile rather than a workshop-manual profile because the retrieval behavior belongs to the knowledge task, not the container format. Vehicle repair questions may need manuals, service bulletins, parts diagrams, owner notes, forum posts, or vendor instructions. A source-type profile would make useful terminology and reranking rules artificially stop at PDFs.

This also keeps the design generalizable. Later business use cases can define profiles for financial metrics, contract review, or operational procedures without changing the generic source model.

### Layered terminology applicability

Terminology is layered because some mappings are broadly true and others are only safe in a narrow context. `LCA` and `lower control arm` are broadly useful vehicle-repair terms; Ferrari bilingual terms may be brand-family behavior; a specific OCR quirk or table convention belongs to one source.

The default save scope is conservative because over-broad mappings create invisible retrieval pollution. Promotion to broader layers should be deliberate and backed by observed evidence across that layer.

### Deterministic torque reranking first

Torque queries need transparent behavior because the answer is a safety-critical numeric claim. A deterministic reranker can expose why a page moved up: it contained `Nm`, a torque-table heading, component terms, and attachment terms. That is easier to inspect, test, and regress than an opaque learned reranker in the first dogfood slice.

This does not rule out model-assisted expansion or reranking later. It sets a reliable baseline that can be evaluated before adding less deterministic behavior.

### Local preview history before persistence

Preview runs start local to the UI because most exploratory term trials are disposable. Persisting every trial would add storage, governance, and cleanup work before the user has proven which interactions are useful.

Server persistence begins at meaningful commitment points: marking a run useful, answering from selected pages, or saving a mapping. Those events are strong signals for eval promotion and future terminology review.

### Eval promotion as part of the loop

Terminology work should feed evals because otherwise fixes remain anecdotal. The lower-control-arm case should become a regression that proves the profile and mappings recover pages 250 or 252 and support the 55 Nm and 60 Nm claims.

The inverse path matters as much as promotion. When an eval fails, opening it in the same workbench gives the user the original question, expected evidence, and observed candidates in the tool they already use to diagnose retrieval.

## Retrieval Profiles And Terminology Layers

`KnowledgeScope` answers which sources are available for a question. `RetrievalProfile` answers how that scope should be searched.

For the current Ferrari 360 dogfood scope, retrieval should compose these layers:

1. `vehicle_repair_v1` - shared vehicle repair behavior and terminology.
2. `ferrari_v1` - Ferrari-specific vocabulary and multilingual repair terms.
3. `ferrari_360_v1` - 360-specific terminology, applicability, and source structure.
4. source-specific mappings - manual OCR quirks, page/table conventions, or forum/source vocabulary.
5. temporary UI terms - unsaved terms applied only to the current preview run.

Examples:

| Layer | Example Mapping | Why |
| --- | --- | --- |
| `vehicle_repair_v1` | `LCA`, `wishbone`, `lower control arm`, `suspension arm` | Broad repair vocabulary useful across many vehicles. |
| `ferrari_v1` | `telaio` = `chassis` or `frame` | Ferrari manuals mix Italian and English terminology. |
| `ferrari_360_v1` | `Challenge Stradale` = `CS` | 360-family variant terminology. |
| source-specific | `p250` contains the F 1.02 torque table | Specific to one manual and should not generalize. |

The default save location for a new mapping should be conservative: `ferrari_360_v1` or source-specific. The UI may allow promotion to `vehicle_repair_v1` or `ferrari_v1` after the user decides the mapping is broadly safe.

Approved mappings should auto-apply within their applicability layer. Auto-application is acceptable only if preview and answer responses expose mapping provenance so the user can see which mappings affected retrieval.

Example provenance:

```json
{
  "applied_mappings": [
    {
      "alias": "LCA",
      "canonical_term": "lower control arm",
      "layer": "vehicle_repair_v1"
    },
    {
      "alias": "telaio",
      "canonical_term": "chassis",
      "layer": "ferrari_v1"
    }
  ]
}
```

## Rejected Alternatives

### Separate Terminology Admin

A standalone dictionary manager would be clean for long-term governance, but it is too disconnected from the moment of failure. The user needs to see a bad answer, try `LCA`, `wishbone`, and `suspension arm`, and immediately observe candidate-page changes.

### Fully Automatic LLM Expansion

LLM-generated synonyms can help later, but automatic mappings are risky. Bad expansions can pollute retrieval invisibly and make eval failures harder to interpret. LLM suggestions should remain proposals until a human accepts them.

### Increase Top-K Only

Sending more pages to the answer provider would sometimes recover the right table, but it does not solve the underlying vocabulary gap and increases noise. The lower-control-arm case needs page 250 to rank for the right reason, not merely be buried in a larger context window.

## User Workflow

### 1. Ask Normally

The user asks a repair question in the existing workshop chat. The answer may be supported, insufficient, ambiguous, or incorrect.

### 2. Open Try Terminology

When the user wants to debug retrieval, they open a `Try terminology` panel from the answer area or candidate evidence area.

The panel shows:

- the original question
- detected terms from simple deterministic extraction
- manually editable term chips
- current candidate pages
- retrieval diagnostics

For the lower-control-arm query, the panel should support terms such as:

- `lower control arm`
- `LCA`
- `wishbone`
- `suspension arm`
- `lower arm`
- `leva inferiore`
- `chassis`
- `frame`
- `telaio`
- `tightening torque`
- `coppie di serraggio`
- `Nm`

### 3. Compare Runs

Each preview run is recorded locally in the panel:

- run label
- original question
- extra terms
- candidate page list
- matched terms
- rank changes when available

Example:

| Run | Extra terms | Top pages | Outcome |
| --- | --- | --- | --- |
| Original | none | p253, p255, p268 | procedure pages, no torque table |
| Expanded A | `lower arm`, `suspension arm`, `Nm` | p250, p252, p266 | torque table found |
| Expanded B | `LCA`, `wishbone` | p250, p252, p253 | useful alias coverage |

### 4. Answer From Selected Run

The user can choose a preview run and ask the existing answer path to answer from those candidate unit ids. This preserves the current evidence-answer contract while giving the user control over the candidate set.

The answer should still cite only the pages it uses. If the selected candidates are weak, the answer provider should still return insufficient evidence.

### 5. Save Mapping Later

After a successful term trial, the UI can offer `Save terminology mapping`.

The mapping should be saved as proposed or approved terminology with provenance:

- canonical term
- aliases
- applicability layer
- scope id when the mapping is scope-specific
- source ids
- optional intent tags
- evidence unit ids that motivated the mapping
- status
- created by
- created at

Example:

```text
canonical_term: suspension arm
aliases: lower control arm, LCA, wishbone, lower arm, leva inferiore
applicability_layer: ferrari_360_v1
scope_id: scope-ferrari-360-modena
source_ids: source-ferrari-360-wsm
intent_tags: torque_spec, procedure
evidence_unit_ids: unit-source-ferrari-360-wsm-p250, unit-source-ferrari-360-wsm-p252
status: approved
```

The first implementation may defer persistence, but the UI and API should be shaped so persistence can be added without redesigning the workflow.

The save UI should require an applicability choice:

```text
Save mapping scope:
[ ] Vehicle repair generally
[ ] Ferrari vehicles
[x] Ferrari 360 only
[ ] This manual/source only
```

The default should be `Ferrari 360 only` or `This manual/source only`, not global vehicle repair.

## API Design

### Retrieval Preview

Add a non-answering endpoint:

```text
POST /knowledge/retrieval-preview
```

Request:

```json
{
  "question": "What are the torque specs for the nuts holding the lower control arms to the chassis?",
  "scope_id": "scope-ferrari-360-modena",
  "extra_terms": ["lower arm", "suspension arm", "Nm"],
  "intent": "torque_spec"
}
```

Response:

```json
{
  "question": "What are the torque specs for the nuts holding the lower control arms to the chassis?",
  "expanded_query": "What are the torque specs for the nuts holding the lower control arms to the chassis? lower arm suspension arm Nm",
  "scope": {
    "scope_id": "scope-ferrari-360-modena",
    "scope_type": "vehicle",
    "display_name": "Ferrari 360 Modena"
  },
  "candidates": [
    {
      "unit_id": "unit-source-ferrari-360-wsm-p250",
      "source_id": "source-ferrari-360-wsm",
      "title": "Ferrari 360 Modena Workshop Manual",
      "page_number": 250,
      "heading": "F 1.02",
      "excerpt": "Nut fastening lower front arm to chassis 55 Nm. Nut fastening lower rear arm to the chassis 60 Nm.",
      "raw_score": 25.44,
      "rerank_score": 42.0,
      "matched_terms": ["lower arm", "chassis", "Nm", "TIGHTENING TORQUES"],
      "evidence_flags": ["torque_table", "numeric_torque"]
    }
  ],
  "applied_mappings": [
    {
      "alias": "lower control arm",
      "canonical_term": "suspension arm",
      "layer": "vehicle_repair_v1"
    }
  ],
  "diagnostics": {
    "raw_candidate_count": 30,
    "returned_candidate_count": 8,
    "stale_hit_count": 0,
    "missing_locator_count": 0
  }
}
```

The endpoint should use the same store, scope resolver, OpenSearch index, and locator validation as `/knowledge/ask`.

`intent` should be a closed enum for the active retrieval profile. For `vehicle_repair_v1`, v1 intents are:

- `torque_spec`
- `procedure`
- `spec_table`

Future domains can define their own profile-specific intents, such as `metric_value`, `definition`, or `clause_lookup`, without changing the preview endpoint shape.

### Answer From Preview

The existing answer route already accepts `candidate_unit_ids`. The UI should use that path for `Answer from these pages`.

This avoids introducing a parallel answer contract. The workbench selects candidates; the answer service determines whether they support an answer.

The web UI should reuse the existing streaming `/knowledge/ask` route for this action. The non-streaming `/knowledge/ask/sync` route remains for MCP, tests, and eval harnesses.

## Retrieval Design

The first workbench version should use a deterministic retrieval strategy:

1. Run the original question.
2. Compose approved mappings from `vehicle_repair_v1`, make/model overlays, source-specific mappings, and any temporary UI terms.
3. Run an expanded query composed of the original question plus composed terms.
4. Merge and deduplicate a larger candidate pool.
5. Apply transparent reranking heuristics.
6. Return the top preview candidates with diagnostic metadata.

Rerank rules should live in retrieval-profile configuration, not hardcoded in generic retrieval code. For `vehicle_repair_v1` torque-intent queries, reranking should boost candidates that contain:

- `TIGHTENING TORQUES`
- `COPPIE DI SERRAGGIO`
- `Nm`
- component terms from the question or extra terms
- attachment terms such as `chassis`, `frame`, or `telaio`

It should penalize pages that only contain generic phrases such as `prescribed torque` without a nearby numeric torque value.

The lower-control-arm query is the first regression case. With extra terms `lower arm`, `suspension arm`, and `Nm`, pages 250 or 252 should appear in the top preview candidates.

Phase 1 multilingual handling is explicitly limited to English plus Italian terms observed in the Ferrari 360 manuals. Later profiles should carry their own language metadata, analyzers, and terminology layers instead of assuming every corpus is bilingual in the same way.

## UI Design

Add a workbench panel to the existing app UI, not a separate admin page.

Recommended placement:

- collapsed `Try terminology` control near insufficient-evidence answers and candidate-only evidence
- `Try terminology` affordance from the `Wrong` feedback path for answered-but-wrong cases
- expandable panel in the existing chat pane or right-side evidence area
- term chips with add/remove controls
- `Preview retrieval` action
- run history list
- candidate page list with page number, heading, excerpt, matched terms, and evidence flags
- `Answer from these pages` action on a selected run

The UI should be utilitarian and dense. It should feel like a diagnostic tool for the garage, not a marketing surface.

Candidate page thumbnails should use `/knowledge/citation-page?rendition=thumb` for inline previews. Full-resolution page images remain available on demand.

The first version does not need mapping persistence. It should make the retrieval failure visible and let the user recover a better candidate set.

Preview run history is local to the UI in Phase 1. Server-side run persistence begins when the user marks a preview run useful, answers from a selected run, or saves a terminology mapping. This avoids storing every exploratory query while still preserving useful dogfood evidence for eval promotion.

## Data Model Direction

When persistence is added, introduce a terminology mapping model that is generic enough for business knowledge:

- id
- canonical term
- aliases
- applicability layer
- retrieval profile id
- scope id
- source ids
- source kind or domain tag
- intent tags
- evidence unit ids
- status: proposed, approved, rejected, disabled
- provenance: created by, created at, originating question, originating answer id

Mappings should be applied at query time, not baked into the source text. This keeps approved terminology inspectable, reversible, and scoped.

Approved mappings auto-apply only inside their applicability layer. The answer and preview contracts should return `applied_mappings` so users and eval records can explain why retrieval changed.

To keep query expansion bounded, each retrieval profile should define caps for automatically applied mappings. Phase 1 should cap applied mappings by intent and term match; disabled or rejected mappings must never be applied.

## Eval And Feedback

Successful terminology trials should be promotable into eval cases. For the lower-control-arm example:

- question: original user wording
- required evidence: pages 250 and/or 252
- required claims: lower front arm to chassis is 55 Nm; lower rear arm to chassis is 60 Nm
- failure mode: original query misses torque table
- expected improvement: expanded terms recover torque table in preview candidates

This turns dogfood failures into retrieval-quality tests and prevents terminology fixes from becoming untracked one-off patches.

The workflow is bidirectional. Failing eval cases should also open in the workbench with their original question, expected evidence, and observed candidates so retrieval failures can be investigated through the same UI used during dogfooding.

## Non-Goals

- Do not build a full terminology administration product in the first slice.
- Do not automatically approve LLM-suggested synonyms.
- Do not make terminology mappings global by default.
- Do not bypass citation enforcement or answer support checks.
- Do not introduce car-specific source entities into the generic KE spine.
- Do not treat source type as the retrieval-profile boundary. A vehicle repair profile may include workshop manuals, parts catalogs, service bulletins, forum posts, notes, and other source types.

## Testing Strategy

Backend tests:

- preview endpoint returns candidates without calling the answer provider
- extra terms are included in expanded retrieval
- lower-control-arm expanded query returns p250 or p252 ahead of generic procedure pages
- torque-intent reranking boosts numeric torque table pages
- preview respects scope filtering
- preview exposes matched terms and evidence flags
- preview exposes applied mapping provenance
- approved mappings auto-apply only within their configured applicability layer

Frontend tests:

- term chips can be added and removed
- preview runs are displayed with candidates
- selected run can call the existing answer route with candidate unit ids
- insufficient-evidence answers and wrong-answer feedback expose the workbench entry point
- inline candidate previews request thumb renditions

Manual verification:

- Ask the lower-control-arm question without extra terms and observe weak candidates.
- Add `lower arm`, `suspension arm`, and `Nm`.
- Confirm p250 or p252 appears in the preview results.
- Answer from the selected run and verify the answer cites 55 Nm and 60 Nm from the manual.

## Rollout

1. Implement retrieval preview behind the workshop API.

   Why first: retrieval must be inspectable before the UI or persistence can be evaluated. This slice proves the backend can return candidate pages, matched terms, evidence flags, diagnostics, and stable candidate ids without invoking the answer provider.

2. Add `vehicle_repair_v1` retrieval-profile configuration with deterministic torque reranking and layered terminology composition.

   Why second: the preview endpoint is only useful if it can exercise the retrieval behavior we actually want to improve. This slice keeps repair-specific rules out of generic retrieval code, makes the Ferrari 360 torque case testable, and establishes the reusable profile boundary for future vehicle sources.

3. Add the temporary term-trial UI.

   Why third: once preview and profile behavior exist, the user can interact with the system in the garage workflow: add terms, compare candidates, inspect page thumbnails, and answer from selected pages. Keeping the first UI temporary avoids turning the initial slice into a terminology administration product.

4. Add mapping persistence with applicability layers, auto-apply behavior, and applied-mapping provenance.

   Why fourth: persistence should follow observed useful trials. By this point the UI will show which terms recover evidence, so saved mappings can include scope, source, originating question, evidence units, status, and provenance instead of becoming ungrounded dictionary entries.

5. Add eval promotion and inverse eval investigation workflows after the workbench has been dogfooded.

   Why fifth: eval cases should be created from real successful and failed investigations. Waiting until the workbench has been used avoids premature eval schema choices while still making regression coverage a required part of the feature before it is considered complete.
