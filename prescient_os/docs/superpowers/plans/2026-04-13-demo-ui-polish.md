# Demo UI/UX Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Polish every page of Prescient OS for a full-tour demo to a potential co-founder (BCG background), fixing broken functionality and elevating visual consistency across the entire application.

**Architecture:** All changes are frontend-only (Next.js/React/Tailwind) except one API bug fix. No new features — only visual refinement and one param fix. Every page gets consistent card surfaces, typography hierarchy, and accent color treatment.

**Tech Stack:** Next.js 15, React 19, Tailwind CSS, shadcn/ui components, FastAPI (one bug fix)

---

### Task 1: Fix Actions Page 422 API Bug

**Files:**
- Modify: `apps/web/src/app/(main)/actions/page.tsx:197-198`

The frontend sends `organization_id` but the API expects `owner_tenant_id` (a UUID). The org ID stored in localStorage is `org-peloton-001` (a string slug), but the API expects a UUID. We need to understand what UUID the backend expects.

- [ ] **Step 1: Check what the API expects**

Run: `docker compose -f infrastructure/docker-compose.yml exec api bash -c "PYTHONPATH=/repo uv run python -c \"from prescient.action_items.api.routes import router; print('checked')\""`

Then check the actual DB to find the correct tenant ID:

```bash
docker compose -f infrastructure/docker-compose.yml exec postgres psql -U prescient -d prescient -c "SELECT id, slug FROM tenants WHERE slug = 'org-peloton-001' OR name ILIKE '%peloton%' LIMIT 5;"
```

- [ ] **Step 2: Check how other pages resolve org ID to tenant UUID**

Look at how triage page sends its request — it works with `organization_id` param. Check the triage API route for comparison:

```bash
grep -n "organization_id\|owner_tenant_id" apps/api/src/prescient/action_items/api/routes.py
grep -n "organization_id\|owner_tenant_id" apps/api/src/prescient/triage/api/routes.py
```

- [ ] **Step 3: Fix the parameter name**

The simplest fix is to change the backend route parameter from `owner_tenant_id` to `organization_id` to match the frontend convention used by every other page. In `apps/api/src/prescient/action_items/api/routes.py`, change the `list_action_items` function signature parameter name from `owner_tenant_id` to `organization_id`, and update the call to the use case accordingly.

Alternatively, if `owner_tenant_id` expects a UUID and the frontend has a string slug, the backend route needs to accept `organization_id` as a string and resolve it. Check what pattern triage uses and follow it.

- [ ] **Step 4: Verify the fix**

Run: `curl -s "http://localhost:8000/actions/items?organization_id=org-peloton-001" | head -200`

Expected: JSON array of action items (not a 422 error)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/action_items/api/routes.py
git commit -m "fix: align actions API param name with frontend convention"
```

---

### Task 2: Add Favicon

**Files:**
- Create: `apps/web/public/favicon.svg`
- Modify: `apps/web/src/app/layout.tsx:11-14`

- [ ] **Step 1: Create a simple SVG favicon**

Create `apps/web/public/favicon.svg`:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
  <rect width="32" height="32" rx="8" fill="#4F46E5"/>
  <text x="16" y="22" text-anchor="middle" font-family="system-ui, sans-serif" font-size="18" font-weight="600" fill="white">P</text>
</svg>
```

- [ ] **Step 2: Add favicon to metadata**

In `apps/web/src/app/layout.tsx`, add icons to the metadata export:

```typescript
export const metadata: Metadata = {
  title: "Prescient OS",
  description: "Operating intelligence platform",
  icons: {
    icon: "/favicon.svg",
  },
};
```

- [ ] **Step 3: Verify favicon loads**

Navigate to `http://localhost:3000` — browser tab should show the "P" icon. Check console: no more 404 for favicon.ico.

- [ ] **Step 4: Commit**

```bash
git add apps/web/public/favicon.svg apps/web/src/app/layout.tsx
git commit -m "feat: add Prescient OS favicon"
```

---

### Task 3: Global CSS — Accent Color & Background

**Files:**
- Modify: `apps/web/src/styles/globals.css:6-27`

Update the CSS variables to introduce an indigo accent and a subtle warm-gray page background.

- [ ] **Step 1: Update CSS variables**

Replace the `:root` block in `apps/web/src/styles/globals.css`:

