# Electrical Ground Retrieval Learning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the "Where are the electrical grounds in the car?" miss a reusable retrieval-learning pattern instead of a one-off synonym patch.

**Architecture:** Add a generic `catalog_lookup` retrieval intent in code, then keep domain-specific electrical ground vocabulary in approved terminology data. Reuse the existing bounded candidate-context expansion path with catalog-specific settings, add strict catalog-answer guidance, and capture the miss in the canonical workshop eval set.

**Tech Stack:** FastAPI, Pydantic v2, existing workshop retrieval profile, layered terminology mappings, Postgres-backed terminology storage target, OpenSearch retrieval preview, pytest integration/unit tests, `eval/questions/workshop_manuals_v1.yaml`, Beads issue `prescient_os-2lp`.

---

## Problem

The Ferrari 360 WSM already contains the answer. Page 463 introduces `L 4 Wiring Diagrams` and says that section identifies all ground connections. Pages 465-489 list ground/earth/massa entries by harness/table identifier.

The system failed because retrieval did not understand that "electrical grounds" should expand to terms like `earth`, `massa`, `general earth`, `power ground`, `electronic ground`, and `wiring diagrams`. Default retrieval returned irrelevant doors/fuel/battery pages, so the answer path correctly refused to claim support. When the right pages were forced in, answer synthesis could summarize examples, but treated the broad car-wide question as ambiguous because the manual provides harness diagram IDs rather than one physical-location list.

## Accepted Decisions

1. **Add a generic code intent: `catalog_lookup`.**
   Do not add `electrical_catalog`, `fastener_catalog`, `connector_catalog`, or similar narrow code-level variants. Those names will multiply by domain and create brittle branching. Code should branch on a small set of retrieval shapes; the domain-specific target belongs in data.

   Example internal classification:

   ```json
   {
     "intent": "catalog_lookup",
     "domain": "vehicle_repair",
     "system": "electrical",
     "target": "ground_connections"
   }
   ```

   **Why:** `catalog_lookup` is reusable across vehicle manuals, business records, contracts, tickets, emails, forums, and future corpora. The code can learn one behavior: retrieve a set/list/catalog across a bounded source region.

2. **Store expansion vocabulary in terminology data, not retrieval code.**
   Terms like `earth`, `massa`, and `general earth` should be approved terminology mappings in the persistent terminology store. Code may contain cue terms for intent detection, but query expansion terms must come through the same mapping path that user-approved and agent-proposed mappings will use.

   Example mapping:

   ```json
   {
     "canonical_term": "ground connection",
     "aliases": ["electrical ground", "earth", "massa", "general earth", "power ground"],
     "retrieval_profile_id": "vehicle_repair_v1",
     "applicability_layers": ["domain:vehicle_repair:v1"],
     "status": "approved"
   }
   ```

   **Why:** This keeps the system general. New domains should add rows/migrations/seeds, not new Python conditionals. It also lets the human-in-the-loop workflow approve, reject, or scope terminology without redeploying retrieval logic.

3. **Reuse the existing candidate context expansion path.**
   `routes_knowledge.py` already expands adjacent units for procedure questions. Refactor that helper into a generic bounded expander with per-intent rules; do not create a second parallel implementation.

   **Why:** Two near-identical expansion mechanisms will drift. One tested primitive with different policies is easier to reason about and easier to refactor into a proper application service later.

4. **Keep `support_state` strict.**
   Do not globally loosen `supported`. Catalog answers can be supported only when the evidence directly supports the entries being listed and the answer explicitly states any limitation, such as "the manual lists harness/table IDs, not a single physical-location list."

   **Why:** The support-state contract is the guardrail against hallucinated answers. The fix should teach the provider how to handle catalog answers, not weaken what "supported" means.

5. **Make the eval YAML canonical.**
   Add the regression to `eval/questions/workshop_manuals_v1.yaml`. Integration tests are still useful for focused behavior, but the scorecard must carry the case.

   **Why:** Dogfood failures should improve the benchmark. If the case only lives in an integration test, answer-quality scorecards can regress without showing it.

