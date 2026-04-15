# Recall & Memory System Design

## Status: Draft — Awaiting Review

## Problem

As an operator accumulates thousands of artifacts and hundreds of conversation threads over months of use, Prescient OS needs to recall the right context — across both structured artifacts and unstructured conversation history — without hallucinating, while being transparent about sources, disambiguating when needed, and admitting gaps.

Current limitations:
- No conversation persistence — copilot sessions are ephemeral
- No mechanism to promote conversation insights into durable artifacts
- No query expansion for domain-specific terminology or internal jargon
- No trust hierarchy distinguishing vetted facts from raw conversation context
- Flat search with no scoping — recall quality degrades as content grows

## Design Principles

Drawn from the AI Memory PRD and operator-first positioning:

1. **Automation proposes; users govern** — The copilot suggests artifacts. The operator approves, edits, or rejects. Nothing is silently persisted as "truth."
2. **Inspectable by default** — Every claim cites its source. The operator can click through to the exact passage.
3. **Structured over vague** — Durable memory lives in explicit artifact types, not raw chat logs.
4. **Source-grounded when possible** — Artifacts link to the conversation exchanges or documents that produced them.
5. **Honest gaps over confident guesses** — The system states what it doesn't know rather than inferring.

## Architecture Overview

### 3-Layer Recall Stack

| Layer | Loaded When | Purpose | Token Budget |
|-------|------------|---------|-------------|
| **L0 — Context Frame** | Every copilot turn | User role, org, current page/module, active KPI owners, recent action items | ~200-400 tokens |
| **L1 — Scoped Search** | On recall query | BM25 search narrowed by artifact type + topic tags, using LLM-generated query variants expanded by knowledge graph | Configurable, ~4000 tokens |
| **L2 — Deep Search** | On L1 miss or explicit request | Full BM25 across all visible artifacts + conversation chunks, broader variant generation | Configurable, ~8000 tokens |

The system starts lean and searches on demand. No large always-loaded summary of everything the operator has discussed.

### Recall Flow

1. Operator sends a message to the copilot
2. LLM classifies intent: `no_recall_needed`, `artifact_recall`, `conversation_recall`, `both`, or `fact_check`
3. For recall queries, LLM generates a search plan:
   - Artifact type scope (e.g., decision records, KPI snapshots)
   - Topic filters
   - 3-8 query variants (synonyms, rephrashings)
   - Whether to expand via knowledge graph
   - Optional time range hint
4. Knowledge graph aliases are fetched and added to variants if `kg_expand` is true
5. BM25 fan-out: each variant runs as a parallel OpenSearch query, scoped by artifact type, topic tags, user_id (for conversations), and org_id
6. Results merged, deduplicated by document ID, ranked by best BM25 score
7. Context assembled with provenance metadata, capped to token budget
8. LLM responds with citations, disambiguation, and gap disclosure
9. On L1 miss (zero or low-scoring results), system auto-escalates to L2 and flags the broadened search

## Data Model

### Conversation Storage

Conversations are stored in Postgres as a source of truth. OpenSearch indexes them asynchronously for recall.

**`conversations`** table:
- `id` (UUID)
- `org_id` (UUID, FK)
- `user_id` (UUID, FK)
- `title` (text, LLM-generated)
- `started_at` (timestamptz)
- `ended_at` (timestamptz, nullable)
- `status` (enum: active, closed)

**`conversation_exchanges`** table:
- `id` (UUID)
- `conversation_id` (UUID, FK)
- `chunk_index` (integer, ordering within conversation)
- `user_message` (text)
- `assistant_message` (text)
- `topic_tags` (text[], LLM-assigned)
- `artifact_refs` (UUID[], IDs of artifacts produced or referenced)
- `created_at` (timestamptz)

Chunking strategy: one exchange pair (user turn + assistant response) = one chunk. Long responses split across continuation chunks sharing the same `conversation_id` with sequential `chunk_index`. The user message is always preserved intact with its response.

