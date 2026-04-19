# Origin Meeting Summary Artifact Design

## Purpose

Create the first point-in-time artifact for the PrescientOS retrieval benchmark from a real founder document instead of a synthetic retrospective memo.

The earliest known source material is:

- a raw transcript: `AI Enablement Strategy for PE Firms and Portfolios.txt`
- an existing human-readable meeting summary: `AI Enablement Strategy for PE Firms and Portfolios.md`

The benchmark should preserve the meeting summary as the first artifact because that is closer to what would realistically exist after the meeting than a newly written founder memo.

## Decision

Artifact `#1` will be the existing meeting summary, lightly normalized into the private corpus format already supported by the benchmark.

This means:

- the markdown body remains substantially intact
- only the required memo frontmatter is added
- edits are limited to normalization and removal of obvious hindsight if any is found
- the raw transcript remains supporting evidence, not the primary artifact body

## Artifact Shape

The artifact will be stored in `prescient_os_data/memos/` as a memo-compatible markdown file so it can be loaded by the current private corpus tooling without schema changes.

Required frontmatter:

- `memo_id`: `ai-enablement-strategy-pe-portfolios-2026-04-10`
- `title`: `AI Enablement Strategy for PE Firms and Portfolios`
- `as_of`: `2026-04-10`
- `topic_bucket`: `Product Strategy`
- `source_refs`: initially empty or limited to schema-supported refs only

The body should preserve the existing sections and wording as much as possible:

- `Executive Summary`
- `The Problem`
- `Themes Discussion`
- `Specific Ideas`
- `Future Directions`

## Provenance Policy

For this first artifact, provenance is handled pragmatically:

- the `.md` meeting summary is the artifact
- the `.txt` transcript is the supporting raw evidence

The current memo schema does not support arbitrary file-path references, and this slice should not widen the schema just to model transcript provenance more richly.

If transcript ingestion becomes benchmark-critical later, it should happen as a separate evidence-import slice rather than inside this artifact-normalization task.

## Chronology Boundary

This artifact is pre-code and pre-platform-build.

It should capture:

- the April 10, 2026 meeting
- the immediate interpretation of the PE / credit / PortCo opportunity
- the reporting and communication pain
- the board-deck / KPI-flow wedge

It should not pull in:

- later operator-first platform framing
- KE-first framing
- retrieval-benchmark framing

The first implementation-phase artifact will be handled separately as artifact `#2`.

## Non-Goals

- no schema changes for memo provenance
- no transcript import work in this slice
- no synthetic founder memo rewrite
- no attempt to merge this artifact with the later `feature/start-demo-build-foundation` phase

## Success Criteria

This slice is successful when:

- the existing meeting summary has been normalized into a memo-compatible artifact
- the body remains faithful to the original summary
- the artifact can be included in the private corpus without changing the loader schema
- the artifact clearly stands as the first chronological benchmark document
