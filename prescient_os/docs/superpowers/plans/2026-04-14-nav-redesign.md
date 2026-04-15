# Navigation Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the crowded horizontal nav with a slim icon rail on the left and an enhanced command palette, making search the hero navigation element.

**Architecture:** Split the current `nav.tsx` into two components — `top-bar.tsx` (simplified header with brand, search, bell, account) and `icon-rail.tsx` (left sidebar with page icons that expands on hover). Enhance `omni-search.tsx` to show quick navigation in its zero state and fuzzy page matching when typing. Update `layout.tsx` to compose the new pieces.

**Tech Stack:** Next.js 14, React, Tailwind CSS, lucide-react icons

**Spec:** `docs/superpowers/specs/2026-04-14-nav-redesign-design.md`

---

## File Structure

```
apps/web/src/
├── components/
│   ├── top-bar.tsx          — NEW: simplified header (brand, search trigger, bell, account)
│   ├── icon-rail.tsx        — NEW: left sidebar with navigation icons, hover-to-expand
│   ├── omni-search.tsx      — MODIFY: add pages section + fuzzy page matching
│   ├── nav.tsx              — DELETE: replaced by top-bar + icon-rail
│   ├── nav-brand.tsx        — KEEP: reused by top-bar
│   ├── nav-actions.tsx      — KEEP: reused by top-bar
│   └── notifications-bell.tsx — KEEP: reused by top-bar
├── app/(main)/
│   └── layout.tsx           — MODIFY: new flex layout with icon rail
```

---

### Task 1: Create Icon Rail Component

**Files:**
- Create: `apps/web/src/components/icon-rail.tsx`

- [ ] **Step 1: Create the icon rail component**

```tsx
// apps/web/src/components/icon-rail.tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import {
  Newspaper,
  Flag,
  CheckCircle,
  Presentation,
  BookOpen,
  Target,
} from "lucide-react";
import type { LucideIcon } from "lucide-react";

interface NavItem {
  href: string;
  label: string;
  icon: LucideIcon;
}

const NAV_ITEMS: NavItem[] = [
  { href: "/brief", label: "Morning Brief", icon: Newspaper },
  { href: "/triage", label: "Triage", icon: Flag },
  { href: "/actions", label: "Actions", icon: CheckCircle },
  { href: "/board-prep", label: "Board Prep", icon: Presentation },
  { href: "/knowledge", label: "Knowledge", icon: BookOpen },
  { href: "/strategy", label: "Strategy", icon: Target },
];

export function IconRail() {
  const pathname = usePathname();

  return (
    <aside className="group/rail sticky top-[53px] flex h-[calc(100vh-53px)] w-12 shrink-0 flex-col border-r border-neutral-200 bg-white transition-all duration-150 hover:w-48">
      <nav className="flex flex-1 flex-col gap-1 px-2 py-3">
        {NAV_ITEMS.map((item) => {
          const active = pathname.startsWith(item.href);
          const Icon = item.icon;
          return (
            <Link
              key={item.href}
              href={item.href}
              className={`relative flex items-center gap-3 rounded-md px-2 py-2 transition-colors ${
                active
                  ? "bg-neutral-100 text-neutral-900"
                  : "text-neutral-400 hover:bg-neutral-50 hover:text-neutral-700"
              }`}
            >
              {active && (
                <span className="absolute left-0 top-1/2 h-4 w-0.5 -translate-y-1/2 rounded-full bg-neutral-900" />
              )}
              <Icon className="h-4.5 w-4.5 shrink-0" />
              <span className="truncate text-sm opacity-0 transition-opacity duration-150 group-hover/rail:opacity-100">
                {item.label}
              </span>
            </Link>
          );
        })}
      </nav>
    </aside>
  );
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cd apps/web && npx next build 2>&1 | tail -5`
Expected: Compiled successfully (no pages use this component yet, but it should have no import errors)

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/components/icon-rail.tsx
git commit -m "feat(web): add icon rail sidebar component with hover-to-expand"
```

---

### Task 2: Create Top Bar Component

**Files:**
- Create: `apps/web/src/components/top-bar.tsx`

- [ ] **Step 1: Create the top bar component**

This is a server component (like the current `Nav`) that fetches companies and renders the simplified header.

```tsx
// apps/web/src/components/top-bar.tsx
import Link from "next/link";

import { NavActions } from "@/components/nav-actions";
import { NavBrand } from "@/components/nav-brand";
import { NotificationsBell } from "@/components/notifications-bell";
import { OmniSearch } from "@/components/omni-search";
import { api } from "@/lib/api";

