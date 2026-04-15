# OpenAPI Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate frontend payload mismatches by generating TypeScript types from the FastAPI OpenAPI spec and providing an MCP server for agents to query API schemas on demand.

**Architecture:** A Python export script extracts `openapi.json` from the FastAPI app. `openapi-typescript` generates `api.generated.ts` from that spec. A Node.js MCP server reads the same spec file to answer agent queries. CI validates both outputs are in sync.

**Tech Stack:** Python (export script), openapi-typescript (codegen), @modelcontextprotocol/sdk + Node.js (MCP server), GitHub Actions (CI)

---

## File Structure

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `scripts/export_openapi.py` | Export OpenAPI JSON from FastAPI app without running server |
| Create | `apps/web/openapi.json` | Static OpenAPI spec (committed, single source of truth) |
| Create | `apps/web/src/types/api.generated.ts` | Generated TypeScript types from OpenAPI spec |
| Create | `tools/openapi-mcp/package.json` | MCP server package definition |
| Create | `tools/openapi-mcp/src/index.ts` | MCP server implementation |
| Create | `tools/openapi-mcp/tsconfig.json` | TypeScript config for MCP server |
| Modify | `apps/web/package.json` | Add `openapi-typescript` dev dep + `generate:types` script |
| Modify | `.github/workflows/ci.yml` | Add OpenAPI spec + types freshness check job |
| Modify | `AGENTS.md` | Add rule requiring generated types for API interfaces |
| Create | `.claude/settings.json` | Configure MCP server for Claude Code sessions |

---

### Task 1: OpenAPI Spec Export Script

**Files:**
- Create: `scripts/export_openapi.py`
- Create: `apps/web/openapi.json`

- [ ] **Step 1: Create the export script**

Create `scripts/export_openapi.py`:

```python
"""Export the FastAPI OpenAPI spec to a static JSON file.

Usage:
    cd apps/api && uv run python ../../scripts/export_openapi.py

The script imports the FastAPI app directly — no running server needed.
Output: ../../apps/web/openapi.json
"""

import json
import sys
from pathlib import Path

# Ensure the API source is importable
api_src = Path(__file__).resolve().parent.parent / "apps" / "api" / "src"
sys.path.insert(0, str(api_src))

# Stub out settings that require env vars so the app can be imported
# without a running database or external services.
import prescient.config.settings as settings_mod  # noqa: E402

_original_get_settings = settings_mod.get_settings


def _get_stub_settings() -> settings_mod.Settings:
    """Return settings with safe defaults for schema export."""
    import os

    env_defaults = {
        "APP_NAME": "prescient-api",
        "DATABASE_URL": "postgresql+asyncpg://stub:stub@localhost:5432/stub",
        "CORS_ORIGINS": "http://localhost:3000",
        "JWT_SECRET": "stub-secret-for-schema-export",
    }
    for key, val in env_defaults.items():
        os.environ.setdefault(key, val)
    return _original_get_settings()


settings_mod.get_settings = _get_stub_settings

from prescient.main import create_app  # noqa: E402

OUTPUT = Path(__file__).resolve().parent.parent / "apps" / "web" / "openapi.json"


def main() -> None:
    app = create_app()
    spec = app.openapi()
    OUTPUT.parent.mkdir(parents=True, exist_ok=True)
    OUTPUT.write_text(json.dumps(spec, indent=2, sort_keys=True) + "\n")
    print(f"Wrote OpenAPI spec to {OUTPUT} ({len(spec.get('paths', {}))} endpoints)")


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Run the export script**

```bash
cd apps/api && uv run python ../../scripts/export_openapi.py
```

Expected: prints `Wrote OpenAPI spec to .../apps/web/openapi.json (N endpoints)` and creates the file.

- [ ] **Step 3: Verify the exported spec**

```bash
python -c "import json; d=json.load(open('apps/web/openapi.json')); print(f'Paths: {len(d[\"paths\"])}  Schemas: {len(d.get(\"components\",{}).get(\"schemas\",{}))}')"
```

Expected: Should show 50+ paths and 30+ schemas matching the 25 routers registered in `main.py`.

- [ ] **Step 4: Commit**

```bash
git add scripts/export_openapi.py apps/web/openapi.json
git commit -m "feat: add OpenAPI spec export script and initial spec snapshot"
```

---

### Task 2: TypeScript Type Generation

**Files:**
- Modify: `apps/web/package.json` (add dev dep + script)
- Create: `apps/web/src/types/api.generated.ts`

- [ ] **Step 1: Install openapi-typescript**

```bash
cd apps/web && npm install -D openapi-typescript
```

- [ ] **Step 2: Add generate:types script to package.json**

In `apps/web/package.json`, add to the `"scripts"` section:

```json
"generate:types": "openapi-typescript openapi.json -o src/types/api.generated.ts"
```

The full scripts section should be:

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "lint": "eslint --max-warnings=0 .",
  "typecheck": "tsc --noEmit",
  "format": "prettier --write .",
  "format:check": "prettier --check .",
  "generate:types": "openapi-typescript openapi.json -o src/types/api.generated.ts"
}
```

