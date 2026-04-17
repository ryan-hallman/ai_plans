# 2026-04-16 Peloton Public Corpus Materialization Design

## Goal

Build the first real, reproducible public corpus for the Peloton benchmark POC.

This slice should produce a frozen public-document dataset that can later be indexed, queried, and compared against hosted baselines. The system of record for this slice is the generated corpus on disk, not OpenSearch.

## Why This Slice Exists

The current POC branch has benchmark scaffolding, but it does not yet have a real corpus. Without a reproducible public corpus, the benchmark cannot honestly test retrieval quality.

The next highest-value step is therefore:

> materialize a real Peloton public corpus from authoritative public sources, freeze it locally, and retain enough normalized metadata to support repeatable indexing and inspection.

This slice should deepen the benchmark where it matters rather than adding more benchmark scaffolding.

## Scope

### In Scope

- Peloton SEC filings fetched from EDGAR
- Peloton investor-relations materials fetched from Peloton IR sources
- raw file freeze
- normalized extracted text
- manifest metadata
- content hashes
- parser/extractor version tracking

### Out Of Scope

- synthetic internal documents
- anonymized long-form real documents
- OpenSearch ingest as a first-class outcome
- dense retrieval
- LLM answer generation
- ChatGPT/Gemini runbooks
- broader KE entity/state modeling

## Corpus Shape

The corpus should be stored in two layers.

### 1. Raw Layer

The raw layer stores the original fetched files exactly as downloaded.

Examples:
- HTML/XHTML filings from EDGAR
- PDF investor presentation
- PDF or HTML shareholder letter

Properties:
- immutable once frozen for a given corpus version
- used as the canonical source for future re-parsing
- preserved even if parsers improve later

### 2. Normalized Layer

The normalized layer stores extracted text and lightweight derived metadata for each raw file.

Properties:
- reproducible from raw files plus the parser version
- human-inspectable
- index-ready
- not the source of truth over the raw file

## Directory Layout

Recommended output layout:

- `corpus/generated/peloton_v1/public/raw/`
- `corpus/generated/peloton_v1/public/normalized/`
- `corpus/generated/peloton_v1/public/manifest.yaml`

The generated corpus should be versioned by directory name, not silently mutated in place.

If regeneration is needed, either:
- overwrite only during an explicit rebuild step for the same version, or
- write a new corpus version directory

The preferred long-term direction is new version directories, but this slice does not need a broad version-management system.

## Manifest Requirements

Each manifest entry should include:

- `document_id`
- `title`
- `source_type`
  - `edgar`
  - `investor_relations`
- `source_url`
- `raw_path`
- `normalized_path`
- `filing_or_report_date` when available
- `form_type` when applicable
- `content_hash`
- `extractor_version`

The manifest should let a later indexing step answer:
- what was fetched
- where it came from
- what local files correspond to it
- which parser version produced the normalized text

## Source Coverage

The first public corpus pass should stay intentionally small but real.

### EDGAR

Target:
- latest 10-K
- latest 2 quarterly 10-Qs
- optionally 1 to 2 recent 8-Ks only if they are clearly material

### Investor Relations

Target:
- latest shareholder letter
- latest investor presentation
- optionally the latest earnings transcript only if it is easy to source cleanly

This gives the benchmark:
- dense formal filings
- more presentation-oriented investor materials
- overlapping public narratives across document types

Do not broaden this slice into general web/news scraping.

## Implementation Approach

Split the pipeline into three narrow parts:

### 1. Fetch

Responsibilities:
- fetch Peloton public materials from authoritative public sources
- cache or freeze the downloaded raw files
- assign stable local paths and document identifiers

The implementation should borrow from the archived `seed/` utilities where useful, especially:
- `seed/edgar.py`
- `seed/parser.py`

Use those as reference or transplant material, not as a reason to pull forward the old app architecture.

### 2. Normalize

Responsibilities:
- parse raw files into extracted text
- preserve lightweight metadata useful for later indexing and inspection

Approach:
- EDGAR HTML/XHTML should use heuristics adapted from the archived filing parser
- PDF IR documents should use a simple first pass via `pymupdf`
- first pass should optimize for reliability and inspectability, not perfect document understanding

This slice does not need advanced table recovery or semantic restructuring.

### 3. Freeze

Responsibilities:
- write raw and normalized outputs into the generated corpus directory
- emit a manifest with hashes and metadata
- make reruns reproducible

The freeze step should be explicit and deterministic enough that later benchmark runs can rely on the generated corpus without depending on live fetches.

## Architecture Guidance

This slice should stay code-native and explicit.

Do not introduce:
- generic source registries
- broad ingestion frameworks
- abstract pipeline orchestration layers

Use straightforward modules for:
- EDGAR fetch
- IR fetch
- normalization
- manifest writing

The point is to create a trustworthy benchmark corpus, not to generalize ingestion architecture prematurely.

## Output Quality Requirements

The public corpus materialization slice is successful only if it produces:

- real Peloton public documents on disk
- normalized extracted text for those documents
- stable manifest metadata with hashes
- repeatable regeneration behavior

This slice is not successful merely because fetch utilities or parsers exist in code.

## Risks And Intentional Constraints

### Known Risk

Investor-relations sources may be less stable than EDGAR, especially around changing URLs or document hosting patterns.

Mitigation:
- keep the first IR set small
- prefer direct downloadable artifacts where possible
- record exact source URLs in the manifest

### Known Constraint

The first normalization pass may be lossy for some PDFs or richly formatted filings.

This is acceptable for this slice because the benchmark still needs a real corpus before parser quality can be tuned deeply.

## Success Condition

This slice is complete when the repo can produce a frozen Peloton public corpus under `corpus/generated/peloton_v1/public/` containing:

- the targeted SEC filings
- the targeted investor-relations documents
- raw source files
- normalized extracted text
- manifest entries with hashes and metadata

At that point, the benchmark has its first real public corpus foundation and the next slice can move to:
- indexing that corpus cleanly
- or wiring the real LLM answer path against it
