# Parts Catalog Extraction V0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first measurable parts-catalog slice: extract a namespaced Ferrari 360 parts catalog graph from ingested PDF source units with row-level confidence factors and reverse-index validation.

**Architecture:** Keep the catalog graph as automotive product logic under `prescient_benchmark.workshop_manuals`, not in generic KE modules. Reuse existing `SourceRecord`, `SourceUnitRecord`, `KnowledgeScope`, and `WorkshopManualStore` primitives. Implement deterministic Ferrari catalog parsing over source-unit text first; no procedure linking, inferred parts, or vendor shopping in this plan.

**Tech Stack:** Python 3.12, Pydantic v2, existing file-backed `WorkshopManualStore`, Typer CLI, PyMuPDF-backed PDF ingestion, pytest.

---

## Governing Spec

This plan implements Milestone 1 from:

- `docs/superpowers/specs/2026-05-07-parts-catalog-procedure-linking-design.md`

Milestone 1 scope only:

- catalog table metadata
- row-level part records
- callout references
- part numbers
- quantities
- Italian and English descriptions when parseable
- source notes such as `Not as spare part` and `Possible fitting`
- reverse numerical index records
- row-level confidence factors

Explicitly excluded:

- procedure linking
- inferred parts
- review UI
- vendor shopping
- cross-catalog dedupe

## File Structure

- Modify `apps/api/src/prescient_benchmark/workshop_manuals/catalog.py`: add the Ferrari 360 Modena spare parts catalog source and allow source configs to specify `source_kind`.
- Modify `apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py`: preserve `source_config.source_kind` in `SourceRecord`.
- Create `apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_models.py`: Pydantic models for namespace, confidence factors, catalog rows, reverse-index entries, and catalog artifacts.
- Create `apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_extraction.py`: deterministic Ferrari catalog extraction from source units.
- Modify `apps/api/src/prescient_benchmark/workshop_manuals/store.py`: persist and load `PartsCatalogArtifact` records in the existing file-backed store.
- Modify `apps/api/src/prescient_benchmark/cli.py`: add `extract-workshop-parts-catalogs` CLI command.
- Modify `tests/unit/test_workshop_catalog.py`: cover spare parts source config and `source_kind`.
- Modify `tests/unit/test_workshop_pdf_ingest.py`: cover ingestion preserving `source_kind`.
- Create `tests/unit/test_parts_catalog_extraction.py`: cover row parsing, note capture, confidence factor polarity, OCR/reverse-index review behavior, and artifact assembly.
- Modify or create `tests/unit/test_parts_catalog_store.py`: cover file-backed artifact persistence.
- Create `tests/unit/test_cli_parts_catalog.py`: cover CLI extraction using the file-backed store.

## Task 1: Seed The Ferrari 360 Parts Catalog Source

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/catalog.py`
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py`
- Test: `tests/unit/test_workshop_catalog.py`
- Test: `tests/unit/test_workshop_pdf_ingest.py`

- [ ] **Step 1: Write failing catalog config tests**

Append these tests to `tests/unit/test_workshop_catalog.py`:

```python
def test_seeded_sources_include_360_modena_spare_parts_catalogue() -> None:
    source = next(
        source
        for source in WORKSHOP_360_SOURCES
        if source.source_id == "source-ferrari-360-modena-parts"
    )

    assert source.title == "Ferrari 360 Modena Spare Parts Catalogue"
    assert source.source_kind == "pdf_parts_catalog"
    assert source.relative_path.as_posix() == "Ferrari 360 Modena/05 - Spare Parts Catalogue.pdf"
    assert "360 Modena parts catalogue" in source.aliases
    assert source.variant_tags == ["360_modena", "parts_catalog"]


def test_default_360_scope_includes_modena_workshop_and_parts_catalog_sources() -> None:
    scope = default_360_scope()

    assert scope.linked_source_ids == [
        "source-ferrari-360-wsm",
        "source-ferrari-360-modena-parts",
    ]
```

- [ ] **Step 2: Write failing ingest source-kind test**

Append this test to `tests/unit/test_workshop_pdf_ingest.py`:

```python
def test_ingest_pdf_source_preserves_configured_source_kind(tmp_path: Path) -> None:
    pdf_path = tmp_path / "fixture.pdf"
    _write_fixture_pdf(pdf_path)
    store = WorkshopManualStore(tmp_path / "store")
    source_config = WorkshopSourceConfig(
        source_id="source-test-parts",
        title="Test Parts Catalog",
        relative_path=Path("parts.pdf"),
        source_kind="pdf_parts_catalog",
    )

    ingest_pdf_source(
        source_config=source_config,
        source_path=pdf_path,
        store=store,
        rendered_pages_root=tmp_path / "rendered",
    )

    source = store.get_source("source-test-parts")

    assert source is not None
    assert source.source_kind == "pdf_parts_catalog"
```

