# Phase 5e: Sharing & Seed Data Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add operator sharing actions to existing pages, seed the demo with triage data (pre-existing Q&A history, triage queue items, shared artifacts), and verify the full end-to-end demo flow.

**Architecture:** "Share with Sponsor" buttons added to existing operator pages (artifact detail, board prep). New seed module creates demo triage data using the established deterministic UUID + upsert pattern. Integration test verifies the full loop.

**Tech Stack:** Python 3.12, SQLAlchemy 2.0, Next.js 15, React 19, pytest

**Depends on:** Phase 5a-d (all prior Phase 5 work)

---

## File Structure

### New files:

```
seed/triage.py                                           — seed triage demo data
apps/web/src/components/share-with-sponsor.tsx            — share button + dialog
```

### Modified files:

```
seed/loader.py                                           — call seed_triage()
apps/web/src/app/(main)/board-prep/page.tsx              — add share button after approve
apps/web/src/components/artifact-block.tsx (or artifact detail page) — add share button
```

---

### Task 1: Share With Sponsor Component

**Files:**
- Create: `apps/web/src/components/share-with-sponsor.tsx`

- [ ] **Step 1: Implement share dialog component**

```typescript
// apps/web/src/components/share-with-sponsor.tsx
"use client";

import { useState } from "react";
import { getToken } from "@/lib/auth";

const API = process.env.NEXT_PUBLIC_API_URL ?? "http://127.0.0.1:8000";

interface ShareWithSponsorProps {
  artifactId: string;
  triageQuestionId?: string;
  onShared?: () => void;
}

export function ShareWithSponsor({ artifactId, triageQuestionId, onShared }: ShareWithSponsorProps) {
  const [open, setOpen] = useState(false);
  const [sharing, setSharing] = useState(false);
  const [shared, setShared] = useState(false);

  const handleShare = async () => {
    setSharing(true);
    try {
      const token = getToken();
      const res = await fetch(`${API}/sharing/items`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          ...(token ? { Authorization: `Bearer ${token}` } : {}),
        },
        body: JSON.stringify({
          artifact_id: artifactId,
          sponsor_org_id: "org-summit-001", // demo: single sponsor
          triage_question_id: triageQuestionId ?? null,
        }),
      });
      if (res.ok) {
        setShared(true);
        setOpen(false);
        onShared?.();
      }
    } finally {
      setSharing(false);
    }
  };

  if (shared) {
    return (
      <span className="text-xs text-green-600 font-medium px-2 py-1">
        Shared with Summit Partners
      </span>
    );
  }

  return (
    <div className="relative">
      <button
        onClick={() => setOpen(!open)}
        className="px-3 py-1.5 text-sm rounded font-medium border border-neutral-300 text-neutral-700 hover:bg-neutral-50 transition-colors"
      >
        Share with Sponsor
      </button>

      {open && (
        <div className="absolute top-full mt-1 right-0 bg-white rounded-lg border border-neutral-200 shadow-lg p-3 z-10 w-64">
          <p className="text-sm font-medium text-neutral-800 mb-2">
            Share with Summit Partners?
          </p>
          <p className="text-xs text-neutral-500 mb-3">
            This will make the artifact visible to Michael Torres and other Summit Partners members.
          </p>
          <div className="flex gap-2 justify-end">
            <button
              onClick={() => setOpen(false)}
              className="px-2 py-1 text-xs text-neutral-500 hover:text-neutral-700"
            >
              Cancel
            </button>
            <button
              onClick={handleShare}
              disabled={sharing}
              className="px-3 py-1 text-xs rounded font-medium bg-blue-600 text-white hover:bg-blue-700 disabled:opacity-50"
            >
              {sharing ? "Sharing..." : "Share"}
            </button>
          </div>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/share-with-sponsor.tsx
git commit -m "feat(web): add ShareWithSponsor button component"
```

---

### Task 2: Add Share Button to Board Prep

