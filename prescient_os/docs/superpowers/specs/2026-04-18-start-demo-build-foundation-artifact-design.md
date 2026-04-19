# Start Demo Build Foundation Artifact Design

## Purpose

Create the second chronological benchmark artifact from a real build-phase document rather than a retrospective summary.

After the April 10 origin meeting summary, the next stable point-in-time artifact is the early `README.md` on `feature/start-demo-build-foundation`. It captures the first concrete translation of the reporting / board-deck wedge into an implementation program.

## Decision

Artifact `#2` will be a lightly normalized version of the early `README.md` from `feature/start-demo-build-foundation`.

This means:

- the README body remains substantially intact
- only the required memo frontmatter is added
- edits are limited to formatting normalization and obvious cleanup needed to fit the current manual-artifact format
- chronology is anchored by the early branch commits rather than rewritten from later memory

## Artifact Shape

The artifact will be stored in `prescient_os_data/memos/` as a memo-compatible markdown file so it can be loaded by the current private corpus tooling without schema changes.

Required frontmatter:

- `memo_id`: `start-demo-build-foundation-readme-2026-04-10`
- `title`: `Prescient OS`
- `as_of`: `2026-04-10`
- `topic_bucket`: `Product Strategy`
- `source_refs`: initially empty or limited to schema-supported refs only

The body should preserve the README content substantially as written, including:

- product framing as a portfolio intelligence platform
- the demo-build context
- the early architecture summary
- the non-negotiable invariants
- the build workflow and navigation pointers

## Provenance Policy

For this artifact, provenance is handled pragmatically:

- the normalized README copy is the artifact
- the source branch and commit history remain the supporting chronology

This slice should not widen the memo schema just to encode git provenance more richly.

If git-backed artifact provenance becomes benchmark-critical later, it should be handled as a separate schema or ingestion enhancement.

## Chronology Boundary

This artifact should capture the first concrete build phase after the April 10 origin meeting summary.

It should capture:

- the initial product framing after the meeting
- the early demo-build orientation
- the first hard architectural commitments
- the transition from wedge insight to implementation program

It should not pull in:

- later operator-first product-surface expansion from April 12 onward
- KE-first framing
- retrieval-benchmark framing

## Non-Goals

- no schema changes for richer git provenance
- no retrospective rewrite of the phase
- no merging with the later operator-first overview doc
- no attempt to summarize the entire branch history into one new memo

## Success Criteria

This slice is successful when:

- the early README has been normalized into a memo-compatible artifact
- the body remains faithful to the original README
- the artifact can be included in the private corpus without changing the loader schema
- the artifact clearly reads as the first implementation-phase document after artifact `#1`