- [ ] **Step 3: Run the type generation**

```bash
cd apps/web && npm run generate:types
```

Expected: Creates `src/types/api.generated.ts` with TypeScript interfaces for all Pydantic models and endpoint schemas.

- [ ] **Step 4: Verify the generated types compile**

```bash
cd apps/web && npx tsc --noEmit
```

Expected: No type errors. The generated file should be valid TypeScript.

- [ ] **Step 5: Commit**

```bash
git add apps/web/package.json apps/web/package-lock.json apps/web/src/types/api.generated.ts
git commit -m "feat: add TypeScript type generation from OpenAPI spec"
```

---

### Task 3: MCP Server

**Files:**
- Create: `tools/openapi-mcp/package.json`
- Create: `tools/openapi-mcp/tsconfig.json`
- Create: `tools/openapi-mcp/src/index.ts`

- [ ] **Step 1: Initialize the MCP server package**

Create `tools/openapi-mcp/package.json`:

```json
{
  "name": "@prescient/openapi-mcp",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.1",
    "fuse.js": "^7.0.0",
    "zod": "^3.25.0"
  },
  "devDependencies": {
    "typescript": "^5.6.3"
  }
}
```

Create `tools/openapi-mcp/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **Step 2: Install dependencies**

```bash
cd tools/openapi-mcp && npm install
```

- [ ] **Step 3: Create the MCP server implementation**

Create `tools/openapi-mcp/src/index.ts`:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { readFileSync } from "fs";
import { resolve } from "path";
import Fuse from "fuse.js";

// --- Load spec ---------------------------------------------------------------

const SPEC_PATH = resolve(
  import.meta.dirname,
  "..",
  "..",
  "..",
  "apps",
  "web",
  "openapi.json",
);

interface OpenAPISpec {
  paths: Record<string, Record<string, EndpointDef>>;
  components?: { schemas?: Record<string, SchemaObject> };
}

interface EndpointDef {
  summary?: string;
  description?: string;
  tags?: string[];
  requestBody?: unknown;
  parameters?: unknown[];
  responses?: Record<string, unknown>;
}

interface SchemaObject {
  type?: string;
  properties?: Record<string, unknown>;
  required?: string[];
  description?: string;
  [key: string]: unknown;
}

function loadSpec(): OpenAPISpec {
  const raw = readFileSync(SPEC_PATH, "utf-8");
  return JSON.parse(raw) as OpenAPISpec;
}

const spec = loadSpec();

// --- Search index ------------------------------------------------------------

interface SearchEntry {
  kind: "endpoint" | "model";
  name: string;
  description: string;
}

function buildSearchEntries(): SearchEntry[] {
  const entries: SearchEntry[] = [];

  for (const [path, methods] of Object.entries(spec.paths)) {
    for (const [method, def] of Object.entries(methods)) {
      entries.push({
        kind: "endpoint",
        name: `${method.toUpperCase()} ${path}`,
        description: def.summary ?? def.description ?? "",
      });
    }
  }

  const schemas = spec.components?.schemas ?? {};
  for (const [name, schema] of Object.entries(schemas)) {
    entries.push({
      kind: "model",
      name,
      description: schema.description ?? "",
    });
  }

  return entries;
}

const searchEntries = buildSearchEntries();
const fuse = new Fuse(searchEntries, {
  keys: ["name", "description"],
  threshold: 0.4,
  includeScore: true,
});

// --- MCP Server --------------------------------------------------------------

const server = new McpServer({
  name: "prescient-openapi",
  version: "0.1.0",
});

server.tool(
  "list_endpoints",
  "List all API endpoints. Optionally filter by tag (e.g. 'kpis', 'strategy').",
  { tag: z.string().optional().describe("Filter by OpenAPI tag") },
  async ({ tag }) => {
    const results: string[] = [];

    for (const [path, methods] of Object.entries(spec.paths)) {
      for (const [method, def] of Object.entries(methods)) {
        if (tag && !(def.tags ?? []).includes(tag)) continue;
        results.push(
          `${method.toUpperCase()} ${path} — ${def.summary ?? "(no summary)"}`,
        );
      }
    }

    return { content: [{ type: "text", text: results.join("\n") }] };
  },
);

server.tool(
  "get_endpoint",
  "Get the full request/response schema for a specific API endpoint.",
  {
    path: z.string().describe("Endpoint path, e.g. /strategy/timeline"),
    method: z
      .string()
      .default("get")
      .describe("HTTP method (get, post, put, delete)"),
  },
  async ({ path, method }) => {
    const methods = spec.paths[path];
    if (!methods) {
      return {
        content: [
          {
            type: "text",
            text: `No endpoint found at path: ${path}\nAvailable paths:\n${Object.keys(spec.paths).join("\n")}`,
          },
        ],
      };
    }

    const def = methods[method.toLowerCase()];
    if (!def) {
      return {
        content: [
          {
            type: "text",
            text: `No ${method.toUpperCase()} method at ${path}\nAvailable methods: ${Object.keys(methods).join(", ")}`,
          },
        ],
      };
    }

    // Resolve $ref pointers inline for readability
    const resolved = resolveRefs(def);
    return {
      content: [
        { type: "text", text: JSON.stringify(resolved, null, 2) },
      ],
    };
  },
);

server.tool(
  "get_model",
  "Get the schema for a specific Pydantic model (OpenAPI component).",
  {
    model_name: z
      .string()
      .describe(
        "Model name, e.g. InitiativeResponse, KpiValueResponse",
      ),
  },
  async ({ model_name }) => {
    const schemas = spec.components?.schemas ?? {};
    const schema = schemas[model_name];
    if (!schema) {
      const available = Object.keys(schemas).join(", ");
      return {
        content: [
          {
            type: "text",
            text: `Model "${model_name}" not found.\nAvailable models: ${available}`,
          },
        ],
      };
    }

    const resolved = resolveRefs(schema);
    return {
      content: [
        { type: "text", text: JSON.stringify(resolved, null, 2) },
      ],
    };
  },
);

server.tool(
  "search_api",
  "Fuzzy search across endpoint paths, descriptions, and model names.",
  {
    query: z.string().describe("Search query, e.g. 'initiative', 'kpi value'"),
  },
  async ({ query }) => {
    const results = fuse.search(query, { limit: 15 });
    if (results.length === 0) {
      return {
        content: [
          { type: "text", text: `No results for "${query}"` },
        ],
      };
    }

    const lines = results.map(
      (r) => `[${r.item.kind}] ${r.item.name} — ${r.item.description}`,
    );
    return { content: [{ type: "text", text: lines.join("\n") }] };
  },
);

// --- $ref resolver -----------------------------------------------------------

function resolveRefs(obj: unknown): unknown {
  if (obj === null || typeof obj !== "object") return obj;

  if (Array.isArray(obj)) return obj.map(resolveRefs);

  const record = obj as Record<string, unknown>;

  if ("$ref" in record && typeof record["$ref"] === "string") {
    const refPath = record["$ref"] as string;
    // e.g. "#/components/schemas/InitiativeResponse"
    const parts = refPath.replace("#/", "").split("/");
    let resolved: unknown = spec;
    for (const part of parts) {
      if (resolved && typeof resolved === "object") {
        resolved = (resolved as Record<string, unknown>)[part];
      }
    }
    return resolved ? resolveRefs(resolved) : { $ref: refPath, _unresolved: true };
  }

  const result: Record<string, unknown> = {};
  for (const [key, val] of Object.entries(record)) {
    result[key] = resolveRefs(val);
  }
  return result;
}

// --- Start -------------------------------------------------------------------

async function main(): Promise<void> {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch(console.error);
```