6. **Verify corpus units before retrieval tests.**
   Check that the expected units/pages exist and contain `ground`, `earth`, or `massa` before asserting retrieval behavior.

   **Why:** This separates ingestion drift from retrieval regressions. A retrieval test should fail because retrieval missed available evidence, not because the local corpus did not ingest the expected pages.

7. **Cover both default and agent retrieval surfaces.**
   The v1 fix targets the default `auto` API path and the explicit `agent` path.

   **Why:** MCP and on-the-go usage may hit the agent path. A fix that works only in the web single-pass path is incomplete for the product direction.

## Design Principles

1. **Code owns shapes; data owns domain vocabulary.**
   `catalog_lookup` is a shape. `electrical ground`, `earth`, and `massa` are data.

2. **Separate failure learning from retrieval execution.**
   A bad answer should become a structured failure case with original query, retrieved pages, expected pages, failure type, and user correction. Retrieval then consumes approved mappings/evals, not arbitrary free text.

3. **Use source structure when the manual gives structure.**
   Page 463 is the guide page; pages 465-489 are the catalog entries. Retrieval should walk from guide pages into the relevant section rather than relying on page-level keyword matches alone.

4. **Answer with caveats instead of false insufficiency.**
   If the evidence gives harness IDs but not physical photos/locations, say that explicitly and cite the pages.

## Target Behavior

Question:

> Where are the electrical grounds in the car?

Expected behavior:

- Retrieval classifies the question as `catalog_lookup` with vehicle-repair/electrical/ground facets.
- Retrieval expands the query using approved terminology mappings, not hardcoded expansion lists.
- Retrieval finds the L4 wiring diagram guide and ground/earth pages without manual extra terms.
- The answer explains that the WSM organizes grounds by wiring harness/table identifiers.
- The answer lists the relevant ground references with page citations.
- The answer clarifies that this is not a clean physical-location list and asks for a subsystem if the user needs exact physical access guidance.
- The response does not cite unrelated fuel, door, or generic battery pages as primary support.

## Task 1: Add Generic Catalog Intent

**Why:** The current `VehicleRepairIntent` set is biased toward procedure/spec-table/torque questions. Electrical grounds are a catalog-style lookup across wiring diagrams. The code should learn the generic retrieval shape, not an electrical-specific intent.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/retrieval_profile.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/unit/test_workshop_retrieval_profile.py`

- [ ] Add `catalog_lookup` to `VehicleRepairIntent`.
- [ ] Add `catalog_lookup` to `VEHICLE_REPAIR_PROFILE.intents`.
- [ ] Update `_effective_vehicle_repair_intent(...)` so "Where are the electrical grounds in the car?" maps to `catalog_lookup`.
- [ ] Keep cue terms in code only for intent detection:
  - `electrical ground`
  - `electrical grounds`
  - `ground connection`
  - `ground connections`
  - `earth`
  - `massa`
  - `wiring diagram`
  - `wiring diagrams`
- [ ] Add a failing unit test:

```python
def test_electrical_ground_questions_use_catalog_lookup_intent() -> None:
    assert (
        _effective_vehicle_repair_intent(
            "Where are the electrical grounds in the car?",
            "procedure",
        )
        == "catalog_lookup"
    )
```

- [ ] Run:

```bash
uv run python -m pytest tests/unit/test_workshop_retrieval_profile.py::test_electrical_ground_questions_use_catalog_lookup_intent -q
```

Expected before implementation: the test fails because the query returns `procedure`.

Expected after implementation: the test passes.

## Task 2: Seed Electrical Ground Terminology as Persistent Data

**Why:** Query expansion terms must live in the DB-backed terminology layer, not in `built_in_vehicle_repair_query_terms`. Built-ins are acceptable for generic intent cues, but not for domain vocabulary that will vary by product, source, and user feedback.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/knowledge/terminology.py` only if the current model cannot represent the seed.
- Modify: the existing Postgres terminology repository/migration/seed path for approved mappings.
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/store.py` only as a compatibility adapter if the dogfood path still reads terminology through `KnowledgeStore`.
- Test: `tests/unit/test_terminology_mappings.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Add or update the DB seed/migration so the active terminology store contains an approved mapping:

