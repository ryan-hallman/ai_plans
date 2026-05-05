# Electrical Ground Retrieval Learning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the "Where are the electrical grounds in the car?" miss a reusable retrieval-learning pattern instead of a one-off synonym patch.

**Architecture:** Add a small, profile-owned query understanding and terminology layer for electrical ground/catalog questions, then make retrieval section-aware enough to pull the L4 wiring diagram context. Capture the failure as an eval and future feedback-learning seed so similar misses in other domains become data, not code churn.

**Tech Stack:** FastAPI, Pydantic v2, existing workshop retrieval profile, OpenSearch retrieval preview, pytest integration/unit tests, Beads issue `prescient_os-2lp`.

---

## Problem

The Ferrari 360 WSM already contains the answer. Page 463 introduces `L 4 Wiring Diagrams` and says that section identifies all ground connections. Pages 465-489 list ground/earth/massa entries by harness/table identifier.

The system failed because retrieval did not understand that "electrical grounds" should expand to terms like `earth`, `massa`, `general earth`, `power ground`, `electronic ground`, and `wiring diagrams`. Default retrieval returned irrelevant doors/fuel/battery pages, so the answer path correctly refused to claim support. When the right pages were forced in, answer synthesis could summarize examples, but treated the broad car-wide question as ambiguous because the manual provides harness diagram IDs rather than one physical-location list.

## Design Principles

1. **Do not hardcode this as a Ferrari-only fix.**
   The fix belongs in retrieval profiles, terminology layers, and evals so future domains can add their own vocabulary without frontend or route changes.

2. **Separate failure learning from retrieval execution.**
   A bad answer should become a structured failure case with original query, retrieved pages, expected pages, failure type, and user correction. Retrieval then consumes approved mappings/evals, not arbitrary free text.

3. **Treat broad catalog questions as a distinct intent.**
   "Where are all the grounds?" is not a procedure, torque spec, or single spec-table lookup. It is a catalog/listing query over a section.

4. **Use source structure when the manual gives structure.**
   Page 463 is the guide page; pages 465-489 are the catalog entries. Retrieval should walk from guide pages into the relevant section rather than relying on page-level keyword matches alone.

5. **Answer with caveats instead of false insufficiency.**
   If the evidence gives harness IDs but not physical photos/locations, say that explicitly and cite the pages.

## Target Behavior

Question:

> Where are the electrical grounds in the car?

Expected behavior:

- Retrieval finds the L4 wiring diagram guide and ground/earth pages without manual extra terms.
- The answer explains that the WSM organizes grounds by wiring harness/table identifiers.
- The answer lists the relevant ground references with page citations.
- The answer clarifies that this is not a clean physical-location list and asks for a subsystem if the user needs exact physical access guidance.
- The response does not cite unrelated fuel, door, or generic battery pages as primary support.

## Task 1: Add Ground/Catalog Query Understanding

**Why:** The current `VehicleRepairIntent` set is biased toward procedure/spec-table/torque questions. Electrical grounds are a catalog-style lookup across wiring diagrams, so retrieval needs a stable intent hook.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/retrieval_profile.py`
- Test: `tests/unit/test_workshop_retrieval_profile.py`

- [ ] Add a vehicle-repair intent or intent helper for electrical catalog questions.
- [ ] Add a failing unit test for:
  - question: `Where are the electrical grounds in the car?`
  - expected intent/effective behavior: electrical catalog lookup.
- [ ] Include cue terms:
  - `electrical ground`
  - `grounds`
  - `ground connection`
  - `earth`
  - `massa`
  - `wiring diagram`
- [ ] Run:

```bash
uv run python -m pytest tests/unit/test_workshop_retrieval_profile.py -q
```

Expected before implementation: the new test fails because the query has no electrical catalog handling.

Expected after implementation: the new test passes.

## Task 2: Add Profile-Owned Electrical Terminology

**Why:** The terminology suggestion endpoint currently returns no electrical/ground terms. Hardcoding these in the frontend would recreate the same scalability problem.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/retrieval_profile.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/unit/test_workshop_retrieval_profile.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Add profile-owned suggestions for electrical ground vocabulary:
  - `electrical ground`
  - `ground connection`
  - `ground connections`
  - `earth`
  - `general earth`
  - `electronic ground`
  - `power ground`
  - `massa`
  - `wiring diagrams`
  - `L 4 wiring diagrams`
- [ ] Add a unit test confirming `vehicle_repair_terminology_suggestions()` includes the electrical vocabulary.
- [ ] Add or update an API integration test confirming `/knowledge/terminology-suggestions?scope_id=scope-ferrari-360-modena` returns at least one electrical ground term.
- [ ] Run:

```bash
uv run python -m pytest \
  tests/unit/test_workshop_retrieval_profile.py \
  tests/integration/test_workshop_api.py::test_terminology_suggestions_route_returns_scope_profile_terms \
  -q
