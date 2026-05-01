# 2026-04-28 Workshop Manual Dogfood Spec — Critical Review

## Context

Requested as a hole-poking review of the workshop manual dogfood design spec before implementation begins.

- Spec under review: `docs/superpowers/specs/2026-04-28-workshop-manual-dogfood-design.md`
- Prior thinking: `docs/superpowers/ideas/2026-04-22-vehicle-manuals-validation-corpus.md` argued for using the manuals as a *validation corpus* against the existing private-retrieval scorer, with explicit guardrails against any product-shaped surface.
- Existing infrastructure: `apps/api/src/prescient_benchmark/{corpus,retrieval,eval}` already implements ingestion, indexing, search, mixed-source selection, scoring. Recent commits (`Harden private retrieval provenance and index reuse`, `feat: add mixed-source retrieval diagnostics`) show active investment.

The spec is a real and disciplined attempt to ship a usable shop slice. This review is about the gaps that will bite during implementation, not the overall direction.

## One-Sentence Risk

The spec implicitly reverses the 2026-04-22 idea doc's "no Ferrari-specific UI, validation only via the scorer" stance and describes a parallel ingestion/retrieval pipeline without acknowledging the existing `prescient_benchmark` one — both moves may be the right call, but neither is named or justified, so the spec ships choices it hasn't made on the page.

## Top Concerns

### 1. Direct conflict with the 2026-04-22 idea doc — needs reconciliation

The idea doc was emphatic:

- "No Ferrari-specific UI. Memory Inbox, Artifact Library, the copilot chat surface are business-shaped and stay business-shaped. Validation happens against the retrieval scorer, not via the product UI."
- "Two hats, separated. Product-user hat: 'does this make me faster on my cars?' Eval-engineer hat: 'does the same query shape work on 10-K filings?' Only the second hat drives product decisions."

This spec walks the opposite direction: a Next.js shop chat, an MCP server with `ask_manual_question` / `resolve_manual_scope` / `record_manual_feedback`, and a contract surface named `AskManualQuestion*`, `ManualAnswer`, `ManualCitation`. The disclaimer on line 7 ("not an automotive vertical") doesn't neutralize the artifacts.

Either:

- supersede the idea doc explicitly with a stated reason for the pivot (real-time shop usability now beats scorer-only validation), or
- pull the UI/MCP surface back to a thin probe over the scorer path.

Right now the spec disagrees with the prior thinking without flagging that it's doing so. Future readers will not be able to tell which document is current.

### 2. Doesn't acknowledge the existing `prescient_benchmark` pipeline

The repo already has retrieval modules (`index.py`, `search.py`, `chunk.py`, `parse.py`, `answer.py`, `rerank.py`, `mixed_source_selection.py`), corpus loaders, and an eval orchestrator with scoring. The idea doc was specific: "Reuse existing infrastructure; do not build parallel pipelines. Tag with `corpus=vehicle-manuals`."

The spec describes ingestion, retrieval, persistence, and contracts as if green-field — Postgres + OpenSearch + filesystem cache, none of which appear in the current code. It does not name a single existing module it extends.

This reads as a parallel pipeline. Either commit to extending `prescient_benchmark` (and say which modules), or argue why a fresh pipeline is warranted for a single-user dogfood slice.

### 3. Postgres + OpenSearch is a heavy v1 infrastructure bet

For ~8 PDFs and one user, OpenSearch is expensive both to stand up and to keep healthy alongside the existing in-process retrieval. If the existing `retrieval/index.py` works, adopt it. If it doesn't scale to PDFs, that is a separable design conversation, not a one-liner under "Persistence." Either way, the choice deserves justification proportional to its operational cost.

### 4. The retrieval flow is a fixed pipeline, not "agent + primitives"

Project thesis (memorized): *"agent + primitives over RAG-first; vectors are one tool, not the spine."* Lines 234–245 of the spec are procedural — scope → ambiguity check → search → expand → read → synthesize. There is no decision loop, no enumerated primitive surface, no point at which the agent chooses what to invoke next. That is RAG with a clarification gate, not agent-driven retrieval.

Either re-frame the section to expose primitives the agent calls (scope extraction, structural walk, long-context read, alias expansion, citation enforcement — all named in the idea doc), or admit v1 is structured RAG and revisit the thesis claim later. Both are defensible; the present text is neither.

## Concrete Design Gaps

### 5. Section governance is the headline corpus challenge — and is handwaved