```css
@layer base {
  :root {
    --background: 0 0% 98%;
    --foreground: 0 0% 9%;
    --card: 0 0% 100%;
    --card-foreground: 0 0% 9%;
    --popover: 0 0% 100%;
    --popover-foreground: 0 0% 9%;
    --primary: 239 84% 67%;
    --primary-foreground: 0 0% 100%;
    --secondary: 0 0% 96%;
    --secondary-foreground: 0 0% 15%;
    --muted: 0 0% 96%;
    --muted-foreground: 0 0% 45%;
    --accent: 239 84% 67%;
    --accent-foreground: 0 0% 100%;
    --destructive: 0 84% 60%;
    --destructive-foreground: 0 0% 98%;
    --border: 0 0% 90%;
    --input: 0 0% 90%;
    --ring: 239 84% 67%;
    --radius: 0.625rem;
  }

  * {
    @apply border-border;
  }

  body {
    @apply bg-background text-foreground;
  }
}
```

Key changes: `--background` to 98% (subtle off-white), `--primary` and `--accent` to indigo (239 84% 67%), `--ring` to match accent.

- [ ] **Step 2: Verify global styling**

Navigate to `http://localhost:3000/brief`. Page background should be very slightly off-white. Primary buttons (like "Sign in") should now be indigo instead of near-black.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/styles/globals.css
git commit -m "style: introduce indigo accent color and subtle page background"
```

---

### Task 4: Navigation Bar Polish

**Files:**
- Modify: `apps/web/src/components/nav.tsx`

- [ ] **Step 1: Update the nav component**

Replace the entire content of `apps/web/src/components/nav.tsx`:

```tsx
import Link from "next/link";

import { NavActions } from "@/components/nav-actions";
import { PortfolioDropdown } from "@/components/portfolio-dropdown";
import { api } from "@/lib/api";

const NAV_LINKS = [
  { href: "/brief", label: "Morning Brief" },
  { href: "/onboarding", label: "Setup" },
  { href: "/triage", label: "Triage" },
  { href: "/actions", label: "Actions" },
  { href: "/chat", label: "Chat" },
  { href: "/board-prep", label: "Board Prep" },
];

export async function Nav() {
  let companies: { slug: string; name: string }[] = [];
  try {
    companies = await api.companies();
  } catch {
    // Backend unavailable — render nav without company links.
  }

  return (
    <header className="border-b border-neutral-200 bg-white">
      <div className="mx-auto flex max-w-6xl items-center justify-between px-6 py-3">
        {/* Branding */}
        <Link href="/brief" className="flex items-center gap-2">
          <span className="text-xs font-medium uppercase tracking-widest text-neutral-400">
            Prescient OS
          </span>
          <span className="text-neutral-300">/</span>
          <span className="text-sm font-semibold text-neutral-900">Peloton Interactive</span>
        </Link>

        {/* Primary navigation */}
        <nav className="flex items-center gap-1 text-sm">
          {NAV_LINKS.map((link) => (
            <Link
              key={link.href}
              href={link.href}
              className="rounded-md px-3 py-1.5 text-neutral-500 transition-colors hover:bg-neutral-100 hover:text-neutral-900"
            >
              {link.label}
            </Link>
          ))}
          <PortfolioDropdown companies={companies} />
        </nav>

        {/* User area */}
        <NavActions />
      </div>
    </header>
  );
}
```

Key changes: "Intelligence" renamed to "Setup", nav links get rounded hover backgrounds, padding tightened from `py-4` to `py-3`, nav links extracted to a data array.

- [ ] **Step 2: Verify nav changes**

Navigate to any page. Confirm: "Setup" label (not "Intelligence"), hover states on nav links show subtle bg, tighter vertical spacing.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/nav.tsx
git commit -m "style: polish navigation — rename Intelligence to Setup, add hover states"
```

---

### Task 5: Login Page Polish

**Files:**
- Modify: `apps/web/src/app/(auth)/login/page.tsx`

- [ ] **Step 1: Update the login page**