- [ ] **Step 3: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_catalog.py::test_seeded_sources_include_360_modena_spare_parts_catalogue tests/unit/test_workshop_catalog.py::test_default_360_scope_includes_modena_workshop_and_parts_catalog_sources tests/unit/test_workshop_pdf_ingest.py::test_ingest_pdf_source_preserves_configured_source_kind -q
```

Expected: FAIL because `WorkshopSourceConfig.source_kind` and the Modena parts source do not exist.

- [ ] **Step 4: Add `source_kind` to `WorkshopSourceConfig`**

In `apps/api/src/prescient_benchmark/workshop_manuals/catalog.py`, add this field to `WorkshopSourceConfig` after `relative_path`:

```python
    source_kind: str = "pdf_manual"
```

- [ ] **Step 5: Add the Modena parts source**

In `WORKSHOP_360_SOURCES`, insert this config immediately after `source-ferrari-360-wsm`:

```python
    WorkshopSourceConfig(
        source_id="source-ferrari-360-modena-parts",
        title="Ferrari 360 Modena Spare Parts Catalogue",
        relative_path=Path("Ferrari 360 Modena/05 - Spare Parts Catalogue.pdf"),
        source_kind="pdf_parts_catalog",
        aliases=["360 Modena parts catalogue", "360 spare parts catalogue"],
        variant_tags=["360_modena", "parts_catalog"],
    ),
```

Update the default 360 scope's `linked_source_ids` to include both source ids:

```python
            linked_source_ids=[
                "source-ferrari-360-wsm",
                "source-ferrari-360-modena-parts",
            ],
```

- [ ] **Step 6: Preserve source kind during PDF ingest**

In `apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py`, change the `SourceRecord` construction from:

```python
        source_kind="pdf_manual",
```

to:

```python
        source_kind=source_config.source_kind,
```

- [ ] **Step 7: Run focused tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_catalog.py tests/unit/test_workshop_pdf_ingest.py::test_ingest_pdf_source_preserves_configured_source_kind -q
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/catalog.py apps/api/src/prescient_benchmark/workshop_manuals/pdf_ingest.py tests/unit/test_workshop_catalog.py tests/unit/test_workshop_pdf_ingest.py
git commit -m "feat(workshop): seed Modena parts catalog source"
```

## Task 2: Add Parts Catalog Domain Models

**Files:**
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_models.py`
- Test: `tests/unit/test_parts_catalog_extraction.py`

- [ ] **Step 1: Write failing model tests**

Create `tests/unit/test_parts_catalog_extraction.py` with:

```python
from prescient_benchmark.workshop_manuals.parts_catalog_models import (
    CatalogNamespace,
    ConfidenceFactors,
    ParsedQuantity,
    PartsCatalogRow,
    RelationshipBasis,
    ReviewBucket,
    TrustState,
)


def test_parts_catalog_row_serializes_reviewable_contract() -> None:
    row = PartsCatalogRow(
        row_id="row-source-ferrari-360-modena-parts-tavola-33-rif-12",
        namespace=CatalogNamespace(
            domain="vehicle_repair",
            manufacturer="Ferrari",
            product_family="360",
            model="360 Modena",
            variant=None,
            year_range=[1999, 2005],
            source_id="source-ferrari-360-modena-parts",
        ),
        source_id="source-ferrari-360-modena-parts",
        source_unit_ids=["unit-source-ferrari-360-modena-parts-p90"],
        table_id="tavola_33",
        table_number="33",
        table_title_it="IMPIANTO FRENANTE",
        table_title_en="BRAKE SYSTEM",
        callout_ref="12",
        part_number="183100",
        quantity=ParsedQuantity(value=4, raw="4", basis="catalog_row"),
        description_it="TUBO FLESSIBILE FRENI",
        description_en="BRAKES HOSE",
        notes=[],
        relationship_basis=RelationshipBasis.EXPLICIT,
        review_bucket=ReviewBucket.INCLUDE_BY_DEFAULT,
        confidence_factors=ConfidenceFactors(
            supports=["required_fields_parsed", "row_shape_matched_expected_profile"],
            weakens=[],
        ),
        trust_state=TrustState.EXTRACTED,
    )

    payload = row.model_dump(mode="json")

    assert payload["namespace"]["manufacturer"] == "Ferrari"
    assert payload["quantity"] == {"value": 4, "raw": "4", "basis": "catalog_row"}
    assert payload["confidence_factors"] == {
        "supports": ["required_fields_parsed", "row_shape_matched_expected_profile"],
        "weakens": [],
    }


def test_parts_catalog_row_rejects_empty_part_number() -> None:
    try:
        PartsCatalogRow(
            row_id="row-1",
            namespace=CatalogNamespace(
                domain="vehicle_repair",
                manufacturer="Ferrari",
                product_family="360",
                model="360 Modena",
                source_id="source-ferrari-360-modena-parts",
            ),
            source_id="source-ferrari-360-modena-parts",
            source_unit_ids=["unit-1"],
            table_id="tavola_33",
            table_number="33",
            callout_ref="12",
            part_number=" ",
            quantity=ParsedQuantity(value=None, raw="-", basis="unknown"),
            description_it=None,
            description_en=None,
        )
    except ValueError as exc:
        assert "part_number" in str(exc)
    else:
        raise AssertionError("expected empty part_number validation failure")
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_parts_catalog_extraction.py -q
```

Expected: FAIL because `parts_catalog_models.py` does not exist.

- [ ] **Step 3: Create model module**

Create `apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_models.py`:

```python
from enum import StrEnum
from typing import Annotated

