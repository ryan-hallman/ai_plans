# Peloton Public Corpus Materialization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a repeatable Peloton public-corpus materialization flow that fetches public documents, normalizes them, and freezes raw files plus extracted text and manifest metadata under `corpus/generated/peloton_v1/public/`.

**Architecture:** Keep the slice code-native and explicit. Extend the existing `prescient_benchmark.corpus` package with focused modules for EDGAR fetch, investor-relations fetch, normalization, and freeze/write logic. Drive the end-to-end flow through a CLI command so the generated corpus on disk becomes the system of record for later benchmark work.

**Tech Stack:** Python 3.12, Typer, httpx, PyMuPDF, BeautifulSoup4, PyYAML, pytest, FastAPI repo scaffolding already present

---

## File Structure

**Create:**
- `apps/api/src/prescient_benchmark/corpus/edgar.py`
- `apps/api/src/prescient_benchmark/corpus/investor_relations.py`
- `apps/api/src/prescient_benchmark/corpus/normalize_public.py`
- `apps/api/src/prescient_benchmark/corpus/materialize_public.py`
- `tests/unit/test_edgar_fetch.py`
- `tests/unit/test_investor_relations.py`
- `tests/unit/test_normalize_public.py`
- `tests/unit/test_materialize_public.py`

**Modify:**
- `apps/api/src/prescient_benchmark/cli.py`
- `apps/api/src/prescient_benchmark/corpus/models.py`
- `apps/api/src/prescient_benchmark/corpus/public_peloton.py`
- `apps/api/src/prescient_benchmark/corpus/storage.py`
- `tests/unit/test_public_peloton.py`

**Notes:**
- Do not add networked integration tests in this slice. Keep tests deterministic with mocked `httpx` responses and local fixture files.
- Do not index into OpenSearch in this slice.
- Do not wire a new API route yet. The materialization entry point is the CLI.

---

### Task 1: Expand Corpus Models And Freeze Storage Helpers

**Files:**
- Modify: `apps/api/src/prescient_benchmark/corpus/models.py`
- Modify: `apps/api/src/prescient_benchmark/corpus/public_peloton.py`
- Modify: `apps/api/src/prescient_benchmark/corpus/storage.py`
- Modify: `tests/unit/test_public_peloton.py`

- [ ] **Step 1: Write the failing plan-shape tests**

`tests/unit/test_public_peloton.py`

```python
from pathlib import Path

import pytest
from pydantic import ValidationError

from prescient_benchmark.corpus.models import CorpusDocumentPlan, FrozenDocumentRecord
from prescient_benchmark.corpus.public_peloton import build_public_document_plan
from prescient_benchmark.corpus.storage import generated_public_root


def test_build_public_document_plan_returns_target_public_documents() -> None:
    plan = build_public_document_plan()

    assert [document.slug for document in plan] == [
        "10-k-latest",
        "10-q-latest",
        "10-q-prior",
        "shareholder-letter-latest",
        "investor-presentation-latest",
    ]
    assert plan[0].source_type == "edgar"
    assert plan[-1].source_type == "investor_relations"


def test_generated_public_root_points_to_versioned_public_directory() -> None:
    root = generated_public_root("peloton_v1")

    assert root.as_posix().endswith("corpus/generated/peloton_v1/public")


def test_frozen_document_record_rejects_scalar_coercion() -> None:
    with pytest.raises(ValidationError):
        FrozenDocumentRecord.model_validate(
            {
                "document_id": 1,
                "title": "Peloton 10-K Latest",
                "source_type": "edgar",
                "source_url": "https://www.sec.gov/Archives/edgar/data/1639825/",
                "raw_path": "corpus/generated/peloton_v1/public/raw/10-k.html",
                "normalized_path": "corpus/generated/peloton_v1/public/normalized/10-k.yaml",
                "content_hash": "abc",
                "extractor_version": "v1",
            }
        )
```