Replace the entire content of `apps/web/src/app/(auth)/login/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";

import { isSponsor, login, setAuth } from "@/lib/auth";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";

const DEMO_ACCOUNTS = [
  { label: "Sarah", role: "Operator · Peloton", email: "sarah@onepeloton.com", password: "demo" },
  { label: "Michael", role: "Sponsor · Summit Partners", email: "michael@summit.com", password: "demo" },
];

export default function LoginPage() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);
  const router = useRouter();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);
    setLoading(true);
    try {
      const result = await login(email, password);
      setAuth(result.access_token, result.user_id, result.organization_id, result.role);
      if (isSponsor()) {
        router.push("/portfolio");
      } else {
        router.push("/brief");
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : "Login failed");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex min-h-screen items-center justify-center bg-neutral-50">
      <div className="w-full max-w-sm space-y-6">
        {/* Branding */}
        <div className="text-center">
          <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">Prescient OS</h1>
          <p className="mt-1 text-sm text-neutral-500">Operating intelligence for portfolio companies</p>
        </div>

        {/* Form card */}
        <div className="rounded-xl border border-neutral-200 bg-white p-6 shadow-sm">
          <form onSubmit={handleSubmit} className="flex flex-col gap-4">
            <div className="space-y-1.5">
              <label htmlFor="email" className="text-xs font-medium text-neutral-600">Email</label>
              <Input
                id="email"
                type="email"
                placeholder="you@company.com"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                required
              />
            </div>
            <div className="space-y-1.5">
              <label htmlFor="password" className="text-xs font-medium text-neutral-600">Password</label>
              <Input
                id="password"
                type="password"
                placeholder="Password"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
                required
              />
            </div>
            {error && <p className="text-sm text-red-600">{error}</p>}
            <Button type="submit" disabled={loading} className="w-full">
              {loading ? "Signing in..." : "Sign in"}
            </Button>
          </form>
        </div>

        {/* Demo accounts */}
        <div className="space-y-2">
          <p className="text-center text-xs text-neutral-400">Quick access</p>
          <div className="flex justify-center gap-2">
            {DEMO_ACCOUNTS.map((acct) => (
              <button
                key={acct.email}
                type="button"
                className="rounded-full border border-neutral-200 bg-white px-4 py-1.5 text-xs font-medium text-neutral-600 transition-colors hover:border-neutral-300 hover:bg-neutral-50"
                onClick={() => {
                  setEmail(acct.email);
                  setPassword(acct.password);
                }}
              >
                {acct.label} <span className="text-neutral-400">· {acct.role.split(" · ")[0]}</span>
              </button>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}
```

Key changes: branding with subtitle, card has proper shadow/border, form fields have labels, demo accounts are pill buttons, cleaner spacing.

- [ ] **Step 2: Verify login page**

Navigate to `http://localhost:3000/login`. Confirm: centered branding, card with shadow, pill-style demo account buttons, "Operating intelligence for portfolio companies" subtitle.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(auth)/login/page.tsx
git commit -m "style: polish login page — branding, card surface, pill demo accounts"
```

---

### Task 6: Morning Brief Cards Transformation

**Files:**
- Modify: `apps/web/src/components/briefing-item.tsx`
- Modify: `apps/web/src/app/(main)/brief/page.tsx:76-79`

- [ ] **Step 1: Update the BriefingItem component**

Replace the entire content of `apps/web/src/components/briefing-item.tsx`:

```tsx
"use client";

import { useState } from "react";
import { ActionItemDialog } from "@/components/action-item-dialog";

export type ActionType = "READ" | "DECIDE" | "FYI";

export interface BriefingItemData {
  id: string;
  title: string;
  summary: string;
  action_type: ActionType;
  source?: string;
  relevance_score?: number;
}

const TYPE_CONFIG: Record<ActionType, { label: string; icon: string; border: string; badge: string }> = {
  READ: {
    label: "Alert",
    icon: "📊",
    border: "border-l-amber-400",
    badge: "bg-amber-50 text-amber-700 border-amber-200",
  },
  DECIDE: {
    label: "Review",
    icon: "✅",
    border: "border-l-indigo-400",
    badge: "bg-indigo-50 text-indigo-700 border-indigo-200",
  },
  FYI: {
    label: "FYI",
    icon: "🛡️",
    border: "border-l-neutral-300",
    badge: "bg-neutral-50 text-neutral-600 border-neutral-200",
  },
};

interface BriefingItemProps {
  item: BriefingItemData;
}

