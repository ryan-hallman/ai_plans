# Board Prep Financial Engine

**Date:** 2026-04-13
**Status:** Approved
**Author:** Ryan + Claude

## Overview

Add structured financial slide types to the board prep deck editor. Replace text-only financial narratives with purpose-built data slides: P&L tables, segment breakdowns, cash flow, KPI trend charts, margin analysis, pipeline funnels, and budget vs. actual comparisons. Introduce new data models for pipeline deals and budget plans, with seed data for the demo.

## Goals

- Replace narrative-only financial slides with structured, visually rich data slides
- Introduce a typed slide system so each slide type has its own data payload and renderer
- Add pipeline and budget data models with full CRUD and seed data
- Cut deck generation time by skipping LLM calls for data slides
- Maintain backward compatibility with existing narrative slides

## Data Models

### Pipeline Deals

New `pipeline` schema with a `deals` table:

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `company_id` | UUID | FK to companies.companies |
| `owner_tenant_id` | UUID | FK to tenants.funds |
| `deal_name` | string(256) | |
| `customer_name` | string(256) | |
| `stage` | string(32) | prospecting, qualification, proposal, negotiation, closed_won, closed_lost |
| `amount` | decimal(20,4) | Deal value |
| `currency` | string(3) | Default "USD" |
| `probability` | decimal(5,2) | 0-100 |
| `expected_close_date` | date | |
| `product_line` | string(128) | Maps to company segments |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### Budget

New `budget` schema with two tables:

**`budget_plans`:**

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `company_id` | UUID | FK to companies.companies |
| `owner_tenant_id` | UUID | FK to tenants.funds |
| `period_label` | string(16) | e.g. "2026-Q2" |
| `period_start` | date | |
| `period_end` | date | |
| `status` | string(16) | draft, approved |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

**`budget_line_items`:**

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `budget_plan_id` | UUID | FK to budget.budget_plans |
| `category` | string(32) | revenue, cogs, opex, capex |
| `subcategory` | string(64) | e.g. subscription_revenue, marketing, r_and_d |
| `label` | string(128) | Display name |
| `planned_amount` | decimal(20,4) | |
| `actual_amount` | decimal(20,4) | Nullable — filled as actuals come in |
| `sort_order` | int | Controls display ordering |

## Typed Slide System

Each slide gains two optional fields alongside the existing `content`, `title`, `number`, and `citations`:

- `slide_type` (string, default `"narrative"`) — determines which renderer to use
- `data` (JSON object, nullable) — structured payload specific to the slide type

### Slide Types

| Type | Data Payload Shape | Renderer |
|------|-------------------|----------|
| `narrative` | None (uses `content` markdown) | Existing SlideContent |
| `pl_summary` | `{ periods: string[], rows: [{ label, values: number[], is_subtotal, indent }] }` | Financial table with P&L formatting |
| `segment_breakdown` | `{ segments: [{ name, revenue, margin, pct_of_total }], period }` | Horizontal bar + summary table |
| `cash_flow` | `{ periods: string[], rows: [{ label, values: number[], is_subtotal }] }` | Table similar to P&L |
| `kpi_trends` | `{ metrics: [{ label, unit, data_points: [{ period, value }] }] }` | Line/bar chart via Recharts |
| `margin_analysis` | `{ product_lines: [{ name, periods: [{ period, gross_margin, op_margin }] }] }` | Multi-line chart + summary table |
| `pipeline` | `{ deals: [{ name, customer, stage, amount, probability, expected_close }], summary: { total_pipeline, weighted, by_stage: { [stage]: { count, amount } } } }` | Funnel summary + deal table |
| `budget_vs_actual` | `{ period, rows: [{ label, category, planned, actual, variance, variance_pct }] }` | Table with green/red variance coloring |

### Backward Compatibility

- Existing slides without `slide_type` default to `"narrative"`
- The `data` field is nullable — narrative slides don't use it
- API response models add `slide_type` and `data` as optional fields
- Zustand store `Slide` type gains optional `slide_type` and `data` fields

## Frontend Rendering

### Charting Library

Add `recharts` to the frontend. React-native, composable, lightweight, works with Tailwind.

### Component Registry

A `slide-renderer.tsx` component switches on `slide_type` and renders the appropriate component:

- `narrative` -> `SlideContent` (existing)
- `pl_summary` -> `PlSummarySlide`
- `cash_flow` -> `CashFlowSlide`
- `segment_breakdown` -> `SegmentBreakdownSlide`
- `kpi_trends` -> `KpiTrendsSlide`
- `margin_analysis` -> `MarginAnalysisSlide`
- `pipeline` -> `PipelineSlide`
- `budget_vs_actual` -> `BudgetVsActualSlide`