export async function TopBar() {
  let companies: { slug: string; name: string }[] = [];
  try {
    companies = await api.companies();
  } catch {
    // Backend unavailable — render header without company name.
  }

  return (
    <header className="sticky top-0 z-30 border-b border-neutral-200 bg-white">
      <div className="flex items-center justify-between px-6 py-3">
        {/* Brand */}
        <Link href="/brief" className="flex items-center gap-2 shrink-0">
          <span className="text-xs font-medium uppercase tracking-widest text-neutral-400">
            Prescient OS
          </span>
          <span className="text-neutral-300">/</span>
          <NavBrand companies={companies} />
        </Link>

        {/* Centered search trigger */}
        <div className="flex flex-1 justify-center px-8">
          <OmniSearch />
        </div>

        {/* Right cluster */}
        <div className="flex items-center gap-2 shrink-0">
          <NotificationsBell />
          <NavActions companies={companies} />
        </div>
      </div>
    </header>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/components/top-bar.tsx
git commit -m "feat(web): add simplified top bar component with centered search"
```

---

### Task 3: Update OmniSearch with Page Navigation

**Files:**
- Modify: `apps/web/src/components/omni-search.tsx`

- [ ] **Step 1: Update the trigger button to be wider**

Replace the trigger button (lines 134-143) with a wider, more prominent version:

```tsx
{/* Collapsed trigger button — wider for centered header placement */}
<button
  onClick={() => setOpen(true)}
  className="flex w-full max-w-sm items-center gap-2 rounded-lg border border-neutral-200 bg-neutral-50 px-4 py-2 text-sm text-neutral-400 transition-colors hover:border-neutral-300 hover:text-neutral-600"
>
  <Search className="h-4 w-4 shrink-0" />
  <span className="flex-1 text-left">Search or type a command...</span>
  <kbd className="hidden rounded border border-neutral-200 bg-white px-1.5 py-0.5 text-[10px] font-medium text-neutral-400 sm:inline-block">
    ⌘K
  </kbd>
</button>
```

- [ ] **Step 2: Add page navigation data and fuzzy matching**

Add the page items array and filter function after the existing imports (before the `BASE_URL` const):

```tsx
import {
  Search,
  Newspaper,
  Flag,
  CheckCircle,
  Presentation,
  BookOpen,
  Target,
  MessageCircle,
} from "lucide-react";
import type { LucideIcon } from "lucide-react";

// ... existing Dialog import ...

const BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:8000";

interface PageItem {
  href: string;
  label: string;
  icon: LucideIcon;
}

const PAGES: PageItem[] = [
  { href: "/brief", label: "Morning Brief", icon: Newspaper },
  { href: "/triage", label: "Triage", icon: Flag },
  { href: "/actions", label: "Actions", icon: CheckCircle },
  { href: "/board-prep", label: "Board Prep", icon: Presentation },
  { href: "/knowledge", label: "Knowledge", icon: BookOpen },
  { href: "/strategy", label: "Strategy", icon: Target },
];

function filterPages(query: string): PageItem[] {
  if (!query.trim()) return PAGES;
  const q = query.toLowerCase();
  return PAGES.filter(
    (p) => p.label.toLowerCase().includes(q) || p.href.toLowerCase().includes(q),
  );
}
```

- [ ] **Step 3: Add the pages section to the results area**

Replace the results `<div>` (lines 164-199) with a version that includes pages:

```tsx
{/* Results */}
<div className="max-h-80 overflow-y-auto">
  {/* Pages section */}
  {(() => {
    const matchedPages = filterPages(query);
    if (matchedPages.length > 0) {
      return (
        <div>
          <div className="px-4 py-1.5 text-[10px] font-semibold uppercase tracking-wider text-neutral-400">
            Pages
          </div>
          {matchedPages.map((page) => {
            const Icon = page.icon;
            return (
              <button
                key={page.href}
                onClick={() => {
                  setOpen(false);
                  router.push(page.href);
                }}
                className="flex w-full items-center gap-3 px-4 py-2 text-left text-sm transition-colors hover:bg-neutral-50"
              >
                <Icon className="h-4 w-4 shrink-0 text-neutral-400" />
                <span className="text-neutral-700">{page.label}</span>
              </button>
            );
          })}
        </div>
      );
    }
    return null;
  })()}

  {/* Knowledge results section */}
  {query.trim() && (
    <div>
      {(results.length > 0 || loading) && (
        <div className="px-4 py-1.5 text-[10px] font-semibold uppercase tracking-wider text-neutral-400">
          Knowledge
        </div>
      )}
      {loading && (
        <div className="px-4 py-2 text-sm text-neutral-400">
          Searching...
        </div>
      )}
      {!loading && query.trim() && results.length === 0 && (
        <div className="px-4 py-2 text-sm text-neutral-400">
          No documents found.
        </div>
      )}
      {results.map((source) => (
        <button
          key={source.id}
          onClick={() => handleSelect(source.id)}
          className="flex w-full items-center gap-3 px-4 py-2 text-left text-sm transition-colors hover:bg-neutral-50"
        >
          <Search className="h-3.5 w-3.5 shrink-0 text-neutral-400" />
          <span className="truncate text-neutral-700">
            {source.title}
          </span>
          {source.source_type && (
            <span className="ml-auto shrink-0 text-xs text-neutral-400">
              {source.source_type}
            </span>
          )}
        </button>
      ))}
    </div>
  )}

  {/* Ask copilot option */}
  {query.trim() && (
    <div className="border-t border-neutral-100">
      <div className="px-4 py-1.5 text-[10px] font-semibold uppercase tracking-wider text-neutral-400">
        Ask Copilot
      </div>
      <button
        onClick={handleAskCopilot}
        className="flex w-full items-center gap-3 px-4 py-2 text-left text-sm transition-colors hover:bg-neutral-50"
      >
        <MessageCircle className="h-4 w-4 shrink-0 text-neutral-400" />
        <span className="text-neutral-600">
          &ldquo;{query.trim()}&rdquo;
        </span>
      </button>
    </div>
  )}
</div>
```

- [ ] **Step 4: Update the placeholder text**

Change the input placeholder (line 159) from `"Search knowledge base..."` to `"Search pages, documents, or ask a question..."`.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/components/omni-search.tsx
git commit -m "feat(web): enhance command palette with page navigation and wider trigger"
```

---

### Task 4: Update Layout and Wire Everything Together

**Files:**
- Modify: `apps/web/src/app/(main)/layout.tsx`
- Delete: `apps/web/src/components/nav.tsx`

- [ ] **Step 1: Update the layout**

Replace the contents of `apps/web/src/app/(main)/layout.tsx`:

```tsx
import type { ReactNode } from "react";

import { TopBar } from "@/components/top-bar";
import { IconRail } from "@/components/icon-rail";
import { RouteGuard } from "@/components/route-guard";

export default function MainLayout({ children }: { children: ReactNode }) {
  return (
    <>
      <RouteGuard />
      <TopBar />
      <div className="flex">
        <IconRail />
        <main className="mx-auto w-full max-w-6xl px-6 py-8">{children}</main>
      </div>
    </>
  );
}
```

- [ ] **Step 2: Delete the old nav component**

```bash
rm apps/web/src/components/nav.tsx
```

- [ ] **Step 3: Check for other imports of nav.tsx**

Run: `grep -rn "from.*[\"']@/components/nav[\"']" apps/web/src/ --include="*.tsx" --include="*.ts"`

If any files still import from `@/components/nav`, update them. The only expected import was in `layout.tsx` which we just replaced.

- [ ] **Step 4: Verify the build**

Run: `cd apps/web && npx next build 2>&1 | tail -10`
Expected: Compiled successfully, no missing import errors.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/app/\(main\)/layout.tsx apps/web/src/components/
git commit -m "feat(web): switch to icon rail + top bar layout, remove old nav"
```

---

### Task 5: Visual Verification with Playwright

**Files:** None (verification only)

- [ ] **Step 1: Start the dev server if not running**

Run: `cd apps/web && npm run dev` (or verify `http://localhost:3000` is reachable)

- [ ] **Step 2: Navigate to the strategy page and take a screenshot**

Use Playwright MCP to navigate to `http://localhost:3000/strategy`, take a snapshot and screenshot. Verify:
- Icon rail visible on the left with 6 icons
- Strategy icon is highlighted as active
- Top bar has brand on left, search pill centered, bell + account on right
- No nav links in the top bar
- Page content is to the right of the icon rail

- [ ] **Step 3: Hover the icon rail and verify expansion**

Use Playwright to hover over the icon rail area and verify labels appear.

- [ ] **Step 4: Open the command palette**

Use Playwright to click the search trigger or press ⌘K. Verify:
- Zero state shows all 6 pages with icons
- Typing "str" filters to show "Strategy"
- The KB search and copilot sections still appear when typing

- [ ] **Step 5: Navigate via command palette**

Click "Strategy" in the command palette results. Verify navigation works.

- [ ] **Step 6: Check other pages for regressions**

Navigate to `/brief`, `/actions`, `/board-prep` and verify they render correctly with the new layout. Take screenshots.

- [ ] **Step 7: Commit any fixes needed**

If visual issues are found, fix and commit.