- [ ] **Step 2: Run the tests to verify the new types and helpers do not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_public_peloton.py -q
```

Expected:
- failures for missing `FrozenDocumentRecord`, missing `generated_public_root`, and missing `source_type` on `CorpusDocumentPlan`

- [ ] **Step 3: Add the public-corpus planning and frozen-record models**

`apps/api/src/prescient_benchmark/corpus/models.py`

```python
from pydantic import BaseModel, ConfigDict, StrictStr


class Scenario(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    scenario_id: StrictStr
    anchor_company: StrictStr
    sections: dict[StrictStr, dict[StrictStr, StrictStr]]
    synthetic_document_categories: list[StrictStr]


class CorpusDocumentPlan(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    slug: StrictStr
    title: StrictStr
    source_type: StrictStr
    source_url: StrictStr
    section: StrictStr


class FrozenDocumentRecord(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    document_id: StrictStr
    title: StrictStr
    source_type: StrictStr
    source_url: StrictStr
    raw_path: StrictStr
    normalized_path: StrictStr
    content_hash: StrictStr
    extractor_version: StrictStr
    filing_or_report_date: StrictStr | None = None
    form_type: StrictStr | None = None
```

`apps/api/src/prescient_benchmark/corpus/public_peloton.py`

```python
from prescient_benchmark.corpus.models import CorpusDocumentPlan


def build_public_document_plan() -> list[CorpusDocumentPlan]:
    return [
        CorpusDocumentPlan(
            slug="10-k-latest",
            title="Peloton 10-K Latest",
            source_type="edgar",
            source_url="https://www.sec.gov/Archives/edgar/data/1639825/",
            section="public_company_materials",
        ),
        CorpusDocumentPlan(
            slug="10-q-latest",
            title="Peloton 10-Q Latest",
            source_type="edgar",
            source_url="https://www.sec.gov/Archives/edgar/data/1639825/",
            section="public_company_materials",
        ),
        CorpusDocumentPlan(
            slug="10-q-prior",
            title="Peloton 10-Q Prior",
            source_type="edgar",
            source_url="https://www.sec.gov/Archives/edgar/data/1639825/",
            section="public_company_materials",
        ),
        CorpusDocumentPlan(
            slug="shareholder-letter-latest",
            title="Peloton Shareholder Letter Latest",
            source_type="investor_relations",
            source_url="https://investor.onepeloton.com/",
            section="public_company_materials",
        ),
        CorpusDocumentPlan(
            slug="investor-presentation-latest",
            title="Peloton Investor Presentation Latest",
            source_type="investor_relations",
            source_url="https://investor.onepeloton.com/",
            section="public_company_materials",
        ),
    ]
```

`apps/api/src/prescient_benchmark/corpus/storage.py`

```python
import os
from pathlib import Path


def _repo_root() -> Path:
    for parent in Path(__file__).resolve().parents:
        candidate = parent / "corpus" / "scenarios"
        if candidate.is_dir():
            return candidate.parent.parent.resolve()
    raise RuntimeError("unable to locate repository root")


def _scenarios_root() -> Path:
    return (_repo_root() / "corpus" / "scenarios").resolve()


def generated_public_root(version: str) -> Path:
    return (_repo_root() / "corpus" / "generated" / version / "public").resolve()


def normalize_scenario_path(path: Path) -> Path:
    if path.suffix not in {".yaml", ".yml"}:
        raise ValueError(f"scenario manifest must be a YAML file: {path}")

    scenarios_root = _scenarios_root()
    repo_root = scenarios_root.parent.parent
    resolved_path = (
        path.resolve(strict=False)
        if path.is_absolute()
        else (repo_root / path).resolve(strict=False)
    )

    if scenarios_root not in resolved_path.parents:
        raise ValueError(f"scenario manifest must live under corpus/scenarios: {path}")

    if not resolved_path.exists():
        raise ValueError(f"scenario manifest does not exist: {path}")

    if not resolved_path.is_file():
        raise ValueError(f"scenario manifest must be a file: {path}")

    if not os.access(resolved_path, os.R_OK):
        raise ValueError(f"scenario manifest must be readable: {path}")

    return resolved_path
```

- [ ] **Step 4: Run the public-plan test file**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_public_peloton.py -q
```

Expected:
- tests pass

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/corpus/models.py apps/api/src/prescient_benchmark/corpus/public_peloton.py apps/api/src/prescient_benchmark/corpus/storage.py tests/unit/test_public_peloton.py
git commit -m "feat: define peloton public corpus records"
```

---

### Task 2: Port A Minimal EDGAR Fetch Client

**Files:**
- Create: `apps/api/src/prescient_benchmark/corpus/edgar.py`
- Test: `tests/unit/test_edgar_fetch.py`

- [ ] **Step 1: Write the failing EDGAR fetch tests**

`tests/unit/test_edgar_fetch.py`

```python
from datetime import date

import httpx

from prescient_benchmark.corpus.edgar import EdgarClient, FilingRef


def test_list_filings_picks_requested_recent_forms(tmp_path) -> None:
    client = EdgarClient(
        user_agent="Prescient Benchmark test@test.local",
        cache_root=tmp_path,
    )
    payload = {
        "filings": {
            "recent": {
                "accessionNumber": ["0001-000001", "0001-000002", "0001-000003"],
                "filingDate": ["2025-02-10", "2024-11-07", "2024-08-22"],
                "reportDate": ["2024-12-31", "2024-09-30", "2024-06-30"],
                "form": ["10-K", "10-Q", "8-K"],
                "primaryDocument": ["peloton-10k.htm", "peloton-10q.htm", "peloton-8k.htm"],
            }
        }
    }

    refs = client._parse_recent_filings("0001639825", payload, {"10-K": 1, "10-Q": 1})

    assert refs == [
        FilingRef(
            cik="0001639825",
            accession_number="0001-000001",
            form="10-K",
            filing_date=date(2025, 2, 10),
            primary_document="peloton-10k.htm",
            report_date=date(2024, 12, 31),
        ),
        FilingRef(
            cik="0001639825",
            accession_number="0001-000002",
            form="10-Q",
            filing_date=date(2024, 11, 7),
            primary_document="peloton-10q.htm",
            report_date=date(2024, 9, 30),
        ),
    ]


def test_fetch_filing_writes_cached_html(monkeypatch, tmp_path) -> None:
    transport = httpx.MockTransport(
        lambda request: httpx.Response(200, content=b"<html><body>Peloton filing</body></html>")
    )
    client = EdgarClient(
        user_agent="Prescient Benchmark test@test.local",
        cache_root=tmp_path,
        http_client=httpx.Client(transport=transport),
    )

    ref = FilingRef(
        cik="0001639825",
        accession_number="0001-000001",
        form="10-K",
        filing_date=date(2025, 2, 10),
        primary_document="peloton-10k.htm",
        report_date=date(2024, 12, 31),
    )

    path = client.fetch_filing(ref)

    assert path.exists()
    assert "Peloton filing" in path.read_text()
```

- [ ] **Step 2: Run the EDGAR tests to verify the module does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_edgar_fetch.py -q
```

Expected:
- import failure for `prescient_benchmark.corpus.edgar`

- [ ] **Step 3: Implement the minimal synchronous EDGAR client**

`apps/api/src/prescient_benchmark/corpus/edgar.py`

```python
from __future__ import annotations

import json
from dataclasses import dataclass
from datetime import date
from pathlib import Path
from typing import Any

import httpx


@dataclass(frozen=True, slots=True)
class FilingRef:
    cik: str
    accession_number: str
    form: str
    filing_date: date
    primary_document: str
    report_date: date | None

    @property
    def accession_no_dashes(self) -> str:
        return self.accession_number.replace("-", "")

    @property
    def cik_no_leading_zeros(self) -> str:
        return str(int(self.cik))


class EdgarClient:
    SUBMISSIONS_URL = "https://data.sec.gov/submissions/CIK{cik}.json"
    ARCHIVE_URL = "https://www.sec.gov/Archives/edgar/data/{cik}/{accession}/{document}"

    def __init__(
        self,
        *,
        user_agent: str,
        cache_root: Path,
        http_client: httpx.Client | None = None,
    ) -> None:
        self._cache_root = cache_root
        self._cache_root.mkdir(parents=True, exist_ok=True)
        self._client = http_client or httpx.Client(
            headers={"User-Agent": user_agent, "Accept-Encoding": "gzip, deflate"},
            timeout=httpx.Timeout(30.0, connect=10.0),
        )

    def fetch_submissions(self, cik: str, *, refresh: bool = False) -> dict[str, Any]:
        cache_path = self._cache_root / cik / "submissions.json"
        if cache_path.exists() and not refresh:
            return json.loads(cache_path.read_text(encoding="utf-8"))

        response = self._client.get(self.SUBMISSIONS_URL.format(cik=cik))
        response.raise_for_status()
        cache_path.parent.mkdir(parents=True, exist_ok=True)
        cache_path.write_bytes(response.content)
        return response.json()

    def _parse_recent_filings(
        self,
        cik: str,
        submissions: dict[str, Any],
        form_counts: dict[str, int],
    ) -> list[FilingRef]:
        recent = submissions.get("filings", {}).get("recent", {})
        keys = ["accessionNumber", "filingDate", "reportDate", "form", "primaryDocument"]
        rows = zip(*(recent.get(key, []) for key in keys), strict=False)

        results: list[FilingRef] = []
        remaining = dict(form_counts)
        for accession, filing_date, report_date, form, primary_document in rows:
            if form not in remaining or remaining[form] <= 0:
                continue
            parsed_report_date = date.fromisoformat(report_date) if report_date else None
            results.append(
                FilingRef(
                    cik=cik,
                    accession_number=accession,
                    form=form,
                    filing_date=date.fromisoformat(filing_date),
                    primary_document=primary_document,
                    report_date=parsed_report_date,
                )
            )
            remaining[form] -= 1
            if all(count <= 0 for count in remaining.values()):
                break
        return results

    def list_filings(
        self,
        cik: str,
        *,
        form_counts: dict[str, int],
        refresh: bool = False,
    ) -> list[FilingRef]:
        submissions = self.fetch_submissions(cik, refresh=refresh)
        return self._parse_recent_filings(cik, submissions, form_counts)

    def fetch_filing(self, filing: FilingRef, *, refresh: bool = False) -> Path:
        cache_dir = self._cache_root / filing.cik / filing.accession_no_dashes
        cache_dir.mkdir(parents=True, exist_ok=True)
        local_path = cache_dir / filing.primary_document
        if local_path.exists() and not refresh:
            return local_path

        response = self._client.get(
            self.ARCHIVE_URL.format(
                cik=filing.cik_no_leading_zeros,
                accession=filing.accession_no_dashes,
                document=filing.primary_document,
            )
        )
        response.raise_for_status()
        local_path.write_bytes(response.content)
        return local_path
```

- [ ] **Step 4: Run the EDGAR tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_edgar_fetch.py -q
```

Expected:
- tests pass

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/corpus/edgar.py tests/unit/test_edgar_fetch.py
git commit -m "feat: add edgar fetch client for benchmark corpus"
```

---

### Task 3: Add Investor-Relations Fetch And Normalization

**Files:**
- Create: `apps/api/src/prescient_benchmark/corpus/investor_relations.py`
- Create: `apps/api/src/prescient_benchmark/corpus/normalize_public.py`
- Test: `tests/unit/test_investor_relations.py`
- Test: `tests/unit/test_normalize_public.py`

- [ ] **Step 1: Write the failing IR and normalization tests**

`tests/unit/test_investor_relations.py`

```python
from pathlib import Path

import httpx

from prescient_benchmark.corpus.investor_relations import discover_latest_ir_documents


def test_discover_latest_ir_documents_finds_direct_document_links() -> None:
    html = """
    <html><body>
      <a href="https://investor.onepeloton.com/files/presentation.pdf">Q2 FY25 Investor Presentation</a>
      <a href="https://investor.onepeloton.com/files/shareholder-letter.pdf">Q2 FY25 Shareholder Letter</a>
    </body></html>
    """
    transport = httpx.MockTransport(lambda request: httpx.Response(200, text=html))
    client = httpx.Client(transport=transport)

    docs = discover_latest_ir_documents(
        base_url="https://investor.onepeloton.com/",
        http_client=client,
    )

    assert docs["investor-presentation-latest"].endswith("presentation.pdf")
    assert docs["shareholder-letter-latest"].endswith("shareholder-letter.pdf")
```

`tests/unit/test_normalize_public.py`

```python
from pathlib import Path

from prescient_benchmark.corpus.normalize_public import normalize_public_document


def test_normalize_public_document_reads_html_text(tmp_path: Path) -> None:
    raw_path = tmp_path / "peloton-10k.htm"
    raw_path.write_text("<html><body><h1>Business</h1><p>Connected fitness subscriptions grew year over year.</p></body></html>")

    normalized = normalize_public_document(
        raw_path=raw_path,
        title="Peloton 10-K",
        source_type="edgar",
    )

    assert normalized["title"] == "Peloton 10-K"
    assert "Connected fitness subscriptions grew year over year." in normalized["text"]
    assert normalized["extractor_version"] == "public-normalize-v1"
```

- [ ] **Step 2: Run the tests to verify these modules do not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_investor_relations.py tests/unit/test_normalize_public.py -q
```

Expected:
- import failures for missing modules

- [ ] **Step 3: Implement investor-relations discovery and normalization**

`apps/api/src/prescient_benchmark/corpus/investor_relations.py`

```python
from urllib.parse import urljoin

import httpx
from bs4 import BeautifulSoup


def discover_latest_ir_documents(
    *,
    base_url: str,
    http_client: httpx.Client,
) -> dict[str, str]:
    response = http_client.get(base_url)
    response.raise_for_status()
    soup = BeautifulSoup(response.text, "html.parser")

    discovered: dict[str, str] = {}
    for link in soup.find_all("a", href=True):
        href = link["href"]
        label = " ".join(link.get_text(" ", strip=True).lower().split())
        absolute_url = urljoin(base_url, href)
        if "presentation" in label and "investor-presentation-latest" not in discovered:
            discovered["investor-presentation-latest"] = absolute_url
        if "shareholder letter" in label and "shareholder-letter-latest" not in discovered:
            discovered["shareholder-letter-latest"] = absolute_url
    return discovered
```

`apps/api/src/prescient_benchmark/corpus/normalize_public.py`

```python
from pathlib import Path

import fitz
from bs4 import BeautifulSoup


EXTRACTOR_VERSION = "public-normalize-v1"


def _normalize_html(raw_bytes: bytes) -> str:
    soup = BeautifulSoup(raw_bytes, "html.parser")
    for tag in soup(["script", "style", "noscript"]):
        tag.decompose()
    return " ".join(soup.get_text("\n", strip=True).split())


def _normalize_pdf(raw_path: Path) -> str:
    with fitz.open(raw_path) as document:
        return "\n".join(page.get_text("text").strip() for page in document)


def normalize_public_document(
    *,
    raw_path: Path,
    title: str,
    source_type: str,
) -> dict[str, str]:
    if raw_path.suffix.lower() in {".htm", ".html", ".xhtml"}:
        text = _normalize_html(raw_path.read_bytes())
    elif raw_path.suffix.lower() == ".pdf":
        text = _normalize_pdf(raw_path)
    else:
        text = raw_path.read_text(errors="ignore")

    return {
        "title": title,
        "source_type": source_type,
        "text": text,
        "extractor_version": EXTRACTOR_VERSION,
    }
```

- [ ] **Step 4: Run the IR and normalization tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_investor_relations.py tests/unit/test_normalize_public.py -q
```

Expected:
- tests pass

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/corpus/investor_relations.py apps/api/src/prescient_benchmark/corpus/normalize_public.py tests/unit/test_investor_relations.py tests/unit/test_normalize_public.py
git commit -m "feat: add public document discovery and normalization"
```

---

### Task 4: Materialize And Freeze The Peloton Public Corpus

**Files:**
- Create: `apps/api/src/prescient_benchmark/corpus/materialize_public.py`
- Modify: `apps/api/src/prescient_benchmark/cli.py`
- Test: `tests/unit/test_materialize_public.py`

- [ ] **Step 1: Write the failing materialization test**

`tests/unit/test_materialize_public.py`

```python
from pathlib import Path

import yaml

from prescient_benchmark.corpus.edgar import FilingRef
from prescient_benchmark.corpus.materialize_public import freeze_public_corpus


def test_freeze_public_corpus_writes_raw_normalized_and_manifest(tmp_path: Path) -> None:
    raw_source = tmp_path / "cache"
    raw_source.mkdir()
    filing_path = raw_source / "peloton-10k.htm"
    filing_path.write_text("<html><body>Peloton filing text</body></html>")

    output_root = tmp_path / "generated/peloton_v1/public"
    manifest_path = freeze_public_corpus(
        version="peloton_v1",
        output_root=output_root,
        documents=[
            {
                "document_id": "10-k-latest",
                "title": "Peloton 10-K Latest",
                "source_type": "edgar",
                "source_url": "https://www.sec.gov/Archives/edgar/data/1639825/",
                "raw_source_path": filing_path,
                "filing_or_report_date": "2025-02-10",
                "form_type": "10-K",
            }
        ],
    )

    assert manifest_path.exists()
    manifest = yaml.safe_load(manifest_path.read_text())
    assert manifest["documents"][0]["document_id"] == "10-k-latest"
    assert (output_root / "raw" / "10-k-latest.htm").exists()
    assert (output_root / "normalized" / "10-k-latest.yaml").exists()
```

- [ ] **Step 2: Run the materialization test to verify the module does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_materialize_public.py -q
```

Expected:
- import failure for `materialize_public`

- [ ] **Step 3: Implement the freeze writer and CLI command**

`apps/api/src/prescient_benchmark/corpus/materialize_public.py`

```python
from __future__ import annotations

import hashlib
import shutil
from pathlib import Path

import yaml

from prescient_benchmark.corpus.models import FrozenDocumentRecord
from prescient_benchmark.corpus.normalize_public import normalize_public_document
from prescient_benchmark.corpus.storage import generated_public_root


def _sha256_bytes(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()


def freeze_public_corpus(
    *,
    version: str,
    output_root: Path | None = None,
    documents: list[dict[str, object]],
) -> Path:
    root = output_root or generated_public_root(version)
    raw_root = root / "raw"
    normalized_root = root / "normalized"
    raw_root.mkdir(parents=True, exist_ok=True)
    normalized_root.mkdir(parents=True, exist_ok=True)

    records: list[dict[str, object]] = []
    for document in documents:
        raw_source_path = Path(document["raw_source_path"])
        suffix = raw_source_path.suffix.lower() or ".bin"
        raw_target = raw_root / f"{document['document_id']}{suffix}"
        shutil.copy2(raw_source_path, raw_target)

        normalized = normalize_public_document(
            raw_path=raw_target,
            title=str(document["title"]),
            source_type=str(document["source_type"]),
        )
        normalized_target = normalized_root / f"{document['document_id']}.yaml"
        normalized_target.write_text(yaml.safe_dump(normalized, sort_keys=False))

        raw_bytes = raw_target.read_bytes()
        record = FrozenDocumentRecord(
            document_id=str(document["document_id"]),
            title=str(document["title"]),
            source_type=str(document["source_type"]),
            source_url=str(document["source_url"]),
            raw_path=raw_target.as_posix(),
            normalized_path=normalized_target.as_posix(),
            content_hash=_sha256_bytes(raw_bytes),
            extractor_version=normalized["extractor_version"],
            filing_or_report_date=document.get("filing_or_report_date"),
            form_type=document.get("form_type"),
        )
        records.append(record.model_dump())

    manifest_path = root / "manifest.yaml"
    manifest_path.write_text(
        yaml.safe_dump(
            {
                "version": version,
                "documents": records,
            },
            sort_keys=False,
        )
    )
    return manifest_path
```

`apps/api/src/prescient_benchmark/cli.py`

```python
from pathlib import Path

import httpx
import typer

from prescient_benchmark.corpus.edgar import EdgarClient
from prescient_benchmark.corpus.investor_relations import discover_latest_ir_documents
from prescient_benchmark.corpus.materialize_public import freeze_public_corpus
from prescient_benchmark.corpus.public_peloton import build_public_document_plan

app = typer.Typer()


@app.callback()
def main() -> None:
    \"\"\"Filesystem benchmark helpers.\"\"\"


@app.command("materialize-peloton-public")
def materialize_peloton_public(
    *,
    version: str = typer.Option("peloton_v1"),
    user_agent: str = typer.Option("Prescient Benchmark dev@prescient.local"),
    cache_root: Path = typer.Option(Path("corpus/cache")),
) -> None:
    client = httpx.Client()
    plan = build_public_document_plan()
    edgar = EdgarClient(user_agent=user_agent, cache_root=cache_root, http_client=client)
    ir_urls = discover_latest_ir_documents(
        base_url="https://investor.onepeloton.com/",
        http_client=client,
    )

    filings = edgar.list_filings("0001639825", form_counts={"10-K": 1, "10-Q": 2})
    documents: list[dict[str, object]] = []
    filing_by_slug = {
        "10-k-latest": filings[0],
        "10-q-latest": filings[1],
        "10-q-prior": filings[2],
    }
    for item in plan:
        if item.source_type == "edgar":
            filing = filing_by_slug[item.slug]
            documents.append(
                {
                    "document_id": item.slug,
                    "title": item.title,
                    "source_type": item.source_type,
                    "source_url": item.source_url,
                    "raw_source_path": edgar.fetch_filing(filing),
                    "filing_or_report_date": filing.filing_date.isoformat(),
                    "form_type": filing.form,
                }
            )
        else:
            target_url = ir_urls[item.slug]
            response = client.get(target_url)
            response.raise_for_status()
            local_path = cache_root / "investor_relations" / Path(target_url).name
            local_path.parent.mkdir(parents=True, exist_ok=True)
            local_path.write_bytes(response.content)
            documents.append(
                {
                    "document_id": item.slug,
                    "title": item.title,
                    "source_type": item.source_type,
                    "source_url": target_url,
                    "raw_source_path": local_path,
                    "filing_or_report_date": None,
                    "form_type": None,
                }
            )

    manifest_path = freeze_public_corpus(version=version, documents=documents)
    typer.echo(manifest_path.as_posix())
```

- [ ] **Step 4: Run the materialization unit test**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_materialize_public.py -q
```

Expected:
- test passes

- [ ] **Step 5: Run the full public-corpus unit test set**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_public_peloton.py tests/unit/test_edgar_fetch.py tests/unit/test_investor_relations.py tests/unit/test_normalize_public.py tests/unit/test_materialize_public.py -q
```

Expected:
- all tests pass

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/cli.py apps/api/src/prescient_benchmark/corpus/materialize_public.py tests/unit/test_materialize_public.py
git commit -m "feat: materialize peloton public corpus"
```

---

### Task 5: Add A CLI Smoke Test For The End-To-End Freeze Path

**Files:**
- Modify: `tests/unit/test_materialize_public.py`
- Modify: `apps/api/src/prescient_benchmark/cli.py`

- [ ] **Step 1: Add the failing CLI smoke test**

Append to `tests/unit/test_materialize_public.py`:

```python
from typer.testing import CliRunner

from prescient_benchmark.cli import app


def test_cli_materialize_peloton_public_prints_manifest_path(monkeypatch, tmp_path: Path) -> None:
    def fake_materialize(**_: object) -> Path:
        path = tmp_path / "corpus/generated/peloton_v1/public/manifest.yaml"
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text("version: peloton_v1\n")
        return path

    monkeypatch.setattr(
        "prescient_benchmark.cli.materialize_peloton_public_corpus",
        fake_materialize,
    )

    runner = CliRunner()
    result = runner.invoke(app, ["materialize-peloton-public"])

    assert result.exit_code == 0
    assert "manifest.yaml" in result.stdout
```

- [ ] **Step 2: Run the CLI smoke test to verify the helper symbol does not exist yet**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_materialize_public.py::test_cli_materialize_peloton_public_prints_manifest_path -q
```

Expected:
- failure because `materialize_peloton_public_corpus` is not exposed for patching

- [ ] **Step 3: Extract the command body into a patchable helper**

Update `apps/api/src/prescient_benchmark/cli.py`:

```python
def materialize_peloton_public_corpus(
    *,
    version: str,
    user_agent: str,
    cache_root: Path,
) -> Path:
    client = httpx.Client()
    plan = build_public_document_plan()
    edgar = EdgarClient(user_agent=user_agent, cache_root=cache_root, http_client=client)
    ir_urls = discover_latest_ir_documents(
        base_url="https://investor.onepeloton.com/",
        http_client=client,
    )

    filings = edgar.list_filings("0001639825", form_counts={"10-K": 1, "10-Q": 2})
    documents: list[dict[str, object]] = []
    filing_by_slug = {
        "10-k-latest": filings[0],
        "10-q-latest": filings[1],
        "10-q-prior": filings[2],
    }
    for item in plan:
        if item.source_type == "edgar":
            filing = filing_by_slug[item.slug]
            documents.append(
                {
                    "document_id": item.slug,
                    "title": item.title,
                    "source_type": item.source_type,
                    "source_url": item.source_url,
                    "raw_source_path": edgar.fetch_filing(filing),
                    "filing_or_report_date": filing.filing_date.isoformat(),
                    "form_type": filing.form,
                }
            )
        else:
            target_url = ir_urls[item.slug]
            response = client.get(target_url)
            response.raise_for_status()
            local_path = cache_root / "investor_relations" / Path(target_url).name
            local_path.parent.mkdir(parents=True, exist_ok=True)
            local_path.write_bytes(response.content)
            documents.append(
                {
                    "document_id": item.slug,
                    "title": item.title,
                    "source_type": item.source_type,
                    "source_url": target_url,
                    "raw_source_path": local_path,
                    "filing_or_report_date": None,
                    "form_type": None,
                }
            )
    return freeze_public_corpus(version=version, documents=documents)


@app.command("materialize-peloton-public")
def materialize_peloton_public(
    *,
    version: str = typer.Option("peloton_v1"),
    user_agent: str = typer.Option("Prescient Benchmark dev@prescient.local"),
    cache_root: Path = typer.Option(Path("corpus/cache")),
) -> None:
    manifest_path = materialize_peloton_public_corpus(
        version=version,
        user_agent=user_agent,
        cache_root=cache_root,
    )
    typer.echo(manifest_path.as_posix())
```

- [ ] **Step 4: Run the full materialization test file**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_materialize_public.py -q
```

Expected:
- tests pass

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/cli.py tests/unit/test_materialize_public.py
git commit -m "test: cover peloton public materialization cli"
```

---

## Self-Review

- Spec coverage:
  - EDGAR fetch covered by Task 2
  - IR fetch covered by Task 3
  - raw + normalized freeze covered by Task 4
  - manifest + hashes covered by Task 4
  - repeatable CLI entry point covered by Tasks 4 and 5
- Placeholder scan:
  - no `TODO`/`TBD`
  - every task includes file paths, code, commands, and expected outcomes
- Type consistency:
  - `CorpusDocumentPlan`, `FrozenDocumentRecord`, `FilingRef`, and `materialize_peloton_public_corpus` names are consistent across tasks
