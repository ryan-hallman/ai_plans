# Strategy Initiative Cards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a grouped initiative card grid below the strategy timeline chart with All/Needs Attention toggle.

**Architecture:** New `InitiativeCards` component rendered below the timeline. Shared constants (`HEALTH_COLORS`, `PRIORITY_STYLES`, `STATUS_LABELS`, `healthColor`, `Pill`) stay in `page.tsx` and are passed as props or duplicated minimally. Parent manages `cardMode` state.

**Tech Stack:** React, TypeScript, Tailwind CSS

---

### Task 1: Create the InitiativeCards component

**Files:**
- Create: `apps/web/src/app/(main)/strategy/components/initiative-cards.tsx`

- [ ] **Step 1: Create the component file**

Create `apps/web/src/app/(main)/strategy/components/initiative-cards.tsx`:

```tsx
"use client";

interface Initiative {
  id: string;
  title: string;
  description: string;
  status: string;
  priority: string;
  start_date: string;
  target_end_date: string;
  health_score: number | null;
  completion_pct: number | null;
}

interface InitiativeCardsProps {
  initiatives: Initiative[];
  onSelect: (id: string) => void;
  mode: "all" | "attention";
}

const HEALTH_COLORS: Record<string, string> = {
  green: "bg-green-500",
  yellow: "bg-amber-400",
  red: "bg-red-500",
  gray: "bg-neutral-400",
};

const PRIORITY_STYLES: Record<string, string> = {
  critical: "bg-red-50 text-red-700 border border-red-200",
  high: "bg-orange-50 text-orange-700 border border-orange-200",
  medium: "bg-amber-50 text-amber-700 border border-amber-200",
  low: "bg-green-50 text-green-700 border border-green-200",
};

const STATUS_LABELS: Record<string, string> = {
  draft: "Draft",
  active: "Active",
  at_risk: "At Risk",
  on_hold: "On Hold",
  completed: "Completed",
  cancelled: "Cancelled",
};

const GROUP_ORDER = ["at_risk", "active", "draft", "on_hold", "completed", "cancelled"];
const ATTENTION_STATUSES = new Set(["at_risk", "on_hold"]);

function healthColor(score: number | null, status: string): string {
  if (status === "on_hold" || status === "cancelled") return "gray";
  if (score === null) return "gray";
  if (score >= 0.7) return "green";
  if (score >= 0.4) return "yellow";
  return "red";
}

export function InitiativeCards({ initiatives, onSelect, mode }: InitiativeCardsProps) {
  const grouped = new Map<string, Initiative[]>();
  for (const init of initiatives) {
    if (mode === "attention" && !ATTENTION_STATUSES.has(init.status)) continue;
    const existing = grouped.get(init.status) ?? [];
    existing.push(init);
    grouped.set(init.status, existing);
  }

  const sortedGroups = GROUP_ORDER.filter((s) => grouped.has(s));

  if (sortedGroups.length === 0) {
    return (
      <div className="rounded-lg border border-dashed border-neutral-300 bg-neutral-50 py-8 text-center text-sm text-neutral-400">
        {mode === "attention" ? "No initiatives need attention right now." : "No initiatives to display."}
      </div>
    );
  }

  return (
    <div className="space-y-6">
      {sortedGroups.map((status) => {
        const items = grouped.get(status)!;
        return (
          <div key={status}>
            <h3 className="mb-3 text-sm font-semibold text-neutral-500">
              {/* eslint-disable-next-line security/detect-object-injection -- status from GROUP_ORDER */}
              {STATUS_LABELS[status] ?? status} <span className="font-normal text-neutral-400">({items.length})</span>
            </h3>
            <div className="grid grid-cols-2 gap-3 lg:grid-cols-3">
              {items.map((init) => {
                const color = healthColor(init.health_score, init.status);
                const pct = init.completion_pct != null ? Math.round(init.completion_pct * 100) : 0;
                return (
                  <button
                    key={init.id}
                    onClick={() => onSelect(init.id)}
                    className="rounded-lg border border-neutral-200 bg-white p-4 text-left transition-colors hover:bg-neutral-50"
                  >
                    <div className="mb-1 flex items-center gap-2">
                      {/* eslint-disable-next-line security/detect-object-injection -- color from controlled fn */}
                      <div className={`h-2.5 w-2.5 shrink-0 rounded-full ${HEALTH_COLORS[color]}`} />
                      <p className="truncate text-sm font-medium text-neutral-900">{init.title}</p>
                    </div>
                    <p className="mb-3 truncate text-xs text-neutral-500">{init.description}</p>
                    <div className="flex items-center justify-between">
                      <div className="flex items-center gap-2">
                        <span
                          className={`inline-flex items-center rounded-md px-2 py-0.5 text-xs font-semibold capitalize ${PRIORITY_STYLES[init.priority] ?? ""}`}
                        >
                          {init.priority}
                        </span>
                        <span className="text-xs text-neutral-400">{pct}%</span>
                      </div>
                      <span className="text-xs text-neutral-400">
                        {init.start_date} — {init.target_end_date}
                      </span>
                    </div>
                  </button>
                );
              })}
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add "apps/web/src/app/(main)/strategy/components/initiative-cards.tsx"
git commit -m "feat(strategy): add InitiativeCards component with grouped card grid"
```

