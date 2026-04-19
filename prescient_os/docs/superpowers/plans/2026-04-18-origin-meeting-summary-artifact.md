# Origin Meeting Summary Artifact Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Normalize the existing April 10 meeting summary into the private corpus as artifact `#1` without rewriting it into a synthetic memo.

**Architecture:** Keep the current private-corpus schema unchanged. Add one manual memo-compatible markdown file under `prescient_os_data/memos/`, sourced from the existing summary markdown, then cut a new self-contained snapshot so the loader can ingest it directly.

**Tech Stack:** Markdown with YAML frontmatter, existing private corpus loader, existing snapshot-cut CLI wrappers

---

### Task 1: Create The Manual Artifact File

**Files:**
- Create: `/home/rhallman/Projects/prescient_os_data/memos/ai-enablement-strategy-pe-portfolios-2026-04-10.md`
- Reference: `/home/rhallman/AI Enablement Strategy for PE Firms and Portfolios.md`
- Reference: `/home/rhallman/Projects/prescient_os_data/templates/manual_point_in_time_memo.md`

- [ ] **Step 1: Read the source summary and template**

Run:

```bash
sed -n '1,220p' "/home/rhallman/AI Enablement Strategy for PE Firms and Portfolios.md"
sed -n '1,120p' "/home/rhallman/Projects/prescient_os_data/templates/manual_point_in_time_memo.md"
```

Expected: the meeting summary body and the required memo frontmatter shape are visible.

- [ ] **Step 2: Create the memo-compatible artifact**

Write a markdown file with this frontmatter:

```yaml
---
memo_id: ai-enablement-strategy-pe-portfolios-2026-04-10
title: AI Enablement Strategy for PE Firms and Portfolios
as_of: 2026-04-10
topic_bucket: Product Strategy
source_refs: []
---
```

Then preserve the existing summary body substantially as-is, using the original section headings and wording unless a clearly post-April-10 hindsight edit must be removed.

- [ ] **Step 3: Inspect the created file**

Run:

```bash
sed -n '1,240p' "/home/rhallman/Projects/prescient_os_data/memos/ai-enablement-strategy-pe-portfolios-2026-04-10.md"
```

Expected: valid frontmatter followed by the meeting-summary body.

- [ ] **Step 4: Commit the data-repo artifact**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os_data add memos/ai-enablement-strategy-pe-portfolios-2026-04-10.md
git -C /home/rhallman/Projects/prescient_os_data commit -m "data: add origin meeting summary artifact"
```

Expected: one new commit in the data repo for the artifact file.

### Task 2: Cut A Snapshot That Includes The Artifact

**Files:**
- Modify indirectly via script: `/home/rhallman/Projects/prescient_os_data/scripts/cut_private_chat_snapshot.sh`
- Create via CLI: `/home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-origin-summary/manifest.yaml`
- Create via CLI: `/home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-origin-summary/memos/ai-enablement-strategy-pe-portfolios-2026-04-10.md`

- [ ] **Step 1: Cut the new snapshot**

Run:

```bash
cd /home/rhallman/Projects/prescient_os_data
./scripts/cut_private_chat_snapshot.sh prescient-os-chats-2026-04-18-with-origin-summary
```

Expected: a new snapshot directory is created that contains the copied memo file.

- [ ] **Step 2: Verify the memo is present in the snapshot**

Run:

```bash
find /home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-origin-summary -maxdepth 3 | sort | sed -n '1,200p'
```

Expected: `manifest.yaml`, `normalized/sessions/`, and `memos/ai-enablement-strategy-pe-portfolios-2026-04-10.md` are present.

- [ ] **Step 3: Validate the snapshot with the CLI**

Run:

```bash
PYTHONPATH=/home/rhallman/Projects/prescient_os/apps/api/src \
uv run python -m prescient_benchmark.cli inspect-private-corpus \
  --source-root /home/rhallman/Projects/prescient_os_data/snapshots/prescient-os-chats-2026-04-18-with-origin-summary
```

Expected output includes:

```text
snapshot_id: prescient-os-chats-2026-04-18-with-origin-summary
memos: 1
```

- [ ] **Step 4: Commit the updated snapshot state**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os_data add snapshots/prescient-os-chats-2026-04-18-with-origin-summary
git -C /home/rhallman/Projects/prescient_os_data commit -m "data: cut snapshot with origin meeting artifact"
```

Expected: one new commit in the data repo for the snapshot.
