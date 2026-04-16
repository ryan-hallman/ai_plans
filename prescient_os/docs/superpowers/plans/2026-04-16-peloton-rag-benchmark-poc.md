# Peloton RAG Benchmark POC Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a narrow, reproducible Peloton-centered benchmark that compares a strong local OpenSearch-based RAG system against ChatGPT, Gemini/NotebookLM, and a CLI filesystem agent on the same corpus.

**Architecture:** Keep the codebase narrow and retrieval-first. Build one FastAPI service that ingests local files, chunks and indexes them into OpenSearch, retrieves with hybrid search plus reranking, and returns synthesized answers with strict citations. Everything else in the plan exists to generate a reproducible corpus and capture eval runs fairly.

**Tech Stack:** Python 3.12, FastAPI, Uvicorn, OpenSearch, Docker Compose, pytest, Typer, httpx, BeautifulSoup4, PyMuPDF, PyYAML, OpenAI-compatible API client for embeddings/answering/generation

---

## File Structure

**Create:**
- `docker-compose.yml`
- `pyproject.toml`
- `apps/api/Dockerfile`
- `apps/api/src/prescient_benchmark/__init__.py`
- `apps/api/src/prescient_benchmark/main.py`
- `apps/api/src/prescient_benchmark/config.py`
- `apps/api/src/prescient_benchmark/api/routes_health.py`
- `apps/api/src/prescient_benchmark/api/routes_corpus.py`
- `apps/api/src/prescient_benchmark/api/routes_ask.py`
- `apps/api/src/prescient_benchmark/corpus/models.py`
- `apps/api/src/prescient_benchmark/corpus/manifest.py`
- `apps/api/src/prescient_benchmark/corpus/public_peloton.py`
- `apps/api/src/prescient_benchmark/corpus/synthetic_docs.py`
- `apps/api/src/prescient_benchmark/corpus/storage.py`
- `apps/api/src/prescient_benchmark/retrieval/models.py`
- `apps/api/src/prescient_benchmark/retrieval/parse.py`
- `apps/api/src/prescient_benchmark/retrieval/chunk.py`
- `apps/api/src/prescient_benchmark/retrieval/index.py`
- `apps/api/src/prescient_benchmark/retrieval/search.py`
- `apps/api/src/prescient_benchmark/retrieval/rerank.py`
- `apps/api/src/prescient_benchmark/retrieval/answer.py`
- `apps/api/src/prescient_benchmark/eval/models.py`
- `apps/api/src/prescient_benchmark/eval/export.py`
- `apps/api/src/prescient_benchmark/eval/scorecard.py`
- `apps/api/src/prescient_benchmark/cli.py`
- `corpus/scenarios/peloton_v1.yaml`
- `eval/questions/peloton_v1.yaml`
- `tests/unit/test_health.py`
- `tests/unit/test_manifest.py`
- `tests/unit/test_public_peloton.py`
- `tests/unit/test_synthetic_docs.py`
- `tests/unit/test_chunking.py`
- `tests/unit/test_answering.py`
- `tests/unit/test_eval_export.py`
- `tests/unit/test_cli_baseline.py`
- `tests/integration/test_opensearch_ingest_and_search.py`
- `tests/integration/test_benchmark_smoke.py`

**Modify:**
- `.gitignore`

**Notes:**
- Do **not** create `apps/web` in this POC. The KE-first branch keeps Next.js as the future frontend choice, but the benchmark itself is backend-only.
- Keep corpus state file-backed in v0. Do not introduce Postgres persistence for manifests or eval runs in this POC.

---

### Task 1: Bootstrap The FastAPI Benchmark Service And Local Infra

**Files:**
- Create: `docker-compose.yml`
- Create: `pyproject.toml`
- Create: `apps/api/Dockerfile`
- Create: `apps/api/src/prescient_benchmark/__init__.py`
- Create: `apps/api/src/prescient_benchmark/main.py`
- Create: `apps/api/src/prescient_benchmark/config.py`
- Create: `apps/api/src/prescient_benchmark/api/routes_health.py`
- Test: `tests/unit/test_health.py`
- Modify: `.gitignore`

- [ ] **Step 1: Write the failing health-route test**

```python
from fastapi.testclient import TestClient

from prescient_benchmark.main import app


def test_healthcheck_returns_ok() -> None:
    client = TestClient(app)

    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {
        "status": "ok",
        "service": "prescient-benchmark-api",
    }
```

