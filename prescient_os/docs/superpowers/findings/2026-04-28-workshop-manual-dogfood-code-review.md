# Workshop Manuals Dogfood Cluster — Code Review

**Date:** 2026-04-28
**Branch:** `poc/retrieval-benchmark-workshop-manuals`
**Range:** `1d513a4..e1e76e9` (11 commits, ~3.3K LoC across 35 files)
**Reviewer:** superpowers:code-reviewer subagent
**Scope:** Workshop manuals dogfood cluster — closed beads `1x9`, `2lx`, `542`, `795`, `7eu`, `k73`, `t7n`, plus follow-on commits (`8bfb0e0`, `1c1f39f`, `f33fd83`, `e1e76e9`).

## Overall Assessment

The slice cleanly separates the layers the spec asks for: generic `knowledge/` contracts, a workshop-specific package, FastAPI/MCP adapters, and a thin Next.js probe. Tests cover the happy path for every module, the answered/insufficient-evidence branch is exercised end-to-end, and the new `--index/--no-index` flow plus the `e1e76e9` stale-hit fix are both genuinely tested. As a "first usable" dogfood, the bones are right.

That said, several focus areas have real correctness or scope-leakage problems that will bite the moment this corpus moves past one PDF page. The retrieval side has at least two data-integrity issues that should block leaning on this as the basis for a public eval until they are fixed.

---

## Critical (data correctness / scope correctness)

### C1. `/knowledge/citation-page` is an unauthenticated arbitrary-file-read endpoint

`apps/api/src/prescient_benchmark/api/routes_knowledge.py:56-58`

```python
@router.get("/citation-page")
def citation_page(target: str) -> FileResponse:
    return FileResponse(target, media_type="image/png")
```

`target` is taken straight from the query string and handed to `FileResponse` with no validation. A request like `GET /knowledge/citation-page?target=/etc/passwd` returns the file with `Content-Type: image/png`. The web UI URL-encodes `selectedLocator.target` directly (`page.tsx:66`), so the API is implicitly trusting an unauthenticated client to provide a path it will then read from disk. This is unauthenticated path traversal / arbitrary local file disclosure.

**Why it matters:** the moment this binds to anything other than `localhost`, it is a file-read primitive. Even on localhost, dogfood gets demoed.

**Fix:** resolve `locator_id` (or unit_id + page) on the server, look up the locator in the store, verify the resolved target is `is_relative_to(rendered_pages_root)`, then serve. Never accept the raw filesystem path from the client.

### C2. Auto-search ignores the request's `scope_id` — every call is implicitly scoped to the seeded 360 Modena

`apps/api/src/prescient_benchmark/api/routes_knowledge.py:30-48`, `mcp/workshop_server.py:23-30`

`AskRouteRequest` accepts `scope_id`, but the handler discards it:

```python
scope = default_360_scope()
```

The MCP `ask_knowledge_question` tool doesn't even surface `scope_id`. The spec is explicit (`scope-design.md` §Web UI / §MCP): the system must distinguish 360 Modena vs. Challenge Stradale vs. Challenge and ask for clarification when it cannot. Today every question is answered against `linked_source_ids` for *all* 360 sources concatenated, regardless of what the caller said. There is no clarification path, no variant detection, and no per-variant scoping — the conservative-answer guarantees in the spec (`§Retrieval And Answering Flow`, `§Success Criteria`) cannot hold.

This is the cross-scope leakage you specifically asked about: scoped search *does* filter on `linked_source_ids`, but the scope itself spans all 360 family manuals, so a Modena question can be answered out of a Challenge Stradale page with the seeded variant tag never consulted. Variant disambiguation is in the spec but not in the code.

**Why it matters:** the scope contract is the entire point of the slice. If you base eval thresholds (90% scope accuracy, 95% citation coverage on numeric claims) on this implementation, you are measuring the wrong thing.

**Fix path:** (a) honor `scope_id` in `ask_knowledge` and `resolve_seeded_scope`; (b) plumb `variant_tags` from the source filter into search (or add a per-variant scope so 360 Modena scope only links to the Modena WSM by default); (c) add a real `needs_clarification` path that returns when the resolved hits cross variant tags.

### C3. `resolve_seeded_scope` lies — both branches of the `if` return the same scope

`apps/api/src/prescient_benchmark/workshop_manuals/catalog.py:101-104`

```python
def resolve_seeded_scope(scope_id: str | None) -> KnowledgeScope:
    if scope_id in (None, DEFAULT_360_SCOPE_ID):
        return default_360_scope()
    return default_360_scope()
```