**Files:**
- Modify: `apps/web/src/app/(main)/board-prep/page.tsx`

- [ ] **Step 1: Add share button to board prep page**

Open `apps/web/src/app/(main)/board-prep/page.tsx` and:

1. Import the component:
```typescript
import { ShareWithSponsor } from "@/components/share-with-sponsor";
```

2. Find the section after the "Approve" button (or where the deck status shows "approved"). Add the share button nearby:

```typescript
<ShareWithSponsor artifactId={deckId} />
```

Where `deckId` is the ID of the board deck artifact. If the board prep page doesn't track an artifact ID yet, use a placeholder string — the share action will still work for the demo since it creates a SharedItem record.

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/app/\(main\)/board-prep/page.tsx
git commit -m "feat(web): add 'Share with Sponsor' button to board prep page"
```

---

### Task 3: Seed Triage Demo Data

**Files:**
- Create: `seed/triage.py`

- [ ] **Step 1: Implement triage seed module**

```python
# seed/triage.py
"""Seed triage demo data.

Creates:
- 2 answered TriageQuestions (Q&A history)
- 1 escalated TriageQuestion (the live demo item)
- 5 TriageItems across types (sponsor questions, KPI anomaly, monitoring finding, action review)
- 3 SharedItems (board deck + 2 intelligence signals shared with Summit Partners)
"""

from __future__ import annotations

import logging
from datetime import UTC, datetime, timedelta
from uuid import UUID, uuid5

from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.triage.infrastructure.tables.shared_item import SharedItemTable
from prescient.triage.infrastructure.tables.triage_item import TriageItemTable
from prescient.triage.infrastructure.tables.triage_question import TriageQuestionTable

logger = logging.getLogger(__name__)

_TRIAGE_NS = UUID("e4f5a6b7-c8d9-4e0f-a1b2-c3d4e5f6a7b8")

now = datetime.now(tz=UTC)


def _uuid(key: str) -> UUID:
    return uuid5(_TRIAGE_NS, key)


# ---------------------------------------------------------------------------
# Demo questions
# ---------------------------------------------------------------------------

_QUESTIONS = [
    {
        "key": "q-revenue-mix",
        "asker_user_id": "michael-torres-001",
        "asker_org_id": "org-summit-001",
        "target_company_org_id": "org-peloton-001",
        "question_text": "How has the revenue mix shifted between hardware and subscriptions over the last year?",
        "categories_detected": ["kpi_metrics"],
        "status": "answered",
        "partial_answer": None,
        "final_answer": (
            "Subscription revenue now accounts for 42% of total revenue, up from 33% a year ago. "
            "Hardware revenue declined 18% YoY as the company shifted focus to content and connected fitness "
            "experiences. The subscription gross margin is 68% vs 5% for hardware, making this shift "
            "margin-accretive overall."
        ),
        "final_citations": [
            {"type": "kpi_value", "label": "Subscription Revenue %"},
            {"type": "kpi_value", "label": "Hardware Revenue"},
        ],
        "priority": "high",
        "escalation_reason": None,
        "created_at": now - timedelta(days=5),
        "answered_at": now - timedelta(days=4),
    },
    {
        "key": "q-competitor-response",
        "asker_user_id": "michael-torres-001",
        "asker_org_id": "org-summit-001",
        "target_company_org_id": "org-peloton-001",
        "question_text": "What's the competitive impact of Apple Fitness+ bundling with Apple One?",
        "categories_detected": ["intelligence_signals", "narrative"],
        "status": "answered",
        "partial_answer": (
            "Apple Fitness+ launched as part of the Apple One bundle at $14.95/month. "
            "Peloton's connected fitness subscriber base has seen increased churn in the "
            "Apple ecosystem demographic."
        ),
        "final_answer": (
            "Apple Fitness+ bundling is a real threat in the casual fitness segment, but Peloton's "
            "core connected fitness community (hardware owners) shows stronger retention. The team is "
            "responding with a tiered subscription model and exclusive content partnerships. Internal "
            "data shows hardware owners churn at 0.6% vs 2.1% for app-only subscribers."
        ),
        "final_citations": [
            {"type": "artifact", "label": "Apple Fitness+ Competitive Profile"},
            {"type": "kpi_value", "label": "Connected Fitness Churn Rate"},
        ],
        "priority": "high",
        "escalation_reason": "question touches gated categories: narrative",
        "created_at": now - timedelta(days=12),
        "answered_at": now - timedelta(days=11),
    },
    {
        "key": "q-churn-drivers",
        "asker_user_id": "michael-torres-001",
        "asker_org_id": "org-summit-001",
        "target_company_org_id": "org-peloton-001",
        "question_text": "What's driving the subscriber churn increase, and what's the team's plan to address it?",
        "categories_detected": ["kpi_metrics", "narrative", "decisions"],
        "status": "escalated",
        "partial_answer": (
            "Connected Fitness subscriber churn increased from 1.1% to 1.4% over the last two quarters. "
            "Total Connected Fitness subscribers declined by approximately 8% YoY to 2.88M."
        ),
        "final_answer": None,
        "final_citations": None,
        "priority": "high",
        "escalation_reason": "question also touches gated categories: narrative, decisions",
        "created_at": now - timedelta(hours=3),
        "answered_at": None,
    },
]

