# Demo UI/UX Polish — Design Spec

**Date:** 2026-04-13
**Context:** Demo for Walter (potential co-founder, former BCG operator/digital transformation lead) on 2026-04-14
**Goal:** Full-tour demo where every page feels like one cohesive, refined product. Visual polish signals execution quality. Walter should think "I want to use this every day" and "this is a great place to spend time."
**Design direction:** Minimal but elevated — better spacing, subtle surfaces, refined typography, one accent color. Linear/Notion tier. Easy to reskin later.

---

## 1. Fix Blockers

### 1a. Actions Page 422 Error
- `GET /actions/items?organization_id=org-peloton-001` returns 422 (Unprocessable Entity)
- Diagnose: likely a missing or renamed query parameter, or a schema mismatch between frontend and API
- Fix the API bug so the page loads with seed data

### 1b. Missing Favicon
- Add a simple favicon — "P" in accent color on white, or a minimal geometric mark
- Eliminates the 404 in the browser tab

### 1c. "Intelligence" Nav Label
- Currently links to `/onboarding` (setup wizard)
- Relabel to "Setup" — calling it "Intelligence" sets the wrong expectation for a demo audience

---

## 2. Global Visual Refinements

### 2a. Typography Hierarchy
- **h1 (page titles):** `text-2xl font-semibold tracking-tight` — authoritative, not shouty
- **h2 (section headings):** `text-lg font-medium` — clear separation from body
- **Body text:** current size, increase `leading-relaxed` for readability
- **Secondary/meta text** (timestamps, labels, subtitles): `text-sm text-muted-foreground` — consistent everywhere

### 2b. Card Surfaces
- All content cards: `border border-border/50 bg-card rounded-lg p-5`
- Interactive cards on hover: slight shadow lift (`hover:shadow-sm`) or border darken (`hover:border-border`)
- Consistent internal padding and spacing

### 2c. Accent Color
- Refined indigo/slate-blue for interactive elements (active nav, primary buttons, links, tags)
- Semantic badge colors (muted tones, not neon):
  - **Critical/High:** muted red (`bg-red-50 text-red-700 border-red-200`)
  - **Medium:** muted amber (`bg-amber-50 text-amber-700 border-amber-200`)
  - **Low/Resolved:** muted green (`bg-green-50 text-green-700 border-green-200`)
  - **Pending/Neutral:** muted gray (`bg-gray-100 text-gray-600`)

### 2d. Page Layout
- Consistent max-width container across all pages
- Consistent page header pattern: title + subtitle + optional actions, with `mb-8` before content
- Standardize gap between cards/sections (`space-y-4` for cards, `space-y-8` for sections)

---

## 3. Navigation & Shell

### 3a. Header Bar
- Add `border-b border-border/50` bottom border to separate nav from content
- Breadcrumb: "Prescient OS" muted (`text-muted-foreground`), "Peloton Interactive" bold (`font-medium`)
- Nav links: `text-muted-foreground` by default, active gets accent color + `border-b-2 border-accent` indicator
- Portfolio dropdown: ensure popover menu has consistent card styling (`border rounded-lg shadow-md`)
- Sign out: keep subtle, de-emphasize (`text-muted-foreground text-sm`)

### 3b. Favicon
- Simple "P" mark or geometric shape in accent color
- Place in `apps/web/public/`

---

## 4. Login Page

- Center card vertically and horizontally on the page
- Card: `border border-border/50 bg-card rounded-xl shadow-sm p-8`, max-width ~400px
- "Prescient OS" heading: `text-2xl font-semibold tracking-tight`
- Add subtitle: "Operating intelligence for portfolio companies" in `text-muted-foreground`
- Email/password inputs: consistent border treatment
- Sign In button: accent color, full-width
- Demo role hints at bottom: style as pill buttons (`border rounded-full px-3 py-1 text-sm`) so Walter sees them as clickable role options, not debug text

---

## 5. Morning Brief (Opening Impression)

### 5a. Page Header
- "Good morning — Monday, April 13" — keep as-is, it's warm and right
- "Here's your week ahead" — keep, style as `text-muted-foreground`

### 5b. Briefing Cards — Core Transformation
Each card becomes a structured, scannable unit:

- **Colored left border strip** (4px) by item type:
  - KPI anomaly: red/amber left border
  - Action item: blue/accent left border
  - Monitoring finding: purple/slate left border
- **Type icon** at top-left: use the same emoji vocabulary as Triage (📊 for KPI, ✅ for action, 🛡️ for monitoring)
- **Headline extraction:** First sentence or a bold summary line, styled as `font-medium` — separate from the body paragraph. Right now it's all one undifferentiated text block.
- **Body text:** The longer description/context paragraph, styled as `text-sm text-muted-foreground`, maybe with a "Show more" truncation for long items
- **Tags:** Style as proper small badges:
  - Type badge: `text-xs px-2 py-0.5 rounded-full` with semantic color
  - Relevance indicator: dot or subtle highlight, not a separate badge that says "High relevance" in plain text