```json
{
  "mapping_id": "mapping-vehicle-repair-ground-connection",
  "canonical_term": "ground connection",
  "aliases": [
    "electrical ground",
    "electrical grounds",
    "ground connections",
    "earth",
    "general earth",
    "electronic ground",
    "power ground",
    "massa",
    "wiring diagrams",
    "L 4 wiring diagrams"
  ],
  "retrieval_profile_id": "vehicle_repair_v1",
  "applicability_layers": ["domain:vehicle_repair:v1"],
  "applicability_layer_kinds": {"domain:vehicle_repair:v1": "domain"},
  "intent_tags": ["catalog_lookup"],
  "status": "approved",
  "created_by": "system_seed"
}
```

- [ ] Add a unit test proving `compose_terminology_for_query(...)` returns the canonical and alias expansion terms when the query contains `electrical grounds` and intent is `catalog_lookup`.
- [ ] Add an integration test proving the API route sees the seeded mapping via `store.list_terminology_mappings()`.
- [ ] Run:

```bash
uv run python -m pytest \
  tests/unit/test_terminology_mappings.py \
  tests/integration/test_workshop_api.py::test_electrical_ground_mapping_seed_is_available \
  -q
```

Expected: expansion terms come from approved terminology data. No electrical expansion list is added to `built_in_vehicle_repair_query_terms`.

## Task 3: Verify Corpus Evidence Before Retrieval Assertions

**Why:** The plan depends on ingested Ferrari 360 WSM pages. Tests should fail clearly if the corpus fixture does not contain the expected units.

**Files:**
- Test: `tests/integration/test_workshop_api.py`
- Modify if useful: `apps/api/src/prescient_benchmark/workshop_manuals/eval.py` for shared helpers only.

- [ ] Add a small corpus-evidence assertion helper in the test module:

```python
def _assert_units_contain_terms(store, expected: dict[str, list[str]]) -> None:
    units_by_id = {unit.unit_id: unit for unit in store.list_units()}
    for unit_id, terms in expected.items():
        assert unit_id in units_by_id
        text = units_by_id[unit_id].text.casefold()
        assert any(term.casefold() in text for term in terms)
```

- [ ] Use it before the electrical-ground retrieval assertions for:
  - `unit-source-ferrari-360-wsm-p463`: `ground connections`, `massa`, or `wiring diagrams`
  - `unit-source-ferrari-360-wsm-p466`: `powerground`, `ground`, or `massa`
  - `unit-source-ferrari-360-wsm-p488`: `general earth`, `massa`, or `ground`
  - `unit-source-ferrari-360-wsm-p489`: `general earth`, `massa`, or `ground`
- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_electrical_ground_corpus_units_exist -q
```

Expected: if the corpus fixture is correct, the helper passes. If it fails, fix ingestion/fixtures before retrieval logic.

## Task 4: Teach Query Expansion to Pull L4 Wiring Pages

**Why:** The right evidence lives under manual language and section structure. Once the query is classified as `catalog_lookup`, expansion must combine approved terminology mappings and reranking so L4 wiring pages outrank unrelated pages.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/retrieval_profile.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Add a failing integration test using a seeded store with:
  - an irrelevant fuel/door page,
  - page 463-style L4 wiring guide text,
  - page 466-style power ground text,
  - page 488-style general earth text,
  - the approved terminology mapping from Task 2.
- [ ] Assert the default `auto` retrieval path selects the L4/ground pages for the electrical grounds query.
- [ ] Assert `applied_mappings` includes the ground connection mapping.
- [ ] Do not add electrical expansion vocabulary to `built_in_vehicle_repair_query_terms`.
- [ ] Add rerank support for `catalog_lookup` that boosts:
  - matched approved mapping terms,
  - `wiring diagram` / `wiring diagrams`,
  - `ground connection`,
  - `earth`,
  - `massa`.
- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_route_retrieves_electrical_ground_pages -q
```