- [ ] **Step 2: Run the test to verify the app does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_health.py -q
```

Expected:
- import failure for `prescient_benchmark.main` or missing module/file

- [ ] **Step 3: Add the minimal FastAPI app, config, and Docker/OpenSearch scaffolding**

`pyproject.toml`

```toml
[project]
name = "prescient-benchmark"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
  "beautifulsoup4>=4.12",
  "fastapi>=0.115",
  "httpx>=0.28",
  "openai>=1.75.0",
  "opensearch-py>=2.6.0",
  "pydantic>=2.11",
  "pydantic-settings>=2.8",
  "pymupdf>=1.25",
  "python-multipart>=0.0.9",
  "pyyaml>=6.0.2",
  "typer>=0.12",
  "uvicorn>=0.34",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.3",
  "pytest-asyncio>=0.25",
]

[tool.pytest.ini_options]
pythonpath = ["apps/api/src"]
testpaths = ["tests"]
```

`docker-compose.yml`

```yaml
services:
  opensearch:
    image: opensearchproject/opensearch:2.15.0
    environment:
      discovery.type: single-node
      plugins.security.disabled: "true"
      OPENSEARCH_JAVA_OPTS: "-Xms512m -Xmx512m"
    ports:
      - "9200:9200"

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    environment:
      OPENSEARCH_URL: http://opensearch:9200
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      ANSWER_MODEL: gpt-5.4-mini
      EMBEDDING_MODEL: text-embedding-3-large
      CORPUS_ROOT: /workspace/corpus
      EVAL_ROOT: /workspace/eval
    volumes:
      - .:/workspace
    working_dir: /workspace
    command: uv run uvicorn apps.api.src.prescient_benchmark.main:app --host 0.0.0.0 --port 8000
    ports:
      - "8000:8000"
    depends_on:
      - opensearch
```

`apps/api/src/prescient_benchmark/config.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    opensearch_url: str = "http://localhost:9200"
    answer_model: str = "gpt-5.4-mini"
    embedding_model: str = "text-embedding-3-large"
    corpus_root: str = "corpus"
    eval_root: str = "eval"


settings = Settings()
```

`apps/api/src/prescient_benchmark/api/routes_health.py`

```python
from fastapi import APIRouter

router = APIRouter()


@router.get("/health")
def healthcheck() -> dict[str, str]:
    return {
        "status": "ok",
        "service": "prescient-benchmark-api",
    }
```

`apps/api/src/prescient_benchmark/main.py`

```python
from fastapi import FastAPI

from prescient_benchmark.api.routes_health import router as health_router

app = FastAPI(title="Prescient Benchmark API")
app.include_router(health_router)
```

`apps/api/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /workspace

RUN pip install --no-cache-dir uv

COPY pyproject.toml /workspace/pyproject.toml
RUN uv sync --frozen || uv sync
```

`.gitignore`

```gitignore
.venv
__pycache__/
.pytest_cache/
.mypy_cache/
apps/api/.venv
apps/api/.pytest_cache
apps/web/node_modules
apps/web/.next
eval/results/
corpus/generated/
```

- [ ] **Step 4: Run unit test and compose config verification**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_health.py -q
docker compose config >/dev/null
```

Expected:
- health test passes
- compose config exits `0`

- [ ] **Step 5: Commit**

```bash
git add .gitignore pyproject.toml docker-compose.yml apps/api tests/unit/test_health.py
git commit -m "feat: bootstrap benchmark api and infra"
```

---

### Task 2: Build The Corpus Manifest Model And Peloton Public Import Tool

**Files:**
- Create: `apps/api/src/prescient_benchmark/corpus/models.py`
- Create: `apps/api/src/prescient_benchmark/corpus/manifest.py`
- Create: `apps/api/src/prescient_benchmark/corpus/public_peloton.py`
- Create: `apps/api/src/prescient_benchmark/corpus/storage.py`
- Create: `corpus/scenarios/peloton_v1.yaml`
- Test: `tests/unit/test_manifest.py`
- Test: `tests/unit/test_public_peloton.py`

- [ ] **Step 1: Write failing tests for manifest loading and Peloton document planning**

`tests/unit/test_manifest.py`

```python
from pathlib import Path

from prescient_benchmark.corpus.manifest import load_scenario


def test_load_scenario_returns_named_documents() -> None:
    scenario = load_scenario(Path("corpus/scenarios/peloton_v1.yaml"))

    assert scenario.anchor_company == "Peloton"
    assert "public_company_materials" in scenario.sections
    assert "synthetic_internal_documents" in scenario.sections
```