export function BriefingItem({ item }: BriefingItemProps) {
  const config = TYPE_CONFIG[item.action_type] ?? TYPE_CONFIG.FYI;
  const [actionDialogOpen, setActionDialogOpen] = useState(false);

  // Extract first sentence as headline
  const dotIndex = item.summary.indexOf(". ");
  const hasMultipleSentences = dotIndex > 0 && dotIndex < item.summary.length - 2;

  return (
    <>
      <div
        className={`rounded-lg border border-neutral-200 bg-white p-4 transition-shadow hover:shadow-sm border-l-4 ${config.border}`}
      >
        <div className="flex items-start gap-3">
          {/* Type icon */}
          <span className="mt-0.5 text-lg leading-none">{config.icon}</span>

          {/* Content */}
          <div className="min-w-0 flex-1">
            {/* Headline */}
            <p className="font-medium text-neutral-900">{item.title}</p>

            {/* Summary with extracted first sentence */}
            {hasMultipleSentences ? (
              <>
                <p className="mt-1 text-sm text-neutral-700">
                  {item.summary.slice(0, dotIndex + 1)}
                </p>
                <p className="mt-1 text-sm text-neutral-500">
                  {item.summary.slice(dotIndex + 2)}
                </p>
              </>
            ) : (
              <p className="mt-1 text-sm text-neutral-600">{item.summary}</p>
            )}

            {/* Action button for DECIDE items */}
            {item.action_type === "DECIDE" && (
              <button
                className="mt-2 rounded-md border border-indigo-200 bg-indigo-50 px-3 py-1 text-xs font-medium text-indigo-700 transition-colors hover:bg-indigo-100"
                onClick={() => setActionDialogOpen(true)}
              >
                Take action
              </button>
            )}
          </div>

          {/* Right side: badges */}
          <div className="flex shrink-0 flex-col items-end gap-1.5">
            <span
              className={`inline-flex items-center rounded-full border px-2 py-0.5 text-[10px] font-semibold uppercase tracking-wide ${config.badge}`}
            >
              {config.label}
            </span>
            {item.source && (
              <span className="inline-flex items-center rounded-full border border-neutral-200 bg-neutral-50 px-2 py-0.5 text-[10px] text-neutral-500">
                {item.source}
              </span>
            )}
            {item.relevance_score != null && item.relevance_score >= 0.8 && (
              <span
                className="inline-block h-2 w-2 rounded-full bg-indigo-400"
                title="High relevance"
              />
            )}
          </div>
        </div>
      </div>

      <ActionItemDialog
        open={actionDialogOpen}
        onClose={() => setActionDialogOpen(false)}
        defaultTitle={item.title}
      />
    </>
  );
}
```

Key changes: colored left border by type, emoji icons, "Alert"/"Review"/"FYI" differentiated labels, first sentence extraction for scannability, pill-shaped badges.

- [ ] **Step 2: Update the brief page header**

In `apps/web/src/app/(main)/brief/page.tsx`, update the header section (lines 74-80) to always show the subtitle (not just on Mondays):

Replace:
```tsx
        <h1 className="text-2xl font-semibold text-neutral-900">Good morning — {todayLabel()}</h1>
        {briefing?.is_monday && (
          <p className="mt-1 text-neutral-500">Here&apos;s your week ahead.</p>
        )}
```

With:
```tsx
        <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">
          Good morning — {todayLabel()}
        </h1>
        <p className="mt-1 text-sm text-neutral-500">
          {briefing?.is_monday
            ? "Here\u2019s your week ahead."
            : "Here\u2019s what needs your attention."}
        </p>
```

- [ ] **Step 3: Verify brief page**

Navigate to `http://localhost:3000/brief`. Confirm: cards have colored left borders, emoji icons per type, "Alert"/"Review"/"FYI" labels differentiated, subtitle always shows.

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/components/briefing-item.tsx apps/web/src/app/(main)/brief/page.tsx
git commit -m "style: transform Morning Brief cards — type borders, icons, differentiated labels"
```

---

### Task 7: Triage Page Polish

**Files:**
- Modify: `apps/web/src/components/triage-queue.tsx`
- Modify: `apps/web/src/components/triage-question-detail.tsx`

- [ ] **Step 1: Polish the triage queue list**

In `apps/web/src/components/triage-queue.tsx`, update the priority badge styles (lines 6-11):

Replace:
```typescript
const PRIORITY_BADGE: Record<string, string> = {
  critical: "bg-red-100 text-red-700 border border-red-200",
  high: "bg-orange-100 text-orange-700 border border-orange-200",
  medium: "bg-yellow-100 text-yellow-700 border border-yellow-200",
  low: "bg-neutral-100 text-neutral-600 border border-neutral-200",
};
```

With:
```typescript
const PRIORITY_BADGE: Record<string, string> = {
  critical: "bg-red-50 text-red-700 border border-red-200",
  high: "bg-red-50 text-red-700 border border-red-200",
  medium: "bg-amber-50 text-amber-700 border border-amber-200",
  low: "bg-neutral-100 text-neutral-600 border border-neutral-200",
};
```

Then update the status badge (line 152) from plain unstyled to colored:

Replace:
```tsx
                  <div className="mt-1.5">
                    <span className="inline-flex items-center rounded bg-neutral-100 px-1.5 py-0.5 text-[10px] text-neutral-500">
                      {item.status.replace("_", " ")}
                    </span>
                  </div>