Expected before implementation: the test retrieves irrelevant pages or returns insufficient evidence.

Expected after implementation: the test retrieves L4/ground pages and reports the applied terminology mapping.

## Task 5: Refactor Existing Candidate Context Expansion

**Why:** `routes_knowledge.py` already has adjacent-ordinal expansion for procedure questions. Catalog lookup needs a wider bounded window, but duplicating the expansion algorithm would create two code paths that drift.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Rename `_expand_procedure_candidate_context(...)` to `_expand_candidate_context(...)`.
- [ ] Parameterize its behavior by intent:
  - `procedure`: existing ±1 window, max 10 units.
  - `catalog_lookup`: include nearby same-source units around high-ranked catalog hits, max 12 units.
  - all other intents: preserve current truncation behavior.
- [ ] Preserve ordering and deduplication.
- [ ] Add a focused test proving a guide page at ordinal 463 expands into nearby L4 pages but does not pull unrelated later bodywork pages.
- [ ] Update every call site to use `_expand_candidate_context(...)`.
- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_catalog_lookup_expands_nearby_wiring_diagram_pages -q
```

Expected: the shared expander handles both procedure and catalog lookup without a new module.

## Task 6: Cover Agent Mode

**Why:** MCP/on-the-go usage can hit the agent path. The fix should not only work in the web/default single-pass path.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Add an integration test that sends:

```json
{
  "question": "Where are the electrical grounds in the car?",
  "answer_mode": "agent"
}
```

- [ ] Assert retrieved/cited candidate units include L4 wiring diagram ground pages.
- [ ] Ensure `_search_agent_query(...)` composes terminology for the effective `catalog_lookup` intent.
- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_agent_mode_retrieves_electrical_ground_pages -q
```

Expected: agent-mode retrieval uses the same terminology and catalog context behavior as default retrieval.

## Task 7: Adjust Answer Policy for Broad Catalog Answers

**Why:** The correct answer is not "all physical ground locations"; it is "the manual gives ground references by harness/table ID." The answer should be useful with a caveat instead of falling into ambiguity, while preserving strict support semantics.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/providers.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Add catalog-specific provider guidance without weakening the global `supported|unsupported|ambiguous` contract:
  - Answer from catalog entries only when the entries are present in evidence.
  - State the manual's representation model when it matters.
  - Mark `supported` only if every listed entry is in cited evidence.
  - Mark `ambiguous` only when sources conflict or scope is genuinely unclear.
- [ ] Add a deterministic test provider for the electrical ground scenario.
- [ ] Assert the API response status is `answered`.
- [ ] Assert the answer text includes a caveat like:

```text
The manual lists these as wiring harness/table ground IDs, not as a single physical-location list.
```

- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_route_answers_electrical_ground_catalog -q
```

Expected: the answer cites L4 pages, lists supported ground references, and explains the harness/table-ID limitation.

## Task 8: Record the Miss in the Canonical Eval Set

**Why:** This was found through dogfooding. It should become a scorecard regression guard, not only an integration test.

**Files:**
- Modify: `eval/questions/workshop_manuals_v1.yaml`
- Modify: `tests/unit/test_workshop_eval.py`
- Test: `tests/unit/test_workshop_eval.py`

- [ ] Add this question to `eval/questions/workshop_manuals_v1.yaml`:

```yaml
  - id: wq_360_modena_electrical_ground_connections
    category: catalog
    prompt: Where are the electrical grounds in the car on a Ferrari 360 Modena?
    scope_id: scope-ferrari-360-modena
    required_unit_ids:
      - unit-source-ferrari-360-wsm-p463
      - unit-source-ferrari-360-wsm-p466
      - unit-source-ferrari-360-wsm-p488
      - unit-source-ferrari-360-wsm-p489
    required_source_ids: [source-ferrari-360-wsm]
    required_claims:
      - claim_id: electrical_ground_catalog
        statement: The L4 wiring diagrams identify the car's ground connections by harness/table reference.
        required_unit_ids:
          - unit-source-ferrari-360-wsm-p463
          - unit-source-ferrari-360-wsm-p466
          - unit-source-ferrari-360-wsm-p488
          - unit-source-ferrari-360-wsm-p489