`tests/unit/test_public_peloton.py`

```python
from prescient_benchmark.corpus.public_peloton import build_public_document_plan


def test_public_document_plan_contains_expected_peloton_sources() -> None:
    plan = build_public_document_plan()

    slugs = {doc.slug for doc in plan}

    assert "10-k-latest" in slugs
    assert "q2-fy25-investor-presentation" in slugs
    assert "q2-fy25-shareholder-letter" in slugs
```

- [ ] **Step 2: Run the tests to verify the corpus layer does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_manifest.py tests/unit/test_public_peloton.py -q
```

Expected:
- import/module failures

- [ ] **Step 3: Implement the manifest schema and Peloton public import planning**

`corpus/scenarios/peloton_v1.yaml`

```yaml
anchor_company: Peloton
scenario_id: peloton_v1
sections:
  public_company_materials:
    description: Public investor and filing materials for Peloton
  synthetic_internal_documents:
    description: Generated internal operating and strategy documents
  long_form_real_documents:
    description: Anonymized long-form dense documents supplied locally
synthetic_document_categories:
  - executive_strategy_memo
  - initiative_update
  - meeting_notes
  - operating_review_summary
  - policy_process_doc
```

`apps/api/src/prescient_benchmark/corpus/models.py`

```python
from pydantic import BaseModel


class Scenario(BaseModel):
    scenario_id: str
    anchor_company: str
    sections: dict[str, dict[str, str]]
    synthetic_document_categories: list[str]


class CorpusDocumentPlan(BaseModel):
    slug: str
    title: str
    source_url: str
    section: str
```

`apps/api/src/prescient_benchmark/corpus/manifest.py`

```python
from pathlib import Path

import yaml

from prescient_benchmark.corpus.models import Scenario


def load_scenario(path: Path) -> Scenario:
    return Scenario.model_validate(yaml.safe_load(path.read_text()))
```

`apps/api/src/prescient_benchmark/corpus/public_peloton.py`

```python
from prescient_benchmark.corpus.models import CorpusDocumentPlan


def build_public_document_plan() -> list[CorpusDocumentPlan]:
    return [
        CorpusDocumentPlan(
            slug="10-k-latest",
            title="Peloton 10-K Latest",
            source_url="https://www.sec.gov/Archives/edgar/data/1639825/",
            section="public_company_materials",
        ),
        CorpusDocumentPlan(
            slug="q2-fy25-investor-presentation",
            title="Peloton Q2 FY25 Investor Presentation",
            source_url="https://investor.onepeloton.com/",
            section="public_company_materials",
        ),
        CorpusDocumentPlan(
            slug="q2-fy25-shareholder-letter",
            title="Peloton Q2 FY25 Shareholder Letter",
            source_url="https://investor.onepeloton.com/",
            section="public_company_materials",
        ),
    ]
```

`apps/api/src/prescient_benchmark/corpus/storage.py`

```python
from pathlib import Path


def ensure_directory(path: Path) -> Path:
    path.mkdir(parents=True, exist_ok=True)
    return path
```

- [ ] **Step 4: Run unit tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_manifest.py tests/unit/test_public_peloton.py -q
```

Expected:
- both tests pass

- [ ] **Step 5: Commit**

```bash
git add corpus/scenarios/peloton_v1.yaml apps/api/src/prescient_benchmark/corpus tests/unit/test_manifest.py tests/unit/test_public_peloton.py
git commit -m "feat: add Peloton corpus manifest and public import planning"
```

---

### Task 3: Add Synthetic Internal Document Generation From Scenario Seeds

**Files:**
- Create: `apps/api/src/prescient_benchmark/corpus/synthetic_docs.py`
- Test: `tests/unit/test_synthetic_docs.py`

- [ ] **Step 1: Write the failing test for deterministic synthetic-doc generation**

```python
from pathlib import Path

from prescient_benchmark.corpus.manifest import load_scenario
from prescient_benchmark.corpus.synthetic_docs import build_generation_requests


def test_build_generation_requests_uses_seeded_categories() -> None:
    scenario = load_scenario(Path("corpus/scenarios/peloton_v1.yaml"))

    requests = build_generation_requests(scenario, seed="peloton-v1")

    assert requests[0].scenario_id == "peloton_v1"
    assert requests[0].seed == "peloton-v1"
    assert requests[0].doc_type == "executive_strategy_memo"
    assert requests[-1].doc_type == "policy_process_doc"
```