- [ ] **Step 4: Build the MCP server**

```bash
cd tools/openapi-mcp && npx tsc
```

Expected: Compiles without errors, creates `dist/index.js`.

- [ ] **Step 5: Smoke-test the MCP server**

```bash
cd tools/openapi-mcp && echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}}}' | node dist/index.js
```

Expected: Returns a JSON-RPC response with server capabilities listing the 4 tools.

- [ ] **Step 6: Commit**

```bash
git add tools/openapi-mcp/
git commit -m "feat: add OpenAPI MCP server with endpoint/model query tools"
```

---

### Task 4: CI Freshness Check

**Files:**
- Modify: `.github/workflows/ci.yml`

- [ ] **Step 1: Add the openapi-check job to ci.yml**

Add this job after the existing `test-unit` job in `.github/workflows/ci.yml`:

```yaml
  # ---------- OpenAPI spec freshness -------------------------------------------
  openapi-check:
    name: OpenAPI Spec Freshness
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
          cache-dependency-path: apps/web/package-lock.json

      - name: Install API dependencies
        working-directory: apps/api
        run: |
          python -m venv .venv
          .venv/bin/pip install --upgrade pip
          .venv/bin/pip install -e ".[dev]"

      - name: Install web dependencies
        working-directory: apps/web
        run: npm ci

      - name: Re-export OpenAPI spec
        run: |
          cd apps/api && .venv/bin/python ../../scripts/export_openapi.py

      - name: Check spec is up to date
        run: |
          if ! git diff --exit-code apps/web/openapi.json; then
            echo "::error::OpenAPI spec is out of date. Run: cd apps/api && uv run python ../../scripts/export_openapi.py"
            exit 1
          fi

      - name: Re-generate TypeScript types
        working-directory: apps/web
        run: npx openapi-typescript openapi.json -o src/types/api.generated.ts

      - name: Check types are up to date
        run: |
          if ! git diff --exit-code apps/web/src/types/api.generated.ts; then
            echo "::error::Generated types are out of date. Run: cd apps/web && npm run generate:types"
            exit 1
          fi
```

