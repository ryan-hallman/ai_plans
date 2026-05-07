# Workshop Complete Manual Eval And Artifact UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the incomplete Ferrari 360 English PDF with the complete multilingual workshop manual, rebuild the eval baseline against the new corpus, then expose artifact-first answers in the web UI without regressing the current manual-retrieval experience.

**Architecture:** Keep `source-ferrari-360-wsm` stable and change only the source path so existing scopes, terminology mappings, and artifact provenance keep their identity. Treat the complete manual as a new corpus baseline because page ordinals may shift. Add scorecard diagnostics before rerunning so the new baseline can guide the following artifact-first UI slice.

**Tech Stack:** FastAPI/Pydantic, Typer CLI, PyMuPDF ingestion, OpenSearch workshop index, pytest, Next.js/React workshop probe UI, Beads issues `prescient_os-vtt` and `prescient_os-5ex`.

---

## Task 1: Complete Manual Source Swap

**Why:** The current catalog points to an incomplete English PDF. The source id should remain stable, but the source path must point to the complete uploaded manual before any eval refresh.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/catalog.py`
- Test: `tests/unit/test_workshop_catalog.py`

- [ ] Write a failing catalog test asserting the canonical 360 WSM source resolves to `Ferrari 360 Modena/ferrari_360_wsm.pdf` and keeps `source_id == "source-ferrari-360-wsm"`.
- [ ] Run `uv run python -m pytest tests/unit/test_workshop_catalog.py -q` and verify the new assertion fails.
- [ ] Update `WORKSHOP_360_SOURCES[0].relative_path` to `Path("Ferrari 360 Modena/ferrari_360_wsm.pdf")`.
- [ ] Run `uv run python -m pytest tests/unit/test_workshop_catalog.py -q` and verify it passes.

## Task 2: Eval Scorecard Diagnostics

**Why:** The refreshed baseline needs enough detail to diagnose multilingual/manual-page shifts, not just aggregate citation coverage.

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/eval.py`
- Test: `tests/unit/test_workshop_eval.py`
- Test: `tests/integration/test_workshop_manuals_cli.py`

- [ ] Write failing tests for category summaries on `WorkshopEvalSummary`, covering at least torque and procedure categories.
- [ ] Write failing tests for failed-question detail including question id, category, missing required unit ids, first required rank, and candidate unit ids.
- [ ] Run `uv run python -m pytest tests/unit/test_workshop_eval.py -q` and verify the new tests fail.
- [ ] Add small Pydantic models for category summary and failed-question detail.
- [ ] Update `summarize_workshop_eval_run(...)` to populate those fields while preserving the existing `failed_questions` list for compatibility.
- [ ] Update CLI output to print one line per category after the existing aggregate metrics.
- [ ] Run `uv run python -m pytest tests/unit/test_workshop_eval.py tests/integration/test_workshop_manuals_cli.py -q`.

## Task 3: Clean Reingest And Reindex Complete Manual

**Why:** The complete multilingual PDF has different pages, so reusing the prior store/index would mix incompatible page ids and rendered page images.

**Files:**
- No code changes expected.
- Local generated corpus under project `corpus/workshop_manuals`.
- OpenSearch index `prescient-workshop-v1`.

- [ ] Stop any assumption that old page ordinals are valid.
- [ ] Move or delete the local generated workshop corpus directory for a clean rebuild.
- [ ] Run `make dev-services` if Postgres/OpenSearch are not already running.
- [ ] Run `make ingest-workshop-manuals` for `~/Projects/workshop_manuals` with indexing enabled.
- [ ] Confirm the new store contains `source-ferrari-360-wsm` with `origin_path` ending in `ferrari_360_wsm.pdf`.
- [ ] Confirm rendered pages exist for the new source.

## Task 4: Repair Eval Evidence Keys

**Why:** Existing required unit ids may now refer to the wrong page after replacing the PDF. The eval file must point to pages in the complete manual.

**Files:**
- Modify: `eval/questions/workshop_manuals_v1.yaml`
- Optional helper commands using existing store JSON/OpenSearch data.

- [ ] For each failed required unit from the old source, search the new store text for the required claim terms.
- [ ] Update only required unit ids that moved.
- [ ] Keep `source-ferrari-360-wsm` stable.
- [ ] Run `uv run python -m pytest tests/integration/test_private_eval_assets.py -q` and any workshop eval model tests that load the YAML.

## Task 5: Run And Record Refreshed Baseline

**Why:** This creates the new current regression gate for the complete manual before artifact-first UI changes.

**Files:**
- Generated: `eval/runs/<new-run>/workshop_scorecard.yaml`
- Modify or create finding if the result needs a durable note under `docs/superpowers/findings/`.

- [ ] Run `make eval-workshop-baseline`.
- [ ] Review category summaries and failed-question detail.
- [ ] If the baseline reveals invalid evidence keys, return to Task 4.
- [ ] Commit source path, eval diagnostics, eval key updates, and Beads changes.

## Task 6: Artifact-First UI Slice

**Why:** Once the complete-manual baseline is coherent, the UI can expose the structured artifact path while preserving the currently working workshop-doc retrieval flow.

**Files:**
- Modify: `apps/web/app/page.tsx`
- Modify: `apps/web/app/sessionTypes.ts`
- Test/verify: `npm run lint` from `apps/web`

- [ ] Add a compact answer-mode control with current retrieval as the default and artifact-first as an explicit opt-in.
- [ ] Send `artifact_mode: "prefer"` only when artifact-first is selected.
- [ ] Render artifact claim reference and trust state when the answer includes `artifact_claim_ref`.
- [ ] Add artifact claim feedback controls for artifact-backed answers using the existing backend artifact feedback route.
- [ ] Preserve current fallback behavior when no artifact matches.
- [ ] Run `npm run lint` from `apps/web`.
- [ ] Commit the UI slice separately from the eval refresh.

## Verification

- [ ] `uv run python -m pytest tests/unit/test_workshop_catalog.py tests/unit/test_workshop_eval.py tests/integration/test_workshop_manuals_cli.py tests/integration/test_private_eval_assets.py -q`
- [ ] `make eval-workshop-baseline`
- [ ] `npm run lint` from `apps/web` after the UI task.
- [ ] `git status --short`
- [ ] Close `prescient_os-vtt` after the baseline refresh commit.
- [ ] Close `prescient_os-5ex` after the artifact UI commit.