- [ ] **Step 2: Run the test to verify the generator layer does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_synthetic_docs.py -q
```

Expected:
- import failure for `synthetic_docs`

- [ ] **Step 3: Implement generation requests and one prompt template path**

`apps/api/src/prescient_benchmark/corpus/synthetic_docs.py`

```python
from pydantic import BaseModel

from prescient_benchmark.corpus.models import Scenario


class SyntheticGenerationRequest(BaseModel):
    scenario_id: str
    seed: str
    doc_type: str
    prompt: str


def build_generation_requests(
    scenario: Scenario,
    *,
    seed: str,
) -> list[SyntheticGenerationRequest]:
    return [
        SyntheticGenerationRequest(
            scenario_id=scenario.scenario_id,
            seed=seed,
            doc_type=doc_type,
            prompt=(
                f"Generate a realistic {doc_type} for {scenario.anchor_company}. "
                f"Use seed '{seed}'. Keep it grounded in a plausible operating context."
            ),
        )
        for doc_type in scenario.synthetic_document_categories
    ]
```

- [ ] **Step 4: Run the synthetic-doc test**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_synthetic_docs.py -q
```

Expected:
- test passes

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/corpus/synthetic_docs.py tests/unit/test_synthetic_docs.py
git commit -m "feat: add seeded synthetic document generation planning"
```

---

### Task 4: Build Parsing, Chunking, And OpenSearch Hybrid Indexing

**Files:**
- Create: `apps/api/src/prescient_benchmark/retrieval/models.py`
- Create: `apps/api/src/prescient_benchmark/retrieval/parse.py`
- Create: `apps/api/src/prescient_benchmark/retrieval/chunk.py`
- Create: `apps/api/src/prescient_benchmark/retrieval/index.py`
- Create: `apps/api/src/prescient_benchmark/retrieval/search.py`
- Create: `apps/api/src/prescient_benchmark/retrieval/rerank.py`
- Test: `tests/unit/test_chunking.py`
- Test: `tests/integration/test_opensearch_ingest_and_search.py`

- [ ] **Step 1: Write a failing chunking test**

```python
from prescient_benchmark.retrieval.chunk import semantic_chunks


def test_semantic_chunks_preserve_document_identity() -> None:
    chunks = semantic_chunks(
        doc_id="peloton-shareholder-letter",
        title="Shareholder Letter",
        text="Section one.\n\nSection two with more detail.\n\nSection three.",
        target_tokens=40,
    )

    assert len(chunks) >= 2
    assert all(chunk.doc_id == "peloton-shareholder-letter" for chunk in chunks)
    assert chunks[0].title == "Shareholder Letter"
```

- [ ] **Step 2: Write a failing OpenSearch integration test**

```python
from prescient_benchmark.retrieval.index import InMemoryDocument
from prescient_benchmark.retrieval.search import hybrid_search


def test_hybrid_search_returns_relevant_chunk(opensearch_client) -> None:
    docs = [
        InMemoryDocument(
            doc_id="operating-agreement",
            title="Operating Agreement",
            text="Transfer restrictions require manager approval before assignment.",
        ),
        InMemoryDocument(
            doc_id="insurance-policy",
            title="Insurance Policy",
            text="Coverage extends to business interruption after named perils.",
        ),
    ]

    results = hybrid_search(
        client=opensearch_client,
        index_name="benchmark-test",
        documents=docs,
        query="What does the operating agreement say about transfer restrictions?",
    )

    assert results[0].doc_id == "operating-agreement"
```

- [ ] **Step 3: Run the tests to verify the retrieval layer does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_chunking.py -q
```

Expected:
- import failure for retrieval modules

- [ ] **Step 4: Implement parsing, chunking, and hybrid retrieval primitives**

`apps/api/src/prescient_benchmark/retrieval/models.py`

```python
from pydantic import BaseModel


class Chunk(BaseModel):
    chunk_id: str
    doc_id: str
    title: str
    text: str
    ordinal: int


class SearchResult(BaseModel):
    chunk_id: str
    doc_id: str
    title: str
    text: str
    score: float
```

`apps/api/src/prescient_benchmark/retrieval/chunk.py`