---

### Task 2: Wire InitiativeCards into the strategy page

**Files:**
- Modify: `apps/web/src/app/(main)/strategy/page.tsx`

- [ ] **Step 1: Add import**

Add after the TimelineChart import (line 9):

```tsx
import { InitiativeCards } from "./components/initiative-cards";
```

- [ ] **Step 2: Add cardMode state**

In the `StrategyPage` component, after the `viewMode` state (line 152), add:

```tsx
const [cardMode, setCardMode] = useState<"all" | "attention">("all");
```

- [ ] **Step 3: Add the card mode toggle and InitiativeCards below the timeline**

Replace the timeline rendering block. Find this section (lines 233-240):

```tsx
          {view === "timeline" ? (
            <TimelineChart
              initiatives={initiatives}
              milestones={timelineData.milestones}
              dependencies={timelineData.dependencies}
              viewMode={viewMode}
              onSelect={selectInitiative}
            />
```

Replace with:

```tsx
          {view === "timeline" ? (
            <div className="space-y-6">
              <TimelineChart
                initiatives={initiatives}
                milestones={timelineData.milestones}
                dependencies={timelineData.dependencies}
                viewMode={viewMode}
                onSelect={selectInitiative}
              />
              <div className="flex items-center justify-between">
                <h2 className="text-sm font-semibold text-neutral-700">Initiatives</h2>
                <div className="flex rounded-md border border-neutral-200">
                  <button
                    onClick={() => setCardMode("all")}
                    className={`px-3 py-1 text-xs font-medium transition-colors ${
                      cardMode === "all"
                        ? "bg-neutral-900 text-white"
                        : "text-neutral-500 hover:text-neutral-900"
                    } rounded-l-md`}
                  >
                    All
                  </button>
                  <button
                    onClick={() => setCardMode("attention")}
                    className={`px-3 py-1 text-xs font-medium transition-colors ${
                      cardMode === "attention"
                        ? "bg-neutral-900 text-white"
                        : "text-neutral-500 hover:text-neutral-900"
                    } rounded-r-md`}
                  >
                    Needs Attention
                  </button>
                </div>
              </div>
              <InitiativeCards
                initiatives={initiatives}
                onSelect={selectInitiative}
                mode={cardMode}
              />
            </div>
```

- [ ] **Step 4: Commit**

```bash
git add "apps/web/src/app/(main)/strategy/page.tsx"
git commit -m "feat(strategy): wire initiative cards with All/Needs Attention toggle"
```

---

### Task 3: Visual verification

- [ ] **Step 1: Verify default (All) mode**

Open `http://localhost:3000/strategy`. Below the timeline chart, confirm:
- "Initiatives" heading with All/Needs Attention toggle
- Cards grouped by status (At Risk first, then Active, Draft, On Hold)
- Each card shows: health dot, title, description, priority pill, completion %, date range
- Cards in 3-column grid

- [ ] **Step 2: Verify Needs Attention mode**

Click "Needs Attention" toggle. Confirm:
- Only At Risk and On Hold groups are shown
- Active and Draft cards are hidden

- [ ] **Step 3: Verify card click opens detail panel**

Click any card. Confirm:
- Detail side panel opens on the right
- Card grid reflows to 2 columns to accommodate the panel

- [ ] **Step 4: Check no console errors**

Open browser console. Confirm no new errors related to the cards.
