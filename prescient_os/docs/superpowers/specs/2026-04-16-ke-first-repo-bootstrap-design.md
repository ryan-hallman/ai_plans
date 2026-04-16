# 2026-04-16 KE-First Repo Bootstrap Design

## Goal

Reset the code repository to a clean greenfield starting point for the KE-first rebuild without losing prior work or confusing future agents.

This bootstrap is intentionally minimal. Its purpose is to:

- preserve the current in-progress platform-heavy direction on an archive branch
- create a fresh code branch for the KE-first rebuild
- rewrite repository context so agents follow the new direction
- keep only the foundational architecture choices that still matter

## Repo Transition

The code repository should be transitioned in three steps:

1. Preserve the current in-progress state on an archive branch.
2. Return local `main` to a clean baseline.
3. Create a fresh `ke-first-greenfield` code branch from clean `main`.

The new branch should not inherit partial implementation from the archived direction. It should start from a clean repository baseline with updated context only.

## New Branch Contents

The fresh `ke-first-greenfield` code branch should contain only:

- the existing dev scaffolding already present on `main`
- a rewritten `AGENTS.md`
- minimal context pointers if needed

It should not contain:

- copied-over product code from the archived branch
- speculative KE package skeletons
- new routes, migrations, or placeholder modules
- stale product/platform assumptions from the prior direction

## Context Hygiene

The new `AGENTS.md` should clearly establish:

- the repo is now being rebuilt around an API-first business knowledge engine
- active design authority lives in the docs repo on branch `ke-first-greenfield`
- archived code and docs branches are reference material only
- future implementation should optimize for KE quality, not platform breadth

It should explicitly tell agents not to treat prior sponsor-heavy, onboarding-heavy, or workflow-heavy specs as the active direction unless intentionally consulted as reference.

## Architecture Choices To Carry Forward

The new branch should preserve only the architectural choices that still support the KE-first rebuild:

- DDD with hexagonal layering
- modular monolith bias unless proven otherwise
- FastAPI for backend
- Next.js for frontend
- Docker-required development
- Postgres as primary store
- OpenSearch as search engine
- API-first development
- generated API contracts
- grounded/provenance-aware outputs
- human approval for canonical publication
- tests required for domain and application logic

These choices remain active technical constraints. They are no longer attached to the previous operator-first platform scope.

## Explicit Non-Goals

This bootstrap should not:

- preserve the old bounded-context map as if it were still authoritative
- preserve the old platform roadmap as the active product direction
- imply that sponsor workflows or onboarding UX are current priorities
- introduce new implementation structure before the greenfield plan is written

## Next Step

After the bootstrap branch and context reset are complete, write the KE-first implementation plan and only then begin rebuilding the codebase.
