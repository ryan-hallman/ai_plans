# Strategy Timeline Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Polish the strategy timeline Gantt chart — flat material colors, readable labels, tighter proportions.

**Architecture:** All changes stay within `gantt-task-react@^0.3.9`. Per-task `styles` props control bar colors. A scoped CSS file overrides library internals that aren't exposed as props (label fill, row striping, project opacity, grid lines).

**Tech Stack:** React, gantt-task-react, CSS

---

### Task 1: Create scoped CSS overrides

**Files:**
- Create: `apps/web/src/app/(main)/strategy/components/timeline-chart.css`
- Modify: `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx:6` (add import)

- [ ] **Step 1: Create the CSS override file**

Create `apps/web/src/app/(main)/strategy/components/timeline-chart.css`:

```css
/* Scoped overrides for gantt-task-react@0.3.9 internal classes.
   These target minified class names — safe because the library is pinned
   and effectively unmaintained. Delete this file if the library is replaced. */

/* Bar label text: white → near-black for readability */
.strategy-gantt ._3zRJQ {
  fill: #1a1a1a !important;
}

/* Project bar opacity: library default 0.6 → 1 so colors render true */
.strategy-gantt ._2RbVy {
  opacity: 1 !important;
}

/* Remove zebra-striped row backgrounds for a cleaner look */
.strategy-gantt ._2dZTy:nth-child(even) {
  fill: #fff !important;
}

/* Soften grid lines */
.strategy-gantt ._3rUKi {
  stroke: #f0f0f2 !important;
}
```

- [ ] **Step 2: Import the CSS file in timeline-chart.tsx**

In `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx`, add the import after the existing library CSS import on line 6:

```typescript
import "gantt-task-react/dist/index.css";
import "./timeline-chart.css";
```

- [ ] **Step 3: Add the scoping class to the wrapper div**

In `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx`, change the wrapper div (line 140) from:

```tsx
<div className="overflow-x-auto rounded-lg border border-neutral-200 bg-white">
```

to:

```tsx
<div className="strategy-gantt overflow-x-auto rounded-lg border border-neutral-200 bg-white">
```

- [ ] **Step 4: Verify in browser**

Open `http://localhost:3000/strategy`. Confirm:
- Bar label text is dark (near-black), not white
- No zebra striping on alternating rows
- Grid lines are subtler
- Bar colors appear at full opacity (no 0.6 wash)

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/app/(main)/strategy/components/timeline-chart.css apps/web/src/app/(main)/strategy/components/timeline-chart.tsx
git commit -m "style(strategy): add scoped CSS overrides for gantt chart readability"
```

---

### Task 2: Flatten the color palette

**Files:**
- Modify: `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx:42-47`

- [ ] **Step 1: Replace the HEALTH_BAR_COLORS constant**

In `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx`, replace lines 42-47:

```typescript
const HEALTH_BAR_COLORS: Record<string, { bg: string; progress: string }> = {
  green: { bg: "#86efac", progress: "#22c55e" },
  yellow: { bg: "#fde68a", progress: "#f59e0b" },
  red: { bg: "#fca5a5", progress: "#ef4444" },
  gray: { bg: "#d4d4d4", progress: "#a3a3a3" },
};
```

with:

```typescript
const HEALTH_BAR_COLORS: Record<string, { bg: string; progress: string }> = {
  green: { bg: "#a7d7be", progress: "#7cc4a4" },
  yellow: { bg: "#e8d5a3", progress: "#d4bc7c" },
  red: { bg: "#d4a0a0", progress: "#c08080" },
  gray: { bg: "#c8c8cc", progress: "#b0b0b8" },
};
```

- [ ] **Step 2: Verify in browser**

Open `http://localhost:3000/strategy`. Confirm:
- Bars are muted sage/sand/rose/gray tones
- Progress/background split is subtle, not a two-tone gradient pop
- Labels remain readable against the new colors

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(main)/strategy/components/timeline-chart.tsx
git commit -m "style(strategy): flatten gantt bar colors to muted material palette"
```

---

### Task 3: Tighten timeline proportions

**Files:**
- Modify: `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx:148-154`

- [ ] **Step 1: Update Gantt component props**

In `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx`, change the three props on the `<Gantt>` component:

Replace:
```tsx
columnWidth={viewMode === ViewMode.Month ? 150 : 60}
```
with:
```tsx
columnWidth={viewMode === ViewMode.Month ? 100 : 60}
```

Replace:
```tsx
barCornerRadius={4}
```
with:
```tsx
barCornerRadius={6}
```

Replace:
```tsx
barFill={70}
```
with:
```tsx
barFill={60}
```

- [ ] **Step 2: Verify in browser**

Open `http://localhost:3000/strategy`. Confirm:
- Timeline is noticeably tighter (~33% less horizontal scrolling in month view)
- Bars are slimmer with more vertical breathing room
- Corner rounding is slightly softer
- No layout breakage — labels still fit, bars still align with months

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(main)/strategy/components/timeline-chart.tsx
git commit -m "style(strategy): tighten timeline proportions — narrower columns, slimmer bars"
```

---

### Task 4: Final visual review

- [ ] **Step 1: Full-page screenshot comparison**

Open `http://localhost:3000/strategy` and review the complete page. Check:
- Overall feel is clean and material, not candy-colored
- All 6 initiative bars are visible and distinguishable
- Label text is readable on every bar color (green, yellow, red, gray)
- Month headers and year labels are legible
- Horizontal scroll is manageable — the full timeline fits or nearly fits in viewport at month scale
- No console errors related to the CSS changes

- [ ] **Step 2: Check other view modes**

Switch the time scale dropdown to Day, Week, and Year. Confirm nothing breaks — the CSS overrides and prop changes should be neutral for non-month views (columnWidth stays at 60 for those).
