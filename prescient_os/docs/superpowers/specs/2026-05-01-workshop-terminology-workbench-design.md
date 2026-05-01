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
4. The normal answer path and the workbench must share retrieval primitives. The workbench should not become a separate search system.
5. Retrieval diagnostics should be visible enough to explain why a candidate moved up or down.

## Chosen Approach

Build the feature in two layers:

1. A retrieval-preview API that accepts a question plus optional extra terms and returns candidate pages with diagnostics. It does not call the answer provider.
2. A UI workbench panel that lets the user edit term chips, rerun retrieval, compare runs, and answer from a selected candidate set.

Durable terminology mappings are a later step on the same surface. The first usable version should optimize for fast manual experimentation rather than dictionary administration.

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
- detected or manually editable terms
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
- scope id
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
scope_id: scope-ferrari-360-modena
source_ids: source-ferrari-360-wsm
intent_tags: torque_spec, procedure
evidence_unit_ids: unit-source-ferrari-360-wsm-p250, unit-source-ferrari-360-wsm-p252
status: approved
```

The first implementation may defer persistence, but the UI and API should be shaped so persistence can be added without redesigning the workflow.

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
  "diagnostics": {
    "raw_candidate_count": 30,
    "returned_candidate_count": 8,
    "stale_hit_count": 0,
    "missing_locator_count": 0
  }
}
```

The endpoint should use the same store, scope resolver, OpenSearch index, and locator validation as `/knowledge/ask`.

### Answer From Preview

The existing answer route already accepts `candidate_unit_ids`. The UI should use that path for `Answer from these pages`.

This avoids introducing a parallel answer contract. The workbench selects candidates; the answer service determines whether they support an answer.

## Retrieval Design

The first workbench version should use a deterministic retrieval strategy:

1. Run the original question.
2. Run an expanded query composed of the original question plus extra terms.
3. Merge and deduplicate a larger candidate pool.
4. Apply transparent reranking heuristics.
5. Return the top preview candidates with diagnostic metadata.

For torque-intent queries, reranking should boost candidates that contain:

- `TIGHTENING TORQUES`
- `COPPIE DI SERRAGGIO`
- `Nm`
- component terms from the question or extra terms
- attachment terms such as `chassis`, `frame`, or `telaio`

It should penalize pages that only contain generic phrases such as `prescribed torque` without a nearby numeric torque value.

The lower-control-arm query is the first regression case. With extra terms `lower arm`, `suspension arm`, and `Nm`, pages 250 or 252 should appear in the top preview candidates.

## UI Design

Add a workbench panel to the existing app UI, not a separate admin page.

Recommended placement:

- collapsed `Try terminology` control near insufficient-evidence answers and candidate-only evidence
- expandable panel in the existing chat pane or right-side evidence area
- term chips with add/remove controls
- `Preview retrieval` action
- run history list
- candidate page list with page number, heading, excerpt, matched terms, and evidence flags
- `Answer from these pages` action on a selected run

The UI should be utilitarian and dense. It should feel like a diagnostic tool for the garage, not a marketing surface.

The first version does not need mapping persistence. It should make the retrieval failure visible and let the user recover a better candidate set.

## Data Model Direction

When persistence is added, introduce a terminology mapping model that is generic enough for business knowledge:

- id
- canonical term
- aliases
- scope id
- source ids
- source kind or domain tag
- intent tags
- evidence unit ids
- status: proposed, approved, rejected, disabled
- provenance: created by, created at, originating question, originating answer id

Mappings should be applied at query time, not baked into the source text. This keeps approved terminology inspectable, reversible, and scoped.

## Eval And Feedback

Successful terminology trials should be promotable into eval cases. For the lower-control-arm example:

- question: original user wording
- required evidence: pages 250 and/or 252
- required claims: lower front arm to chassis is 55 Nm; lower rear arm to chassis is 60 Nm
- failure mode: original query misses torque table
- expected improvement: expanded terms recover torque table in preview candidates

This turns dogfood failures into retrieval-quality tests and prevents terminology fixes from becoming untracked one-off patches.

## Non-Goals

- Do not build a full terminology administration product in the first slice.
- Do not automatically approve LLM-suggested synonyms.
- Do not make terminology mappings global by default.
- Do not bypass citation enforcement or answer support checks.
- Do not introduce car-specific source entities into the generic KE spine.

## Testing Strategy

Backend tests:

- preview endpoint returns candidates without calling the answer provider
- extra terms are included in expanded retrieval
- lower-control-arm expanded query returns p250 or p252 ahead of generic procedure pages
- torque-intent reranking boosts numeric torque table pages
- preview respects scope filtering
- preview exposes matched terms and evidence flags

Frontend tests:

- term chips can be added and removed
- preview runs are displayed with candidates
- selected run can call the existing answer route with candidate unit ids
- insufficient-evidence answers expose the workbench entry point

Manual verification:

- Ask the lower-control-arm question without extra terms and observe weak candidates.
- Add `lower arm`, `suspension arm`, and `Nm`.
- Confirm p250 or p252 appears in the preview results.
- Answer from the selected run and verify the answer cites 55 Nm and 60 Nm from the manual.

## Rollout

1. Implement retrieval preview and deterministic reranking behind the workshop API.
2. Add the temporary term-trial UI.
3. Use real garage questions to identify the smallest useful set of diagnostics.
4. Add persistence only after the temporary workflow proves useful.
5. Promote successful trials into eval cases before expanding the mapping system.
