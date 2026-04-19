# Start Demo Build Foundation Artifact Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Normalize the early `feature/start-demo-build-foundation` README into the private corpus as artifact `#2` without rewriting it into a retrospective memo.

**Architecture:** Keep the current private-corpus schema unchanged. Copy the early README body into a memo-compatible markdown artifact under `prescient_os_data/memos/`, then cut a new self-contained snapshot so the loader can ingest both artifact `#1` and artifact `#2` together.

**Tech Stack:** Markdown with YAML frontmatter, git history as source material, existing private corpus loader, existing snapshot-cut CLI wrappers

---

### Task 1: Create The Build-Phase Artifact File

**Files:**
- Create: `/home/rhallman/Projects/prescient_os_data/memos/start-demo-build-foundation-readme-2026-04-10.md`
- Reference: `feature/start-demo-build-foundation:README.md`

- [ ] **Step 1: Read the source README**

Run:

```bash
git show feature/start-demo-build-foundation:README.md | sed -n '1,240p'
```

Expected: the early build-phase README is visible.

- [ ] **Step 2: Create the memo-compatible artifact**

Write a markdown file with this frontmatter:

```yaml
---
memo_id: start-demo-build-foundation-readme-2026-04-10
title: Prescient OS
as_of: 2026-04-10
topic_bucket: Product Strategy
source_refs: []
---
```

Then preserve the README body substantially as written, including product framing, demo-build context, architecture summary, invariants, and workflow/navigation pointers.

- [ ] **Step 3: Inspect the created file**

Run:

```bash
sed -n '1,260p' "/home/rhallman/Projects/prescient_os_data/memos/start-demo-build-foundation-readme-2026-04-10.md"
```

Expected: valid frontmatter followed by the README body.

- [ ] **Step 4: Commit the data-repo artifact**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os_data add memos/start-demo-build-foundation-readme-2026-04-10.md
git -C /home/rhallman/Projects/prescient_os_data commit -m "data: add start-demo-build artifact"
```

Expected: one new commit in the data repo for the artifact file.

### Task 2: Cut A Snapshot That Includes Both Artifacts

**Files:**
- Create via CLI: `/home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-first-two-artifacts/manifest.yaml`
- Create via CLI: `/home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-first-two-artifacts/memos/ai-enablement-strategy-pe-portfolios-2026-04-10.md`
- Create via CLI: `/home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-first-two-artifacts/memos/start-demo-build-foundation-readme-2026-04-10.md`

- [ ] **Step 1: Cut the new snapshot**

Run:

```bash
cd /home/rhallman/Projects/prescient_os_data
./scripts/cut_private_chat_snapshot.sh prescient-os-chats-2026-04-18-with-first-two-artifacts
```

Expected: a new snapshot directory is created that contains both memo files.

- [ ] **Step 2: Verify both memos are present in the snapshot**

Run:

```bash
find /home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-first-two-artifacts/memos -maxdepth 1 -type f | sort
```

Expected:

```text
.../ai-enablement-strategy-pe-portfolios-2026-04-10.md
.../start-demo-build-foundation-readme-2026-04-10.md
```

- [ ] **Step 3: Validate the snapshot with the CLI**

Run:

```bash
PYTHONPATH=/home/rhallman/Projects/prescient_os/apps/api/src \
uv run python -m prescient_benchmark.cli inspect-private-corpus \
  --source-root /home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-first-two-artifacts
```

Expected output includes:

```text
snapshot_id: prescient-os-chats-2026-04-18-with-first-two-artifacts
memos: 2
```

- [ ] **Step 4: Commit the updated snapshot state**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os_data add snapshots/prescient-os-chats-2026-04-18-with-first-two-artifacts
git -C /home/rhallman/Projects/prescient_os_data commit -m "data: cut snapshot with first two artifacts"
```

Expected: one new commit in the data repo for the snapshot.
