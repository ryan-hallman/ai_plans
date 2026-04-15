# OpenAPI Pipeline: Type Generation + MCP Server

**Date:** 2026-04-14
**Status:** Approved
**Branch:** feature/operator-first-demo-build-foundation

## Problem

Frontend agents building React components frequently guess API payload shapes instead of checking the actual backend contracts. This leads to field naming mismatches (`id` vs `initiative_id`, `organization_id` vs `company_id`), invented response shapes, and hard-coded data formats that don't match the API. Analysis of the branch history shows 24 fix/align commits out of 366 total (6.6%), with 75% of mistakes occurring in React components.

The root cause: no shared type definitions between frontend and backend, and no easy way for agents to look up API schemas at development time.

## Solution

An OpenAPI-first pipeline with two outputs from a single source of truth:

1. **Generated TypeScript types** from the FastAPI OpenAPI spec for compile-time safety
2. **An MCP server** that serves the spec for runtime querying by any AI coding agent

Both consume the same static `openapi.json` file exported from the FastAPI app.

## Architecture

```
Pydantic models (source of truth)
        |
        v
  FastAPI app.openapi()
        |
        v
  openapi.json (static, committed)
       / \
      /   \
     v     v
openapi-typescript    MCP Server
     |                    |
     v                    v
api.generated.ts    Agent queries
(committed)         (list, get, search)
```

## Component 1: OpenAPI Spec Export

### Export Script

A Python script at `scripts/export-openapi.py` that imports the FastAPI app and calls `app.openapi()` directly. No running server required.

- Outputs to `apps/web/openapi.json`
- The exported spec is committed to the repo so agents always have a reference without Docker running
- Deterministic output (sorted keys, stable formatting) to minimize noisy diffs

### CI Integration

- On every PR that touches files under `apps/api/src/`, CI re-exports the spec
- CI fails if the freshly exported spec differs from the committed `openapi.json`
- This catches forgotten regeneration before merge

## Component 2: TypeScript Type Generation

### Setup

- `openapi-typescript` installed as a dev dependency in `apps/web`
- npm script: `"generate:types": "openapi-typescript openapi.json -o src/types/api.generated.ts"`
- Generated file committed to the repo for agent accessibility

### Generated Output

The `api.generated.ts` file provides:

- Interfaces for every Pydantic model (request bodies, response schemas)
- Typed path parameters and query parameters per endpoint
- Organized by OpenAPI component names matching the backend model names

### CI Integration

- CI regenerates types and fails if the committed `api.generated.ts` doesn't match
- Runs as part of the same check that validates the spec export

### Usage in Components

Components import from the generated types:

```typescript
import type { components } from '@/types/api.generated';

type Initiative = components['schemas']['InitiativeResponse'];
type KpiValue = components['schemas']['KpiValueResponse'];
```

### Migration Strategy

- Existing inline interfaces are replaced incrementally as files are touched
- No big-bang rewrite — the generated types file is additive
- AGENTS.md updated to require using generated types for any new API-consuming code

## Component 3: MCP Server

### Location and Runtime

- Lives at `tools/openapi-mcp/`
- Node.js MCP server using `@modelcontextprotocol/sdk` (lightweight, fast startup, matches the web app's JS toolchain)
- Reads the static `openapi.json` file — no runtime dependency on the API server

### Tools Exposed

| Tool | Input | Output |
|------|-------|--------|
| `list_endpoints` | (none, or optional tag filter) | All endpoints: method, path, summary |
| `get_endpoint` | `path`, `method` | Full request/response schema with field descriptions |
| `get_model` | `model_name` | Schema for a specific Pydantic model |
| `search_api` | `query` (text) | Fuzzy search across paths, descriptions, model names |

### Agent Configuration

- Added to `.claude/settings.json` for automatic availability in Claude Code sessions
- Documented in README for configuration with other MCP-compatible tools (Copilot, Cursor, etc.)

## Component 4: Integration & Workflow

### Developer Workflow

1. Backend dev changes a Pydantic model
2. Run `scripts/export-openapi.py` (or let CI catch it)
3. Run `npm run generate:types` in `apps/web`
4. Commit updated `openapi.json` and `api.generated.ts`

### Agent Workflow

1. Agent needs to write a component consuming an API endpoint
2. Imports types from `@/types/api.generated` for compile-time correctness
3. If exploring or unsure, queries the MCP server to discover endpoints and shapes
4. Uses exact field names and types from the generated definitions

### AGENTS.md Rules

Add to AGENTS.md:

- Frontend agents must use generated types from `@/types/api.generated` for all API interfaces
- Do not define inline interfaces for API request/response shapes
- When unsure about an API shape, query the OpenAPI MCP server before guessing

## Scope Boundaries

### In Scope

- OpenAPI spec export script with CI validation
- TypeScript type generation with CI validation
- MCP server with 4 query tools
- AGENTS.md rule updates
- Claude Code MCP configuration

### Out of Scope

- BFF route type validation (not needed — 75% of mistakes are in components)
- Full client SDK generation (heavier than needed, conflicts with BFF pattern)
- Runtime request/response validation middleware
- Automated type migration of existing components (done incrementally)