```

With:
```tsx
                  <div className="mt-1.5">
                    <span className={`inline-flex items-center rounded-full px-2 py-0.5 text-[10px] font-medium ${
                      item.status === "resolved" ? "bg-green-50 text-green-700" :
                      item.status === "dismissed" ? "bg-neutral-100 text-neutral-500" :
                      item.status === "in_progress" ? "bg-blue-50 text-blue-700" :
                      "bg-neutral-100 text-neutral-600"
                    }`}>
                      {item.status.replace("_", " ")}
                    </span>
                  </div>
```

- [ ] **Step 2: Polish the triage question detail**

In `apps/web/src/components/triage-question-detail.tsx`, update the "Open Brainstorm in Chat" link (lines 116-121) from a bare link to a button:

Replace:
```tsx
      <div>
        <Link
          href={`/chat?triage=${item.id}`}
          className="inline-flex items-center gap-1 text-sm text-blue-600 hover:underline"
        >
          💬 Open Brainstorm in Chat
        </Link>
      </div>
```

With:
```tsx
      <div>
        <Link
          href={`/chat?triage=${item.id}`}
          className="inline-flex items-center gap-2 rounded-lg border border-neutral-200 bg-white px-4 py-2 text-sm font-medium text-neutral-700 transition-colors hover:bg-neutral-50"
        >
          <span>💬</span> Open Brainstorm in Chat
        </Link>
      </div>
```

Also update the "Approve & Send" button (lines 138-144) to use the accent color:

Replace:
```tsx
          <button
            onClick={() => void handleApproveAndSend()}
            disabled={sending || !response.trim()}
            className="rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700 disabled:cursor-not-allowed disabled:opacity-50"
          >
```

With:
```tsx
          <button
            onClick={() => void handleApproveAndSend()}
            disabled={sending || !response.trim()}
            className="rounded-lg bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700 disabled:cursor-not-allowed disabled:opacity-50"
          >
```

- [ ] **Step 3: Verify triage page**

Navigate to `http://localhost:3000/triage`. Confirm: priority badges are colored, status badges are colored pills, brainstorm link is a proper button, Approve & Send is indigo.

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/components/triage-queue.tsx apps/web/src/components/triage-question-detail.tsx
git commit -m "style: polish triage — colored badges, button-style brainstorm link"
```

---

### Task 8: Chat Page — Empty State with Suggested Prompts

**Files:**
- Modify: `apps/web/src/app/(main)/chat/page.tsx:166-185`
- Modify: `apps/web/src/components/chat-input.tsx:40-66`

- [ ] **Step 1: Add suggested prompts to the chat empty state**

In `apps/web/src/app/(main)/chat/page.tsx`, replace the empty state block (lines 178-185):

Replace:
```tsx
            <div className="flex h-full flex-col items-center justify-center text-center">
              <p className="text-sm font-medium text-neutral-900">Ask anything about Peloton</p>
              <p className="mt-1 max-w-md text-xs text-neutral-500">
                The copilot can search company artifacts, KPIs, monitoring data, and action items to
                answer your questions with cited sources.
              </p>
            </div>
```

With:
```tsx
            <div className="flex h-full flex-col items-center justify-center text-center">
              <p className="text-lg font-semibold text-neutral-900">Ask anything about Peloton</p>
              <p className="mt-2 max-w-md text-sm text-neutral-500">
                The copilot searches company artifacts, KPIs, monitoring data, and action items to
                answer your questions with cited sources.
              </p>
              <div className="mt-6 flex flex-wrap justify-center gap-2">
                {[
                  "What's driving subscriber churn?",
                  "Summarize the competitive landscape",
                  "What are our top risks this quarter?",
                  "How is Project Pulse progressing?",
                ].map((prompt) => (
                  <button
                    key={prompt}
                    onClick={() => handleSend(prompt)}
                    disabled={isStreaming}
                    className="rounded-full border border-neutral-200 bg-white px-4 py-2 text-sm text-neutral-600 transition-colors hover:border-neutral-300 hover:bg-neutral-50"
                  >
                    {prompt}
                  </button>
                ))}
              </div>
            </div>
```

- [ ] **Step 2: Polish the chat header**

In the same file, update the header section (lines 171-174):

Replace:
```tsx
        <div className="border-b border-neutral-200 px-4 py-3">
          <h1 className="text-sm font-semibold text-neutral-900">Copilot</h1>
          <p className="text-xs text-neutral-500">Peloton Interactive</p>
        </div>
