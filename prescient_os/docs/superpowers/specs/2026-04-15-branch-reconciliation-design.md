# Branch Reconciliation: Remote-First Reset + Slug-to-UUID Re-application

**Date:** 2026-04-15
**Status:** Approved
**Context:** Two machines diverged after a force-push history rewrite (superpowers docs removal). 329 shared commits exist on both sides with different SHAs. Each side has ~45-55 unique commits.

## Problem

The `feature/operator-first-demo-build-foundation` branch diverged between two machines:

- **Remote (new dev machine):** Knowledge Engine deep implementation, research engine, onboarding interview, shared chat components, history rewrite removing docs from repo
- **Local (this machine):** Slug-to-UUID migration, OpenAPI pipeline, design specs/plans (now in external repo)

A standard merge produces 65+ file conflicts across knowledge_mgmt, research, onboarding, intelligence, and web — most are add/add conflicts from the history divergence. Resolving them individually is error-prone and unverifiable.

## Decision

Reset local to remote, then re-apply the slug-to-UUID migration fresh against the current codebase. OpenAPI pipeline re-application deferred to a later phase.

## Phase 1: Reset to Remote

1. Create backup branch `backup/local-pre-reconciliation-2026-04-15` from current HEAD
2. Delete local untracked files:
   - `docs/superpowers/specs/2026-04-14-architecture-scale-review.md`
   - `sweep/` directory (demo screenshots, no longer needed)
3. Hard reset to `origin/feature/operator-first-demo-build-foundation`
4. Create symlink: `docs/superpowers` → `../ai_plans/prescient_os/docs/superpowers/`
5. Verify clean `git status` (symlink will show as untracked if not gitignored)

## Phase 2: Slug-to-UUID Migration (Re-applied)

The original migration was a mechanical refactor across 6 layers. Re-apply the same pattern to the remote codebase, which includes new modules (research, onboarding interview, KE deep implementation) that also need UUID treatment.

### Layer 1: Backend Auth Foundation
- `CompanyIdPath` validator in `prescient/shared/validators.py` — annotated UUID type for FastAPI path params
- `primary_company_id` added to `CurrentUser` in `prescient/auth/base.py`
- `primary_company_id` in JWT encode/decode (`auth/infrastructure/jwt.py`)
- `primary_company_id` in login use case and `TokenResponse` (`auth/application/use_cases/login.py`, `auth/api/routes.py`)
- `primary_company_id` in demo middleware (`auth/demo.py`)

### Layer 2: Backend Route Params
Replace `company_slug: str` path params with `company_id: CompanyIdPath` in all route modules:
- `companies/api/routes.py`
- `intelligence/api/routes.py`
- `sponsor/api/routes.py`
- `board_prep/api/routes.py`
- `research/api/routes.py` (new on remote)
- `onboarding/api/routes.py` (new on remote)
- `knowledge_mgmt/api/routes.py` (refactored on remote)
- `knowledge_mgmt/api/department_routes.py` (refactored on remote)

### Layer 3: Copilot/Intelligence Internals
Replace slug-based company identity with UUID in:
- `intelligence/infrastructure/tools/base.py` — ToolContext uses UUID
- `intelligence/infrastructure/tools/registry.py`
- `intelligence/api/copilot_stream.py` — already partially UUID on remote, complete the migration
- `intelligence/application/planner_stream.py` — same
- `intelligence/application/planner.py`
- `intelligence/prompts/__init__.py` and `system.md`

### Layer 4: Frontend Routing
- Rename `[slug]` route directories to `[companyId]` under `apps/web/src/app/(main)/companies/` and `apps/web/src/app/(sponsor)/portfolio/companies/`
- Update all `params.slug` references to `params.companyId`

### Layer 5: Frontend Auth & Components
- `primaryCompanyId` in auth storage (`lib/auth.ts`), provider (`components/auth-provider.tsx`), login page
- Replace `companySlug` with `companyId` in all URL builder functions and component links
- Update copilot store, chat-stream, deck-store to use UUID

### Layer 6: Board-Prep Migration
- Alembic migration: rename `company_slug` column to `company_id` (UUID) on `board_prep_board_decks`
- Update board-prep domain entity, ORM table, repository, assembler, routes

## Phase 3: OpenAPI Pipeline (Deferred)

Separate effort — adds MCP server, TypeScript type generation, CI freshness check. Standalone new files, minimal conflict risk. Will be re-applied after Phase 2 stabilizes.

## Verification

After each phase:
- `git status` clean (no unexpected changes)
- Docker dev environment starts (`docker compose up`)
- API tests pass
- Web app builds and loads

## Rollback

Backup branch `backup/local-pre-reconciliation-2026-04-15` preserves the full local state. If reconciliation fails, `git checkout` that branch to return to the pre-reset state.
