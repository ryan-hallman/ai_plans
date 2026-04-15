# Branch Reconciliation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reset local branch to the remote (which has deeper KE work + cleaned history), set up the docs symlink, and clean up stale untracked files.

**Architecture:** The remote branch already has the slug-to-UUID migration ~95% complete. Phase 1 resets to remote. Phase 2 audits remaining slug references and fills gaps. Phase 3 (OpenAPI pipeline) is deferred to a separate plan.

**Tech Stack:** Git, bash, symlinks

**Spec:** `docs/superpowers/specs/2026-04-15-branch-reconciliation-design.md`

---

### Task 1: Create backup branch

**Files:**
- None (git-only operation)

- [ ] **Step 1: Create backup branch from current HEAD**

```bash
git branch backup/local-pre-reconciliation-2026-04-15
```

- [ ] **Step 2: Verify backup branch exists**

Run: `git log --oneline backup/local-pre-reconciliation-2026-04-15 -3`

Expected: Shows the same 3 most recent commits as current HEAD:
```
5287c7a docs: update KE plan for slug-to-UUID migration, update spec index naming
6fbaddb fix(auth): add primary_company_id to login TokenResponse
0cbef1a fix(migration): use correct index name ix_board_prep_board_decks_company
```

- [ ] **Step 3: Commit — N/A (no file changes)**

---

### Task 2: Delete stale untracked files

**Files:**
- Delete: `docs/superpowers/specs/2026-04-14-architecture-scale-review.md`
- Delete: `sweep/` (entire directory)

- [ ] **Step 1: Delete the stale spec file**

```bash
rm docs/superpowers/specs/2026-04-14-architecture-scale-review.md
```

- [ ] **Step 2: Delete the sweep screenshots directory**

```bash
rm -rf sweep/
```

- [ ] **Step 3: Verify deletions**

Run: `ls docs/superpowers/specs/2026-04-14-architecture-scale-review.md sweep/ 2>&1`

Expected: Both paths report "No such file or directory"

---

### Task 3: Hard reset to remote branch

**Files:**
- None (git-only operation, replaces entire working tree)

- [ ] **Step 1: Fetch latest remote state**

```bash
git fetch origin
```

- [ ] **Step 2: Hard reset to remote branch tip**

```bash
git reset --hard origin/feature/operator-first-demo-build-foundation
```

- [ ] **Step 3: Verify HEAD matches remote**

Run: `git log --oneline -3`

Expected: Shows the remote branch's most recent commits:
```
e90262f fix(web): pass extraBody through streamChat properly
16f33f1 fix(research): move /sources route before /{task_id} to fix route matching
198249f Merge branch 'feature/research-onboarding-phase2' into feature/research-onboarding-frontend
```

- [ ] **Step 4: Verify clean working tree**

Run: `git status --short`

Expected: No modified or staged files. May show untracked files for `docs/superpowers` if the directory existed before (now gitignored).

---

### Task 4: Create docs/superpowers symlink

**Files:**
- Create: `docs/superpowers` (symlink → `../../ai_plans/prescient_os/docs/superpowers/`)

Note: The remote `.gitignore` already contains `docs/superpowers`, so the symlink will not be tracked by git.

- [ ] **Step 1: Ensure the docs directory exists**

```bash
mkdir -p docs
```

- [ ] **Step 2: Remove any leftover docs/superpowers directory or broken symlink**

```bash
rm -rf docs/superpowers
```

- [ ] **Step 3: Create the symlink**

The symlink must be relative from `docs/` to `../../ai_plans/prescient_os/docs/superpowers/`:

```bash
ln -s ../../ai_plans/prescient_os/docs/superpowers docs/superpowers
```

- [ ] **Step 4: Verify symlink resolves correctly**

Run: `ls -la docs/superpowers && ls docs/superpowers/specs/ | head -5 && ls docs/superpowers/plans/ | head -5`

Expected: Symlink points to `../../ai_plans/prescient_os/docs/superpowers`, and listing shows spec/plan files from the ai_plans repo.

- [ ] **Step 5: Verify git ignores the symlink**

Run: `git status --short docs/`

Expected: No output (docs/superpowers is gitignored)

---

### Task 5: Verify Docker dev environment starts

**Files:**
- None (verification only)

- [ ] **Step 1: Build and start the Docker dev environment**

```bash
cd infrastructure && docker compose up -d --build
```

- [ ] **Step 2: Verify API container is healthy**

Run: `docker compose ps` (from `infrastructure/`)

Expected: API and web containers show "running" or "healthy" status.

- [ ] **Step 3: Verify web app is accessible**

Run: `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000`

Expected: `200` or `302` (redirect to login)

- [ ] **Step 4: Verify API is accessible**

Run: `curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/health`

Expected: `200`

---

### Task 6: Audit remaining slug references for Phase 2 readiness

This is a read-only audit task to document what slug references remain on the remote codebase and classify them as intentional vs. migration gaps. No code changes — output feeds the Phase 2 plan.

**Files:**
- None (audit only)

- [ ] **Step 1: Catalog backend slug references**

Run:
```bash
grep -rn "company_slug\|\.slug\b" apps/api/src/prescient/ \
  --include="*.py" \
  | grep -v "__pycache__" \
  | grep -v "test" \
  | grep -v "alembic"
```

Classify each hit as:
- **Intentional:** slug as display field in response models, slug for OpenSearch index naming, slug column in ORM tables/entities, slug-based DB lookups in auth (resolving slug→UUID at login)
- **Migration gap:** slug used as a primary identifier in path params, slug used where UUID should be, slug in config keys that should be UUID-keyed

- [ ] **Step 2: Catalog frontend slug references**

Run:
```bash
grep -rn "slug\|companySlug" apps/web/src/ \
  --include="*.ts" --include="*.tsx" \
  | grep -v "node_modules" \
  | grep -v ".generated.ts"
```

Classify each hit as:
- **Intentional:** slug as display text, slug field in API response types (matches backend response models)
- **Migration gap:** slug used for routing, slug used in URL construction, slug used where companyId should be

- [ ] **Step 3: Document findings**

Write a brief summary of:
1. How many intentional slug references exist (expected, no action needed)
2. How many migration gaps exist (to be addressed in Phase 2)
3. Specific files and line numbers for each gap

This output determines whether Phase 2 is needed or if the remote is already complete.

---

## Phase 2: Slug-to-UUID Gap Fill (Conditional)

If Task 6 identifies migration gaps, a follow-up plan will be written targeting those specific files. Based on preliminary analysis, likely gaps include:

- `apps/api/src/prescient/intelligence/api/routes.py` — `StrategicFocusResponse.company_slug` field and response construction
- `apps/api/src/prescient/sponsor/api/routes.py` — response model `company_slug` fields
- `apps/api/src/prescient/board_prep/application/assembler.py` — `COMPANY_KPI_CONFIG` keyed by slug (has explicit TODOs)
- `apps/web/src/types/api.generated.ts` — stale generated types with `{slug}` paths (regenerated from OpenAPI, Phase 3)
- `apps/web/src/lib/api.ts` — response type interfaces with slug fields

## Phase 3: OpenAPI Pipeline (Deferred)

Separate plan. Adds MCP server, TypeScript type generation, CI freshness check.