from pydantic import BaseModel, ConfigDict, Field, StringConstraints


NonEmptyString = Annotated[str, StringConstraints(strip_whitespace=True, min_length=1)]


class TrustState(StrEnum):
    EXTRACTED = "extracted"
    REVIEWED = "reviewed"
    DISPUTED = "disputed"
    SUPERSEDED = "superseded"
    STALE = "stale"


class RelationshipBasis(StrEnum):
    EXPLICIT = "explicit"
    STRONG_INFERRED = "strong_inferred"
    WEAK_INFERRED = "weak_inferred"
    UNCERTAIN = "uncertain"


class ReviewBucket(StrEnum):
    INCLUDE_BY_DEFAULT = "include_by_default"
    SHOW_AS_SUGGESTED = "show_as_suggested"
    NEEDS_REVIEW = "needs_review"


class QuantityBasis(StrEnum):
    CATALOG_ROW = "catalog_row"
    PROCEDURE_TEXT = "procedure_text"
    KIT_CONTENTS = "kit_contents"
    INFERRED = "inferred"
    UNKNOWN = "unknown"


class CatalogNamespace(BaseModel):
    model_config = ConfigDict(extra="forbid")

    domain: NonEmptyString
    manufacturer: NonEmptyString
    product_family: NonEmptyString
    model: NonEmptyString
    variant: str | None = None
    year_range: list[int] = Field(default_factory=list)
    source_id: NonEmptyString


class ConfidenceFactors(BaseModel):
    model_config = ConfigDict(extra="forbid")

    supports: list[NonEmptyString] = Field(default_factory=list)
    weakens: list[NonEmptyString] = Field(default_factory=list)


class ParsedQuantity(BaseModel):
    model_config = ConfigDict(extra="forbid")

    value: int | None = None
    raw: NonEmptyString
    basis: QuantityBasis


class CatalogRef(BaseModel):
    model_config = ConfigDict(extra="forbid")

    table_id: NonEmptyString
    table_title: str | None = None
    callout_ref: str | None = None
    source_unit_id: NonEmptyString


class PartsCatalogRow(BaseModel):
    model_config = ConfigDict(extra="forbid")

    row_id: NonEmptyString
    namespace: CatalogNamespace
    source_id: NonEmptyString
    source_unit_ids: list[NonEmptyString] = Field(min_length=1)
    table_id: NonEmptyString
    table_number: NonEmptyString
    table_title_it: str | None = None
    table_title_en: str | None = None
    callout_ref: str | None = None
    part_number: NonEmptyString
    quantity: ParsedQuantity
    description_it: str | None = None
    description_en: str | None = None
    notes: list[str] = Field(default_factory=list)
    relationship_basis: RelationshipBasis = RelationshipBasis.EXPLICIT
    review_bucket: ReviewBucket = ReviewBucket.INCLUDE_BY_DEFAULT
    confidence_factors: ConfidenceFactors = Field(default_factory=ConfidenceFactors)
    trust_state: TrustState = TrustState.EXTRACTED


class PartsCatalogTable(BaseModel):
    model_config = ConfigDict(extra="forbid")

    table_id: NonEmptyString
    table_number: NonEmptyString
    title_it: str | None = None
    title_en: str | None = None
    source_unit_ids: list[NonEmptyString] = Field(default_factory=list)
    rows: list[PartsCatalogRow] = Field(default_factory=list)


class ReverseIndexEntry(BaseModel):
    model_config = ConfigDict(extra="forbid")

    part_number: NonEmptyString
    table_number: NonEmptyString
    callout_ref: str | None = None
    source_unit_id: NonEmptyString


class PartsCatalogArtifact(BaseModel):
    model_config = ConfigDict(extra="forbid")

    artifact_id: NonEmptyString
    namespace: CatalogNamespace
    source_id: NonEmptyString
    source_fingerprint: NonEmptyString
    tables: list[PartsCatalogTable] = Field(default_factory=list)
    reverse_index: list[ReverseIndexEntry] = Field(default_factory=list)
    trust_state: TrustState = TrustState.EXTRACTED
```

- [ ] **Step 4: Run tests to verify pass**

Run:

```bash
uv run python -m pytest tests/unit/test_parts_catalog_extraction.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_models.py tests/unit/test_parts_catalog_extraction.py
git commit -m "feat(parts): add catalog graph models"
```

## Task 3: Parse Ferrari Parts Catalog Tables

**Files:**
- Create: `apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_extraction.py`
- Modify: `tests/unit/test_parts_catalog_extraction.py`

- [ ] **Step 1: Add failing extraction tests**

Append to `tests/unit/test_parts_catalog_extraction.py`:

```python
from prescient_benchmark.knowledge.models import SourceRecord, SourceUnitKind, SourceUnitRecord
from prescient_benchmark.workshop_manuals.parts_catalog_extraction import (
    extract_ferrari_parts_catalog,
)