Exchange pairs are written in real-time during the conversation. OpenSearch indexing happens asynchronously.

### Knowledge Graph

Postgres tables behind a `KnowledgeGraphPort` protocol. Designed for Postgres at current scale (low thousands of entities per org), with a clean migration path to a dedicated graph DB when cross-portfolio pattern recognition requires it.

**`kg_entities`** table:
- `id` (UUID)
- `org_id` (UUID, FK)
- `name` (text)
- `entity_type` (enum: person, company, initiative, metric, term)
- `aliases` (text[])
- `metadata` (jsonb)
- `created_at` (timestamptz)

**`kg_relationships`** table:
- `id` (UUID)
- `org_id` (UUID, FK)
- `subject_id` (UUID, FK to kg_entities)
- `predicate` (text, e.g., owns, leads, tracks, synonym_of, replaced_by, part_of, decided_in)
- `object_id` (UUID, FK to kg_entities)
- `valid_from` (timestamptz)
- `valid_to` (timestamptz, nullable — null means current)
- `source_artifact_id` (UUID, FK, nullable)
- `created_at` (timestamptz)

**`kg_aliases`** table (denormalized for fast query expansion):
- `id` (UUID)
- `org_id` (UUID, FK)
- `canonical_term` (text)
- `alias` (text)
- `context` (text, e.g., "Internal codename used by ops team")

Knowledge graph population:
- **At artifact creation** — LLM extracts entities and relationships as a side effect
- **At onboarding** — setup wizard captures company terminology, team members, KPI ownership
- **Operator correction** — when the operator says "we call that Project Redline," the system adds an alias (most valuable source — captures jargon no model knows)

### Artifact Modifications

Existing artifact schema extended with:
- `confidence` (enum: verified, approved, inferred, uncertain)
- `status` lifecycle (draft -> approved -> archived -> deleted)
- `source_type` (enum: conversation, upload, manual)
- `source_conversation_id` (UUID, FK, nullable)
- `source_exchange_ids` (UUID[], specific exchanges that produced this artifact)
- `source_excerpts` (jsonb[], exact passages with positional references)

### OpenSearch Index Updates

Artifact index adds:
- `topic_tags` (keyword array, for filtering)
- `confidence` (keyword, for filtering and ranking)
- `status` (keyword, for filtering)
- `title` field boosted in BM25 scoring

New conversation exchange index:
- `user_id` (keyword, mandatory filter — conversations are private)
- `org_id` (keyword, tenant scoping)
- `conversation_id` (keyword, grouping)
- `chunk_index` (integer, ordering)
- `user_message` (text, BM25-searchable)
- `assistant_message` (text, BM25-searchable)
- `topic_tags` (keyword array, for filtering)
- `artifact_refs` (keyword array, for join-like queries)
- `created_at` (date, for time-range filtering)

## Conversation to Artifact Pipeline

### Flow

```
Conversation (private, ephemeral working space)
  -> Copilot detects artifact-worthy content
    -> Suggests inline: "This looks like a decision record — Save | Preview | Dismiss"
      -> Save: artifact draft lands in Memory Inbox
      -> Preview: expands to show structured artifact, then Edit & Save | Dismiss
      -> Dismiss: nothing saved, conversation continues
        -> Memory Inbox: operator reviews pending drafts
          -> Approve / Edit / Merge / Reject
            -> Approved artifact: durable, searchable, visible per org permissions
              -> Source excerpt links back to conversation passage
```

### Detection Triggers

After generating each response, the LLM evaluates whether the exchange produced durable content:
- A decision was made ("let's go with value-based pricing")
- An action item was assigned ("Sarah needs to pull the Q1 numbers")
- A KPI insight emerged ("revenue is down 12% because of channel mix")
- A fact was established ("Project Redline is our warehouse consolidation initiative")
- A strategic position was articulated (thesis, rationale, risk assessment)

### Inline Suggestion UX