```

Expected: terms come from backend profile/scope data, not frontend constants.

## Task 3: Teach Query Expansion to Pull L4 Wiring Pages

**Why:** The right evidence lives under manual language and section structure. Expansion terms should bring the L4 guide and ground catalog pages into the raw candidate set.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/retrieval_profile.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Add a failing integration test using a seeded store with:
  - an irrelevant fuel/door page,
  - page 463-style L4 wiring guide text,
  - page 466-style power ground text,
  - page 488-style general earth text.
- [ ] Assert the retrieval path selects the L4/ground pages for the electrical grounds query.
- [ ] Add built-in expansion terms for electrical catalog questions:
  - `wiring diagrams`
  - `ground connections`
  - `earth`
  - `massa`
  - `general earth`
  - `power ground`
  - `electronic ground`
- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_route_retrieves_electrical_ground_pages -q
```

Expected before implementation: the test retrieves irrelevant pages or returns insufficient evidence.

Expected after implementation: the test retrieves L4/ground pages.

## Task 4: Add Section-Aware Expansion for Wiring Diagram Catalogs

**Why:** The L4 guide page establishes the section semantics, while later pages carry the actual entries. A scalable solution needs a small structure-walk primitive, not just stronger keywords.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/context_expansion.py`
- Test: `tests/unit/test_workshop_context_expansion.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Add a context expansion helper that can take high-ranked guide/catalog pages and include adjacent pages in the same source section.
- [ ] Keep it bounded; for v1, cap expanded catalog context at 12 units.
- [ ] Add a unit test proving a guide page at ordinal 463 expands into nearby L4 pages but does not pull unrelated later bodywork pages.
- [ ] Route electrical catalog queries through this expansion helper after initial retrieval.
- [ ] Run:

```bash
uv run python -m pytest \
  tests/unit/test_workshop_context_expansion.py \
  tests/integration/test_workshop_api.py::test_ask_knowledge_route_retrieves_electrical_ground_pages \
  -q
```

Expected: section expansion includes enough L4 pages to answer without overloading the answer provider with unrelated sections.

## Task 5: Adjust Answer Policy for Broad Catalog Answers

**Why:** The correct answer is not "all physical ground locations"; it is "the manual gives ground references by harness/table ID." The answer should be useful with a caveat instead of falling into ambiguity.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/providers.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Add answer-provider guidance for catalog/listing questions:
  - Answer from the catalog entries when evidence supports them.
  - State the manual's representation model when it matters.
  - Use `supported` when the evidence provides a partial-but-useful catalog with clear limitations.
  - Use `ambiguous` only when sources conflict or scope is genuinely unclear.
- [ ] Add a deterministic test provider for the electrical ground scenario.
- [ ] Assert the API response status is `answered`.
- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py::test_ask_knowledge_route_answers_electrical_ground_catalog -q
```

Expected: the answer cites L4 pages and explains the harness/table-ID limitation.

## Task 6: Record the Miss as an Eval Case

**Why:** This was found through dogfooding. It should become a regression guard so future retrieval changes cannot silently degrade it.

**Files:**
- Modify: `tests/integration/test_workshop_api.py`
- Reference Beads issue: `prescient_os-2lp`

- [ ] Add the query:

```text
Where are the electrical grounds in the car?
```

- [ ] Required evidence pages include:
  - p. 463 L4 wiring guide,
  - at least two of p. 465, p. 466, p. 469, p. 475, p. 481, p. 482, p. 485, p. 488, p. 489.
- [ ] Negative guard: primary citations must not be limited to door, fuel, or generic battery pages.
- [ ] Run:

```bash
uv run python -m pytest tests/integration/test_workshop_api.py -q
```

Expected: the new regression passes with the broader workshop API test suite.

## Task 7: Prepare Feedback-Learning Follow-Up

**Why:** The next clean step is to turn user corrections into proposed terminology/eval data. That is broader than this query fix, so it should be tracked separately unless already covered by an active issue.

**Files:**
- Beads only unless an existing spec needs a link.

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
uv run python -m pytest tests/integration/test_workshop_api.py -q
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
- no primary reliance on irrelevant fuel/door/generic battery pages.

## Non-Goals

- Do not build a full electrical-system ontology in this task.
- Do not extract every wiring diagram into canonical artifacts yet.
- Do not add frontend-specific terminology constants.
- Do not make user feedback automatically authoritative.
- Do not solve forum/email/non-manual source trust in this task.

## Review Questions

1. Does `answered` with explicit caveats feel right for partial catalog answers, or should this plan be revised to keep a cautious non-answered status?
2. Should L4 wiring diagram pages become source-derived artifacts now, or should that wait for the broader artifact extraction pipeline?
3. Should electrical catalog intent be a new typed intent, or should it live as a helper over the existing `spec_table` intent for v1?
