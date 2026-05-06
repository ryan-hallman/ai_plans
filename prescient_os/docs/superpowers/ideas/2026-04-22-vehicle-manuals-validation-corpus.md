# 2026-04-22 Vehicle Manuals as a Validation Corpus

## Purpose of this document

Propose using a personal collection of vehicle workshop manuals (`../manuals/`) as a private validation corpus for Prescient OS's retrieval spine. This is an idea doc, not a spec — it argues the case, names the hard problems the corpus exercises, and draws the line between what it is (retrieval stress test) and what it isn't (a pivot toward automotive verticals). If we proceed, concrete pieces fork into specs (ingestion, question set, evidence key).

Expected lifespan: short. Either we run the validation and promote findings, or we decide not to.

## Why this corpus, now

Two recurring problems for Prescient OS:

1. **Thin real data.** The engine is being built faster than the operator-first pilots are generating dogfood material. Synthetic eval and the small Peloton private corpus are useful but narrow.
2. **The thesis's hardest claims are under-tested.** The retrieval thesis (`ideas/2026-04-18-retrieval-thesis.md`) makes specific claims about scope extraction, structural walk, table context, and proprietary vocabulary. The Peloton corpus is earnings-call-shaped — it exercises some of these but not all. We need a corpus where the canonical RAG failure modes are obvious and ground truth is judgeable.

Workshop manuals hit both. They are:

