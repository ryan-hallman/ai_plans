# Strategy Timeline Polish

**Date:** 2026-04-14
**Status:** Draft
**Scope:** Visual polish for the Strategy page timeline (Gantt chart)

## Problem

The current strategy timeline has three visual issues:
1. **Cheesy gradient effect** — the two-tone bar rendering (light background + saturated progress overlay) looks dated
2. **Poor text contrast** — white label text on light bar backgrounds is hard to read
3. **Timeline too wide** — 150px column width in month view creates excessive horizontal scrolling

## Approach

Stay within `gantt-task-react@^0.3.9` — use per-task style props for colors and a scoped CSS override block for properties the library doesn't expose as props (label fill, row striping, project bar opacity).

## Changes

### 1. Flat Color Palette

Replace saturated Tailwind colors with muted, material-inspired tones. Narrow the gap between `backgroundColor` and `progressColor` so the progress/background split is subtle rather than a two-tone gradient.

**File:** `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx`

| Health | Old bg | Old progress | New bg | New progress |
|--------|--------|-------------|--------|-------------|
| Green (healthy) | `#86efac` | `#22c55e` | `#a7d7be` (sage) | `#7cc4a4` (deeper sage) |
| Yellow (warning) | `#fde68a` | `#f59e0b` | `#e8d5a3` (warm sand) | `#d4bc7c` (deeper sand) |
| Red (at-risk) | `#fca5a5` | `#ef4444` | `#d4a0a0` (dusty rose) | `#c08080` (deeper rose) |
| Gray (inactive) | `#d4d4d4` | `#a3a3a3` | `#c8c8cc` (cool gray) | `#b0b0b8` (deeper gray) |

### 2. Dark Label Text

Override the library's internal bar label class to use dark text instead of white.

**Target:** `._3zRJQ` (bar label text, currently `fill: #fff`)
**New value:** `fill: #1a1a1a`

This is the single biggest readability win. The external labels (`._3KcaM`, `fill: #555`) are already fine.

### 3. Timeline Width & Proportions

Prop changes on the `<Gantt>` component:

| Prop | Old | New | Effect |
|------|-----|-----|--------|
| `columnWidth` (month) | 150 | 100 | ~33% tighter timeline |
| `barFill` | 70 | 60 | Slimmer bars, more row breathing room |
| `barCornerRadius` | 4 | 6 | Softer, more material feel |

### 4. CSS Overrides (scoped)

Add a wrapper class `.strategy-gantt` to the timeline container div and define these overrides in a CSS module or inline style block:

```css
/* Bar label text: white → near-black */
.strategy-gantt ._3zRJQ {
  fill: #1a1a1a !important;
}

/* Project bar opacity: 0.6 → 1 (render colors true) */
.strategy-gantt ._2RbVy {
  opacity: 1 !important;
}

/* Remove zebra-striped row backgrounds */
.strategy-gantt ._2dZTy:nth-child(even) {
  fill: #fff !important;
}

/* Soften grid lines */
.strategy-gantt ._3rUKi {
  stroke: #f0f0f2 !important;
}
```

**Fragility note:** These are minified class names from `gantt-task-react@0.3.9`. The library is pinned and effectively unmaintained (last npm publish was years ago). If the library is ever replaced, these overrides get deleted with it — no maintenance burden.

## Files Changed

1. `apps/web/src/app/(main)/strategy/components/timeline-chart.tsx` — color palette, prop values, wrapper class
2. `apps/web/src/app/(main)/strategy/components/timeline-chart.css` — scoped CSS overrides for library internals

## Out of Scope

- Replacing `gantt-task-react` with a custom timeline component
- Moving labels to a sidebar (`TaskListTable`)
- Changes to the list view, detail panel, or summary bar
- Tooltip styling