The function pretends to dispatch but always returns the default. `tests/unit/test_workshop_catalog.py:25-29` explicitly asserts this behavior ("unknown scope falls back to default for v1"). That is a tautology test masquerading as the scope resolver: the fallback is the only branch.

**Why it matters:** `resolve_seeded_scope` is the only seam where "another scope shows up later" gets implemented. As written, it's actively misleading — adding a second scope requires rewriting the function and the test that locks the bug in.

**Fix:** either return `None` / raise on unknown ids and let callers decide, or build a real registry map keyed on `scope_id`. Re-do the test so the unknown-scope path checks something a future change can break.

### C4. Stale-hit fix is a band-aid that silently masks index/store divergence

`apps/api/src/prescient_benchmark/workshop_manuals/retrieval.py:96-104`

The fix turns `units[hit["_source"]["unit_id"]]` into `units.get(...)` and `continue`s on miss. The integration test (`test_workshop_retrieval.py:85-114`) confirms the swallow but never produces a log, metric, or surfaced warning — and the `locator_for_unit is None` branch above does the same silent skip.

The contract risk is real. `ensure_workshop_index` does a delete+create on every call (`retrieval.py:26-27`), but `client.index(...)` calls are not idempotent against the *store*: any unit removed from the store between ingest passes (e.g. fingerprint-rebuild path you don't implement yet) becomes a permanent ghost in OpenSearch until someone re-runs ensure. Today the fix means: search returns 5 hits, 3 of them are stale, the user sees 2 results, no one knows the index drifted.

**Why it matters:** "tolerate" is fine; "tolerate silently and produce zero observability" is data-loss-shaped. For a quality-over-speed retrieval engine, this is the kind of thing that makes eval scores untrustworthy.

**Fix:** at minimum, log a warning on each skip with `(index_name, unit_id)`. Better, count stale hits per query and put the count in the retrieval trace; ideally, re-index from the store atomically (alias swap) so this race cannot happen.

### C5. Citations are constructed from `units[0]` only — they don't actually cite the answer

`apps/api/src/prescient_benchmark/workshop_manuals/service.py:61-73`

```python
first_unit = units[0]
first_locator = locators[0]
...
citation = EvidenceCitation(
    ...
    unit_ids=[unit.unit_id for unit in units],
    locator_ids=[locator.locator_id for locator in locators],
    title=source.title if source else first_unit.source_id,
    page_number=first_locator.page_number,
    ...
    excerpt=first_unit.text[:500],
)
```

A single citation is fabricated: title/page/excerpt come from `units[0]`, but `unit_ids` and `locator_ids` lump *every* candidate together. If the LLM said "25 Nm" and the support is on `units[3]`, the citation says page 1 with the page-1 excerpt and a `unit_ids` list that mixes pages from different sources. The web UI then renders only `answer.locators[0]` (`page.tsx:104`), which is also `units[0]`'s page, regardless of which citation button the user clicks (the click handler is hardcoded to `answer.locators[0]`).

This violates `§Design Principles 4` ("every numeric or procedural claim must cite source evidence") and `§Success Criteria` ("answers either cite exact manual pages or refuse to answer definitively").

**Why it matters:** the citation primitive is what makes this corpus worth dogfooding. Today citations are decorative, not load-bearing.

**Fix:** the service needs `enforce_citations` for real — at minimum, return one `EvidenceCitation` per supporting unit with that unit's own page/excerpt/locator, and have the LLM provider report which unit indices it relied on. The web UI's selected-locator click handler also needs to thread `citation.locator_ids[0]` (or per-citation locators) instead of always picking `locators[0]`.

### C6. Auto-search test in the web/API path is tautological — it doesn't prove explicit-id mode still works distinctly

`tests/integration/test_workshop_api.py:79-100`

The "searches when candidates are omitted" test seeds one unit, indexes it, asks a question, gets the unit. The "explicit candidate_unit_ids" test (`test_workshop_api.py:58-77`) seeds the same unit and supplies its id. Both pass; nothing actually checks that explicit ids *bypass* search (e.g. with the index empty, or with a candidate that wouldn't score on the query).

Bead 795 specifically calls out "deterministic candidate_unit_ids still supported." A real test would: (a) provide an id whose text doesn't match the query, then verify it is still returned; or (b) provide ids while the index is empty/missing, then verify the answer still resolves. Without that, the "auto-search" change can silently start ignoring `candidate_unit_ids` and the suite stays green.

---

## Important (architecture / maintenance / spec drift)

### I1. Store boundary is not actually replaceable — file paths leak into every adapter

`apps/api/src/prescient_benchmark/api/routes_knowledge.py:31,63`, `mcp/workshop_server.py:17,48`

The plan and bead 2lx say the store sits behind a port so a Postgres adapter can replace the file manifest later. In practice, every caller does `WorkshopManualStore(Path(settings.corpus_root) / "workshop_manuals" / "store")` inline. There is no `WorkshopManualStore` Protocol/ABC, no factory, and the constructor takes a filesystem `Path`. Replacing this with Postgres means hand-editing four call sites in three files — that's not a port boundary, that's a class.

**Why it matters:** the spec calls this out as the explicit forward-compatibility test. If we don't have it now, retrofitting later costs more than building it now.

**Fix:** define a `KnowledgeStore` protocol in `knowledge/` with the methods the service actually needs (`get_units(ids)`, `locator_for_unit`, `get_source`, `append_feedback`, etc.), have FastAPI/MCP take it via dependency injection, and keep `WorkshopManualStore(Path)` as one implementation. The store path string in routes is a smell that there is no boundary today.

### I2. Store performance is O(N²) on every call

`apps/api/src/prescient_benchmark/workshop_manuals/store.py:59-104`

Every `upsert_*`, `get_source`, `units_for_source`, and `locator_for_unit` re-reads the entire JSON file from disk and re-validates every record. `ensure_workshop_index` calls `store.locator_for_unit(unit.unit_id)` indirectly through `units = {unit.unit_id: unit for unit in store.list_units()}` (OK — one read), but `search_workshop_units` calls `store.locator_for_unit(unit.unit_id)` *per hit* (`retrieval.py:102`), which means one full file read per hit. The 360 WSM is hundreds of pages — at top_k=5 you take 5 full re-reads of `locators.json` per query. `ingest_pdf_source` similarly re-reads `units.json` once per `upsert_units` and once per `upsert_locators` (twice across the whole list).

**Why it matters:** dogfood-tier performance is fine, but the in-loop usage in retrieval will get noticeable on the real manuals, and once you replace JSON with Postgres these patterns become "one round-trip per hit" — not OK.

**Fix:** load once per request and pass dicts/lists into the helpers; or have the store cache list reads behind an in-memory map keyed on the file mtime.

### I3. PyMuPDF documents and pixmaps are not closed on exception paths; `fitz.open(Path)` is type-fragile

`apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py:52-105`

`doc = fitz.open(source_path)` is not in a `with` (PyMuPDF supports `with fitz.open(...) as doc`), and the `try/finally doc.close()` covers the doc but `pixmap` is never explicitly released. On a long PDF this leaks native memory. `fitz.open` also expects `str`/bytes/PathLike — PyMuPDF generally accepts `Path` but versions diverge; coercing with `str(source_path)` is more portable.

Also: ingestion is documented as "fingerprint-aware" (bead 2lx, store has `content_fingerprint`), but the ingester recomputes the fingerprint and re-renders all pages every run. The `if not rendered_path.exists()` guard skips re-rendering individual pages, but text extraction, structures, and OpenSearch indexing run unconditionally. Bead 2lx promised "skip unchanged sources." Today it's idempotent in outcome, not in cost.

**Fix:** look up existing source via `store.get_source(source_id)`, compare `content_fingerprint`, short-circuit if equal and rendered files all exist.

### I4. Heading regex is too aggressive; structures are noise on most pages

`apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py:125-130`

`^[A-Z0-9][A-Z0-9. -]{2,}$` matches things like part numbers (`360F-12345`), table-of-contents entries, ALL-CAPS warnings, and page banners. The fixture text "A. ENGINE" trivially matches; a real Ferrari WSM page has dozens of all-caps fragments per page and the regex will pick whichever one happens to be first. The spec (`§Section Governance`) explicitly calls out PDF outline/bookmarks → page labels → heading-range as the order, but the implementation only does heading-range and uses a regex that conflates "heading" with "any uppercase line." There is no `doc.get_toc()` / outline path, even though PyMuPDF exposes it freely.

The structure produced by this is `confidence=0.6` always. The conservative answer policy is supposed to weight this; nothing in `service.py` or `retrieval.py` ever reads confidence.

**Fix path:** read `doc.get_toc()` first; only fall back to heading regex when the outline is missing, and tighten the regex (length, percentage of letters, blacklist of common false positives like part numbers).

### I5. CORS allow_origins are hardcoded; web `apiBase` defaults to `http://localhost:8000`

`apps/api/src/prescient_benchmark/main.py:10-15`, `apps/web/app/page.tsx:26`

`allow_origins=["http://localhost:3000", "http://127.0.0.1:3000"]` is fine for dogfood, but it's not configurable, and `NEXT_PUBLIC_API_BASE` defaults to `http://localhost:8000`. The moment this is exposed off-box (even on a Tailscale network), the front-end and CORS lists drift. There is no env-driven `BACKEND_ORIGINS`/`NEXT_PUBLIC_API_BASE` enforcement and no test guarding it.

**Fix:** drive both from settings/env, fail closed if `NEXT_PUBLIC_API_BASE` is unset in production builds.

### I6. `KnowledgeAnswer` validator allows other failure modes to skip citations silently

`apps/api/src/prescient_benchmark/knowledge/models.py:67-70`

The validator only enforces `ANSWERED` → citations. But the spec says (`§Retrieval And Answering Flow`): "if candidate evidence exists but does not answer directly, present the candidate sections as evidence to inspect." Today an `INSUFFICIENT_EVIDENCE` response can return zero citations even when candidates were found (`service.py:50-59` does exactly this — citations is empty list, but locators is populated). The contract doesn't model "candidates-as-evidence-to-inspect" vs. "no candidates at all." A future caller can't distinguish "we have no idea" from "we found three pages but couldn't synthesize."

**Fix:** add an explicit support_role for `candidate_evidence` (the citation model already has `support_role`), and emit citations on the insufficient-evidence path when `units` and `locators` are non-empty.

### I7. Tests mock the LLM with a substring match on "25 Nm"

`apps/api/src/prescient_benchmark/workshop_manuals/providers.py:16-25`

`DeterministicLlmProvider` returns `supported=True` iff the literal string `"25 Nm"` is in the evidence. Every test case happens to contain "25 Nm" — that is the only thing being tested in the integration suite for "answered" status. There is no real `LlmProvider` adapter (no OpenAI/Anthropic implementation lives in the repo), so what runs in production is genuinely unknown from this branch.

The spec calls out provider boundaries (`§Provider Boundary`) and required test coverage (`§Testing Strategy`: "Mocked `LlmProvider` behavior for supported, unsupported, ambiguous, and provider-failure answers"). The current provider has supported/unsupported but not ambiguous, not provider-failure, and there is no real LLM at all. The `/ask` route serves answers that say "The cited evidence states 25 Nm" verbatim regardless of question.

**Why it matters:** this is fine *as scaffolding*, but the closed-bead 3d2ea88 ("knowledge answer service") implies a working answer service. The dogfood UI is currently a regex match, not a question answerer.

**Fix:** at minimum file a follow-up bead and gate the auto-search/web probe behind a real provider. As reviewed today, the slice cannot satisfy bead 795's intent.

---

## Minor (tidiness, conventions)

### M1. `EvidenceCitation` has no validator constraining `unit_ids`/`locator_ids` to be non-empty

`knowledge/models.py:41-51`

Easy to construct a citation with empty `unit_ids` and `locator_ids`. Tighten with a `model_validator` that requires both to be length ≥1, and that `len(locator_ids) == len(unit_ids)` (or a documented relationship), since the service's `[locator.locator_id for locator in locators]` pattern only works when they're parallel.

### M2. `support_role` is `str` rather than an enum

`knowledge/models.py:50`. Three free-text values float around (`"answer_support"`, `"scope_support"`, `"clarification_support"`). The spec lists them; codify them as `StrEnum`.

### M3. `KnowledgeScope.scope_type` and `SourceLocator.locator_kind`/`SourceUnitRecord.unit_kind` are also free-text strings

Same problem. With `extra="forbid"` set everywhere, the discipline is partly there — finish the job by enumerating these values.

### M4. `_extract_heading` returns the first regex match per page

`pdf_ingest.py:125-130`. Pages with multiple headings collapse into the first. Even for the dogfood path, prefer the longest match or first-after-non-whitespace-padding.

### M5. Web UI citation buttons are non-functional

`apps/web/app/page.tsx:101-109`. Every `<button>` calls `setSelectedLocatorId(answer.locators[0]?.locator_id ?? null)`, so clicking citation #2 still selects citation #1. Should use the citation's own `locator_ids[0]`.

### M6. `tsconfig.json` has `"strict": false`

`apps/web/tsconfig.json:11`. Given the UI's small surface, turn strict mode on now while it's cheap.

### M7. `WorkshopManualStore` `_write_models` is not atomic

`store.py:120-122`. A crash mid-write corrupts the entire JSON file (and there is no backup). Use `Path.write_text` to a `.tmp` file and `os.replace`.

### M8. `default_360_scope` is rebuilt on every call

`catalog.py:91-98`. Pure function, so it's not wrong — but caching with `@lru_cache` (or storing the tuple of ids once) makes the intent clearer and avoids repeated string list construction in hot paths.

### M9. CLI `index` flag passes through but doesn't expose `corpus_version`

`cli.py:158`. Hardcoded to `"v1"`. The retrieval module exposes `workshop_index_name(corpus_version)` precisely so this can change; the CLI should accept it.

### M10. `tests/integration/test_workshop_api.py:103-117` reads a path served via the unsafe route

The "citation page route serves rendered page" test passes a path under `tmp_path`, which is "fine" — but it also confirms there is zero validation. Add a negative test asserting a request for `/etc/passwd`-style paths is rejected once C1 is fixed.

---

## What's done well

- **Layering is correct.** `knowledge/` truly does not import `workshop_manuals/`; the workshop package is the only thing that names "Ferrari" or "PDF." The contract names (`AskKnowledgeQuestionRequest`, `KnowledgeAnswer`, `EvidenceCitation`, `SourceLocator`) match the spec's `§Contract Stability` list verbatim, so a future business-document corpus can plug in without an API rename.
- **The MCP and FastAPI adapters are genuinely thin.** Both call `AskKnowledgeService.answer_from_candidates` with the same arguments, and the MCP server doesn't reimplement retrieval. That's exactly what bead t7n asked for.
- **`ensure_workshop_index` rebuilds mappings deterministically** (`retrieval.py:25-44`) — explicit fields, no dynamic mapping surprises.
- **CLI `--index/--no-index` is properly tested** (`test_workshop_manuals_cli.py:44-72`) with `monkeypatch` injection of the OpenSearch client and the index function — a real seam, not a tautology.
- **Stale-hit test is a real reproducer** (`test_workshop_retrieval.py:85-114`): it indexes a synthetic ghost unit and asserts the search returns `[]`. The fix itself is too quiet (C4), but the test is honest.
- **Spec/plan alignment is high on the structural axes.** File placement matches the plan's "File Structure" section to the letter; the bead-by-bead deliverables are implemented.

---

## Ready-to-merge verdict

**Not ready as the foundation for a public eval.** Three issues materially affect the very thing the corpus is meant to validate:

- **C2 (scope_id ignored / no variant detection)** breaks the spec's headline scope guarantee and will skew any retrieval-support score the eval produces.
- **C5 (fabricated single-page citations + UI ignores per-citation locators)** means citation accuracy is not actually measurable from the system's outputs.
- **C1 (path-traversal on `/citation-page`)** is unsafe to expose anywhere off-loopback, including a demo.

C3, C4, C6 each undermine confidence in test signal and forward extensibility. I'd want at least C1, C2, C4, C5 fixed before scoring this against the 40–60 promoted-question target.

**Acceptable as a personal dogfood** if (a) you bind the API to localhost only, (b) you mentally treat the answer text as "first-unit excerpt, not synthesized," and (c) you don't draw eval conclusions from the variant-disambiguation success rate.

Suggested unblocking order: **C1 → C2 → C5 → I1 → C4 → C3/C6 → I7**. C1, C2, C5 are roughly half-day each; I1 is a one-shot rewrite of the store boundary that will pay back across every later change.

---

## Files referenced

- `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- `apps/api/src/prescient_benchmark/main.py`
- `apps/api/src/prescient_benchmark/knowledge/models.py`
- `apps/api/src/prescient_benchmark/mcp/workshop_server.py`
- `apps/api/src/prescient_benchmark/workshop_manuals/catalog.py`
- `apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py`
- `apps/api/src/prescient_benchmark/workshop_manuals/providers.py`
- `apps/api/src/prescient_benchmark/workshop_manuals/retrieval.py`
- `apps/api/src/prescient_benchmark/workshop_manuals/service.py`
- `apps/api/src/prescient_benchmark/workshop_manuals/store.py`
- `apps/api/src/prescient_benchmark/cli.py`
- `apps/web/app/page.tsx`
- `apps/web/tsconfig.json`
- `tests/integration/test_workshop_api.py`
- `tests/integration/test_workshop_retrieval.py`
- `tests/integration/test_workshop_manuals_cli.py`
- `tests/unit/test_workshop_catalog.py`
- `tests/unit/test_workshop_service.py`
- `docs/superpowers/specs/2026-04-28-workshop-manual-dogfood-design.md`
- `docs/superpowers/plans/2026-04-28-workshop-manual-dogfood.md`
