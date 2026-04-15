# Slug-to-UUID Migration Design

## Problem

Company identity flows through slugs (human-readable strings like `"peloton"`) in URLs, API path params, copilot stream requests, and ToolContext. This creates ambiguity — agents and code must constantly decide whether to pass a slug or a UUID, and frequently get it wrong, causing rework. Slugs can also collide across tenants and break when company names change.

## Goal

Eliminate slugs as identifiers. Every position that identifies a company uses a UUID. The user's primary company is resolved once at login and carried in the session — never passed as a parameter.

## Architecture

Three-tier identity model:

1. **Primary company** — the user's own org, resolved at login, carried in JWT and `CurrentUser`. Always available, never passed.
2. **Target company** — a specific company being viewed or queried. Passed as a UUID in the URL path or request body.
3. **Scope** — the set of companies visible in a copilot conversation. Already exists as `ConversationScope.visible_company_ids`.

Slugs remain in the database and API responses as display labels but are never used as identifiers.

## Design

### 1. Identity Model

`CurrentUser` expands to include the primary company:

```python
@dataclass(frozen=True, slots=True)
class CurrentUser:
    user_id: UserId
    fund_id: FundId
    primary_company_id: CompanyId  # NEW
```

JWT payload expands:

```json
{
  "sub": "sarah",
  "fund_id": "...",
  "primary_company_id": "a1b2c3d4-...",
  "exp": 1234567890
}
```

Login response adds `primary_company_id`. The frontend stores it in localStorage as `prescient_primary_company_id` and exposes it via `useAuth()`.

The `DemoAuthMiddleware` fallback (used when no JWT is present) also resolves `primary_company_id` from the demo user's membership, so the identity contract holds even in demo mode.

### 2. Route & API Surface

All routes change from slug to UUID:

| Current | New |
|---------|-----|
| `/companies/[slug]/...` | `/companies/[id]/...` |
| `/portfolio/companies/[slug]/...` | `/portfolio/companies/[id]/...` |
| `GET /companies/{slug}` | `GET /companies/{company_id}` |
| `GET /companies/{slug}/kpis` | `GET /companies/{company_id}/kpis` |
| `GET /companies/{slug}/artifacts` | `GET /companies/{company_id}/artifacts` |
| `GET /companies/{slug}/documents` | `GET /companies/{company_id}/documents` |
| `GET /companies/{slug}/relationships` | `GET /companies/{company_id}/relationships` |
| `GET /intelligence/strategic-focus/{company_slug}` | `GET /intelligence/strategic-focus/{company_id}` |
| `GET /sponsor/companies/{slug}/kpis` | `GET /sponsor/companies/{company_id}/kpis` |
| `GET /sponsor/companies/{slug}/briefing` | `GET /sponsor/companies/{company_id}/briefing` |

Backend route handlers receive the UUID directly and call `repository.get(company_id)` instead of `get_by_slug(slug)`.

Next.js API proxy routes follow the same rename — `/api/companies/[slug]/...` becomes `/api/companies/[id]/...`, forwarding the UUID to FastAPI.

All components that build company URLs (`nav-actions`, `portfolio-dropdown`, `competitor-sidebar`, `artifacts-list`, `sponsor-portfolio-card`) switch from `c.slug` to `c.id` in href templates.

### 3. Copilot & ToolContext

Copilot stream request fields change:

| Current | New |
|---------|-----|
| `company_slug: str \| None` | `company_id: UUID \| None` |
| `focus_company_slug: str \| None` | `focus_company_id: UUID \| None` |

Resolution logic simplifies: if `company_id` / `focus_company_id` provided, verify the company is in the user's visible set. If neither provided, fall back to `ctx.user.primary_company_id`.

ToolContext aligns:

```python
@dataclass(frozen=True, slots=True)
class ToolContext:
    tenant_id: FundId
    user_id: UserId
    scope: ConversationScope
    primary_company_id: CompanyId  # was: primary_company_slug: str
    session: AsyncSession
    opensearch: AsyncOpenSearch
```

Frontend copilot store changes `companySlug` → `companyId`. `chat-stream.ts` sends `company_id` / `focus_company_id`.

Board prep `GenerateDeckRequest.company_slug` becomes `company_id: UUID`.

### 4. UUID Validation at API Boundary

A reusable FastAPI dependency validates that any company ID path/body param is a valid UUID. If it receives something that looks like a slug, it rejects with a clear error:

```python
def parse_company_id(company_id: str) -> CompanyId:
    try:
        val = UUID(company_id)
    except ValueError:
        raise HTTPException(
            status_code=422,
            detail=f"'{company_id}' is not a valid company UUID. "
                   "Company slugs are no longer accepted as identifiers — "
                   "use the company's UUID instead.",
        )
    return CompanyId(val)

CompanyIdPath = Annotated[CompanyId, Depends(parse_company_id)]
```

Applied to every route that accepts a company identifier. The error message is intentionally explicit so agents and developers immediately understand what went wrong.

Same pattern for `focus_company_id` in copilot stream — Pydantic's UUID type rejects non-UUID strings, with a custom error message override.

### 5. What Stays, What Goes

**Stays:**
- `slug` column in companies table — useful display label
- `Company` domain entity keeps its `slug` field
- API responses include `slug` alongside `id` for display
- `get_by_slug()` in the repository — demoted to display/search utility

**Goes:**
- All `get_company_by_slug()` / `get_company_id_by_slug()` calls in route handlers
- `primary_company_slug` in ToolContext
- `company_slug` / `focus_company_slug` in copilot stream request/response
- `companySlug` in frontend copilot store and chat-stream options
- Slug-based path params in all route definitions

**Frontend auth storage expands:**
- New key: `prescient_primary_company_id`
- `useAuth()` hook exposes `primaryCompanyId`

### 6. Scope of Changes

| Layer | Files affected | Nature of change |
|-------|---------------|-----------------|
| Auth/Login | `login.py`, `demo.py`, `jwt.py`, `base.py` | Add `primary_company_id` to JWT, `CurrentUser`, login response |
| Frontend auth | `auth.ts`, `auth-provider.tsx` | Store and expose `primaryCompanyId` |
| Next.js routes | ~8 page dirs `[slug]` → `[id]` | Rename dirs, update param extraction |
| Next.js API proxies | ~3 proxy route dirs | Rename dirs, forward UUID |
| FastAPI routes | `companies/`, `intelligence/`, `sponsor/`, `briefing/`, `board_prep/` route files | Path params slug→company_id, use `CompanyIdPath` |
| Copilot | `copilot_stream.py`, `base.py`, `chat-stream.ts`, `copilot-store.ts`, `use-copilot-context.ts` | Slug fields → UUID fields |
| URL-building components | `nav-actions.tsx`, `portfolio-dropdown.tsx`, `competitor-sidebar.tsx`, `artifacts-list.tsx`, `sponsor-portfolio-card.tsx` | `c.slug` → `c.id` in hrefs |
| Shared validator | New `CompanyIdPath` dependency | UUID format validation with clear error |
| Queries module | `companies/queries.py` | Slug query functions kept but no longer called from routes |

**Not touched:** Database schema (no migration), RLS/tenant isolation, `Company` domain entity structure, external integrations.

**Risk:** Low. No data migration, no schema change, no backward compat needed. The UUID validator catches any missed references at runtime with a clear error message.