Lightweight by default, expandable on demand:
- **Save** — straight to Memory Inbox as draft, conversation continues uninterrupted
- **Preview** — expands to show the structured artifact (title, body, tags, source excerpt) with Edit & Save | Dismiss
- **Dismiss** — suggestion disappears

## Recall Trust Hierarchy

The LLM treats recalled content differently based on its source and curation status.

| Tier | Source | LLM Behavior | Example Phrasing |
|------|--------|-------------|-------------------|
| 1 — Verified | Approved artifacts with source excerpts | Cite as established fact | "According to the Q1 pricing decision..." |
| 2 — Approved | Approved artifacts without source excerpts (manually created) | Cite with attribution | "Per the project brief created by Sarah..." |
| 3 — Draft | Pending artifacts in Memory Inbox | Mention with caveat | "There's a pending note about warehouse timelines — it hasn't been reviewed yet" |
| 4 — Conversation | Raw conversation exchanges | Present as context, not fact | "In a conversation on Feb 12, you mentioned..." |
| 5 — Inferred | Knowledge graph relationships without artifact backing | Flag as inference | "Based on the relationship between Project Redline and the ops restructuring..." |

### Response Behaviors

**Multiple hits, same topic:** Disambiguate rather than pick one. "I found two decision records about pricing — the B2B channel restructure from January and the retail margin adjustment from March. Which are you asking about?"

**Conflicting information:** Surface the conflict. "The Q1 decision record says we chose value-based pricing, but a later conversation from March mentions reverting to cost-plus for enterprise accounts. These may need reconciliation."

**No hits:** State the gap clearly. "I don't have any artifacts or conversation history about warehouse consolidation. Would you like me to create a research task for this?"

**Stale hits:** Flag recency. "The most recent information I have on this is from October 2025 — it may be outdated."

**Cross-tier synthesis:** Never synthesize across tiers without disclosure. If an answer combines a verified artifact with a conversation excerpt, both sources are labeled explicitly so the operator knows which parts are vetted and which are raw context.

## Query Variant Generation

The LLM generates search variants using the same pattern as command-line agents doing multi-keyword grep.

### Search Plan Structure

```json
{
  "scope": ["decision_record", "conversation"],
  "topic_filter": ["pricing", "revenue"],
  "variants": ["pricing model change", "pricing decision", "revenue model"],
  "kg_expand": true,
  "time_hint": "2026-Q1"
}
```

### Knowledge Graph Expansion

When `kg_expand` is true:
1. Extract key terms from variants
2. Query `kg_aliases` for each term
3. Query `kg_entities` aliases array for each term
4. Add discovered aliases to the variant list
5. Example: "Project Redline" -> also search "warehouse consolidation", "DC rationalization", "logistics restructuring"

This provides semantic breadth without vector embeddings. The LLM does the "understanding" step, BM25 does the "does this exist" step. The knowledge graph fills the jargon gap that neither the LLM nor an embedding model would know about.

## Memory Inbox UX

A review queue for artifact proposals. Accessible from the main nav with a badge count for pending items.

### What Lands Here
- Copilot-suggested artifacts from conversations (operator clicked Save or Edit & Save)
- System-proposed artifacts from uploaded documents (future phase)
- Artifacts in draft status awaiting promotion to approved

### Per-Item View
- Artifact type label (decision record, action item, KPI insight, etc.)
- Title and body preview
- Source link — click through to the conversation exchange that produced it
- Topic tags (editable)
- Actions: Approve | Edit | Merge (if duplicate detected) | Reject

### Behavior
The inbox is not urgent. It's a curation surface the operator visits at their own rhythm — daily, weekly, whatever fits. Draft artifacts are still searchable during recall at Tier 3 trust, so nothing is lost if review is delayed.

## Artifact Library UX

The curated knowledge base. All approved artifacts browsable in one cross-cutting view.

### Filters
- Artifact type
- Topic tags
- Status (approved, archived)
- Created by
- Date range
- Source type (from conversation, from upload, manually created)

