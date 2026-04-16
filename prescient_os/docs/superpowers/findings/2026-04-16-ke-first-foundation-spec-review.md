# 2026-04-16 KE-First Foundation Spec — Critical Review

## Context

Requested as a hole-poking review of the KE-first greenfield foundation spec before any implementation work begins.

- Spec under review: `docs/superpowers/specs/2026-04-16-ke-first-greenfield-foundation-design.md`
- Prior direction (archived): branch `feature/ke-governance-review` in the code repo. That branch had grown to include sponsor workflows, claim/release queues, onboarding interviews, profile drafts, review notifications, strategy/initiatives, action items with state machines, and full DDD layering across ~15 bounded contexts.
- Stated reason for the pivot: "I was building too many shallow-and-wide features; the important thing is whether the knowledge engine works. I don't need a UI for that."

The pivot diagnosis is correct. This review is about whether the *new* spec actually solves the problem the pivot was meant to solve.

## One-Sentence Risk

The new spec risks repeating the shallow-and-wide mistake at a different altitude: last time, many features shallowly at the product layer; this time, many concerns shallowly at the architecture layer (ingestion + normalization + curation + timeline + visibility + retrieval + governance) — and still not answering *"does business-entity-anchored knowledge curation produce demonstrably better answers than vanilla RAG on my real questions?"*

## Where the New Spec Still Fails Its Own Test

### 1. The thesis is never stated in falsifiable terms

The spec's goal is "retrieve current company context better than generic chat systems." That is a wish, not a thesis. It cannot be validated because:

- No query set — what questions should the KE answer?
- No baseline — better than what: ChatGPT + file upload, Glean, vanilla RAG?
- No metric — better on what: groundedness, freshness, coverage, specificity?
- No failure condition — if X is true after 4 weeks, the thesis is dead.

**Before writing another line of code:** pick 20–50 real questions a CEO/operator would ask about a company, write the ideal grounded answers, and define what "winning" looks like against a dumb RAG baseline. If the eval set cannot be written, the thesis isn't real yet.

### 2. The spec bets hard on curation-first against industry consensus

The core architectural bet is: **durable entity-backed primary artifacts beat on-the-fly retrieval-augmented synthesis.** That is a contrarian bet. Glean, ChatGPT Enterprise, Perplexity, Notion AI, and most current enterprise-knowledge products went retrieval-first with light entity modeling. The spec never defends *why* curation wins.

Steel-man the other side: curated artifacts may just be a cache for answers that retrieval could produce directly, and the cache may not be worth the curation overhead. If that is true, the Observation → Artifact → Revision pipeline is premature optimization.

A clean articulation is needed of (a) when curation-first beats retrieval-first, and (b) a concrete failure mode of retrieval-first that the curation bet is designed to fix. Without this, the expensive side of the fork is being built on faith.

### 3. Scope is still too wide for "validate the thesis"

V1 as written carries:

- 4 ingestion channels (docs, chat, email, web)
- ~12 entity types
- 7 shared blocks + per-entity extension blocks
- Primary artifact + drafts + approval flow
- Timeline (events, revisions, approvals, merges)
- Unresolved inbox (ambiguous matches, candidate entities, conflicts)
- 3-layer retrieval pipeline
- 4-level visibility scopes
- Auto-apply policy with provenance trail
- Priority as a structured block
- Packages

If the question is "does the KE work?", **none of visibility, packages, priority, or the unresolved inbox actually test that.** They test governance. The thesis can be proved or disproved with: 1 channel, 2 entity types, 1 artifact shape, no drafts, no timeline, no visibility. Everything else is the same trap the pivot was meant to escape.

### 4. "No UI" contradicts the design

The spec requires human review in several places:

- Candidate entity confirmation
- Unresolved inbox triage
- Primary artifact approval
- Request-changes loops on drafts
- Human-authored priority

A human review loop without UI is a CLI — which is fine, but either (a) admit a minimal review surface is needed and design it, or (b) remove human-in-the-loop from v1 and accept lower precision. Pretending "no UI" and "human approval is a core concern" can coexist is how scope creep starts.

### 5. Blocks are a UI abstraction hiding as a data model