```

With:
```tsx
        <div className="border-b border-neutral-200 px-6 py-4">
          <h1 className="text-lg font-semibold tracking-tight text-neutral-900">Copilot</h1>
          <p className="text-sm text-neutral-500">Peloton Interactive</p>
        </div>
```

- [ ] **Step 3: Polish the chat input bar**

In `apps/web/src/components/chat-input.tsx`, update the container div (line 41):

Replace:
```tsx
    <div className="border-t border-neutral-200 bg-white px-4 py-3">
```

With:
```tsx
    <div className="border-t border-neutral-200 bg-white px-6 py-4 shadow-[0_-1px_3px_rgba(0,0,0,0.05)]">
```

Also update the send button color (lines 52-57):

Replace:
```tsx
        <Button
          onClick={handleSubmit}
          disabled={disabled || !value.trim()}
          size="sm"
          className="bg-indigo-600 hover:bg-indigo-700"
        >
```

This is already indigo, which is good. Keep it.

- [ ] **Step 4: Verify chat page**

Navigate to `http://localhost:3000/chat`. Confirm: larger heading, suggested prompt pills visible, clicking a pill sends the message, input bar has subtle top shadow.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/app/(main)/chat/page.tsx apps/web/src/components/chat-input.tsx
git commit -m "style: polish chat — suggested prompts, elevated header and input bar"
```

---

### Task 9: Board Prep Page Polish

**Files:**
- Modify: `apps/web/src/app/(main)/board-prep/page.tsx:119-152`

- [ ] **Step 1: Update the board prep page**

In `apps/web/src/app/(main)/board-prep/page.tsx`, replace the header and controls section (lines 119-152):

Replace:
```tsx
    <div className="space-y-6">
      {/* Header */}
      <div>
        <h1 className="text-2xl font-semibold text-neutral-900">Board Prep</h1>
        <p className="mt-1 text-sm text-neutral-500">Peloton Interactive</p>
      </div>

      {/* Controls */}
      <div className="flex items-center gap-4">
        <select
          value={quarter}
          onChange={(e) => {
            setQuarter(e.target.value);
            setApproved(false);
          }}
          className="rounded-md border border-neutral-300 px-3 py-2 text-sm text-neutral-700 focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
        >
          {quarterOptions.map((opt) => (
            <option key={opt.value} value={opt.value}>
              {opt.label}
            </option>
          ))}
        </select>

        <button
          type="button"
          onClick={handleGenerate}
          disabled={isGenerating}
          className="rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white transition-colors hover:bg-indigo-700 disabled:cursor-not-allowed disabled:opacity-50"
        >
          {isGenerating ? "Generating..." : "Generate Board Deck"}
        </button>
      </div>
```

With:
```tsx
    <div className="space-y-6">
      {/* Header */}
      <div>
        <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">Board Prep</h1>
        <p className="mt-1 text-sm text-neutral-500">
          Generate a board-ready narrative from your portfolio data, KPIs, and strategic priorities.
        </p>
      </div>

      {/* Controls */}
      <div className="rounded-lg border border-neutral-200 bg-white p-5">
        <div className="flex items-center gap-4">
          <select
            value={quarter}
            onChange={(e) => {
              setQuarter(e.target.value);
              setApproved(false);
            }}
            className="rounded-md border border-neutral-200 bg-neutral-50 px-3 py-2 text-sm text-neutral-700 focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500"
          >
            {quarterOptions.map((opt) => (
              <option key={opt.value} value={opt.value}>
                {opt.label}
              </option>
            ))}
          </select>

          <button
            type="button"
            onClick={handleGenerate}
            disabled={isGenerating}
            className="rounded-md bg-indigo-600 px-4 py-2 text-sm font-medium text-white transition-colors hover:bg-indigo-700 disabled:cursor-not-allowed disabled:opacity-50"
          >
            {isGenerating ? "Generating..." : "Generate Board Deck"}
          </button>
        </div>
      </div>

      {/* Empty state when no sections */}
      {!isGenerating && !error && sections.length === 0 && (
        <div className="py-16 text-center">
          <p className="text-sm text-neutral-400">
            No board decks generated yet. Select a quarter and generate your first deck.
          </p>
        </div>
      )}
