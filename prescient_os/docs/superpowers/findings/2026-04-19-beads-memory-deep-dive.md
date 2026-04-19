# Beads Memory System — Deep Dive

**Date:** 2026-04-19
**Source:** `/home/rhallman/Projects/vendor/beads` (sha at time of review: check `git -C ../vendor/beads rev-parse HEAD`)
**Motivation:** Prescient OS thesis — full-text / keyword search will outperform vector databases for horizontal knowledge retrieval over business, legal, and research documents. Beads is an agent-facing memory system that deliberately uses keyword search over embeddings, so its design choices are directly relevant.

---

## 1. Storage mechanism

Memories live in the shared `config` key-value table — no dedicated schema.

- **Schema:** `config(key VARCHAR(255) PRIMARY KEY, value TEXT)`
  — `internal/storage/schema/migrations/0006_create_config.up.sql:1-4`
- **Key convention:** `kv.memory.<slug>` (e.g. `kv.memory.auth-jwt`, `kv.memory.dolt-phantoms`)
  — prefix constants at `cmd/bd/memory.go:14`, `cmd/bd/kv.go:13`
- **No metadata columns:** no timestamps, no author, no tags, no embedding. Just `key` and `value`.
- **No indexes** beyond the primary key.

### Write path (`bd remember`)

1. `cmd/bd/memory.go:45-108` — `rememberCmd`
2. Slug generation (`slugify`, `memory.go:19-42`): lowercase, non-alphanumerics → `-`, first ~8 "words", cap 60 chars. E.g. `"always run tests with -race flag"` → `always-run-tests-with-race`.
3. `store.SetConfig(ctx, storageKey, insight)` — wraps `REPLACE INTO config (key, value) VALUES (?, ?)` in a Dolt transaction (`internal/storage/issueops/config_metadata.go:16`).
4. `store.CommitPending(ctx, getActor())` — write becomes a Dolt commit.

### Read path (`bd memories [keyword]`)

1. `cmd/bd/memory.go:111-192` — `memoriesCmd`
2. `store.GetAllConfig(ctx)` — fetches the **entire** config table into memory.
3. Filter in-process by prefix: `strings.HasPrefix(k, "kv.memory.")`.
4. Search (if keyword given): `strings.Contains(strings.ToLower(k), q) || strings.Contains(strings.ToLower(v), q)` — case-insensitive substring match on key **or** value.
5. Sort: alphabetical by key. No ranking, no relevance scoring.

### Direct recall (`bd recall <key>`)

`cmd/bd/memory.go:251-294` — single `GetConfig` lookup, returns raw value.

### Forget (`bd forget <key>`)

`cmd/bd/memory.go:194-249` → `DELETE FROM config WHERE key = ?` (`internal/storage/issueops/bulk_ops.go:93`), then commit.

### Session-start injection (`bd prime`)

This is the feature that matters most. Memories are **pushed into context at session start**, not pulled on demand:

- `cmd/bd/prime.go:228-280` — `formatMemoriesForPrime()`
- MCP mode (Claude Code): compact one-line-per-memory list, truncated to ~150 chars.
- CLI mode: full markdown sections.
- All memories are included, sorted alphabetically.

Agents don't normally search; they read their whole memory set at session start.

---

## 2. Why keyword search, not vectors

There is **no explicit ADR** rejecting embeddings. Zero references to `vector`, `embedding`, `semantic`, or `similarity` in memory code or design docs. The KV-store design doc (`docs/design/kv-store.md`) lists "Future Considerations" and embeddings are never mentioned.

Inferred rationale from the code and git history:

1. **Determinism.** Same query → same results, same order. Agents can't trust a drifting ML ranker.
2. **Zero external dependencies.** No embedding model, no vector DB, no re-embedding on import/export.
3. **Agent-driven keys.** Slug keys are derived from the first ~8 words of content, making them human-readable and agent-reasonable (`auth-jwt`, `dolt-phantoms`). The agent can cite/search by keyword it already remembers.
4. **Simplicity parity with predecessor.** Feature commit (Feb 2026, `ed8dc567`) explicitly replaces Claude Code's filesystem-based `MEMORY.md` — matching that simplicity.
5. **Scale assumption.** Typical agent memory counts are 10s–100s; full-table scan is cheap.

---

## 3. Why Dolt

What Dolt buys beads, specifically for memories:

- **Versioning as first-class.** Every memory write is a commit; supports blame, rollback, merge.
- **Sync for free.** `bd dolt push/pull` ships memories between agents/machines with no custom sync logic.
- **Shared multi-agent view.** `docs/design/dolt-concurrency.md:34-40` — "shared state must live on main." Agent A's memory is visible to Agent B on next sync.
- **Audit trail via commit metadata.** Timestamp/actor come from Dolt commits, not extra columns.
- **Portable export.** Memories round-trip through `.beads/memories.jsonl` with no backend-specific state (no embedding cache to invalidate).

