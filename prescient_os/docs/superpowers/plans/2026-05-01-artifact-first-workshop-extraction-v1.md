# Artifact-First Workshop Extraction V1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first Postgres-backed artifact-first workshop-manual slice for Ferrari 360 component torque/spec claims.

**Architecture:** Keep the existing workshop JSON store and OpenSearch retrieval path intact as the raw evidence/fallback layer. Add a focused artifact module with Pydantic domain models, a Postgres repository, deterministic-first torque/spec extraction, component-spec assembly, and an explicit artifact answer mode that can be compared against raw retrieval before becoming default.

**Tech Stack:** FastAPI, Pydantic v2, Typer, psycopg 3, Postgres 16, existing workshop manual JSON store, existing OpenSearch retrieval, pytest.

---

## Boundary Rules

This slice should be easy to refactor into the future DDD structure:

- Domain models in `artifacts/models.py` must not import FastAPI, psycopg, SQL, filesystem stores, or route modules.
- Application services such as extraction, assembly, feedback triage, and answering should depend on the `ArtifactRepository` protocol, not on `PostgresArtifactRepository`.
- Postgres-specific SQL, connection handling, JSON serialization, and schema bootstrap stay in `artifacts/postgres.py`.
- API routes and CLI commands are adapters. They may construct repositories and call application services, but they should not contain artifact assembly logic or SQL.
- Workshop-specific rules live in `artifacts/workshop_*` modules so the core artifact model remains usable for business docs, chats, emails, and forums later.
- Existing JSON workshop store and OpenSearch retrieval remain evidence/fallback adapters, not dependencies of the artifact domain.

## File Map

- `pyproject.toml`: add `psycopg[binary]` for Postgres access.
- `docker-compose.yml`: add local `postgres` service and `ARTIFACT_DATABASE_URL` for the API container.
- `apps/api/src/prescient_benchmark/config.py`: add `artifact_database_url`.
- `apps/api/src/prescient_benchmark/artifacts/models.py`: domain models/enums for observations, artifacts, claims, events, validation.
- `apps/api/src/prescient_benchmark/artifacts/repository.py`: repository protocol plus disabled/null repository.
- `apps/api/src/prescient_benchmark/artifacts/postgres.py`: schema bootstrap and Postgres repository implementation.
- `apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py`: deterministic-first torque/spec observation extraction from `SourceUnitRecord`.
- `apps/api/src/prescient_benchmark/artifacts/workshop_assembly.py`: component-spec artifact assembly from observations.
- `apps/api/src/prescient_benchmark/artifacts/workshop_answer.py`: artifact-first answer service that returns `KnowledgeAnswer`.
- `apps/api/src/prescient_benchmark/api/routes_knowledge.py`: add explicit artifact answer mode and repository provider.
- `apps/api/src/prescient_benchmark/cli.py`: add artifact schema bootstrap and extraction/assembly command.
- `tests/unit/test_artifact_models.py`: domain validation tests.
- `tests/unit/test_workshop_artifact_extraction.py`: extraction and assembly tests.
- `tests/unit/test_workshop_artifact_answer.py`: artifact answer tests.
- `tests/unit/test_workshop_api.py`: explicit artifact mode route tests using fake repository.

## Task 1: Postgres Configuration

**Files:**
- Modify: `pyproject.toml`
- Modify: `docker-compose.yml`
- Modify: `apps/api/src/prescient_benchmark/config.py`

- [x] **Step 1: Write the failing settings test**

Create `tests/unit/test_artifact_config.py`:

```python
from prescient_benchmark.config import Settings


def test_artifact_database_url_is_configurable() -> None:
    settings = Settings(artifact_database_url="postgresql://user:pass@db:5432/prescient")

    assert settings.artifact_database_url == "postgresql://user:pass@db:5432/prescient"
```

- [x] **Step 2: Run test to verify it fails**

Run: `uv run python -m pytest tests/unit/test_artifact_config.py -q`

Expected: fails because `artifact_database_url` does not exist.

- [x] **Step 3: Add config and dependency**

Add `psycopg[binary]>=3.2` to `pyproject.toml`.

Add to `Settings` in `apps/api/src/prescient_benchmark/config.py`:

```python
artifact_database_url: str | None = None
```

Add `postgres` to `docker-compose.yml` and set the API env var:

```yaml
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: prescient
      POSTGRES_USER: prescient
      POSTGRES_PASSWORD: prescient
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  api:
    environment:
      ARTIFACT_DATABASE_URL: postgresql://prescient:prescient@postgres:5432/prescient
    depends_on:
      - opensearch
      - postgres

volumes:
  postgres_data:
```

- [x] **Step 4: Verify config test passes**

Run: `uv run python -m pytest tests/unit/test_artifact_config.py -q`

Expected: pass.

- [x] **Step 5: Refresh lockfile**

Run: `uv lock`

Expected: `uv.lock` updates with psycopg packages.