def _parts_source() -> SourceRecord:
    return SourceRecord(
        source_id="source-ferrari-360-modena-parts",
        title="Ferrari 360 Modena Spare Parts Catalogue",
        source_kind="pdf_parts_catalog",
        origin_path="/manuals/05 - Spare Parts Catalogue.pdf",
        content_fingerprint="parts123",
        aliases=["360 Modena parts catalogue"],
        variant_tags=["360_modena", "parts_catalog"],
    )


def _unit(unit_id: str, ordinal: int, text: str) -> SourceUnitRecord:
    return SourceUnitRecord(
        unit_id=unit_id,
        source_id="source-ferrari-360-modena-parts",
        unit_kind=SourceUnitKind.PDF_PAGE,
        ordinal=ordinal,
        text=text,
    )


def test_extract_ferrari_parts_catalog_parses_table_rows() -> None:
    unit = _unit(
        "unit-source-ferrari-360-modena-parts-p90",
        90,
        """
        IMPIANTO FRENANTE                                     TAV. 33

        BRAKE SYSTEM                                          DATA: FEBBRAIO 2000

               Dis.N°                                                                               Dis.N°
         Rif
               Part.N°
                          Q.ty   Denominazione                 Description                    Rif   Part.N°
                                                                                                               Q.ty   Denominazione                   Description
          12     183100     4     TUBO FLESSIBILE FRENI         BRAKES HOSE                    13     113670     4     RACCORDO                         UNION
          18    10260160    8     GUARNIZIONE                   GASKET

        Tavola 33 - IMPIANTO FRENANTE             BRAKE SYSTEM
        """,
    )

    artifact = extract_ferrari_parts_catalog(
        source=_parts_source(),
        units=[unit],
        namespace_model="360 Modena",
    )

    table = artifact.tables[0]
    assert table.table_number == "33"
    assert table.title_it == "IMPIANTO FRENANTE"
    assert table.title_en == "BRAKE SYSTEM"
    assert [row.part_number for row in table.rows] == ["183100", "113670", "10260160"]
    assert table.rows[0].quantity.value == 4
    assert table.rows[0].description_it == "TUBO FLESSIBILE FRENI"
    assert table.rows[0].description_en == "BRAKES HOSE"
    assert table.rows[0].confidence_factors.supports
    assert table.rows[0].confidence_factors.weakens == []


def test_extract_ferrari_parts_catalog_attaches_notes_to_previous_row() -> None:
    unit = _unit(
        "unit-source-ferrari-360-modena-parts-p10",
        10,
        """
        BASAMENTO                                                                         TAV. 1
        CRANKCASE                                                                         DATA: FEBBRAIO 2000
         Rif   Part.N°   Q.ty   Denominazione             Description
          1    173885     1     BASAMENTO                 CRANKCASE
                            -Non a ricambio-                -Not as spare part-
                            -Con il basamento completo-     -With the complete crankcase-
        Tavola 1 - BASAMENTO           CRANKCASE
        """,
    )

    artifact = extract_ferrari_parts_catalog(
        source=_parts_source(),
        units=[unit],
        namespace_model="360 Modena",
    )

    row = artifact.tables[0].rows[0]
    assert row.notes == [
        "Non a ricambio / Not as spare part",
        "Con il basamento completo / With the complete crankcase",
    ]
    assert "source_notes_present" in row.confidence_factors.supports
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_parts_catalog_extraction.py::test_extract_ferrari_parts_catalog_parses_table_rows tests/unit/test_parts_catalog_extraction.py::test_extract_ferrari_parts_catalog_attaches_notes_to_previous_row -q
```

Expected: FAIL because `parts_catalog_extraction.py` does not exist.

- [ ] **Step 3: Implement extraction module**

Create `apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_extraction.py`:

```python
import re
from collections import defaultdict
from collections.abc import Iterable

from prescient_benchmark.knowledge.models import SourceRecord, SourceUnitRecord
from prescient_benchmark.workshop_manuals.parts_catalog_models import (
    CatalogNamespace,
    ConfidenceFactors,
    ParsedQuantity,
    PartsCatalogArtifact,
    PartsCatalogRow,
    PartsCatalogTable,
    QuantityBasis,
    ReviewBucket,
    ReverseIndexEntry,
)


_TABLE_HEADER_RE = re.compile(r"(?P<title>[A-ZÀ-Ú0-9][A-ZÀ-Ú0-9 .,'’\\-/]+?)\\s+TAV\\.\\s*(?P<number>\\d+)")
_FOOTER_RE = re.compile(r"Tavola\\s+(?P<number>\\d+)\\s+-\\s+(?P<title_it>.+?)\\s{2,}(?P<title_en>[A-Z0-9].+)$")
_ROW_RE = re.compile(
    r"(?P<rif>\\d+|-)\\s+"
    r"(?P<part>\\d{5,8})\\s+"
    r"(?P<qty>\\d+|-)\\s+"
    r"(?P<it>[A-ZÀ-Ú0-9][A-ZÀ-Ú0-9 .,'’\\-/]+?)\\s{2,}"
    r"(?P<en>[A-Z0-9][A-Z0-9 .,'’\\-/]+?)(?=\\s{2,}(?:\\d+|-)\\s+\\d{5,8}\\s+|$)"
)
_NOTE_PAIR_RE = re.compile(r"-(?P<it>[^-]+?)-\\s+-(?P<en>[^-]+?)-")
_INDEX_ROW_RE = re.compile(r"(?P<part>\\d{5,8})\\s+(?P<table>\\d{1,3})\\s+(?P<rif>\\d+|-)")


