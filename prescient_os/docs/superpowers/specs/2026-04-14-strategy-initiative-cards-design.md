# Strategy Initiative Cards

**Date:** 2026-04-14
**Status:** Draft
**Scope:** Initiative card grid below the strategy timeline chart

## Problem

The timeline chart shows initiative timelines but lacks at-a-glance detail about each initiative. Users need to click into individual bars to see status, priority, and progress. A card grid below the chart surfaces this information immediately and provides a second entry point to the detail panel.

## Design

### Card Grid

A responsive grid of initiative cards rendered below the timeline chart (timeline view only). Cards are grouped by status with section headers.

**Grid layout:**
- No detail panel open: 3 cards per row (`grid-cols-2 lg:grid-cols-3`)
- Detail panel open: 2 cards per row (the flex parent squeezes the grid naturally)

### Card Contents

Each card displays:
- Health dot (colored circle using existing `healthColor()`) + title (bold, truncated to one line)
- Description (one line, truncated, muted text)
- Priority pill (using existing `PRIORITY_STYLES`) + completion % (right-aligned)
- Date range (small, muted)

Cards use the existing `border border-neutral-200 bg-white rounded-lg` pattern with `hover:bg-neutral-50` and `cursor-pointer`.

### Grouping & Filtering

A toggle control above the card grid switches between two modes:

- **"All"** (default): Shows all initiatives grouped by status. Group order: At Risk, Active, Draft, On Hold, Completed, Cancelled. Each group has a section header with the status label and count.
- **"Needs Attention"**: Only shows At Risk + On Hold groups.

The toggle uses the same visual style as the existing Timeline/List toggle (pill buttons, `bg-neutral-900 text-white` for active, `text-neutral-500` for inactive).

### Interaction

Clicking a card calls `selectInitiative(id)`, opening the existing detail side panel. This is the same handler used by the timeline bar click and the list view row click.

## Component Structure

### New: `InitiativeCards`

**File:** `apps/web/src/app/(main)/strategy/components/initiative-cards.tsx`

**Props:**
```typescript
interface InitiativeCardsProps {
  initiatives: Initiative[];
  onSelect: (id: string) => void;
  mode: "all" | "attention";
}
```

**Responsibilities:**
- Group initiatives by status
- Filter groups based on mode
- Render section headers + card grid
- Delegate click to `onSelect`

Uses the `healthColor()` function, `HEALTH_COLORS`, `PRIORITY_STYLES`, and `Pill` component already defined in `page.tsx`. These will be extracted to a shared location or imported directly.

### Modified: `page.tsx`

**File:** `apps/web/src/app/(main)/strategy/page.tsx`

**Changes:**
- Add `cardMode` state: `useState<"all" | "attention">("all")`
- Add toggle control between SummaryBar and the chart/cards area
- Render `<InitiativeCards>` below the `<TimelineChart>` (inside the same `min-w-0 flex-1` container)
- Extract `healthColor`, `HEALTH_COLORS`, `PRIORITY_STYLES`, `Pill` so both `page.tsx` and `initiative-cards.tsx` can use them (co-located utils file or inline in the cards component)

## Files Changed

1. Create: `apps/web/src/app/(main)/strategy/components/initiative-cards.tsx` — card grid component
2. Modify: `apps/web/src/app/(main)/strategy/page.tsx` — add card mode toggle, render InitiativeCards, extract shared utilities

## Out of Scope

- Cards in list view (list view already shows initiatives as rows)
- Drag-and-drop reordering of cards
- Card actions (edit, delete, status change) — those go through the detail panel
- Persistence of the All/Needs Attention toggle preference
