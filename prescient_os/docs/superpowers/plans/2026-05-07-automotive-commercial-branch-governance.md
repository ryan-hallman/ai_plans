# Automotive Commercial Branch Governance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish the automotive commercial branch charter and update agent guidance so future work keeps the automotive product focused over a generic KE core.

**Architecture:** This is a documentation and project-memory change. The durable spec defines product scope and architecture boundaries; root agent guidance and Beads memory make the rule available during everyday work.

**Tech Stack:** Markdown docs in the docs repo, root `AGENTS.md`, and Beads memory.

---

### Task 1: Add Governance Spec

**Files:**
- Create: `docs/superpowers/specs/2026-05-07-automotive-commercial-branch-governance-design.md`

- [ ] **Step 1: Write the spec**

Add a spec that defines the active branch as the automotive commercial product track, keeps KE core contracts generic, and records the feature admission test.

- [ ] **Step 2: Review the spec**

Run:

```bash
sed -n '1,260p' docs/superpowers/specs/2026-05-07-automotive-commercial-branch-governance-design.md
```

Expected: the spec contains no placeholders, no contradiction between automotive product focus and generic core boundaries, and no implementation work beyond governance.

### Task 2: Update Root Agent Guidance

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Update project direction**

Change the opening direction from horizontal business KE only to automotive commercial product over a generic KE core.

- [ ] **Step 2: Add a feature admission test**

Add branch rules requiring each feature to improve the automotive product, a KE primitive used by it, or reliability/provenance/eval/deployment/reviewability for those areas.

- [ ] **Step 3: Review the guidance**

Run:

```bash
sed -n '1,180p' AGENTS.md
```

Expected: root guidance points to the governance spec and does not instruct agents to optimize for unrelated workflow breadth.

### Task 3: Update Persistent Beads Memory

**Files:**
- Modify through Beads: `.beads/issues.jsonl`

- [ ] **Step 1: Replace stale product-scope memory**

Run:

```bash
bd remember --key prescient-os-product-scope-horizontal-knowledge-memory-engin "Prescient OS is currently commercializing an automotive workshop knowledge product. Build the product surface for automotive users, while keeping the KE core domain-neutral so later vertical expansion does not require a major refactor."
```

Expected: the memory no longer describes the active product as horizontal-only.

### Task 4: Verify And Commit

**Files:**
- Product repo: `AGENTS.md`, `.beads/issues.jsonl`
- Docs repo: `docs/superpowers/specs/2026-05-07-automotive-commercial-branch-governance-design.md`, `docs/superpowers/plans/2026-05-07-automotive-commercial-branch-governance.md`

- [ ] **Step 1: Verify content**

Run:

```bash
rg -n "automotive commercial|Feature Admission|horizontal-only|prod/automotive|generic KE core|domain-neutral" AGENTS.md docs/superpowers/specs/2026-05-07-automotive-commercial-branch-governance-design.md docs/superpowers/plans/2026-05-07-automotive-commercial-branch-governance.md
```

Expected: the new branch scope and generic-core boundary are present in the updated docs.

- [ ] **Step 2: Check status**

Run:

```bash
git status --short --branch
git -C ../ai_plans status --short --branch
```

Expected: only the governance docs, root guide, and Beads files are changed.

- [ ] **Step 3: Commit docs repo**

Run from `../ai_plans`:

```bash
git add prescient_os/docs/superpowers/specs/2026-05-07-automotive-commercial-branch-governance-design.md prescient_os/docs/superpowers/plans/2026-05-07-automotive-commercial-branch-governance.md
git commit -m "docs: define automotive commercial branch governance"
```

- [ ] **Step 4: Commit product repo**

Run from the product repo:

```bash
git add AGENTS.md .beads/issues.jsonl
git commit -m "docs: align agents with automotive product scope"
```