```

- [ ] Update `tests/unit/test_workshop_eval.py` only if it needs explicit category expectations.
- [ ] Run:

```bash
uv run python -m pytest tests/unit/test_workshop_eval.py -q
```

Expected: the eval parser accepts the new catalog question and required evidence IDs.

## Task 9: Add Integration Regression Guard

**Why:** The eval YAML validates benchmark assets; integration tests validate the runtime API behavior.

**Files:**
- Modify: `tests/integration/test_workshop_api.py`

- [ ] Add an integration test for the full `/knowledge/ask/sync` path.
- [ ] Required evidence pages include:
  - p. 463 L4 wiring guide,
  - at least two of p. 466, p. 488, p. 489.
- [ ] Negative guard: primary citations must not be limited to door, fuel, or generic battery pages.
- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py -q
```

Expected: the new regression passes with the broader workshop API test suite.

## Task 10: Prepare Feedback-Learning Follow-Up

**Why:** The next clean step is to turn user corrections into proposed terminology/eval data. That is broader than this query fix, so it should be tracked separately.

**Files:**
- No files in this implementation.
- Beads only.

- [ ] Create or update a Beads issue for a feedback-learning loop:
  - wrong answer -> triage reason,
  - user correction -> proposed terminology mapping/eval seed,
  - approval -> active mapping/eval.
- [ ] Link it to `prescient_os-2lp` as a follow-up, not a blocker.
- [ ] Do not implement this loop in the electrical-ground fix unless the user explicitly expands scope.

## Verification

Run the focused checks first:

```bash
uv run python -m pytest tests/unit/test_workshop_retrieval_profile.py -q
uv run python -m pytest tests/unit/test_terminology_mappings.py -q
uv run python -m pytest tests/unit/test_workshop_eval.py -q
uv run python -m pytest tests/integration/test_workshop_api.py -q
```

Then run the workshop eval path if local services are available:

```bash
uv run python -m prescient_benchmark.cli run-workshop-eval-baseline \
  --question-set-path eval/questions/workshop_manuals_v1.yaml
```

Then smoke-check the local API:

```bash
curl -sS http://localhost:8000/knowledge/ask/sync \
  -H 'Content-Type: application/json' \
  -d '{"question":"Where are the electrical grounds in the car?"}' | jq .
```

Expected:

- response cites L4 wiring diagram pages,
- text explains harness/table ground IDs,
- no false claim of a single universal physical location list,
- no primary reliance on irrelevant fuel/door/generic battery pages,
- `answer_mode: "agent"` works for the same query.

## Non-Goals

- Do not build a full electrical-system ontology in this task.
- Do not extract every wiring diagram into canonical artifacts yet.
- Do not add frontend-specific terminology constants.
- Do not add narrow code intents like `electrical_catalog`.
- Do not add electrical expansion vocabulary to `built_in_vehicle_repair_query_terms`.
- Do not make user feedback automatically authoritative.
- Do not solve forum/email/non-manual source trust in this task.

## Execution Notes

- If the current dogfood path still uses JSON-backed `WorkshopManualStore` for terminology, keep the implementation behind the existing `KnowledgeStore` protocol and seed the active local store through the same repository interface. The durable design target remains DB-backed terminology data.
- Make small commits per task.
- Keep unrelated dirty files out of commits.