## Task 2: Artifact Domain Models

**Files:**
- Create: `apps/api/src/prescient_benchmark/artifacts/__init__.py`
- Create: `apps/api/src/prescient_benchmark/artifacts/models.py`
- Test: `tests/unit/test_artifact_models.py`

- [x] **Step 1: Write failing model tests**

Create `tests/unit/test_artifact_models.py` with tests that construct:

```python
from prescient_benchmark.artifacts.models import (
    ArtifactClaim,
    ArtifactTrustState,
    Observation,
    ObservationClaimType,
    SourceAuthority,
)


def test_observation_preserves_extraction_confidence_factors() -> None:
    observation = Observation(
        observation_id="obs-1",
        extraction_run_id="run-1",
        source_id="source-ferrari-360-wsm",
        source_authority=SourceAuthority.AUTHORITATIVE_REFERENCE,
        unit_ids=["unit-p1"],
        claim_type=ObservationClaimType.TORQUE_SPEC,
        claim_text="Lower arm to chassis: 55 Nm",
        value=55,
        unit="Nm",
        component_terms=["lower arm"],
        fastener_terms=["chassis nuts"],
        scope={"domain": "vehicle_repair", "make": "Ferrari", "model": "360 Modena"},
        extraction_method="deterministic_torque_table_v1",
        extraction_confidence=0.86,
        confidence_factors={"numeric_unit_parsed": True, "known_alias": True},
    )

    assert observation.extraction_confidence == 0.86
    assert observation.confidence_factors["known_alias"] is True


def test_artifact_claim_uses_three_v1_trust_states() -> None:
    claim = ArtifactClaim(
        claim_id="lower_arm_to_chassis_torque",
        claim_type=ObservationClaimType.TORQUE_SPEC,
        trust_state=ArtifactTrustState.EXTRACTED,
        value=55,
        unit="Nm",
        provenance_observation_ids=["obs-1"],
        provenance_unit_ids=["unit-p1"],
        source_authority=SourceAuthority.AUTHORITATIVE_REFERENCE,
        extraction_confidence=0.86,
    )

    assert claim.trust_state is ArtifactTrustState.EXTRACTED
```

- [x] **Step 2: Run tests to verify they fail**

Run: `uv run python -m pytest tests/unit/test_artifact_models.py -q`

Expected: import failure for missing `prescient_benchmark.artifacts`.

- [x] **Step 3: Implement models**

Create Pydantic models/enums for `SourceAuthority`, `ObservationClaimType`, `ArtifactTrustState`, `ExtractionRun`, `Observation`, `ArtifactClaim`, `Artifact`, `ArtifactEvent`, `ValidationEvent`, and `ArtifactAnswerMode`.

- [x] **Step 4: Verify model tests pass**

Run: `uv run python -m pytest tests/unit/test_artifact_models.py -q`

Expected: pass.

## Task 3: Postgres Artifact Repository

**Files:**
- Create: `apps/api/src/prescient_benchmark/artifacts/repository.py`
- Create: `apps/api/src/prescient_benchmark/artifacts/postgres.py`
- Test: `tests/unit/test_artifact_repository.py`

- [x] **Step 1: Write repository contract tests with a fake connection**

Use a focused fake or monkeypatch around `PostgresArtifactRepository` methods to verify SQL-independent behavior:

```python
from prescient_benchmark.artifacts.repository import DisabledArtifactRepository


def test_disabled_repository_returns_no_artifact() -> None:
    repository = DisabledArtifactRepository()

    assert repository.find_best_component_spec_artifact(
        question="lower control arm torque",
        scope_id="scope-ferrari-360-modena",
    ) is None
```

- [x] **Step 2: Run test to verify it fails**

Run: `uv run python -m pytest tests/unit/test_artifact_repository.py -q`

Expected: import failure.

- [x] **Step 3: Implement repository protocol and Postgres schema bootstrap**

`repository.py` should define `ArtifactRepository` and `DisabledArtifactRepository`.

`postgres.py` should define:

```python
def connect_artifact_repository(database_url: str | None) -> ArtifactRepository: ...
def initialize_artifact_schema(database_url: str) -> None: ...
class PostgresArtifactRepository: ...
```

The schema should include tables for `artifact_extraction_runs`, `artifact_observations`, `artifacts`, `artifact_claims`, `artifact_events`, `artifact_validation_events`, and `artifact_current_projection`.

- [x] **Step 4: Verify repository tests pass**

Run: `uv run python -m pytest tests/unit/test_artifact_repository.py -q`

Expected: pass.

## Task 4: Deterministic Torque/Spec Extraction And Assembly

**Files:**
- Create: `apps/api/src/prescient_benchmark/artifacts/workshop_extraction.py`
- Create: `apps/api/src/prescient_benchmark/artifacts/workshop_assembly.py`
- Test: `tests/unit/test_workshop_artifact_extraction.py`