```python
from prescient_benchmark.retrieval.models import Chunk


def semantic_chunks(
    *,
    doc_id: str,
    title: str,
    text: str,
    target_tokens: int = 400,
) -> list[Chunk]:
    paragraphs = [part.strip() for part in text.split("\n\n") if part.strip()]
    chunks: list[Chunk] = []
    current: list[str] = []
    ordinal = 0

    for paragraph in paragraphs:
        current.append(paragraph)
        if sum(len(part.split()) for part in current) >= target_tokens:
            chunks.append(
                Chunk(
                    chunk_id=f"{doc_id}:{ordinal}",
                    doc_id=doc_id,
                    title=title,
                    text="\n\n".join(current),
                    ordinal=ordinal,
                )
            )
            ordinal += 1
            current = []

    if current:
        chunks.append(
            Chunk(
                chunk_id=f"{doc_id}:{ordinal}",
                doc_id=doc_id,
                title=title,
                text="\n\n".join(current),
                ordinal=ordinal,
            )
        )

    return chunks
```

`apps/api/src/prescient_benchmark/retrieval/index.py`

```python
from pydantic import BaseModel


class InMemoryDocument(BaseModel):
    doc_id: str
    title: str
    text: str
```

`apps/api/src/prescient_benchmark/retrieval/search.py`

```python
from opensearchpy import OpenSearch

from prescient_benchmark.retrieval.index import InMemoryDocument
from prescient_benchmark.retrieval.models import SearchResult


def hybrid_search(
    *,
    client: OpenSearch,
    index_name: str,
    documents: list[InMemoryDocument],
    query: str,
) -> list[SearchResult]:
    # Minimal integration-test path for v0: create, index, and run BM25 search.
    if not client.indices.exists(index=index_name):
        client.indices.create(
            index=index_name,
            body={
                "mappings": {
                    "properties": {
                        "doc_id": {"type": "keyword"},
                        "title": {"type": "text"},
                        "text": {"type": "text"},
                    }
                }
            },
        )

    for doc in documents:
        client.index(index=index_name, body=doc.model_dump())
    client.indices.refresh(index=index_name)

    hits = client.search(
        index=index_name,
        body={"query": {"multi_match": {"query": query, "fields": ["title^2", "text"]}}},
    )["hits"]["hits"]

    return [
        SearchResult(
            chunk_id=hit["_id"],
            doc_id=hit["_source"]["doc_id"],
            title=hit["_source"]["title"],
            text=hit["_source"]["text"],
            score=float(hit["_score"]),
        )
        for hit in hits
    ]
```

`apps/api/src/prescient_benchmark/retrieval/rerank.py`

```python
from prescient_benchmark.retrieval.models import SearchResult


def rerank(results: list[SearchResult]) -> list[SearchResult]:
    return sorted(results, key=lambda result: result.score, reverse=True)
```

- [ ] **Step 5: Run unit and integration tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
docker compose up -d opensearch
uv run pytest tests/unit/test_chunking.py -q
uv run pytest tests/integration/test_opensearch_ingest_and_search.py -q
```

Expected:
- chunking test passes
- OpenSearch integration test passes

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/retrieval tests/unit/test_chunking.py tests/integration/test_opensearch_ingest_and_search.py
git commit -m "feat: add retrieval chunking and OpenSearch search path"
```

---

### Task 5: Add Synthesized Answers With Strict Citations

**Files:**
- Create: `apps/api/src/prescient_benchmark/retrieval/answer.py`
- Create: `apps/api/src/prescient_benchmark/api/routes_ask.py`
- Modify: `apps/api/src/prescient_benchmark/main.py`
- Test: `tests/unit/test_answering.py`

- [ ] **Step 1: Write the failing answer-service test**

```python
from prescient_benchmark.retrieval.answer import synthesize_answer
from prescient_benchmark.retrieval.models import SearchResult


def test_synthesize_answer_requires_citations() -> None:
    results = [
        SearchResult(
            chunk_id="agreement:0",
            doc_id="agreement",
            title="Operating Agreement",
            text="Transfer restrictions require manager approval before assignment.",
            score=1.0,
        )
    ]

    answer = synthesize_answer(
        question="Can units be assigned freely?",
        evidence=results,
    )

    assert "manager approval" in answer.text.lower()
    assert answer.citations == [
        {
            "doc_id": "agreement",
            "chunk_id": "agreement:0",
            "title": "Operating Agreement",
        }
    ]
```