```

- [ ] **Step 2: Verify board prep page**

Navigate to `http://localhost:3000/board-prep`. Confirm: description text below title, controls wrapped in a card, empty state message shown.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(main)/board-prep/page.tsx
git commit -m "style: polish Board Prep — description, card wrapper, empty state"
```

---

### Task 10: Actions Page Polish (Post Bug Fix)

**Files:**
- Modify: `apps/web/src/app/(main)/actions/page.tsx:228-337`

- [ ] **Step 1: Update the actions page header and table styling**

In `apps/web/src/app/(main)/actions/page.tsx`, update the header (lines 229-233):

Replace:
```tsx
      <div>
        <h1 className="text-2xl font-semibold text-neutral-900">Actions</h1>
        <p className="mt-1 text-neutral-500">Review and approve pending action items.</p>
      </div>
```

With:
```tsx
      <div>
        <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">Actions</h1>
        <p className="mt-1 text-sm text-neutral-500">Review and approve pending action items.</p>
      </div>
```

Then add overdue date highlighting. In the due date cell (lines 294-301), replace:

```tsx
                    <TableCell className="text-neutral-500">
                      {item.due_date
                        ? new Date(item.due_date).toLocaleDateString("en-US", {
                            month: "short",
                            day: "numeric",
                          })
                        : "—"}
                    </TableCell>
```

With:
```tsx
                    <TableCell>
                      {item.due_date ? (
                        <span className={new Date(item.due_date) < new Date() ? "font-medium text-red-600" : "text-neutral-500"}>
                          {new Date(item.due_date).toLocaleDateString("en-US", {
                            month: "short",
                            day: "numeric",
                          })}
                          {new Date(item.due_date) < new Date() && (
                            <span className="ml-1 text-[10px]">overdue</span>
                          )}
                        </span>
                      ) : (
                        <span className="text-neutral-400">—</span>
                      )}
                    </TableCell>
```

- [ ] **Step 2: Verify actions page**

Navigate to `http://localhost:3000/actions`. Confirm: page loads (no 422 error), overdue dates show in red with "overdue" label, consistent header styling.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(main)/actions/page.tsx
git commit -m "style: polish Actions page — tracking-tight header, overdue date highlighting"
```

---

### Task 11: Company Page Polish

**Files:**
- Modify: `apps/web/src/components/kpi-strip.tsx:70-103`
- Modify: `apps/web/src/components/recent-documents.tsx:83-115`
- Modify: `apps/web/src/app/(main)/companies/[slug]/page.tsx:60-87`

- [ ] **Step 1: Polish KPI sparkline bars**

In `apps/web/src/components/kpi-strip.tsx`, update the sparkline bar color (line 93):

Replace:
```tsx
                      className="flex-1 rounded-sm bg-neutral-300"
```

With:
```tsx
                      className="flex-1 rounded-sm bg-indigo-200"
```

- [ ] **Step 2: Polish the recent documents table with hover states**

In `apps/web/src/components/recent-documents.tsx`, update the table rows. Add hover state to each `TableRow` (line 85):

Replace:
```tsx
              <TableRow key={doc.id}>
```

With:
```tsx
              <TableRow key={doc.id} className="transition-colors hover:bg-neutral-50">
```

- [ ] **Step 3: Polish company page header**

In `apps/web/src/app/(main)/companies/[slug]/page.tsx`, update the heading (line 67):

Replace:
```tsx
          <h1 className="mt-1 text-3xl font-semibold tracking-tight">{company.name}</h1>
```

With:
```tsx
          <h1 className="mt-1 text-2xl font-semibold tracking-tight text-neutral-900">{company.name}</h1>
```

And update the "Ask about this company" link to be more prominent (lines 84-86):

Replace:
```tsx
        <Button variant="outline" size="sm" asChild>
          <Link href="/chat">Ask about this company</Link>
        </Button>
```

With:
```tsx
        <Button variant="outline" size="sm" asChild className="gap-1.5">
          <Link href="/chat">
            <span>💬</span> Ask about this company
          </Link>
        </Button>
```

- [ ] **Step 4: Verify company page**

Navigate to `http://localhost:3000/companies/peloton`. Confirm: indigo sparkline bars, table row hover states, consistent heading size, chat button has emoji.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/components/kpi-strip.tsx apps/web/src/components/recent-documents.tsx apps/web/src/app/(main)/companies/[slug]/page.tsx
git commit -m "style: polish company page — indigo sparklines, table hovers, consistent heading"
```

---

### Task 12: Onboarding Page Consistency

**Files:**
- Modify: `apps/web/src/app/(main)/onboarding/page.tsx:33-35`

- [ ] **Step 1: Update the onboarding page header**

In `apps/web/src/app/(main)/onboarding/page.tsx`, update the heading (lines 33-38):

Replace:
```tsx
        <h1 className="text-2xl font-semibold text-neutral-900">Welcome to Prescient OS</h1>
        <p className="mt-2 text-neutral-500">
          Complete these three steps to set up your intelligence workspace.
        </p>