# ---------------------------------------------------------------------------
# Demo triage items
# ---------------------------------------------------------------------------

_TRIAGE_ITEMS = [
    {
        "key": "ti-q-revenue",
        "organization_id": "org-peloton-001",
        "item_type": "sponsor_question",
        "source_key": "q-revenue-mix",
        "title": "Revenue mix question from Summit Partners",
        "summary": "How has the revenue mix shifted between hardware and subscriptions?",
        "priority": "high",
        "priority_signals": ["from operating_partner"],
        "status": "resolved",
        "created_at": now - timedelta(days=5),
        "resolved_at": now - timedelta(days=4),
    },
    {
        "key": "ti-q-competitor",
        "organization_id": "org-peloton-001",
        "item_type": "sponsor_question",
        "source_key": "q-competitor-response",
        "title": "Competitive impact question from Summit Partners",
        "summary": "What's the competitive impact of Apple Fitness+ bundling?",
        "priority": "high",
        "priority_signals": ["from operating_partner"],
        "status": "resolved",
        "created_at": now - timedelta(days=12),
        "resolved_at": now - timedelta(days=11),
    },
    {
        "key": "ti-q-churn",
        "organization_id": "org-peloton-001",
        "item_type": "sponsor_question",
        "source_key": "q-churn-drivers",
        "title": "Subscriber churn question from Summit Partners",
        "summary": "What's driving churn and what's the plan? (escalated — touches narrative + decisions)",
        "priority": "high",
        "priority_signals": ["from operating_partner", "touches gated categories"],
        "status": "pending",
        "created_at": now - timedelta(hours=3),
        "resolved_at": None,
    },
    {
        "key": "ti-kpi-churn-spike",
        "organization_id": "org-peloton-001",
        "item_type": "kpi_anomaly",
        "source_key": None,
        "title": "Churn rate spike detected",
        "summary": "Connected Fitness churn increased 27% vs previous quarter (1.1% → 1.4%)",
        "priority": "high",
        "priority_signals": ["severity: critical", "KPI anomaly detection"],
        "status": "pending",
        "created_at": now - timedelta(hours=6),
        "resolved_at": None,
    },
    {
        "key": "ti-monitoring-sec",
        "organization_id": "org-peloton-001",
        "item_type": "monitoring_finding",
        "source_key": None,
        "title": "New SEC filing detected",
        "summary": "Peloton filed 10-Q for Q2 FY2025 with updated risk factors",
        "priority": "medium",
        "priority_signals": ["monitoring: SEC EDGAR"],
        "status": "pending",
        "created_at": now - timedelta(hours=12),
        "resolved_at": None,
    },
    {
        "key": "ti-action-overdue",
        "organization_id": "org-peloton-001",
        "item_type": "action_review",
        "source_key": None,
        "title": "Action item overdue: Finalize content partnership terms",
        "summary": "Due 3 days ago, assigned to VP Content. Priority: high",
        "priority": "medium",
        "priority_signals": ["3 days overdue", "assigned owner: VP Content"],
        "status": "pending",
        "created_at": now - timedelta(days=1),
        "resolved_at": None,
    },
]