- [ ] **Step 2: Run the test to verify the answer layer does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_answering.py -q
```

Expected:
- import failure for `answer`

- [ ] **Step 3: Implement the strict-citation answer shape and `/ask` route**

`apps/api/src/prescient_benchmark/retrieval/answer.py`

```python
from pydantic import BaseModel

from prescient_benchmark.retrieval.models import SearchResult


class Answer(BaseModel):
    text: str
    citations: list[dict[str, str]]


def synthesize_answer(
    *,
    question: str,
    evidence: list[SearchResult],
) -> Answer:
    if not evidence:
        return Answer(
            text="Insufficient evidence to answer from the indexed corpus.",
            citations=[],
        )

    top = evidence[0]
    return Answer(
        text=f"Based on {top.title}, {top.text}",
        citations=[
            {
                "doc_id": top.doc_id,
                "chunk_id": top.chunk_id,
                "title": top.title,
            }
        ],
    )
```

`apps/api/src/prescient_benchmark/api/routes_ask.py`

```python
from fastapi import APIRouter
from pydantic import BaseModel

from prescient_benchmark.retrieval.answer import synthesize_answer
from prescient_benchmark.retrieval.models import SearchResult

router = APIRouter()


class AskRequest(BaseModel):
    question: str


@router.post("/ask")
def ask(request: AskRequest) -> dict:
    placeholder_evidence = [
        SearchResult(
            chunk_id="placeholder:0",
            doc_id="placeholder",
            title="Placeholder Document",
            text="No retrieval pipeline connected yet.",
            score=1.0,
        )
    ]
    answer = synthesize_answer(question=request.question, evidence=placeholder_evidence)
    return answer.model_dump()
```

`apps/api/src/prescient_benchmark/main.py`

```python
from fastapi import FastAPI

from prescient_benchmark.api.routes_ask import router as ask_router
from prescient_benchmark.api.routes_health import router as health_router

app = FastAPI(title="Prescient Benchmark API")
app.include_router(health_router)
app.include_router(ask_router)
```

- [ ] **Step 4: Run the answer test**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_answering.py -q
```

Expected:
- test passes

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/retrieval/answer.py apps/api/src/prescient_benchmark/api/routes_ask.py apps/api/src/prescient_benchmark/main.py tests/unit/test_answering.py
git commit -m "feat: add synthesized answering with strict citations"
```

---

### Task 6: Add Eval Questions, Export Workflow, And Human Scorecards

**Files:**
- Create: `apps/api/src/prescient_benchmark/eval/models.py`
- Create: `apps/api/src/prescient_benchmark/eval/export.py`
- Create: `apps/api/src/prescient_benchmark/eval/scorecard.py`
- Create: `eval/questions/peloton_v1.yaml`
- Test: `tests/unit/test_eval_export.py`

- [ ] **Step 1: Write the failing export test**

```python
from pathlib import Path

from prescient_benchmark.eval.export import export_question_packet


def test_export_question_packet_writes_markdown_bundle(tmp_path: Path) -> None:
    output = export_question_packet(
        question_id="q1",
        prompt="What are the major transfer restrictions?",
        output_dir=tmp_path,
    )

    assert output.exists()
    assert "What are the major transfer restrictions?" in output.read_text()
```

- [ ] **Step 2: Run the test to verify the eval layer does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_eval_export.py -q
```

Expected:
- import failure for `eval.export`

- [ ] **Step 3: Implement the eval question schema and export tools**

`eval/questions/peloton_v1.yaml`

```yaml
questions:
  - id: q1
    category: dense_long_document
    prompt: What are the major transfer restrictions in the operating agreement?
  - id: q2
    category: operator_business
    prompt: What seem to be Peloton's current top operating risks across public and internal materials?
  - id: q3
    category: fact_lookup_and_synthesis
    prompt: What do the public materials say about subscription trends and how does internal planning frame them?
```

`apps/api/src/prescient_benchmark/eval/models.py`

```python
from pydantic import BaseModel


class EvalQuestion(BaseModel):
    id: str
    category: str
    prompt: str
```

`apps/api/src/prescient_benchmark/eval/export.py`

```python
from pathlib import Path


def export_question_packet(
    *,
    question_id: str,
    prompt: str,
    output_dir: Path,
) -> Path:
    output_dir.mkdir(parents=True, exist_ok=True)
    output = output_dir / f"{question_id}.md"
    output.write_text(f"# {question_id}\n\n{prompt}\n")
    return output
```

