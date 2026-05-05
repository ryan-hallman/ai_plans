# Postgres Terminology Storage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Store terminology mappings in Postgres while preserving the existing `KnowledgeStore` terminology methods used by retrieval and API routes.

**Architecture:** Add a focused Postgres terminology repository under the knowledge layer with schema initialization and upsert/list methods. Wrap the existing `WorkshopManualStore` with a terminology-backed adapter so non-terminology workshop data remains JSON-backed for now, while terminology mappings can move to Postgres without changing retrieval call sites. Use `settings.terminology_database_url` with fallback to `settings.artifact_database_url`.

**Tech Stack:** Python, Pydantic, psycopg, FastAPI route providers, pytest with fake psycopg connections.

---

### Task 1: Postgres Terminology Repository

**Files:**
- Create: `apps/api/src/prescient_benchmark/knowledge/postgres_terminology.py`
- Modify: `apps/api/src/prescient_benchmark/config.py`
- Test: `tests/unit/test_terminology_postgres.py`

- [ ] Write failing tests for schema initialization, disabled fallback, upsert/list round-trip using fake psycopg rows, and seed loading through `seed_vehicle_repair_terminology`.
- [ ] Implement schema SQL for `terminology_mappings(mapping_id, status, retrieval_profile_id, scope_id, payload, updated_at)`.
- [ ] Implement `PostgresTerminologyRepository.upsert_terminology_mapping()` and `list_terminology_mappings()`.
- [ ] Implement `connect_terminology_repository(database_url)` returning a disabled repository when no URL is configured.
- [ ] Add `terminology_database_url` to settings.

### Task 2: KnowledgeStore Adapter

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/store.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Modify: `apps/api/src/prescient_benchmark/mcp/workshop_server.py`
- Test: `tests/unit/test_workshop_store.py`
- Test: `tests/integration/test_workshop_api.py`

- [ ] Write failing tests proving a wrapped workshop store delegates terminology reads/writes to the Postgres repository while preserving JSON source/unit/feedback behavior.
- [ ] Add `TerminologyBackedKnowledgeStore` wrapper or equivalent provider helper.
- [ ] Update API and MCP store providers to use Postgres terminology when configured, with JSON fallback when not configured.
- [ ] Keep existing `KnowledgeStore` method names unchanged.

### Task 3: CLI Bootstrap

**Files:**
- Modify: `apps/api/src/prescient_benchmark/cli.py`
- Test: `tests/integration/test_workshop_manuals_cli.py`

- [ ] Write failing tests for a terminology schema init command and seed command that can target Postgres without writing JSON mapping files.
- [ ] Add `terminology-schema-init`.
- [ ] Update or add a terminology seed command option to use the configured terminology DB.

### Task 4: Verification and Closeout

**Files:**
- Beads metadata
- Git commit

- [ ] Run targeted unit/integration tests.
- [ ] Run `git diff --check`.
- [ ] Close `prescient_os-3sg`.
- [ ] Commit and push code; commit and push this docs plan from the docs repo if tracked.

### Bootstrap Commands

Use explicit terminology database configuration so artifact storage and terminology storage can move independently:

```bash
uv run python -m prescient_benchmark.cli terminology-schema-init --database-url "$TERMINOLOGY_DATABASE_URL"
uv run python -m prescient_benchmark.cli seed-workshop-terminology --database-url "$TERMINOLOGY_DATABASE_URL"
```

For ingestion runs that should seed terminology into Postgres instead of JSON:

```bash
uv run python -m prescient_benchmark.cli ingest-workshop-manuals \
  --data-root corpus/workshop_manuals \
  --terminology-database-url "$TERMINOLOGY_DATABASE_URL"
```