def extract_ferrari_parts_catalog(
    *,
    source: SourceRecord,
    units: Iterable[SourceUnitRecord],
    namespace_model: str,
) -> PartsCatalogArtifact:
    namespace = CatalogNamespace(
        domain="vehicle_repair",
        manufacturer="Ferrari",
        product_family="360",
        model=namespace_model,
        variant=None,
        year_range=[1999, 2005],
        source_id=source.source_id,
    )
    tables_by_number: dict[str, PartsCatalogTable] = {}
    reverse_index: list[ReverseIndexEntry] = []

    for unit in sorted(units, key=lambda item: item.ordinal):
        text = unit.text
        if "INDICE NUMERICO" in text or "NUMERICAL INDEX" in text:
            reverse_index.extend(_parse_reverse_index(unit))
            continue

        table_number, header_title = _detect_table_header(text)
        if table_number is None:
            continue
        title_it, title_en = _detect_footer_titles(text, table_number)
        if title_it is None:
            title_it = header_title
        table = tables_by_number.get(table_number)
        if table is None:
            table = PartsCatalogTable(
                table_id=f"tavola_{table_number}",
                table_number=table_number,
                title_it=title_it,
                title_en=title_en,
                source_unit_ids=[],
                rows=[],
            )
            tables_by_number[table_number] = table
        if unit.unit_id not in table.source_unit_ids:
            table.source_unit_ids.append(unit.unit_id)
        table.rows.extend(
            _parse_rows(
                source=source,
                unit=unit,
                namespace=namespace,
                table=table,
            )
        )

    _apply_reverse_index_validation(tables_by_number.values(), reverse_index)
    return PartsCatalogArtifact(
        artifact_id=f"parts-catalog-{_slug(source.source_id)}",
        namespace=namespace,
        source_id=source.source_id,
        source_fingerprint=source.content_fingerprint,
        tables=list(tables_by_number.values()),
        reverse_index=reverse_index,
    )


def _detect_table_header(text: str) -> tuple[str | None, str | None]:
    match = _TABLE_HEADER_RE.search(text)
    if match is None:
        return None, None
    return match.group("number"), _clean(match.group("title"))


def _detect_footer_titles(text: str, table_number: str) -> tuple[str | None, str | None]:
    for line in text.splitlines():
        match = _FOOTER_RE.search(line.strip())
        if match and match.group("number") == table_number:
            return _clean(match.group("title_it")), _clean(match.group("title_en"))
    return None, None


def _parse_rows(
    *,
    source: SourceRecord,
    unit: SourceUnitRecord,
    namespace: CatalogNamespace,
    table: PartsCatalogTable,
) -> list[PartsCatalogRow]:
    rows: list[PartsCatalogRow] = []
    lines = unit.text.splitlines()
    for line_index, line in enumerate(lines):
        for match in _ROW_RE.finditer(line):
            quantity = _quantity(match.group("qty"))
            notes = _notes_after(lines, line_index)
            supports = [
                "required_fields_parsed",
                "row_shape_matched_expected_profile",
            ]
            if quantity.basis is QuantityBasis.CATALOG_ROW:
                supports.append("quantity_parsed_as_valid_value_or_known_marker")
            if notes:
                supports.append("source_notes_present")
            part_number = match.group("part")
            rows.append(
                PartsCatalogRow(
                    row_id=(
                        f"row-{_slug(source.source_id)}-"
                        f"tavola-{table.table_number}-rif-{_slug(match.group('rif'))}-"
                        f"part-{part_number}"
                    ),
                    namespace=namespace,
                    source_id=source.source_id,
                    source_unit_ids=[unit.unit_id],
                    table_id=table.table_id,
                    table_number=table.table_number,
                    table_title_it=table.title_it,
                    table_title_en=table.title_en,
                    callout_ref=None if match.group("rif") == "-" else match.group("rif"),
                    part_number=part_number,
                    quantity=quantity,
                    description_it=_clean(match.group("it")),
                    description_en=_clean(match.group("en")),
                    notes=notes,
                    review_bucket=ReviewBucket.INCLUDE_BY_DEFAULT,
                    confidence_factors=ConfidenceFactors(supports=supports, weakens=[]),
                )
            )
    return rows


def _notes_after(lines: list[str], row_line_index: int) -> list[str]:
    notes: list[str] = []
    for line in lines[row_line_index + 1 :]:
        stripped = line.strip()
        if not stripped:
            continue
        if _ROW_RE.search(stripped) or stripped.startswith("Tavola "):
            break
        note_match = _NOTE_PAIR_RE.search(stripped)
        if note_match:
            notes.append(f"{_clean(note_match.group('it'))} / {_clean(note_match.group('en'))}")
    return notes