- [x] **Step 1: Write failing extraction and assembly tests**

Test that a source unit containing `Lower arm to chassis nuts 55 Nm` emits a torque observation with normalized `lower control arm` terms, confidence factors, and exact unit provenance. Test that assembly produces a `component_spec_artifact` with an extracted claim.

- [x] **Step 2: Run tests to verify they fail**

Run: `uv run python -m pytest tests/unit/test_workshop_artifact_extraction.py -q`

Expected: import failure.

- [x] **Step 3: Implement extractor**

Implement deterministic parsing for `Nm` torque lines using explicit regexes, source authority defaults for workshop manuals, and alias normalization for `LCA`, `wishbone`, `lower arm`, and `lower control arm`.

- [x] **Step 4: Implement assembler**

Group torque observations by scope/component/fastener/value and create `Artifact` records with `ArtifactClaim` records in `extracted` state.

- [x] **Step 5: Verify extraction tests pass**

Run: `uv run python -m pytest tests/unit/test_workshop_artifact_extraction.py -q`

Expected: pass.

## Task 5: Artifact-First Answer Path

**Files:**
- Create: `apps/api/src/prescient_benchmark/artifacts/workshop_answer.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_knowledge.py`
- Test: `tests/unit/test_workshop_artifact_answer.py`
- Test: `tests/integration/test_workshop_api.py`

- [x] **Step 1: Write failing answer service test**

Create a fake artifact repository containing a Ferrari 360 lower-control-arm claim and assert the answer:

- returns `AnswerStatus.ANSWERED`
- includes trust text saying extracted/not validated
- cites the source unit locator
- includes `artifact_first` in retrieval trace

- [x] **Step 2: Run test to verify it fails**

Run: `uv run python -m pytest tests/unit/test_workshop_artifact_answer.py -q`

Expected: import failure.

- [x] **Step 3: Implement artifact answer service**

The service should perform clarification gate checks before using an artifact, return `None` when no relevant artifact exists, and build `KnowledgeAnswer` with existing `EvidenceCitation` and `SourceLocator` types.

- [x] **Step 4: Add explicit request flag**

Extend `AskRouteRequest` with:

```python
artifact_mode: Literal["off", "prefer"] = "off"
```

In `/knowledge/ask/sync`, try artifact-first only when `artifact_mode == "prefer"`. If no artifact answer exists, continue through the existing retrieval path. Keep streaming behavior on raw retrieval until a later UI task adds streamed artifact events.

- [x] **Step 5: Verify answer tests pass**

Run: `uv run python -m pytest tests/unit/test_workshop_artifact_answer.py tests/integration/test_workshop_api.py -q`

Expected: pass.

## Task 6: CLI Bootstrap And Extraction Command

**Files:**
- Modify: `apps/api/src/prescient_benchmark/cli.py`
- Test: `tests/integration/test_workshop_manuals_cli.py`

- [x] **Step 1: Write failing CLI test**

Add a test that invokes:

```bash
artifact-schema-init --database-url postgresql://...
extract-workshop-artifacts --data-root corpus/workshop_manuals --scope-id scope-ferrari-360-modena
```

For unit-level verification, monkeypatch repository connection and assert the command extracts observations and assembles artifacts from the existing JSON store.

- [x] **Step 2: Run test to verify it fails**

Run: `uv run python -m pytest tests/integration/test_workshop_manuals_cli.py -q`

Expected: fails because command does not exist.

- [x] **Step 3: Implement commands**

Add:

```python
@app.command("artifact-schema-init")
def artifact_schema_init(database_url: str | None = typer.Option(None)) -> None: ...

@app.command("extract-workshop-artifacts")
def extract_workshop_artifacts_command(data_root: Path, scope_id: str, database_url: str | None) -> None: ...
```

- [x] **Step 4: Verify CLI tests pass**

Run: `uv run python -m pytest tests/integration/test_workshop_manuals_cli.py -q`

Expected: pass.

## Task 7: Focused Verification And Commits

**Files:**
- All files above.

- [x] **Step 1: Run focused tests**

Run:

```bash
uv run python -m pytest \
  tests/unit/test_artifact_config.py \
  tests/unit/test_artifact_models.py \
  tests/unit/test_artifact_repository.py \
  tests/unit/test_workshop_artifact_extraction.py \
  tests/unit/test_workshop_artifact_answer.py \
  tests/integration/test_workshop_api.py \
  tests/integration/test_workshop_manuals_cli.py \
  -q
```

Expected: all pass.

- [x] **Step 2: Run broader backend tests if focused tests pass**

Run: `uv run python -m pytest tests/unit tests/integration/test_workshop_api.py tests/integration/test_workshop_manuals_cli.py -q`

Expected: all selected tests pass.

- [x] **Step 3: Commit and push**

Commit docs plan, code, tests, lockfile, and Beads updates. Push both the docs repo and main repo.
