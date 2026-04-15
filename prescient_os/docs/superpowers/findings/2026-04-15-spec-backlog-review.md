# Spec Backlog Review

**Date:** 2026-04-15
**Scope reviewed:** `docs/superpowers/` plans/specs in `ai_plans` plus current `prescient_os` implementation status

## Summary

Most demo-track work is materially present in the codebase: foundation, intelligence/onboarding, operating core, strategy, triage/sponsor portal, board prep, knowledge management, generated API types, and enriched seed data all have real API and/or web surface area.

The remaining backlog is concentrated in follow-on workflow completion, knowledge-engine hardening, and idea-stage copilot/product expansions.

## Remaining Work

### 1. Brainstorming mode is still an idea, not an implementation

The brainstorming mode write-up exists in the old idea backlog, but the current codebase does not appear to include:

- a brainstorm-specific planner
- brainstorm prompts
- brainstorm domain models
- brainstorm mode routing / response types
- brainstorm artifact generation flow

This is still open work if it remains desired for MVP or post-demo.

### 2. User-authored skills is future work only

The user-authored skills concept is explicitly future-facing and was not scheduled for implementation. There is no corresponding runtime/catalog/review-queue implementation in the product codebase.

This should be treated as post-demo / future product expansion unless reprioritized.

### 3. Knowledge engine follow-on work remains partially open

The core knowledge-management system is implemented, but the deeper knowledge-engine spec still has meaningful unfinished slices:

- upload path still uses local file storage while the ingestion worker uses S3-backed storage
- no evident malware scan service abstraction is wired through the ingestion flow
- no evident prompt-injection detection layer in the KM pipeline
- sponsor-share retrieval plane / sponsor-visible projection work does not appear complete
- span-level / chunk-granular citation model changes do not appear complete
- trust-aware ranking and deeper retrieval-plane unification appear incomplete
- processing-profile registry and advanced staged processing are still follow-on work

This is the largest remaining spec-driven technical backlog area.

### 4. Strategic initiatives proactive workflow is only partially wired

The strategy context is implemented and usable, but the more ambitious proactive-agent behavior is not fully landed:

- cadence engine logic exists, but it does not appear scheduled/running as a background task
- no strategy task package appears wired into Celery autodiscovery / beat
- the copilot milestone-update tool still describes the approval-request flow as pending
- the full assignee follow-up -> owner approval -> downstream propagation loop appears incomplete

The base strategy module is present; the autonomous follow-up layer remains open.

### 5. KPI live-data completion is still incomplete

KPI reporting exists, along with CSV intake endpoints and templates, but the end-state live-data pipeline is not complete:

- overdue KPI cadence checking is still a stub
- the main KPI UI still appears centered on manual reporting, not the richer intake/template workflow
- webhook / connector stubs and broader ingestion operationalization remain follow-on work

### 6. OpenAPI pipeline is mostly present, but local MCP wiring is incomplete

The export script, committed `openapi.json`, generated frontend types, MCP server code, and CI freshness check are present.

What still appears incomplete is local repo-level MCP registration for the OpenAPI server. Current local MCP config only wires Chrome DevTools, so the intended `search_api` / `get_endpoint` / `get_model` workflow is not yet fully wired into day-to-day local usage.

### 7. Cross-context cleanup debt remains

There are still explicit TODOs and temporary boundary violations in the implementation, especially around:

- briefing querying across contexts
- sponsor routes querying triage tables directly
- board-prep assembler querying across documents / intelligence / knowledge tables directly

These do not necessarily block the demo, but they are still open architectural cleanup from the plans/specs.

## Priority Cut

### Demo-critical / near-term

- knowledge-engine hardening gaps
- strategy cadence / approval workflow completion
- KPI overdue workflow and intake UX completion
- cross-context cleanup in the most load-bearing routes

### Post-demo / likely future

- brainstorming mode
- user-authored skills platform
- advanced sponsor-share retrieval plane and deep citation UX beyond current demo needs

## Notes

- This review is an implementation-status snapshot, not a plan rewrite.
- The old `.ai/ideas` material that still matters should be migrated into `ai_plans` explicitly rather than left as the source of truth in the product repo.