Costs:

- Single-writer bottleneck in embedded mode (flock-serialized).
- `GetAllConfig()` full-table scan on every search — fine at memory scale, not document scale.

Why not a dedicated table? `docs/design/kv-store.md:76-82` considered it and deferred — namespace prefixing in `config` was deemed sufficient and avoided migration overhead.

---

## 4. Agent-facing contract

- **Session start:** `bd prime` injects all memories into the context window. Agents don't have to remember to look.
- **On-demand search:** `bd memories <keyword>` — substring match, alphabetical. Agent chooses the keyword.
- **Direct recall:** `bd recall <key>` when the agent knows the key.
- **No ranking exposed to the agent** — results are unordered by relevance.

Implicit assumptions:
1. Agents read memories at session start.
2. Agents keep memory content concise (prime truncates in MCP mode).
3. Agents search with terms that literally appear in the content (no semantic bridging).
4. Agents manage their own keys (remember what they created, or rely on push-at-prime).

---

## 5. Lessons for Prescient OS

### Translate directly

1. **Push, don't pull.** The biggest lesson. Inject top-N relevant documents into context at session start rather than making agents search. Cheap to implement, large quality improvement.
2. **Human-readable slug keys** derived from content. Agents reason about `legal-employee-noncompete-2024` better than UUIDs.
3. **Deterministic retrieval with a stable tie-breaker.** Even if vectors are added later, ordering must be reproducible.
4. **Portable export format** (JSONL of raw text + metadata) so storage backend stays swappable.
5. **Version-controlled writes** for attribution/rollback when multiple agents collaborate.

### Where beads breaks at document scale

1. **Full-table scan per query.** Fine at ~100 memories; fatal at 10k+ documents. Need real FTS (Tantivy / Meilisearch / Postgres FTS / Dolt FTS).
2. **No ranking.** Alphabetical is useless past a few hundred items. **Ship BM25 from day one** — this is where the FTS-beats-vectors thesis actually gets tested.
3. **No tokenization / stemming / normalization.** `Contains("dolt")` won't match punctuation variants, plurals, or synonyms.
4. **No structured metadata.** Documents need author/date/type/tags/sections as first-class fields, not text blobs.
5. **Text-only content.** No way to model citations, evidence links, or document structure.

### What the beads evidence actually validates

Beads proves a **well-designed agent contract** (proactive injection + human-readable keys + deterministic retrieval) often matters more than the retrieval algorithm. It does **not** validate FTS-beats-vectors at document scale — the corpus is too small for the algorithm choice to be tested seriously.

The thesis still needs to be validated on Prescient OS's actual corpus with a real ranker (BM25 + field boosts + phrase matching), not by copying beads' substring search.

---

## 6. Side-by-side

| Aspect | Beads memories | Prescient OS (recommended direction) |
|---|---|---|
| Storage | `config` KV table, `kv.memory.*` prefix | Dedicated `documents` table with schema |
| Search | Case-insensitive substring, in-process filter | FTS index with BM25 ranking |
| Indexing | None | Inverted index or dedicated FTS table |
| Ranking | Alphabetical by key | BM25 / field boosts / phrase match |
| Versioning | Dolt commit per write | Dolt/Git history or audit table |
| Metadata | None (text only) | Author, date, type, tags, sections, citations |
| Injection | All memories at `bd prime` | Top-N relevant docs at session start |
| Sync | `bd dolt push/pull` + JSONL export | Git/Dolt + index rebuild |
| Write pattern | Upsert via `--key`, in-place update | Versioned insert + soft delete |

---

## 7. Key file references

Core implementation:
- `cmd/bd/memory.go:1-314` — all memory commands
- `internal/storage/issueops/config_metadata.go:10-54` — SQL layer
- `cmd/bd/prime.go:228-280` — session-start injection
- `internal/storage/dolt/config.go:16-82` — Dolt transaction wrapper

Design / rationale:
- `docs/design/kv-store.md` — namespace + schema decisions (and explicit "why not config?" discussion)
- `docs/design/dolt-concurrency.md:1-56` — why Dolt, shared-view requirements
- `CHANGELOG.md` — grep for "memory"

Tests:
- `cmd/bd/memory_embedded_test.go:112-350` — search, update, concurrency patterns

---

## 8. Open questions / followups

- [ ] Prototype a BM25-ranked FTS baseline over the current eval corpus. Compare against any vector baseline to actually test the thesis.
- [ ] Decide whether Prescient OS wants Dolt-style versioned storage, or whether documents are immutable-with-revisions.
- [ ] Design the "session-start push" contract: what's the equivalent of `bd prime` for an agent briefed on a document set?
- [ ] Revisit: at what corpus size does unranked substring match become unusable? (Beads doesn't hit that limit; we will.)