- Heterogeneous (text, tables, exploded diagrams, wiring schematics, revision-dated sections)
- Strongly scoped (vehicle → system → sub-system → procedure)
- Full of context-dependent tables (torque specs listed at the start of a procedure apply to that procedure's bolts, not the manual as a whole)
- Written in proprietary vocabulary (OEM part codes, chassis codes, workshop shorthand)
- Not in public LLM training data (the important ones are manufacturer-internal)
- Used in anger by the author several hours a day, with ground-truth expertise to judge answer quality

No synthetic benchmark matches that combination.

## What is actually in the corpus (inventory)

`../manuals/`, ~1.0 GB, 12 PDFs:

**Ferrari 360 family**
- `ferrari_360_wsm.pdf` — complete multilingual 360 workshop manual
- `206990-Ferrari_360_Challenge_Stradale_1999-2005.pdf` (79 MB) — Challenge Stradale (road, trackday)
- `223037-Ferrari_360_Gearbox.pdf` (15 MB) — 360 gearbox-specific
- `01–05 ...` (28 MB total) — the Ferrari 360 **Challenge** (race car) documentation set: Technical Regulations 2000, Technical Features & Usage, Data Acquisition System, Gearbox & Differential Overhaul, Spare Parts Catalogue

**Ferrari F430 family**
- `127583-Ferrari_F430_Spider_2004-2009.pdf` (201 MB) — F430 Spider workshop manual
- `Ferrari F430 Challenge Workshop Manual.pdf` (37 MB) — F430 Challenge race variant

**Land Rover Defender**
- `55859-Land_Rover_Defender_90_110_1983-1990.pdf` (29 MB)
- `103270-Land-Rover_Defender_TD5_EWD - Electrical Wiring Diagrams.pdf` (1.2 MB)

The variety is a feature, not a bug — it forces the retrieval spine to handle real model-level disambiguation rather than a single-vehicle toy problem.

## Thesis propositions this corpus stress-tests

Each of these is a specific claim from the retrieval thesis or recall/memory design that this corpus exercises directly.

### 1. Scope extraction is first-class

A question like "what is the torque spec for the cam cap bolts" is useless without scope. The correct first step is extracting (or asking for) the vehicle and engine variant. With three "360 Challenge"-adjacent cars (360 Modena vs 360 Challenge Stradale vs 360 Challenge race car) and a second-generation successor (F430, which inherits part of the 360's architecture), the corpus makes scope-mismatch errors visible. If the agent hands back an F430 Spider spec in response to a 360 CS question, that is an unambiguous, judgeable failure.

### 2. Long-context read solves chunk-boundary failures

Torque-spec tables typically appear at the head of a procedure section. In naïve RAG, chunking splits the table away from the procedural context, producing answers like "the torque spec is 25 Nm" without "...for which bolts." The thesis's prescription is: pick the right section, read it whole, answer with citation. This corpus tests that directly and the failure mode is obvious to a domain expert (the answer either names the right bolts or it doesn't).

### 3. Structural walk as a primitive

Workshop manuals are hierarchically organized: section → subsection → procedure → step. Many questions require walking the structure rather than top-k'ing across the whole manual ("before I can torque the cam cap, what prerequisite steps does the procedure list?"). This is where the "structural walk" primitive named in the thesis earns or loses its keep.

### 4. Knowledge graph / alias expansion on proprietary vocabulary

OEM part codes, workshop shorthand, and chassis codes (F131, F136, F133, etc.) are vocabulary no public LLM reliably knows. The system needs an alias path: user says "the Modena V8" → catalog knows that's the F131 engine → manual section for F131 head assembly. This exercises the KG alias layer from the recall/memory design specifically, without any overlap with business vocabulary.

### 5. Citation-first answers

Wrong torque specs damage engines. Provenance is non-negotiable. Every numeric or procedural claim must cite (manual, section, page, figure). A question-answer pass where the agent can't point to a page is a failure even if the numeric answer is correct — that is exactly the stance the recall spec takes for verified artifacts.

### 6. Model / variant disambiguation (the "which car" problem)

Stated explicitly as a user concern. The corpus makes this concrete:
- `360 Challenge` ≠ `360 Challenge Stradale` ≠ `F430 Challenge`
- `360 Modena` shares parts with both `360 Spider` and `360 CS` but has distinct torque specs in some cases
- `F430 Spider` inherits architecture from 360 but has divergent service intervals

The agent's first move on any query should be to disambiguate. If it cannot, it should ask — not guess. This is the scope-extraction primitive exercised at entity granularity.

## Validation plan

Reuse existing infrastructure; do not build parallel pipelines.

1. **Ingest manuals into the private corpus** alongside the Peloton set. The pipeline already exists (`plans/2026-04-20-private-retrieval-baseline-runner.md` and adjacent specs). Tag with `corpus=vehicle-manuals`.
2. **Extend the private-corpus question set and evidence key** (`specs/2026-04-18-private-corpus-question-set-and-evidence-key-design.md`) with a vehicle-specific suite. Target 40–60 questions, spread across:
   - Direct-lookup questions (torque spec for a named bolt — tests citation and table context)
   - Scope-disambiguation questions (same part name in two variants — tests the "which car" primitive)
   - Structural-walk questions (procedure dependencies — tests hierarchical read)
   - Proprietary-vocabulary questions (OEM codes, workshop shorthand — tests KG aliases)
   - Honest-gap questions (a bolt the manual does not spec — agent must say "I can't find this" rather than hallucinate)
3. **Score with the existing private retrieval scorer** (`specs/2026-04-18-private-retrieval-scorer-design.md`). No new scoring infra.
4. **Compare two paths on the same queries**: (a) the agent + primitives path (BM25 + scope + structural walk + long-read); (b) a classical RAG baseline over the same docs. This is the comparison the thesis demands.
5. **Dogfood daily** during active shop work. Track frustrations as candidate failure cases to add to the question set.

## Scope discipline — what this is NOT

This is the single most important section of the doc. Automotive is **not** a product direction. To prevent vertical drift:

- **No new artifact types.** "Torque spec" is not joining `decision_record`, `action_item`, `kpi_insight`. The retrieval spine is generic; what it retrieves is domain-agnostic chunks and documents.
- **No Ferrari-specific UI.** Memory Inbox, Artifact Library, the copilot chat surface are business-shaped and stay business-shaped. Validation happens against the retrieval scorer, not via the product UI.
- **No shop-workflow skills.** Do not add a `repair_procedure_planner` or similar. If the agent can answer "what is the torque spec" correctly, and also "what was Peloton's margin decomposition in Q3," the spine works. Neither deserves a vertical feature.
- **No corpus-specific extractors.** The moment we find ourselves writing a `detect_torque_table()` function, we stop and ask: is this a generic "tables inherit section scope" primitive, or is it domain-specific? If domain-specific, it does not ship.
- **Two hats, separated.** Product-user hat: "does this make me faster on my cars?" Eval-engineer hat: "does the same query shape work on 10-K filings?" Only the second hat drives product decisions.

A useful mental test: if we swap the Ferrari manuals for, say, FDA drug-insert PDFs or SEC filings, do the same primitives handle the questions with the same code path? If yes, we're validating the spine. If no, we're building a vertical and we stop.

## Ingestion considerations

Practical notes gathered from the actual files:

- **Several PDFs are very large.** The main 360 WSM is 673 MB; F430 Spider is 201 MB. Ingestion must stream, not load-in-memory, and must tolerate hour-plus processing per manual.
- **Most are scanned or image-heavy.** OCR quality will vary and should be tracked as a per-doc metric, not assumed uniform. Citation text that was OCR-garbled is a different failure class from retrieval-got-the-wrong-section.
- **Page references are load-bearing.** Citation must include page number (and figure number where applicable). Loss of page-level provenance is a blocker, not a polish item.
- **Figures are sometimes the answer.** An exploded parts diagram *is* the right answer to "what is the assembly order of the rocker arm." Figure references and captions need to be indexed, not stripped.
- **Revisions and dates matter.** The 360 manual spans model years; specs change. The catalog should capture revision dates where the manual exposes them.
- **Some manuals are cross-referenced.** The F430 Challenge manual refers back to the F430 Spider manual in places; the 360 CS manual references the base 360 WSM. Cross-manual references should resolve, not dead-end.
- **Local only.** Manuals are copyrighted manufacturer material. They do not leave the machine — no external embedding APIs with data retention, no third-party OCR services that keep inputs.

## What "good" looks like

The validation passes if, on the vehicle question set:

- Scope extraction correctly identifies the vehicle/variant ≥ 90% of the time on questions where the query names or implies a vehicle; asks for clarification on questions where it doesn't.
- Agent + primitives path beats the classical RAG baseline on retrieval-support score. Margin size matters less than direction + consistency.
- Citation coverage ≥ 95% — almost every numeric or procedural claim is traceable to a specific page and section.
- On honest-gap questions, the agent says "not found" at least 80% of the time rather than hallucinating. (The thesis bets on the "honest gaps" disposition; this is where that bet gets tested.)
- Dogfood: the author reports that asking Prescient is faster and more trusted than opening the PDF directly for at least half of the questions they had that day. This is a softer signal but the one the product ultimately has to deliver.

The validation fails (revise the thesis) if:

- Scope extraction works on 90% of business queries but < 70% of vehicle queries → scope-first is not horizontal, it's domain-tuned in ways we haven't recognized.
- The agent + primitives path underperforms classical RAG on retrieval-support → the primitives aren't earning their complexity on a corpus the thesis explicitly claims to favor.
- Honest-gap discipline collapses under proprietary vocabulary → we need a different strategy for low-confidence answers.

## Open questions

- **Does long-context read scale economically on 673 MB scanned PDFs?** The thesis assumes premium latency/cost; the 360 WSM may stress even that assumption. Need per-query cost telemetry.
- **OCR quality variance.** If the main 360 manual OCRs poorly in exactly the places the torque specs live, we are testing OCR pipelines, not retrieval. Mitigation: sample a few sections manually, measure OCR accuracy, flag as a known failure source before drawing thesis conclusions.
- **Question set authorship.** The author (domain expert) should write the initial question set from real shop frustrations, not from a synthetic template. How to keep this bounded in time?
- **Is the Land Rover subset load-bearing?** Two docs in a different domain. They add adversarial scope diversity (a Defender brake question must not match a Ferrari brake answer) but the question set may not justify the ingestion cost. Decide whether to include them in v1 or hold back as a second-round test.
- **Do we need exemplar-based retrieval here?** "Show me the procedure for the F430 that is analogous to this 360 procedure" is a natural user question and one of the narrow places the thesis says vectors earn their keep. This corpus could be a rare clean test of that narrower vector role.