def _parse_reverse_index(unit: SourceUnitRecord) -> list[ReverseIndexEntry]:
    entries: list[ReverseIndexEntry] = []
    for match in _INDEX_ROW_RE.finditer(unit.text):
        entries.append(
            ReverseIndexEntry(
                part_number=match.group("part"),
                table_number=match.group("table"),
                callout_ref=None if match.group("rif") == "-" else match.group("rif"),
                source_unit_id=unit.unit_id,
            )
        )
    return entries


def _apply_reverse_index_validation(
    tables: Iterable[PartsCatalogTable],
    reverse_index: list[ReverseIndexEntry],
) -> None:
    entries_by_key = {
        (entry.part_number, entry.table_number, entry.callout_ref)
        for entry in reverse_index
    }
    if not entries_by_key:
        return
    for table in tables:
        for row in table.rows:
            key = (row.part_number, table.table_number, row.callout_ref)
            if key in entries_by_key:
                row.confidence_factors.supports.append("reverse_index_agrees")
            else:
                row.confidence_factors.weakens.append("reverse_index_missing_or_disagrees")
                row.review_bucket = ReviewBucket.NEEDS_REVIEW


def _quantity(raw: str) -> ParsedQuantity:
    if raw.isdigit():
        return ParsedQuantity(value=int(raw), raw=raw, basis=QuantityBasis.CATALOG_ROW)
    return ParsedQuantity(value=None, raw=raw, basis=QuantityBasis.UNKNOWN)


def _clean(value: str) -> str:
    return " ".join(value.strip(" -").split())


def _slug(value: str) -> str:
    return re.sub(r"[^a-z0-9]+", "-", value.casefold()).strip("-")
```

- [ ] **Step 4: Run extraction tests**

Run:

```bash
uv run python -m pytest tests/unit/test_parts_catalog_extraction.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_extraction.py tests/unit/test_parts_catalog_extraction.py
git commit -m "feat(parts): parse Ferrari catalog tables"
```

## Task 4: Parse Reverse Index And Downgrade Disagreements

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_extraction.py`
- Test: `tests/unit/test_parts_catalog_extraction.py`

- [ ] **Step 1: Add failing reverse-index tests**

Append to `tests/unit/test_parts_catalog_extraction.py`:

```python
def test_extract_ferrari_parts_catalog_parses_reverse_index_and_validates_rows() -> None:
    table_unit = _unit(
        "unit-source-ferrari-360-modena-parts-p90",
        90,
        """
        IMPIANTO FRENANTE                                     TAV. 33
        BRAKE SYSTEM                                          DATA: FEBBRAIO 2000
         Rif   Part.N°   Q.ty   Denominazione             Description
          12     183100     4     TUBO FLESSIBILE FRENI     BRAKES HOSE
        Tavola 33 - IMPIANTO FRENANTE             BRAKE SYSTEM
        """,
    )
    index_unit = _unit(
        "unit-source-ferrari-360-modena-parts-p7",
        7,
        """
        INDICE NUMERICO - NUMERICAL INDEX
        N. CODICE   TAV.   POS.
        183100       33     12
        """,
    )

    artifact = extract_ferrari_parts_catalog(
        source=_parts_source(),
        units=[index_unit, table_unit],
        namespace_model="360 Modena",
    )

    assert artifact.reverse_index[0].part_number == "183100"
    row = artifact.tables[0].rows[0]
    assert "reverse_index_agrees" in row.confidence_factors.supports
    assert row.review_bucket == "include_by_default"


def test_extract_ferrari_parts_catalog_sends_reverse_index_disagreement_to_review() -> None:
    table_unit = _unit(
        "unit-source-ferrari-360-modena-parts-p90",
        90,
        """
        IMPIANTO FRENANTE                                     TAV. 33
        BRAKE SYSTEM                                          DATA: FEBBRAIO 2000
         Rif   Part.N°   Q.ty   Denominazione             Description
          12     183100     4     TUBO FLESSIBILE FRENI     BRAKES HOSE
        Tavola 33 - IMPIANTO FRENANTE             BRAKE SYSTEM
        """,
    )
    index_unit = _unit(
        "unit-source-ferrari-360-modena-parts-p7",
        7,
        """
        INDICE NUMERICO - NUMERICAL INDEX
        N. CODICE   TAV.   POS.
        183100       34     12
        """,
    )

    artifact = extract_ferrari_parts_catalog(
        source=_parts_source(),
        units=[index_unit, table_unit],
        namespace_model="360 Modena",
    )

    row = artifact.tables[0].rows[0]
    assert "reverse_index_missing_or_disagrees" in row.confidence_factors.weakens
    assert row.review_bucket == "needs_review"
```

- [ ] **Step 2: Run tests to verify behavior before implementation**

Run:

```bash
uv run python -m pytest tests/unit/test_parts_catalog_extraction.py::test_extract_ferrari_parts_catalog_parses_reverse_index_and_validates_rows tests/unit/test_parts_catalog_extraction.py::test_extract_ferrari_parts_catalog_sends_reverse_index_disagreement_to_review -q
```