### Table Slide Styling (P&L, Cash Flow, Budget)

- Fixed left column for labels, period columns to the right
- Bold rows for subtotals (Revenue, Gross Profit, EBITDA, Net Income)
- Indented rows for sub-items
- Right-aligned numbers with financial formatting ($XXX.XM, XX.X%)
- Budget variance: green for favorable, red for unfavorable
- Consistent indigo/neutral color palette

### Chart Slide Styling (KPI Trends, Margins, Segments)

- Indigo/neutral color palette matching the app theme
- Clean axis labels, no chart junk
- Tooltips on hover with formatted values
- Segment breakdown uses horizontal stacked bars for revenue contribution

### Data Slide Interaction

- Data slides are read-only (no double-click-to-edit like narrative slides)
- Slide title remains editable
- Users can modify data slides through the AI assistant (e.g., "remove the ARPU line from the KPI chart")
- AI assistant mutations can update the `data` payload to filter/adjust what's shown

### PPTX Export

Data slides export as formatted tables and chart images. Each slide type's export logic produces appropriate python-pptx elements:
- Table slides -> pptx table objects with styling
- Chart slides -> render Recharts to canvas, capture as PNG, embed in pptx slide

## Default Deck Structure

When generating a new deck, the assembler produces this 10-slide structure:

| Slide | Title | Type | Data Source |
|-------|-------|------|------------|
| 1 | Executive Summary | `narrative` | LLM synthesis of all slides |
| 2 | P&L Summary | `pl_summary` | KPI data + budget actuals |
| 3 | Revenue & Margins by Segment | `segment_breakdown` + `margin_analysis` | KPI data by product line |
| 4 | Cash Flow | `cash_flow` | Cash flow KPIs |
| 5 | KPI Trends | `kpi_trends` | 8-quarter trailing KPIs |
| 6 | Pipeline & Go-Get | `pipeline` | Pipeline deals |
| 7 | Budget vs. Actual | `budget_vs_actual` | Budget line items |
| 8 | Competitive Landscape | `narrative` | LLM from artifacts/findings |
| 9 | Risk & Legal | `narrative` | LLM from artifacts/observations |
| 10 | People & Organization | `narrative` | LLM from artifacts/action items |

Data slides (2-7) are assembled directly from database queries — no LLM call needed. Only narrative slides (1, 8, 9, 10) require LLM calls, cutting generation time roughly in half.

Slide 3 combines two visualizations: a segment breakdown (revenue contribution by product line) and a margin analysis (margin trends by product line). These render as two sections within one slide.

## Seed Data

### Pipeline Deals (8-10 deals for Peloton)

Mix of stages and product lines:
- 2 prospecting deals ($50K-$200K, Corporate Wellness)
- 2 qualification deals ($100K-$500K, Connected Fitness + Subscription)
- 2 proposal deals ($200K-$1M, Corporate Wellness + Connected Fitness)
- 1 negotiation deal ($1.5M, Enterprise Subscription)
- 1 closed_won deal ($800K, Corporate Wellness) — recent win for positive signal

Expected close dates spanning current and next quarter. Product lines: Connected Fitness, Subscription, Corporate Wellness.

### Budget Plan (current quarter)

Revenue lines:
- Subscription Revenue: $441M planned
- Connected Fitness Revenue: $240M planned
- Corporate/Other Revenue: $15M planned

COGS lines:
- Hardware COGS: $180M planned
- Content & Software: $95M planned
- Logistics & Fulfillment: $35M planned

Opex lines:
- Sales & Marketing: $120M planned
- Research & Development: $85M planned
- General & Administrative: $65M planned

Actuals populated for completed months with realistic variances:
- Subscription slightly ahead of plan (+2%)
- Hardware below plan (-8%)
- Marketing under-spend (-5%)
- R&D on track

Numbers align with existing KPI seed data so the P&L slide ties to KPI trends.

## Technical Notes

- The `slide_type` and `data` fields are added to `BoardSection` domain entity, `SlidePayload` API model, the Zustand `Slide` type, and the `board_prep.board_decks` JSONB `slides` column
- The assembler gains new methods for each data slide type that query the database and return structured payloads (no LLM)
- Pipeline and budget API endpoints follow the existing CRUD pattern (router + repository)
- Pipeline and budget routes are registered on the main app alongside board-prep routes
- Recharts is the only new frontend dependency
- No changes to the AI chat or inline action system — they continue to work on narrative slides and can modify `data` payloads via mutations