Lines 13 and 252 acknowledge that "tables at the beginning govern later procedural steps." The retrieval flow says "expand to enclosing section." How? PDF outline parsing? Heading-text heuristics? Page-range inference? This is the failure mode the corpus was specifically chosen to expose. Either commit to an approach or list it as an explicit v1 known-limitation with a downstream entry. Right now it is the most important problem the spec doesn't address.

### 6. "Detect adjacent variant" is load-bearing but unspecified

Line 240 says "Detect whether the question names or implies an adjacent variant." Alias match against `VehicleProfile.aliases`? LLM classifier? Both have failure modes; both have very different cost profiles. The conservative-answer policy and the clarification gate both depend on this working.

### 7. `VehicleProfile` ↔ `KnowledgeScope` relationship is undefined

Subtype? Referenced from `KnowledgeScope.linked_entities`? Standalone table joined by id? This determines whether adding a second scope type later (project, account, legal matter) reuses the same code path or forks it. One paragraph would settle it.

### 8. Citation viewer doesn't pick a strategy

Lines 263–264 list both "open the cited PDF page" and "rendered page image." Browsers cannot reliably open `file://~/Projects/...` PDFs from an HTTP origin, so the likely real answer is: render PNG/WEBP per page at ingestion, serve from the app, link to the original PDF as a secondary affordance. The spec should commit.

### 9. `provider processing permissions` is named but undefined

Line 108 lists this on `Source` without semantics. For copyrighted manuals it matters operationally — does each source carry its own allow/deny list for which providers may process it? Required to honor the idea doc's "no external APIs with data retention" constraint and the spec's own "controlled environment" exclusion (line 42).

### 10. Feedback → eval is a one-way drop

`record_manual_feedback` is in MCP, "Evaluation Capture" lists what's stored, but the workflow that promotes a captured interaction into a graded eval case is missing. Triage step? Promotion criteria? Without it, "real shop interactions create reusable eval candidates" stays aspirational.

## Smaller Calls

### 11. Contract names start manual-specific then plan to migrate

Lines 372–381 plan to start at `AskManualQuestion*` and rename to `AskKnowledgeQuestion*` later. Migrations are costly and the internal model (`Source`, `SourceUnit`, `EvidencePassage`) is already generic. Naming the contracts generic from day one (`AskKnowledgeQuestion` with a `scope` field) avoids the rename entirely.

### 12. Success criteria are softer than the idea doc's

The idea doc had concrete thresholds: 40–60 question target, ≥90% scope extraction on scoped queries, ≥95% citation coverage, ≥80% honest-gap discipline, agent path beats classical RAG baseline. The spec's "real shop interactions create reusable eval candidates" is fuzzier. Inherit the idea doc's numbers rather than restating weaker ones.

### 13. Testing strategy doesn't address the LLM step

How do integration tests assert conservative-answer behavior without real LLM calls? Cassettes? Mocked `LlmProvider` returning canned outputs? Property-based fixtures over a frozen scenario set? The current "Testing Strategy" section lists domain coverage but skips this.

### 14. No placement statement

The spec doesn't say where this code lives in the existing layout. Sibling app under `apps/`? Module under `prescient_benchmark/`? New package? One line would prevent the implementer from picking arbitrarily.

## What's Right

- Generic `Source` / `SourceUnit` / `SourceLocator` / `EvidencePassage` model is the correct shape and reusable beyond manuals.
- Locators kept separate from extracted text — survives re-indexing and parser changes.
- Conservative answer status enum (`answered`, `needs clarification`, `insufficient evidence`, `error`) is the right contract.
- Provider port boundary is motivated correctly and the recorded provider metadata (model id, input unit ids, output fingerprint, timestamp) is the right minimum.
- Excluding vehicle CRUD and `TorqueSpec` as a first-class type is correct discipline against vertical drift.
- Sequencing — ship usable slice first, promote eval cases from real usage second — is the right order if the conflict in Concern #1 is acknowledged.

## Recommended Next Steps

1. Decide: does the dogfood UI exist as a product surface, or only as a thin probe? Update or supersede the 2026-04-22 idea doc accordingly.
2. Read `apps/api/src/prescient_benchmark/{corpus,retrieval,eval}` and rewrite the "Ingestion Design," "Retrieval And Answering Flow," and "Persistence" sections to name what is being extended versus what is new.
3. Decide on the agent/primitives surface or relabel v1 as structured RAG with a follow-on item.
4. Pick a section-governance approach (PDF outline + heading-range heuristic is a reasonable starting point) or list it as a known v1 limitation.
5. Inherit the idea doc's quantitative success criteria.
6. Commit to image-rendered citation viewing.
7. Either rename the contracts to generic from the start, or document why the rename is worth its cost.