Expected: FAIL because the reverse-index validation behavior is not complete until this task is implemented.

- [ ] **Step 3: Implement reverse-index behavior**

Update `_INDEX_ROW_RE`, `_parse_reverse_index`, and `_apply_reverse_index_validation` in `parts_catalog_extraction.py` to match the code shown in Task 3. Keep the rule strict: when a reverse index exists for the source and a parsed row does not match it by `(part_number, table_number, callout_ref)`, append `reverse_index_missing_or_disagrees` to `weakens` and set `review_bucket=ReviewBucket.NEEDS_REVIEW`.

- [ ] **Step 4: Run all extraction tests**

Run:

```bash
uv run python -m pytest tests/unit/test_parts_catalog_extraction.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/parts_catalog_extraction.py tests/unit/test_parts_catalog_extraction.py
git commit -m "feat(parts): validate catalog rows with reverse index"
```

## Task 5: Persist Parts Catalog Artifacts In The Workshop Store

**Files:**
- Modify: `apps/api/src/prescient_benchmark/workshop_manuals/store.py`
- Test: `tests/unit/test_parts_catalog_store.py`

- [ ] **Step 1: Write failing store tests**

Create `tests/unit/test_parts_catalog_store.py`:

```python
from prescient_benchmark.workshop_manuals.parts_catalog_models import (
    CatalogNamespace,
    PartsCatalogArtifact,
)
from prescient_benchmark.workshop_manuals.store import WorkshopManualStore


def test_workshop_store_persists_parts_catalog_artifacts(tmp_path) -> None:
    store = WorkshopManualStore(tmp_path / "store")
    artifact = PartsCatalogArtifact(
        artifact_id="parts-catalog-source-ferrari-360-modena-parts",
        namespace=CatalogNamespace(
            domain="vehicle_repair",
            manufacturer="Ferrari",
            product_family="360",
            model="360 Modena",
            source_id="source-ferrari-360-modena-parts",
        ),
        source_id="source-ferrari-360-modena-parts",
        source_fingerprint="parts123",
        tables=[],
        reverse_index=[],
    )

    store.upsert_parts_catalog_artifact(artifact)

    assert store.get_parts_catalog_artifact(artifact.artifact_id) == artifact
    assert store.list_parts_catalog_artifacts() == [artifact]
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_parts_catalog_store.py -q
```

Expected: FAIL because store methods do not exist.

- [ ] **Step 3: Add store methods**

In `apps/api/src/prescient_benchmark/workshop_manuals/store.py`, import the model:

```python
from prescient_benchmark.workshop_manuals.parts_catalog_models import PartsCatalogArtifact
```

In `WorkshopManualStore.__init__`, add:

```python
        self._parts_catalog_artifacts_path = self.root / "parts_catalog_artifacts.json"
```

Add these methods after `list_structures`:

```python
    def upsert_parts_catalog_artifact(self, artifact: PartsCatalogArtifact) -> None:
        artifacts = {item.artifact_id: item for item in self.list_parts_catalog_artifacts()}
        artifacts[artifact.artifact_id] = artifact
        self._write_models(self._parts_catalog_artifacts_path, list(artifacts.values()))

    def list_parts_catalog_artifacts(self) -> list[PartsCatalogArtifact]:
        return self._read_models(self._parts_catalog_artifacts_path, PartsCatalogArtifact)

    def get_parts_catalog_artifact(self, artifact_id: str) -> PartsCatalogArtifact | None:
        return next(
            (
                artifact
                for artifact in self.list_parts_catalog_artifacts()
                if artifact.artifact_id == artifact_id
            ),
            None,
        )
```

- [ ] **Step 4: Run store tests**

Run:

```bash
uv run python -m pytest tests/unit/test_parts_catalog_store.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient_benchmark/workshop_manuals/store.py tests/unit/test_parts_catalog_store.py
git commit -m "feat(parts): persist catalog artifacts"
```

## Task 6: Add CLI Extraction Command

**Files:**
- Modify: `apps/api/src/prescient_benchmark/cli.py`
- Test: `tests/unit/test_cli_parts_catalog.py`

- [ ] **Step 1: Write failing CLI test**

Create `tests/unit/test_cli_parts_catalog.py`:

