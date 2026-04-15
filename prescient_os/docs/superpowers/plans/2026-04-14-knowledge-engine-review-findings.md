# Knowledge Engine Review Findings

**Date:** 2026-04-14
**Reference plan:** `docs/superpowers/plans/2026-04-14-knowledge-engine-foundation.md`
**Reference spec:** `docs/superpowers/specs/2026-04-13-knowledge-management-design.md`

## Findings

### 1. API-created KM sources are not searchable by the copilot

**Severity:** High

The KM source creation routes create sources with `company_id=None`, which means ingestion propagates `None` to chunks and indexing stores `company_id=null` in OpenSearch. Search then filters all queries by the user's active `company_id`, so these sources will not be returned.

**Plan mismatch:**
- The spec says KM queries are scoped to the active company context.
- The foundation plan keeps `company_id` on `sources`, `source_versions`, and `chunks` specifically for access filtering and retrieval.

**Evidence:**
- [apps/api/src/prescient/knowledge_mgmt/api/routes.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/api/routes.py:147)
- [apps/api/src/prescient/knowledge_mgmt/api/routes.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/api/routes.py:186)
- [apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingestion.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/infrastructure/tasks/ingestion.py:67)
- [apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/infrastructure/services/search.py:149)
- [apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py:55)

### 2. Department routes still use the wrong identity model

**Severity:** High

`create_department` derives `organization_id` from `ctx.user.fund_id` and uses a caller-provided `organization_id` field as the `company_id`. `list_departments` also accepts an `organization_id` query parameter but passes it to `list_by_company()`. This does not match the post-foundation plan, which moved KM isolation to `AccessContext.org_id` and keeps company context separate.

**Plan mismatch:**
- The foundation plan explicitly says KM routes should use `organization_id` from `AccessContext`.
- The original KM design models departments per company, not by caller-supplied organization/company substitution.

**Evidence:**
- [apps/api/src/prescient/knowledge_mgmt/api/department_routes.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/api/department_routes.py:84)
- [apps/api/src/prescient/knowledge_mgmt/api/department_routes.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/api/department_routes.py:104)

### 3. `department_memberships` foundation changes are not wired through ORM and request handling

**Severity:** High

The foundation migration adds `organization_id` plus RLS to `km.department_memberships`, but the ORM model does not include that column. Department routes also do not set `app.current_org_id` before accessing KM tables, and the copilot-side membership lookup runs through a session that only gets `app.current_fund_id`. That leaves department membership access control incomplete and likely broken for department-scoped retrieval.

**Plan mismatch:**
- The foundation plan requires RLS on all KM tables keyed on `organization_id`.
- The same plan states KM routes should set `app.current_org_id` explicitly.

**Evidence:**
- [apps/api/alembic/versions/20260414_ke_foundation.py](/home/rhallman/Projects/prescient_os/apps/api/alembic/versions/20260414_ke_foundation.py:64)
- [apps/api/alembic/versions/20260414_ke_foundation.py](/home/rhallman/Projects/prescient_os/apps/api/alembic/versions/20260414_ke_foundation.py:111)
- [apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/department.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/infrastructure/tables/department.py:27)
- [apps/api/src/prescient/knowledge_mgmt/api/department_routes.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/api/department_routes.py:84)
- [apps/api/src/prescient/db.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/db.py:43)
- [apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/intelligence/infrastructure/tools/search_knowledge_base.py:62)

### 4. `ORG_WIDE` visibility is not honored by search filtering

**Severity:** Medium

The domain access check allows `ORG_WIDE` sources before the company equality check, but the OpenSearch filter builder always constrains retrieval to a single `company_id`. That prevents org-wide knowledge from being retrieved outside the active company even though the visibility model now includes cross-company org-wide access.

**Plan mismatch:**
- The foundation plan adds `Visibility.ORG_WIDE`.
- The current filter implementation still behaves as if all searchable content must belong to one company.

**Evidence:**
- [apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py:26)
- [apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py](/home/rhallman/Projects/prescient_os/apps/api/src/prescient/knowledge_mgmt/infrastructure/services/access_policy.py:55)

## Notes

- I did not treat the `tenant_id` bridge inside `search_knowledge_base` as a finding because the foundation plan explicitly calls that out as a temporary step until `ToolContext` carries `organization_id` natively.
- I attempted to run the KM unit suite in Docker, but the current `api` container image does not include the `tests/` tree, so containerized `pytest` could not run against `tests/unit/knowledge_mgmt`.