- **Differentiated action labels** instead of all "FYI":
  - KPI anomalies: "Alert" (amber)
  - Action items: "Review" (blue)
  - Monitoring findings: "FYI" (gray)

### 5c. Card Layout
- Cards in a `space-y-3` stack
- Each card: `border border-border/50 bg-card rounded-lg p-4` + colored left border
- Enough visual variety that a 2-second glance reveals "3 alerts, 4 reviews" without reading

---

## 6. Triage Page

### 6a. Left Panel (List)
- Cards match global card treatment
- Priority badges: colored (`bg-red-50 text-red-700` for high, `bg-amber-50 text-amber-700` for medium)
- Status badges: "pending" neutral gray, "resolved" muted green
- Timestamps: `text-xs text-muted-foreground`
- Emoji type indicators: vertically aligned, consistent size
- Selected item: accent left border + subtle `bg-accent/5` background

### 6b. Right Panel (Detail)
- Section labels ("Question", "Agent Draft Answer", "Escalation Reason"): `text-xs font-medium uppercase tracking-wide text-muted-foreground` with `border-b` underneath — structured document feel
- Gated category tags: proper badge styling
- "Open Brainstorm in Chat": secondary button style, not bare emoji link
- Response textarea: proper card wrapper
- "Approve & Send" button: accent color, prominent
- "Resolve" / "Dismiss" buttons: ghost/outline style, less prominent

---

## 7. Actions Page

- Fix 422 bug first (Section 1a)
- Once loading, each action item as a card with:
  - Status indicator (badge)
  - Assignee
  - Due date (with overdue indicator in red if past due)
  - Same card treatment as Morning Brief items

---

## 8. Chat / Copilot Page

### 8a. Empty State
- "Ask anything about Peloton" + description: center vertically in the chat area, wrap in subtle treatment
- Add 3-4 **suggested prompt pills** below the description:
  - "What's driving subscriber churn?"
  - "Summarize the competitive landscape"
  - "What are our top risks this quarter?"
  - "Compare Peloton IQ engagement trends"
- Pills styled as `border rounded-full px-4 py-2 text-sm hover:bg-accent/10 cursor-pointer`
- Clicking a pill fills the input and optionally auto-sends

### 8b. Input Bar
- Subtle border + slight shadow — premium chat input feel
- Keep "Ctrl+Enter to send" hint but style it as `text-xs text-muted-foreground`

---

## 9. Board Prep Page

- Add description below title: "Generate a board-ready narrative from your portfolio data, KPIs, and strategic priorities." in `text-muted-foreground`
- Wrap quarter selector + Generate button in a card with context
- Below: empty state message — "No board decks generated yet. Select a quarter and generate your first deck." in a muted, centered treatment
- "Generate Board Deck" button already has good accent styling — keep it

---

## 10. Company / Portfolio Pages

### 10a. Peloton Page (Flagship)
- **Key metrics tiles:** wrap each in a card (`border rounded-lg p-4`). Current value larger/bolder (`text-2xl font-semibold`). Label + trend arrow on same line. Sparkline below.
- **Strategic Focus card:** priority badges get colored backgrounds (`bg-accent/10 text-accent`)
- **Artifacts section:** consistent card spacing, tighten artifact tile padding
- **Recent documents table:** add `hover:bg-muted/50` row hover, subtle row borders (`divide-y`)
- **Competitors list:** style as small horizontal cards with threat-type badge, not a bullet list
- **"Ask about this company"** link: style as a secondary button

### 10b. Competitor Pages (Apple Fitness+, etc.)
- Empty sections get intentional empty states: "No KPI data tracked for this competitor" / "Competitive intelligence is generated from SEC filings, earnings calls, and public data"
- Consistent section headers even when content is sparse — frames emptiness as "monitoring" not "forgot to build"

---

## Out of Scope

- No new features or functionality
- No design system / component library abstraction
- No dark mode
- No responsive/mobile optimization
- No animation or transitions beyond hover states
- No changes to API response shapes (except fixing the Actions 422 bug)
- No changes to data or seed content

---

## Success Criteria

Walter walks through every page and:
1. Never sees a broken page or error state
2. Can instantly scan the Morning Brief and distinguish alert types
3. Feels visual consistency across every page — "same product"
4. Notices the quality — spacing, typography, surfaces feel intentional
5. Wants to click around more — suggested prompts in Chat, interactive cards in Triage
6. Leaves thinking "this team ships quality"