`apps/api/src/prescient_benchmark/eval/scorecard.py`

```python
from pathlib import Path


def write_scorecard_template(*, output_dir: Path) -> Path:
    output_dir.mkdir(parents=True, exist_ok=True)
    scorecard = output_dir / "scorecard.csv"
    scorecard.write_text(
        "question_id,system,groundedness,usefulness,specificity,completeness,long_doc_handling,notes\n"
    )
    return scorecard
```

- [ ] **Step 4: Run the eval export test**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_eval_export.py -q
```

Expected:
- test passes

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/eval eval/questions/peloton_v1.yaml tests/unit/test_eval_export.py
git commit -m "feat: add eval question export and scorecard tooling"
```

---

### Task 7: Add The CLI Filesystem Baseline Runner

**Files:**
- Create: `apps/api/src/prescient_benchmark/cli.py`
- Test: `tests/unit/test_cli_baseline.py`

- [ ] **Step 1: Write the failing CLI baseline test**

```python
from typer.testing import CliRunner

from prescient_benchmark.cli import app


def test_cli_baseline_searches_raw_files(tmp_path) -> None:
    doc = tmp_path / "agreement.txt"
    doc.write_text("Transfer restrictions require manager approval before assignment.")

    runner = CliRunner()
    result = runner.invoke(
        app,
        [
            "answer-from-files",
            "--corpus-root",
            str(tmp_path),
            "--question",
            "What do transfer restrictions require?",
        ],
    )

    assert result.exit_code == 0
    assert "manager approval" in result.stdout.lower()
```

- [ ] **Step 2: Run the test to verify the CLI baseline does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_cli_baseline.py -q
```

Expected:
- import failure for `cli`

- [ ] **Step 3: Implement the raw-filesystem CLI baseline**

`apps/api/src/prescient_benchmark/cli.py`

```python
from pathlib import Path

import typer

app = typer.Typer()


@app.command("answer-from-files")
def answer_from_files(*, corpus_root: Path, question: str) -> None:
    lowered = question.lower()
    for path in corpus_root.rglob("*"):
        if not path.is_file():
            continue
        text = path.read_text(errors="ignore")
        if any(token in text.lower() for token in lowered.split()):
            typer.echo(f"{path.name}: {text}")
            return

    typer.echo("Insufficient evidence in raw files.")
```

- [ ] **Step 4: Run the CLI baseline test**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/unit/test_cli_baseline.py -q
```

Expected:
- test passes

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/cli.py tests/unit/test_cli_baseline.py
git commit -m "feat: add raw filesystem cli baseline"
```

---

### Task 8: End-To-End Benchmark Smoke Test

**Files:**
- Create: `apps/api/src/prescient_benchmark/api/routes_corpus.py`
- Modify: `apps/api/src/prescient_benchmark/api/routes_ask.py`
- Test: `tests/integration/test_benchmark_smoke.py`

- [ ] **Step 1: Write the failing smoke test**

```python
from fastapi.testclient import TestClient

from prescient_benchmark.main import app


def test_benchmark_smoke_flow() -> None:
    client = TestClient(app)

    response = client.post("/ask", json={"question": "What does the operating agreement say about transfer restrictions?"})

    assert response.status_code == 200
    body = response.json()
    assert "citations" in body
    assert isinstance(body["citations"], list)
```

- [ ] **Step 2: Run the smoke test to verify the route is still placeholder quality**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run pytest tests/integration/test_benchmark_smoke.py -q
```

Expected:
- test fails because the route is still placeholder or not wired to retrieval data

- [ ] **Step 3: Wire `/ask` through the retrieval path with corpus fixtures**

Implement the minimal benchmark path:

```python
# route shape target
@router.post("/ask")
def ask(request: AskRequest) -> dict:
    evidence = search_local_index(question=request.question, top_k=8)
    reranked = rerank(evidence)[:5]
    answer = synthesize_answer(question=request.question, evidence=reranked)
    return answer.model_dump()
```

Also add a small corpus-loading route or startup fixture path so the integration test can seed a few files before asking a question.

- [ ] **Step 4: Run the full benchmark smoke test set**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
docker compose up -d opensearch
uv run pytest tests/unit -q
uv run pytest tests/integration -q
```

Expected:
- unit tests pass
- integration tests pass

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/api tests/integration/test_benchmark_smoke.py
git commit -m "feat: wire end-to-end benchmark smoke path"
```
