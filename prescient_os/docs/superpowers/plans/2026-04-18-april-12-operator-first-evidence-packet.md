# April 12 Operator-First Evidence Packet Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first raw-evidence phase packet for the April 12 operator-first pivot so later chronology artifacts can be grounded in primary sources instead of derived docs.

**Architecture:** Add a markdown evidence packet under `prescient_os_data/evidence_packets/` with a narrow, reviewable structure. Populate it from the strongest private Claude sessions and a small set of supporting commits. Do not revise existing memo artifacts in this slice.

**Tech Stack:** Markdown, private normalized chat sessions, git history from `prescient_os`

---

### Task 1: Create The Evidence Packet Directory And Draft Packet

**Files:**
- Create: `/home/rhallman/Projects/prescient_os_data/evidence_packets/2026-04-12-operator-first-pivot.md`
- Create: `/home/rhallman/Projects/prescient_os_data/evidence_packets/README.md`
- Reference: `/home/rhallman/Projects/prescient_os_data/normalized/sessions/anthropic__5d270129-0515-458d-b9cc-a2515ffab359.yaml`
- Reference: `/home/rhallman/Projects/prescient_os_data/normalized/sessions/anthropic__16789efa-e5ed-41ec-8a47-5070b73e653f.yaml`
- Reference: `/home/rhallman/Projects/prescient_os_data/normalized/sessions/anthropic__923e627c-7d70-4bd0-952b-c5f4e8ddeeb5.yaml`

- [ ] **Step 1: Create the packet directory and a brief README**

Create `evidence_packets/README.md` explaining that packets are upstream chronology-support documents used to decide whether existing artifacts are trustworthy.

Minimal content:

```md
# Evidence Packets

Evidence packets capture the primary sources behind a chronological phase before
we promote any derived artifact as canonical.

Use them to:
- collect the strongest chat sessions, commits, and existing docs for a phase
- explain what changed and why
- decide whether to use an existing artifact, write a reconstructed summary, or
  keep the phase as evidence-only for now
```

- [ ] **Step 2: Re-read the primary source sessions**

Run:

```bash
sed -n '1,220p' /home/rhallman/Projects/prescient_os_data/normalized/sessions/anthropic__5d270129-0515-458d-b9cc-a2515ffab359.yaml
sed -n '1,220p' /home/rhallman/Projects/prescient_os_data/normalized/sessions/anthropic__16789efa-e5ed-41ec-8a47-5070b73e653f.yaml
sed -n '180,280p' /home/rhallman/Projects/prescient_os_data/normalized/sessions/anthropic__923e627c-7d70-4bd0-952b-c5f4e8ddeeb5.yaml
```

Expected: the packet author can identify the PE-first framing, the operator-first rationale, and the later corroborating summary.

- [ ] **Step 3: Collect the supporting commit anchors**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os log --date=short --pretty=format:'%ad %h %s' feature/start-demo-build-foundation -- AGENTS.md | sed -n '1,10p'
git -C /home/rhallman/Projects/prescient_os log --reverse --date=short --pretty=format:'%ad %h %s' feature/start-demo-build-foundation | sed -n '1,12p'
```

Expected: concrete commit anchors around the April 10-12 build/focus shift.

- [ ] **Step 4: Write the phase evidence packet**

Create `2026-04-12-operator-first-pivot.md` with these sections:

```md
# April 12 Operator-First Pivot

## Phase Identity
- phase_id: 2026-04-12-operator-first-pivot
- title: April 12 Operator-First Pivot
- date_window: 2026-04-12 to 2026-04-13
- status: draft

## Primary Sources
- chat_session: anthropic__5d270129-0515-458d-b9cc-a2515ffab359
  - why it matters: strongest articulation of the operator-first thesis and the
    board-prep / monthly-reporting pain
- chat_session: anthropic__16789efa-e5ed-41ec-8a47-5070b73e653f
  - why it matters: captures the earlier PE-first / portfolio-intelligence
    framing still present at the edge of the pivot

## Supporting Sources
- chat_session: anthropic__923e627c-7d70-4bd0-952b-c5f4e8ddeeb5
- git_commit: 7e61ca6
- git_commit: e27a4ef
- existing_doc: feature/start-demo-build-foundation:AGENTS.md

## What Changed
[Write a concise phase summary]

## Why It Changed
[Write the rationale recovered from chats]

## Candidate Artifact Decision
- decision: create reconstructed summary
- reason: existing README is too implementation-heavy and existing overview docs
  are later than the pivot; the strongest rationale lives in chat history
```

Use short quoted excerpts where helpful, but keep the packet concise and source-oriented.

- [ ] **Step 5: Inspect the packet**

Run:

```bash
sed -n '1,260p' /home/rhallman/Projects/prescient_os_data/evidence_packets/2026-04-12-operator-first-pivot.md
```

Expected: the packet clearly distinguishes primary sources, supporting sources, what changed, and why.

- [ ] **Step 6: Commit the evidence packet**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os_data add evidence_packets
git -C /home/rhallman/Projects/prescient_os_data commit -m "data: add april 12 operator-first evidence packet"
```

Expected: one commit in the private data repo containing the first packet.

### Task 2: Verify The Packet Is The New Chronology Anchor

**Files:**
- Verify: `/home/rhallman/Projects/prescient_os_data/evidence_packets/2026-04-12-operator-first-pivot.md`

- [ ] **Step 1: Re-read the final packet and confirm the decision**

Run:

```bash
sed -n '1,260p' /home/rhallman/Projects/prescient_os_data/evidence_packets/2026-04-12-operator-first-pivot.md
```

Expected: the packet ends with a clear recommendation that the phase needs a reconstructed summary rather than relying on the current README-derived artifact.

- [ ] **Step 2: Verify clean repo state**

Run:

```bash
git -C /home/rhallman/Projects/prescient_os_data status --short
git -C /home/rhallman/Projects/prescient_os_data log --oneline -n 3
```

Expected: clean status and the new packet commit visible at HEAD.