# ---------------------------------------------------------------------------
# Demo shared items
# ---------------------------------------------------------------------------

_SHARED_ITEMS = [
    {
        "key": "shared-board-deck",
        "artifact_id": "board-deck-placeholder",  # would be real artifact ID in production
        "sponsor_org_id": "org-summit-001",
        "shared_by_user_id": "sarah-chen-001",
        "shared_at": now - timedelta(days=2),
        "triage_question_id": None,
    },
    {
        "key": "shared-apple-profile",
        "artifact_id": "apple-competitor-profile",
        "sponsor_org_id": "org-summit-001",
        "shared_by_user_id": "sarah-chen-001",
        "shared_at": now - timedelta(days=11),
        "triage_question_id_key": "q-competitor-response",
    },
    {
        "key": "shared-peloton-profile",
        "artifact_id": "peloton-company-profile",
        "sponsor_org_id": "org-summit-001",
        "shared_by_user_id": "sarah-chen-001",
        "shared_at": now - timedelta(days=7),
        "triage_question_id": None,
    },
]


async def seed_triage(session: AsyncSession) -> None:
    """Seed triage demo data — questions, items, shared artifacts."""

    # 1. Questions
    for q in _QUESTIONS:
        qid = _uuid(q["key"])
        stmt = pg_insert(TriageQuestionTable).values(
            id=qid,
            asker_user_id=q["asker_user_id"],
            asker_org_id=q["asker_org_id"],
            target_company_org_id=q["target_company_org_id"],
            question_text=q["question_text"],
            categories_detected=q["categories_detected"],
            status=q["status"],
            partial_answer=q.get("partial_answer"),
            partial_citations=None,
            escalation_reason=q.get("escalation_reason"),
            final_answer=q.get("final_answer"),
            final_citations=q.get("final_citations"),
            brainstorm_conversation_id=None,
            priority=q["priority"],
            created_at=q["created_at"],
            answered_at=q.get("answered_at"),
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[TriageQuestionTable.id],
            set_={
                "status": stmt.excluded.status,
                "final_answer": stmt.excluded.final_answer,
                "final_citations": stmt.excluded.final_citations,
                "answered_at": stmt.excluded.answered_at,
            },
        )
        await session.execute(stmt)
    logger.info("upserted %d triage questions", len(_QUESTIONS))

    # 2. Triage items
    for item in _TRIAGE_ITEMS:
        iid = _uuid(item["key"])
        source_id = str(_uuid(item["source_key"])) if item.get("source_key") else str(iid)
        stmt = pg_insert(TriageItemTable).values(
            id=iid,
            organization_id=item["organization_id"],
            item_type=item["item_type"],
            source_id=source_id,
            title=item["title"],
            summary=item["summary"],
            priority=item["priority"],
            priority_signals=item["priority_signals"],
            status=item["status"],
            assigned_to_user_id=None,
            created_at=item["created_at"],
            resolved_at=item.get("resolved_at"),
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[TriageItemTable.id],
            set_={
                "status": stmt.excluded.status,
                "resolved_at": stmt.excluded.resolved_at,
            },
        )
        await session.execute(stmt)
    logger.info("upserted %d triage items", len(_TRIAGE_ITEMS))

    # 3. Shared items
    for shared in _SHARED_ITEMS:
        sid = _uuid(shared["key"])
        triage_qid = None
        if shared.get("triage_question_id_key"):
            triage_qid = _uuid(shared["triage_question_id_key"])
        elif shared.get("triage_question_id"):
            triage_qid = shared["triage_question_id"]

        stmt = pg_insert(SharedItemTable).values(
            id=sid,
            artifact_id=shared["artifact_id"],
            sponsor_org_id=shared["sponsor_org_id"],
            shared_by_user_id=shared["shared_by_user_id"],
            shared_at=shared["shared_at"],
            triage_question_id=triage_qid,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[SharedItemTable.id],
            set_={
                "shared_at": stmt.excluded.shared_at,
            },
        )
        await session.execute(stmt)
    logger.info("upserted %d shared items", len(_SHARED_ITEMS))
