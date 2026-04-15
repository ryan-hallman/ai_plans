# Actions Inbox Redesign

**Date:** 2026-04-13
**Status:** Draft

## Problem

The current actions page is a flat sortable table that shows ALL action items for the entire organization. This creates two problems:

1. **Information overload** — no separation between items that need your attention and items that don't involve you
2. **No user scoping** — the backend has no per-user filtering; the `ActionItemTable` has a `created_by` column but it's never used for filtering, and there's no `assigned_to` field

The primary user (an operator) arrives with 5 minutes between meetings and needs to quickly approve/reject what's waiting, then leave.

## Design

### Data Model Changes

Add `assigned_to` column to `ActionItemTable`:
- Nullable UUID, FK to `tenants.users.id`
- Alembic migration required

"My items" = items where `created_by = :user_id OR assigned_to = :user_id`.

### API Changes

**GET /actions/items** — new parameters:
- `user_id` (optional) — filters to items created by or assigned to this user
- `scope` (optional, `mine` | `org`, default `mine`) — `mine` applies user filter, `org` returns all items for the tenant (existing behavior)
- Existing `organization_id` and `status` params unchanged

**POST /actions/items** — accept optional `assigned_to` field.

No new endpoints needed. Existing PATCH for status and POST for decisions work as-is.

### Inbox Layout

The page has three vertical zones:

#### 1. Summary Bar (top, single row)
Compact stats: "3 needs action / 1 overdue / 8 resolved this week". "This week" = since Monday 00:00 UTC of the current week. Always visible, updates live as you act on items.

A "View all org actions" link at the top right routes to `/actions?scope=org` which renders the existing table view.

#### 2. Needs Action Queue (main area)
Cards stacked vertically, sorted by: overdue first, then by priority (high > medium > low), then by due date ascending.

When the queue is empty: a "You're all caught up" message.

#### 3. Resolved Section (bottom, collapsed by default)
A compact list (not cards) of items you've acted on in the last 7 days. Expandable. Shows title, action taken, and when.

### Card Design

Each inbox card is a contained row:

**Left side:**
- Priority indicator — colored left border (red=high, amber=medium, green=low)
- Title in medium weight
- Subtitle: owner name + due date (e.g. "Sarah Chen · Due Apr 18") or "Overdue · Apr 10" in red

**Right side:**
- Approve (green outline), Reject (red outline), Defer (neutral outline) buttons
- "Log Decision" as a small text link below — secondary action

**States:**
- Overdue cards: subtle red-tinted background
- Patching: loading state on the clicked button
- After action: card fades out over ~300ms (CSS opacity + height collapse)

### Frontend Architecture

**Files:**
- `app/(main)/actions/page.tsx` — rewritten as inbox; existing table view preserved behind `?scope=org` query param
- `components/actions/inbox-card.tsx` — single action card with inline buttons
- `components/actions/summary-bar.tsx` — stats row
- `components/actions/resolved-list.tsx` — collapsed section for resolved items
- Existing `components/action-item-dialog.tsx` and `components/decision-dialog.tsx` unchanged

**State:** Local `useState` (or `useReducer`) — no global store needed. Optimistic UI updates on card actions.

**Animation:** CSS transitions only, no motion library dependency.

## Out of Scope

- Notifications / push alerts for new action items
- Drag-and-drop reordering
- Kanban board view
- Real-time updates (polling or websocket)