```python
from typer.testing import CliRunner

from prescient_benchmark.cli import app
from prescient_benchmark.knowledge.models import SourceRecord, SourceUnitKind, SourceUnitRecord
from prescient_benchmark.workshop_manuals.store import WorkshopManualStore


def test_extract_workshop_parts_catalogs_command_writes_artifact(tmp_path) -> None:
    data_root = tmp_path / "corpus"
    store = WorkshopManualStore(data_root / "store")
    store.upsert_source(
        SourceRecord(
            source_id="source-ferrari-360-modena-parts",
            title="Ferrari 360 Modena Spare Parts Catalogue",
            source_kind="pdf_parts_catalog",
            origin_path="/manuals/05 - Spare Parts Catalogue.pdf",
            content_fingerprint="parts123",
            aliases=[],
            variant_tags=["360_modena", "parts_catalog"],
        )
    )
    store.upsert_units(
        [
            SourceUnitRecord(
                unit_id="unit-source-ferrari-360-modena-parts-p90",
                source_id="source-ferrari-360-modena-parts",
                unit_kind=SourceUnitKind.PDF_PAGE,
                ordinal=90,
                text="""
                IMPIANTO FRENANTE                                     TAV. 33
                BRAKE SYSTEM                                          DATA: FEBBRAIO 2000
                 Rif   Part.N°   Q.ty   Denominazione             Description
                  12     183100     4     TUBO FLESSIBILE FRENI     BRAKES HOSE
                Tavola 33 - IMPIANTO FRENANTE             BRAKE SYSTEM
                """,
            )
        ]
    )

    result = CliRunner().invoke(
        app,
        [
            "extract-workshop-parts-catalogs",
            "--data-root",
            data_root.as_posix(),
            "--scope-id",
            "scope-ferrari-360-modena",
        ],
    )

    assert result.exit_code == 0
    assert "parts catalog artifacts: 1" in result.output
    artifacts = store.list_parts_catalog_artifacts()
    assert len(artifacts) == 1
    assert artifacts[0].tables[0].rows[0].part_number == "183100"
```

- [ ] **Step 2: Run test to verify failure**

Run:

```bash
uv run python -m pytest tests/unit/test_cli_parts_catalog.py -q
```

Expected: FAIL because CLI command does not exist.

- [ ] **Step 3: Add CLI imports**

In `apps/api/src/prescient_benchmark/cli.py`, add:

```python
from prescient_benchmark.workshop_manuals.parts_catalog_extraction import (
    extract_ferrari_parts_catalog,
)
```

- [ ] **Step 4: Add CLI command**

Add this command after `extract_workshop_artifacts_command`:

```python
@app.command("extract-workshop-parts-catalogs")
def extract_workshop_parts_catalogs_command(
    *,
    data_root: Path = typer.Option(Path("corpus/workshop_manuals")),
    scope_id: str = typer.Option("scope-ferrari-360-modena"),
) -> None:
    store = WorkshopManualStore(data_root / "store")
    scope = resolve_seeded_scope(scope_id)
    sources = {source.source_id: source for source in store.list_sources()}
    extracted_count = 0
    for source_id in scope.linked_source_ids:
        source = sources.get(source_id)
        if source is None or source.source_kind != "pdf_parts_catalog":
            continue
        artifact = extract_ferrari_parts_catalog(
            source=source,
            units=store.units_for_source(source.source_id),
            namespace_model=scope.display_name,
        )
        store.upsert_parts_catalog_artifact(artifact)
        extracted_count += 1
    typer.echo(f"parts catalog artifacts: {extracted_count}")
```

- [ ] **Step 5: Run CLI test**

Run:

```bash
uv run python -m pytest tests/unit/test_cli_parts_catalog.py -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient_benchmark/cli.py tests/unit/test_cli_parts_catalog.py
git commit -m "feat(parts): add catalog extraction CLI"
```

## Task 7: Final Verification For Milestone 1

**Files:**
- Verify only; no file edits expected.

- [ ] **Step 1: Run focused unit suite**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_catalog.py tests/unit/test_workshop_pdf_ingest.py tests/unit/test_parts_catalog_extraction.py tests/unit/test_parts_catalog_store.py tests/unit/test_cli_parts_catalog.py -q
```

Expected: PASS.

- [ ] **Step 2: Run adjacent workshop unit tests**

Run:

```bash
uv run python -m pytest tests/unit/test_workshop_catalog.py tests/unit/test_workshop_pdf_ingest.py tests/unit/test_workshop_artifact_extraction.py tests/unit/test_workshop_providers.py tests/unit/test_workshop_service.py -q
```

Expected: PASS.

- [ ] **Step 3: Inspect git diff**

Run:

```bash
git status --short
git log --oneline -6
```

Expected: clean working tree after task commits; recent commits correspond to this plan.

- [ ] **Step 4: Update Beads**

Run:

```bash
bd update prescient_os-xby --notes="Parts catalog extraction v0 implementation plan written; implementation worktree is .worktrees/parts-catalog-linking on work/parts-catalog-linking."
```

- [ ] **Step 5: Push feature branch**

Run:

```bash
git push -u origin work/parts-catalog-linking
```

Expected: branch pushed. This branch should later open a PR into `prod/automotive-workshop`.

## Plan Self-Review

Spec coverage:

- Milestone 1 catalog extraction is covered by Tasks 1-7.
- Namespaced catalog graph is covered by Task 2.
- Deterministic Ferrari table extraction is covered by Task 3.
- Reverse-index validation is covered by Task 4.
- File-backed current projection for dogfood use is covered by Task 5.
- CLI execution path is covered by Task 6.
- Verification gates begin with representative unit fixtures in Task 7. The full 20-table hand-coded ground truth set remains a follow-up after this v0 parser exists.

Known deferrals:

- Explicit source-unit linking is Milestone 2 and should get its own plan after this branch proves catalog extraction.
- Inferred linking, review surface, and vendor shopping are later milestones.
- Full 20-table parser benchmark is not created here because this first code slice creates the parser and storage substrate needed to host that benchmark.