```

- [ ] **Step 2: Commit**

```bash
git add seed/triage.py
git commit -m "feat(seed): add triage demo data — 3 questions, 6 items, 3 shared artifacts"
```

---

### Task 4: Wire Triage Seed into Loader

**Files:**
- Modify: `seed/loader.py`

- [ ] **Step 1: Add triage seed call to the loader**

Open `seed/loader.py` and:

1. Import the seed function:
```python
from seed.triage import seed_triage
```

2. Find the `_run()` async function and add after the last seed call (e.g., after `await seed_dual_write(session)` or similar):

```python
    await seed_triage(session)
    logger.info("triage seed complete")
```

- [ ] **Step 2: Test the seed**

Run: `cd infrastructure && docker compose run --rm -e PYTHONPATH=/repo api uv run python -m seed.loader`
Expected: Seed completes with triage data logged.

- [ ] **Step 3: Verify data in database**

Run: `docker exec prescient-postgres psql -U prescient -d prescient -c "SELECT count(*) FROM triage.triage_questions; SELECT count(*) FROM triage.triage_items; SELECT count(*) FROM triage.shared_items;"`
Expected: 3, 6, 3

- [ ] **Step 4: Commit**

```bash
git add seed/loader.py
git commit -m "feat(seed): wire triage seed into main loader"
```

---

### Task 5: End-to-End Demo Verification

**Files:** None (verification only)

- [ ] **Step 1: Start all services**

Run: `cd infrastructure && docker compose up -d`
Wait for all services to be healthy.

- [ ] **Step 2: Run migrations and seed**

Run: `cd infrastructure && docker compose run --rm migrate && docker compose run --rm -e PYTHONPATH=/repo api uv run python -m seed.loader`

- [ ] **Step 3: Test operator flow**

1. Open `http://localhost:3000/login`
2. Login as Sarah Chen: `sarah@onepeloton.com` / `demo`
3. Verify redirect to `/brief`
4. Navigate to `/triage`
5. Verify: 4 pending items visible (churn question, KPI anomaly, SEC filing, overdue action)
6. Click the churn question — see partial answer and escalation reason
7. Click "Open Brainstorm" — verify it opens chat

- [ ] **Step 4: Test sponsor flow**

1. Open `http://localhost:3000/login` in incognito
2. Login as Michael Torres: `michael@summit.com` / `demo`
3. Verify redirect to sponsor portal (portfolio dashboard)
4. Verify: Peloton card shows with KPI sparklines
5. Click into Peloton → KPI Dashboard tab shows metrics
6. Shared Artifacts tab shows 3 shared items
7. Q&A tab shows 2 answered questions + 1 pending (churn question with partial answer)
8. Submit a new question — verify it appears in the list

- [ ] **Step 5: Test cross-user flow**

1. As Michael (sponsor): submit a new question about "hardware unit economics"
2. Switch to Sarah's tab → refresh triage queue
3. Verify: new question appears as high-priority item
4. Respond to the question directly
5. Switch to Michael's tab → refresh Q&A
6. Verify: answer appears

- [ ] **Step 6: Document any issues and commit fixes**

```bash
git add -A
git commit -m "fix: resolve integration issues from Phase 5e demo verification"
```

---