`overview`, `status`, `risks`, `decisions`, `open_questions` are Notion/Confluence-shaped. With no UI, "blocks" are just a nested JSON schema with extra ceremony. Either a UI is coming (in which case blocks are speculative) or it isn't (in which case plain domain schemas are simpler). Right now the choice is cosmetic and adds serialization and migration surface area.

### 6. DDD + hexagonal + modular monolith is architectural overhead for a thesis prototype

These patterns pay off over 1–3 years of team evolution. For proving whether the KE thesis is real, they are scaffolding tax. The prior branch is Exhibit A: full DDD layering across 15+ bounded contexts and the thesis still wasn't validated.

For a thesis proof, a single-package Python project with hand-rolled persistence and no hexagon is faster and more honest. Refactor to DDD *after* the domain is validated. Designing DDD boundaries before the domain is real is backwards.

### 7. The retrieval/LLM stack — the thing that actually determines success — is unspecified

The spec names Postgres + OpenSearch and "layered retrieval." That is not a retrieval design. What is missing:

- Embedding model and dimensionality
- Chunking strategy (per-block? per-observation? hybrid?)
- Dense vs. sparse vs. hybrid retrieval
- Reranker, or none
- Query rewriting / decomposition
- How entity-awareness enters retrieval (entity-filtered search, entity-aware rerank, graph expansion?)
- How "approved primary artifact first" is physically implemented
- Grounding verification before returning an answer
- Hallucination guardrails

**This is the entire ballgame and it is not designed.** Everything else could ship perfectly and retrieval quality could still lose.

### 8. Entity resolution is the hardest unsolved problem and the spec punts

"Candidate entities must be confirmed by humans" is a correctness story, not an architecture. In practice:

- How are mentions matched to entities — fuzzy name + type, embeddings + rerank, LLM-judge?
- What precision/recall target would justify auto-apply?
- What happens when an observation is ambiguously about 2 entities?
- Alias handling, merges, splits

Entity resolution is where most knowledge-graph projects die. Without a picked approach and a measurement, the "unresolved inbox" becomes an infinite queue no one touches.

### 9. Append-and-revise with timeline is the wrong default for a prototype

The write semantics (append-and-revise, auto-apply policy, revision history, timeline events) are being designed before a single retrieval has proven durable curated artifacts are useful. If curation-first loses, all this write machinery is dead weight. **Get retrieval working against trivially-written artifacts first. Design update semantics once the read side earns them.**

### 10. "API-first" with no API consumer is premature contract work

With no UI and no external integrator yet, OpenAPI contract generation, client codegen, and strict API boundaries are ceremony. The first consumer of this API is the eval harness. Build for that. Real API contract design happens when a real second consumer exists.

## Recommended V0

A ruthlessly narrow v0, designed against a concrete eval:

1. **Write the eval first.** 30 real questions + ideal answers + a scoring rubric. Nothing else gets built until this exists.
2. **Pick ONE channel.** Files only. Email/chat/web are later.
3. **Pick TWO entity types.** Whichever two are most load-bearing for the eval questions (likely `company_context` + one of initiative/customer/employee).
4. **One artifact shape.** Flat fields, not blocks. No drafts, no approval — "latest state, overwrite on revise."
5. **No timeline, no inbox, no visibility, no priority, no packages** in v0.
6. **Design the retrieval stack explicitly.** Embedding model, chunking, hybrid search, rerank, grounding check, entity-aware query rewriting. This is the product.
7. **Single Python package.** No DDD, no hexagon, no modular monolith boundary. Refactor when it's earned.
8. **Run the eval weekly.** Score against the dumb-RAG baseline. If v0 isn't beating baseline by week 4–6, the thesis is in trouble — and that's exactly when that signal is most valuable.

Everything in the current spec past step 7 is a v1+ concern and should be moved to a "future considerations" section so it stops shaping v0 implementation choices.

## Suggested Next Steps

- Either revise the foundation spec to a v0 shape along the lines above, or supersede it with an eval-first thesis doc that becomes the real v0 north star.
- Decide explicitly whether curation-first or retrieval-first is the bet, and write the one-paragraph defense of that choice.
- Keep the current foundation spec as a v1+ aspirational design; do not treat it as the implementation target.