- [ ] **Step 2: Verify the CI file is valid YAML**

```bash
python -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" && echo "Valid YAML"
```

Expected: prints `Valid YAML`.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add OpenAPI spec and generated types freshness check"
```

---

### Task 5: AGENTS.md Update + Claude Code MCP Config

**Files:**
- Modify: `AGENTS.md`
- Create: `.claude/settings.json`

- [ ] **Step 1: Add API types rule to AGENTS.md**

Add the following section to `AGENTS.md` after the existing rules:

```markdown
## API Type Contracts

- **Use generated types for all API interfaces.** Import from `@/types/api.generated` — do not define inline interfaces for API request or response shapes.
- **When unsure about an API shape,** query the OpenAPI MCP server (`search_api`, `get_endpoint`, `get_model`) before guessing field names.
- **After changing a Pydantic model,** re-export the spec and regenerate types:
  ```bash
  cd apps/api && uv run python ../../scripts/export_openapi.py
  cd apps/web && npm run generate:types
  ```
- **Usage pattern for generated types:**
  ```typescript
  import type { components } from '@/types/api.generated';

  type Initiative = components['schemas']['InitiativeResponse'];
  ```
```

- [ ] **Step 2: Create Claude Code MCP configuration**

Create `.claude/settings.json`:

```json
{
  "mcpServers": {
    "prescient-openapi": {
      "command": "node",
      "args": ["tools/openapi-mcp/dist/index.js"],
      "cwd": "."
    }
  }
}
```

- [ ] **Step 3: Commit**

```bash
git add AGENTS.md .claude/settings.json
git commit -m "docs: add API type contract rules and MCP server config"
```

---

### Task 6: End-to-End Verification

**Files:** (none — validation only)

- [ ] **Step 1: Run the full pipeline from scratch**

Delete the generated outputs and re-run everything:

```bash
rm apps/web/openapi.json apps/web/src/types/api.generated.ts
cd apps/api && uv run python ../../scripts/export_openapi.py
cd ../../apps/web && npm run generate:types
```

Expected: Both files are regenerated cleanly.

- [ ] **Step 2: Verify types compile with the rest of the project**

```bash
cd apps/web && npx tsc --noEmit
```

Expected: No type errors.

- [ ] **Step 3: Verify MCP server starts and responds**

```bash
cd tools/openapi-mcp && echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}}}' | node dist/index.js
```

Expected: JSON-RPC response with server info and tool listings.

- [ ] **Step 4: Verify generated files match committed versions**

```bash
git diff --stat
```

Expected: No differences — the regenerated files should be identical to what was committed.

- [ ] **Step 5: Final commit (if any cleanup needed)**

```bash
git status
# If clean, no commit needed
# If any changes: git add -A && git commit -m "chore: end-to-end pipeline verification cleanup"
```
