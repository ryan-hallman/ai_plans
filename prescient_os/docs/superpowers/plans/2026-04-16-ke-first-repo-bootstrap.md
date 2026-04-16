# KE-First Repo Bootstrap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Preserve the current code-repo direction on an archive branch, reset the active branch to a clean KE-first baseline, and rewrite repository agent context so future work follows the greenfield knowledge-engine direction.

**Architecture:** The transition is branch-first and context-first. First preserve the existing work-in-progress state so nothing is lost, then create a clean `ke-first-greenfield` branch from `main`, and only then rewrite repo guidance so agents and future implementation work inherit the new KE-first authority chain instead of the archived operator-first platform assumptions.

**Tech Stack:** Git, Markdown, FastAPI, Next.js, Docker, Postgres, OpenSearch

---

### File Structure

**Files:**
- Modify: `AGENTS.md`
- Reference: `docs/superpowers/specs/2026-04-16-ke-first-greenfield-foundation-design.md`
- Reference: `docs/superpowers/specs/2026-04-16-ke-first-repo-bootstrap-design.md`

The code repo stays intentionally minimal on the new branch. No KE module scaffolding should be introduced during this bootstrap.

### Task 1: Preserve Current Code-Repo State On An Archive Branch

**Files:**
- Modify: current git branch state only

- [ ] **Step 1: Inspect current code-repo branch and working tree**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os branch -a --verbose --no-abbrev
git -C /home/rhallman/Projects/prescient_os status --short
```

Expected:
- current branch is the in-progress KE governance/platform-heavy direction
- working tree contains tracked modifications and possibly untracked files

- [ ] **Step 2: Create or switch to the archive branch**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
git switch -c archive/operator-first-platform 2>/dev/null || git switch archive/operator-first-platform
```

Expected:
- branch switches successfully
- no worktree changes are lost

- [ ] **Step 3: Commit the preserved WIP state**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
git add -A
git commit -m "archive: preserve pre-pivot operator-first platform state"
```

Expected:
- commit succeeds
- the current direction is fully preserved on `archive/operator-first-platform`

- [ ] **Step 4: Verify the archive branch is clean**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os status --short
git -C /home/rhallman/Projects/prescient_os log --oneline --decorate -1
```

Expected:
- `git status --short` prints nothing except any intentionally ignored/untracked local machine files
- latest commit message is `archive: preserve pre-pivot operator-first platform state`

### Task 2: Create The Clean KE-First Code Branch

**Files:**
- Modify: current git branch state only

- [ ] **Step 1: Return local `main` to the clean remote baseline**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
git switch main
git reset --hard origin/main
```

Expected:
- local `main` matches `origin/main`
- the prior WIP state is no longer present on `main`

- [ ] **Step 2: Create the new KE-first branch**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
git switch -c ke-first-greenfield
```

Expected:
- new branch `ke-first-greenfield` is created from clean `main`

- [ ] **Step 3: Verify the new branch is clean and minimal**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os status --short
git -C /home/rhallman/Projects/prescient_os branch --show-current
git -C /home/rhallman/Projects/prescient_os rev-parse HEAD
```

Expected:
- current branch is `ke-first-greenfield`
- worktree is clean except local ignored/untracked machine files such as `.codex` or `.mcp.json`
- HEAD matches the clean `main` baseline before context edits

### Task 3: Rewrite `AGENTS.md` For KE-First Greenfield

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Write the new KE-first `AGENTS.md` content**

Replace the current repository-state and product-direction content with KE-first guidance that includes:

```md
# AGENTS.md

Working guide for AI coding agents in this repository.

## What this project is

**Prescient OS** is being rebuilt as an API-first business knowledge engine.

Current goal:
- ingest business inputs from docs/files, chat, email/Outlook, and public web/news
- extract observations
- model business entities
- curate durable primary artifacts
- update those artifacts over time
- retrieve current business context with provenance and visibility awareness

This repository is no longer operating under the previous operator-first platform buildout as its active direction.

## Active Design Authority

The active design authority is in the external docs repo on branch `ke-first-greenfield`:

- `docs/superpowers/specs/2026-04-16-ke-first-greenfield-foundation-design.md`
- `docs/superpowers/specs/2026-04-16-ke-first-repo-bootstrap-design.md`

Use archived specs only as reference material, not as the active roadmap.

## Reference Material

Historical work remains useful, but it is secondary:

- code reference branch: `archive/operator-first-platform`
- docs reference branch: `archive/operator-first-platform`

Only pull ideas forward from the archive when they clearly improve the KE-first design.

## Architecture Choices That Still Stand

- DDD with hexagonal layering
- modular monolith bias unless proven otherwise
- FastAPI for backend
- Next.js for frontend
- Docker-required local development
- Postgres as primary store
- OpenSearch as search engine
- API-first development
- generated API contracts
- grounded/provenance-aware outputs
- human approval for canonical publication
- tests required for domain and application logic

## Current Non-Goals

Do not treat these as active priorities during the greenfield rebuild:

- sponsor workflows
- claim/release flows
- onboarding UX as a primary product surface
- notification-heavy platform behavior
- the prior bounded-context map as authoritative
- the old operator-first platform roadmap
```

- [ ] **Step 2: Review `AGENTS.md` for stale assumptions**

Check that the rewritten file no longer describes:
- the previous platform as an active MVP buildout
- the old bounded contexts as the current target architecture
- the 2026-04-12 redesign spec as the active source of truth

Run:

```bash
sed -n '1,260p' /home/rhallman/Projects/prescient_os/AGENTS.md
```

Expected:
- the file is KE-first
- archive branches are described as reference only
- FastAPI, Next.js, Docker, Postgres, and OpenSearch remain explicit active choices

- [ ] **Step 3: Commit the context reset**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
git add AGENTS.md
git commit -m "docs: reset repo context for KE-first rebuild"
```

Expected:
- commit succeeds on `ke-first-greenfield`

### Task 4: Final Verification And Handoff

**Files:**
- Modify: none

- [ ] **Step 1: Verify branch topology**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os branch -a --verbose --no-abbrev
```

Expected:
- `archive/operator-first-platform` exists
- `main` is clean and aligned to remote baseline
- `ke-first-greenfield` exists and is checked out

- [ ] **Step 2: Verify the new branch contains only bootstrap-level changes**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os log --oneline --decorate --max-count=5
git -C /home/rhallman/Projects/prescient_os status --short
```

Expected:
- recent commits include the archive preservation commit and the `AGENTS.md` reset commit
- worktree is clean except local ignored/untracked machine files

- [ ] **Step 3: Record the new authority chain for implementation**

Confirm in the final handoff that:
- code work should proceed from branch `ke-first-greenfield`
- design authority lives on docs branch `ke-first-greenfield`
- archived code/docs branches are reference only

- [ ] **Step 4: Commit**

No additional commit is required if all previous commits succeeded.
