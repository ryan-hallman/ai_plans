# Phase Evidence Packet Design

## Purpose

Define a lightweight, repeatable structure for reconstructing PrescientOS phase chronology from primary sources before promoting any point-in-time artifact as canonical.

The current memo workflow is too eager to promote derived artifacts like READMEs and summaries. That is acceptable when a real artifact is obviously authoritative, but it breaks down when the deeper rationale lives in chat history and commit sequencing.

Phase evidence packets exist to fix that.

## Decision

Before creating or revising a major chronological artifact, build a small evidence packet for the phase.

The packet should collect:

- primary chat sessions
- supporting chat sessions
- key commits
- optional derived artifacts that existed at the time
- a short phase summary that explains what changed and why

The packet is not the final public-facing artifact. It is the evidence bundle used to decide what the final artifact should be and how trustworthy existing derived documents are.

## Packet Scope

Each packet should cover one clear phase boundary only.

Good examples:

- April 10 origin / wedge thesis
- April 12 operator-first pivot
- April 16 KE-first pivot
- April 16-18 retrieval-benchmark narrowing

Bad examples:

- “everything before KE-first”
- “all early product history”

## Packet Contents

Each phase evidence packet should be a markdown document saved in the private data repo, separate from the final memo artifacts.

Recommended sections:

### 1. Phase Identity

- `phase_id`
- `title`
- `date_window`
- `status`
  - `draft`
  - `reviewed`
  - `artifact-derived`

### 2. Primary Sources

The smallest set of sources that best explain the phase.

For each source:

- source type
  - `chat_session`
  - `git_commit`
  - `existing_doc`
- identifier
- why it matters

### 3. Supporting Sources

Additional evidence that corroborates or contextualizes the primary sources.

### 4. What Changed

A concise explanation of the shift or state that defines the phase.

### 5. Why It Changed

The key rationale recovered from the evidence, especially from chats if that reasoning is not captured in the code or docs.

### 6. Candidate Artifact Decision

One of:

- `use existing artifact`
- `create reconstructed summary`
- `keep as evidence-only for now`

And a short reason.

## Storage Location

Store packets in the private data repo so they can reference private chat evidence without leaking into the product repo.

Recommended path:

- `prescient_os_data/evidence_packets/<phase_id>.md`

They are benchmark-support artifacts, not product code.

## Relationship To Memo Artifacts

Evidence packets are upstream of memo artifacts.

Workflow:

1. gather evidence packet
2. review packet
3. decide whether an existing artifact is sufficient
4. if not, create or revise the memo artifact from the packet

This keeps the chronology honest and reduces accidental hindsight.

## Non-Goals

- no new schema for corpus loading
- no automatic packet generation
- no attempt to replace the memo artifacts entirely
- no requirement that every tiny branch or commit cluster get a packet

## Success Criteria

This design is successful when:

- major chronology decisions are grounded in primary evidence
- existing docs are no longer treated as authoritative by default
- the rationale behind each phase can be traced back to specific sessions and commits
- memo artifacts become easier to trust and easier to revise
