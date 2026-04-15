# Navigation Redesign — Icon Rail + Command Palette

**Date:** 2026-04-14
**Status:** Draft
**Context:** Prescient OS operator-first platform

## Overview

The top nav bar is overcrowded — 10 elements (brand, company name, 6 nav links, bell, search, account) competing for one horizontal row, and the page list will only grow. This redesign moves page navigation into a slim icon rail on the left edge and elevates the command palette (OmniSearch) as the primary navigation mechanism, reinforcing the platform's "human language first" positioning.

## Goals

- Declutter the top bar so it scales as pages are added
- Make the command palette the hero navigation element — type what you want, not hunt through menus
- Maintain discoverability for new users via always-visible icon rail
- Keep company name visible at all times for multi-portco context

## Non-Goals (this phase)

- Mobile/responsive layout (desktop-first SaaS, structure should accommodate it later)
- Company switcher dropdown (future — for sponsors managing multiple portcos)
- Icon rail drag-to-reorder or pinning customization
- Collapsing the rail to fully hidden

---

## Icon Rail

A `<aside>` element pinned to the left edge, below the top bar, full remaining viewport height.

**Dimensions:** 48px wide collapsed (default), expands to ~200px on hover.

**Navigation items (top to bottom):**

| Page | Icon (lucide-react) |
|------|-------------------|
| Morning Brief | `Newspaper` |
| Triage | `Flag` |
| Actions | `CheckCircle` |
| Board Prep | `Presentation` |
| Knowledge | `BookOpen` |
| Strategy | `Target` |

**Behavior:**
- Hover over the rail → smooth expand to ~200px showing icon + text label (~150ms transition)
- Mouse leaves → collapses back to 48px
- Active page: left accent bar (2px, primary color) + slightly filled background on the icon
- Each icon is a Next.js `<Link>` — standard client-side navigation
- Active page detected via `usePathname()` matching against each item's `href`

**Styling:** White background, subtle right border (`border-neutral-200`). Icons in `text-neutral-400`, active icon in `text-neutral-900`. Hover state on individual items: `bg-neutral-50`.

---

## Top Bar

Simplified from the current 3-zone layout to a cleaner arrangement.

**Layout (left to right):**
- **Brand** (~150px): "Prescient OS / {Company Name}" — same content as today, more room to breathe with nav links removed
- **Search trigger** (centered, ~350px): Wider pill than current. Text: "Search or type a command... ⌘K". Visually prominent — this is the hero element.
- **Right cluster** (~80px): Notifications bell icon, account avatar/button

**What's removed from the top bar:**
- All 6 nav links (moved to icon rail)

**What stays the same:**
- Height (~52px)
- White background with bottom border
- Full viewport width (spans above the icon rail)

---

## Enhanced Command Palette

Enhances the existing `OmniSearch` component to add page navigation alongside KB search and copilot handoff.

### Zero state (opened with no query)

Shows a "Quick navigation" section listing all 6 pages with their icons. Keyboard-navigable with arrow keys. This is the discoverability backstop — even users who never notice the icon rail see all destinations immediately on ⌘K.

### With query text

Three sections appear in priority order:

1. **Pages** — fuzzy match on page names. "str" matches Strategy, "act" matches Actions, "board" matches Board Prep. Instant, client-side filtering (no API call). Appears at the top of results.
2. **Knowledge** — existing KB search via `/knowledge/sources` API. Debounced, same as today.
3. **Ask copilot** — always at the bottom when there's a query. Same behavior as today (navigates to `/companies/{slug}/chat?q=...`).

### Keyboard behavior

- Arrow up/down navigates results across all sections as one continuous list
- Enter selects the highlighted result (navigates to page, KB doc, or triggers copilot)
- Escape closes the palette
- Section headers ("Pages", "Knowledge", "Ask copilot") are visual dividers, not selectable

### No other changes

Modal size, position (top 20%), animation, and styling remain the same. The trigger button styling updates to be wider and more prominent in its new centered position.

---

## Layout Integration

### Current structure
```
<Nav />              ← full-width top bar with everything
<main max-w-6xl>    ← centered content
```

### New structure
```
<TopBar />           ← full-width simplified header
<div flex>
  <IconRail />       ← fixed left, 48px, full remaining height
  <main max-w-6xl>  ← content, left edge starts after rail
</div>
```

### Component changes

| Action | File | Description |
|--------|------|-------------|
| Create | `components/top-bar.tsx` | Simplified header: brand, search trigger, bell, account |
| Create | `components/icon-rail.tsx` | Left sidebar with navigation icons, hover-to-expand |
| Modify | `components/omni-search.tsx` | Add pages section to zero state + fuzzy page matching on query |
| Modify | `app/(main)/layout.tsx` | New flex layout with TopBar + IconRail + main |
| Delete | `components/nav.tsx` | Replaced by top-bar.tsx + icon-rail.tsx |

### Active page detection

`usePathname()` from `next/navigation` — match the current path against the icon rail's href values. Exact prefix match (e.g., `/strategy` matches `/strategy` and `/strategy/...`).

### Spacing

The main content area's `max-w-6xl` constraint stays the same. The icon rail sits outside this constraint — it's a fixed-position element that doesn't affect the content flow. Content starts 48px from the left viewport edge.

---

## Acceptance Criteria

- Top bar contains only: brand with company name, centered search trigger, bell icon, account button
- Icon rail visible on the left with 6 page icons, highlights active page
- Hovering the icon rail expands it to show labels
- ⌘K opens the command palette
- Empty command palette shows all 6 pages as quick navigation
- Typing in the palette filters pages (client-side), searches KB (API), and offers copilot
- Selecting a page result navigates to that page
- Existing KB search and copilot handoff behavior unchanged
- No visual regressions on any existing page