### Artifact Detail View
- Structured content (artifact body)
- Source citations — exact excerpts with links to source conversation or document
- Related artifacts (cross-references)
- Version history (diffs between revisions)
- Knowledge graph entities linked to this artifact
- Status controls: archive, edit, delete

### Relationship to Existing Module Pages
The Actions page, KPI dashboard, and Briefing page already show specific artifact types in context. The Artifact Library is the cross-cutting view — all types in one place with search and filtering. Module-specific pages remain the primary working surface; the Library is for exploration and curation.

## Scope Boundaries

### Builds on (no changes needed)
- 8 artifact types with cross-references and versioning
- Artifact store with OpenSearch indexing
- Org/role-based visibility rules
- BM25 search infrastructure
- Copilot planner loop

### Needs Modification
- OpenSearch artifact index — add topic_tags, confidence, status fields, boost title
- Artifact domain entity — add confidence level, source_type, source_excerpts[], lifecycle status
- Copilot planner — add intent classification, query expansion with KG lookup, trust-tier-aware context assembly

### New Bounded Contexts
- **conversations** — conversation + exchange storage, OpenSearch indexing, privacy enforcement (user_id scoping)
- **memory** — Memory Inbox (artifact proposal review queue), artifact promotion workflow, merge/dedup
- **knowledge_graph** — entities, relationships, aliases, temporal queries, behind a KnowledgeGraphPort protocol

### New UI Surfaces
- Memory Inbox page
- Artifact Library page
- Inline artifact suggestion component (in copilot chat)
- Response provenance annotations (source citations in copilot responses)

### Not In Scope
- File/document uploads as source material (future phase)
- Enterprise connectors (Slack, Drive, Notion)
- Branching/merging of artifacts
- Cross-portfolio recall (PE user searching across companies)
- Retention policy engine
- Cold storage for aged content
- Vector embeddings or semantic search (BM25 + LLM query variants + knowledge graph expansion is the search strategy)

## Build Order

1. **Conversations context** — exchange storage, OpenSearch indexing (prerequisite for everything)
2. **Knowledge graph context** — entities, relationships, aliases, KnowledgeGraphPort (enables query expansion)
3. **Recall pipeline** — intent classification, query expansion, scoped BM25 fan-out, context assembly with trust tiers
4. **Artifact modifications** — confidence, lifecycle status, source_type, source_excerpts
5. **Memory Inbox** — artifact proposal flow, review queue, promotion workflow
6. **Artifact Library UI** — cross-cutting artifact browser with filters and detail view
7. **Inline suggestion component** — copilot artifact proposals during conversation
8. **Provenance annotations** — source citations in copilot responses

## Technical Decisions

- **Search engine:** BM25 via OpenSearch. No vector embeddings. LLM-generated query variants provide semantic breadth. Knowledge graph provides jargon expansion.
- **Knowledge graph storage:** Postgres tables behind KnowledgeGraphPort protocol. Migrate to dedicated graph DB when cross-portfolio features require complex traversals.
- **Conversation privacy:** Conversations are filtered by user_id at the OpenSearch query level. A user's search never returns another user's conversation exchanges. Artifacts created from conversations follow org visibility rules.
- **Conversation indexing:** Asynchronous. Exchanges are written to Postgres in real-time and indexed in OpenSearch with a slight delay.
- **Context window management:** Token budgets per layer (L0: ~400, L1: ~4000, L2: ~8000). Results ranked by BM25 score, then included top-down until budget is exhausted. Remaining results are dropped with a count note to the LLM ("12 additional results omitted"). The system searches narrowly first (L1) and broadens only on miss (L2).

## Success Criteria

- Operator can ask about a decision made 6 months ago in a conversation and get the right answer with source citation
- System correctly says "I don't know" when no relevant artifacts or conversations exist
- Knowledge graph aliases enable recall of content using internal jargon the LLM wouldn't otherwise know
- Memory Inbox review takes less time than manually creating the artifacts would
- Artifact Library enables discovery across all knowledge types in a single search
- No degradation in recall quality as artifact count grows from hundreds to thousands