```

With:
```tsx
        <h1 className="text-2xl font-semibold tracking-tight text-neutral-900">Welcome to Prescient OS</h1>
        <p className="mt-2 text-sm text-neutral-500">
          Complete these three steps to set up your intelligence workspace.
        </p>
```

- [ ] **Step 2: Verify**

Navigate to `http://localhost:3000/onboarding`. Confirm: tracking-tight on heading, consistent text-sm on subtitle.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(main)/onboarding/page.tsx
git commit -m "style: onboarding page header consistency"
```

---

### Task 13: Competitor Pages — Intentional Empty States

**Files:**
- Modify: `apps/web/src/app/(main)/companies/[slug]/page.tsx`

The competitor pages (Apple Fitness+, etc.) use the same `CompanyOverviewPage` component. The empty strategic focus state already exists. We need to make the empty KPI state and empty artifacts state feel intentional.

- [ ] **Step 1: Update the KPI section empty state**

In `apps/web/src/components/kpi-strip.tsx`, update the error/no-data state (lines 53-55):

Replace:
```tsx
  if (error || !data) {
    return <p className="text-sm text-neutral-500">Unable to load KPI data.</p>;
  }
```

With:
```tsx
  if (error || !data) {
    return (
      <div className="rounded-lg border border-dashed border-neutral-200 py-8 text-center text-sm text-neutral-400">
        No KPI data tracked for this company.
      </div>
    );
  }
```

Also handle the case when `displayed` is empty (after the filtering). After line 61, add:

```tsx
  if (displayed.length === 0) {
    return (
      <div className="rounded-lg border border-dashed border-neutral-200 py-8 text-center text-sm text-neutral-400">
        No KPI data tracked for this company.
      </div>
    );
  }
```

- [ ] **Step 2: Verify competitor page**

Navigate to `http://localhost:3000/companies/apple_fitness`. Confirm: KPI section shows a styled empty state with dashed border instead of an error message.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/kpi-strip.tsx
git commit -m "style: intentional empty states for competitor pages"
```

---

### Task 14: Triage Page Header Consistency

**Files:**
- Modify: `apps/web/src/app/(main)/triage/page.tsx:81-85`

- [ ] **Step 1: Update the triage page header**

In `apps/web/src/app/(main)/triage/page.tsx`, update the heading (lines 82-85):

Replace:
```tsx
        <h1 className="text-xl font-semibold text-neutral-900">Triage Queue</h1>
        <p className="mt-0.5 text-sm text-neutral-500">
          Review escalated items, sponsor questions, and alerts that need your attention.
        </p>
```

With:
```tsx
        <h1 className="text-xl font-semibold tracking-tight text-neutral-900">Triage Queue</h1>
        <p className="mt-0.5 text-sm text-neutral-500">
          Review escalated items, sponsor questions, and alerts that need your attention.
        </p>
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/app/(main)/triage/page.tsx
git commit -m "style: triage header tracking-tight consistency"
```

---

### Task 15: Final Visual QA Pass

**Files:** None — browser testing only.

- [ ] **Step 1: Full tour walkthrough**

Navigate through every page in demo order and verify visually:

1. `http://localhost:3000/login` — branded, centered, pill buttons
2. Log in as Sarah → redirects to `/brief`
3. `/brief` — colored left borders, emoji icons, differentiated labels, subtitle
4. `/triage` — colored badges, selected item highlight, button-style brainstorm link
5. `/actions` — loads without error, overdue dates in red
6. `/chat` — suggested prompt pills, elevated header/input
7. `/board-prep` — description, card wrapper, empty state
8. Portfolio → Peloton → company page with indigo sparklines
9. `/companies/peloton/strategic-focus` — existing styling (already good)
10. Portfolio → Apple Fitness+ → intentional empty states

- [ ] **Step 2: Check console for errors**

On each page, check that there are no console errors (except the known favicon one which should now be resolved).

- [ ] **Step 3: Fix any regressions found**

If any visual issues are found during the walkthrough, fix them inline.

- [ ] **Step 4: Final commit if any fixes were needed**

```bash
git add -A
git commit -m "fix: visual QA fixes from final walkthrough"
```
