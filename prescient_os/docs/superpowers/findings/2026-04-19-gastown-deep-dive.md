# Gas Town — Deep Dive

**Date:** 2026-04-19
**Source:** `/home/rhallman/Projects/vendor/gastown`
**Companion:** `2026-04-19-beads-memory-deep-dive.md` (gastown sits on top of beads)
**Motivation:** Gastown is a multi-agent orchestration layer built on beads. Prescient OS is a knowledge/memory engine, not an orchestrator — but gastown's agent ↔ state contract (hooks, mailboxes, identity, proactive injection) is directly adjacent to the surface Prescient OS has to expose. Question: what do we steal, what do we skip?

---

## 1. What gastown is, stripped of metaphor

| Gastown term | What it actually is |
|---|---|
| **Rig** | Project container = directory wrapping a git repo + its own beads/Dolt database namespace |
| **Polecat** | Persistent worker identity bound to a git **worktree** (not a full clone). Sessions are ephemeral (tmux); the worktree and identity survive |
| **Hook** | Durability primitive. A bead with `status=hooked` + agent's `hook_bead` field pointing to it. "The hook is the work assignment that survives restart." |
| **Mailbox** | A query over beads of `issue_type=message` where `assignee` matches the agent. Persistent async message queue built on beads |
| **Nudge** | Synchronous tmux send-keys to a running session. Ephemeral — doesn't survive restart |
| **Convoy** | Bead that bundles multiple work beads for tracking/visibility |
| **Mayor** | An agent role (not a data structure). One per town; reads full workspace state and dispatches work |
| **Witness** | Health monitor agent that detects stalled/zombie polecats |
| **Refinery** | Merge-queue worker (Bors-style). Code-specific; ignore for our purposes |

Core insight: **everything is a bead.** Work, messages, agents, roles, convoys, escalations. The Dolt `issues` table is the single source of truth. Agent identity is a string (`gastown/polecats/toast`), state is a row, handoffs are INSERTs.

---

## 2. Persistent state model

### Storage

- **One Dolt SQL server per town**, port 3307, MySQL protocol. Data at `~/.gt/.dolt-data/`.
- **One Dolt database per rig** (namespace). `hq` for town, `gastown` for gastown rig, etc.
- **All writes on `main`** (`docs/design/dolt-storage.md:75-95`). Transaction discipline (`BEGIN` / `DOLT_COMMIT` / `COMMIT`) instead of branch proliferation. Chose this explicitly to fix cross-agent visibility gaps from an earlier per-polecat-branch model.
- **Worktrees don't have their own DBs.** `.beads/redirect` symlinks them back to `mayor/rig/.beads`. Single shared view per rig.
- **Cross-rig routing** via `routes.jsonl` prefix map (`hq-` → town, `gt-` → gastown rig, etc.). Agents never hardcode paths; the bead ID prefix encodes the database.

### Schema (`issues` table)

`docs/design/dolt-storage.md:101-122`. Key columns beyond standard beads:

- `hook_bead` — which bead is on this agent's hook
- `role_bead` — reference to role-template bead
- `agent_state` — polecat lifecycle (working, idle, stalled, zombie, done, stuck)
- `wisp_type` — TTL-based compaction class
- `metadata` — JSON blob for extensibility

### Clean restart flow

1. Tmux session dies (crash, compaction, intentional).
2. Next session starts; Claude Code's SessionStart hook fires.
3. Hook runs `gt prime [--role]` which injects:
   - Role directives (hardcoded + town-level + rig-level markdown overrides)
   - The agent's hooked bead (if any)
   - Pending mail, prioritized
   - Molecule checkpoint / wisp (mid-workflow state)
4. `.beads/PRIME.md:1-16` — "Propulsion Principle": **if there's a hook, execute immediately, no confirmation.**

No local checkpoint files. No session-scoped caches. State is in Dolt; injection is deterministic.

### Code paths

- `internal/cmd/hook.go:27-49` — hook attach/detach + the "durability primitive" doc comment
- `internal/cmd/prime_output.go:25-95` — session-start context rendering
- `internal/polecat/types.go:8-63` — polecat lifecycle state machine
- `internal/mail/mailbox.go:37-150` — mailbox query over beads + prioritization
- `internal/cmd/sling.go:25-110` — work dispatch (spawn polecat, hook bead, spawn tmux)

---

## 3. Coordination contract

### Two message channels

- **`gt nudge`** — send-keys into a running tmux pane. Immediate, ephemeral, lost on restart. For synchronous pokes ("try X approach").
- **`gt mail send`** — INSERT a bead of `issue_type=message`. Persistent, re-injected on next session start via `gt mail check --inject`. For task handoffs, context delivery.

### Handoff pattern

```
Mayor: gt convoy create "task X" <bead-ids...>
Mayor: gt mail send <target> --stdin <<<"context + instructions"
Mayor: gt sling <bead> <rig>
         → auto-spawn polecat if needed
         → hook bead on polecat
         → spawn tmux session
         → SessionStart → gt prime injects hook + mail
         → agent executes immediately
Agent:  gt done → push branch, submit MR, clear hook, mark status
Mayor:  monitors via bead queries
```

### Discovery is push, not pull

Agents never poll. They wait for:
1. Hooks (injected at session start)
2. Mail (injected at session start)
3. Nudges (only if session is live)
4. External human / mayor instructions

This matches beads' memory model: the system pushes context at session start; agents don't search.

---

## 4. Intersection with retrieval/memory

**Gastown has no retrieval engine.** It doesn't rank, tokenize, index, or search documents. Beads are metadata-rich work items, not a document store.

What gastown owns:
- Work distribution + lifecycle
- Agent identity + routing
- Context injection at session start
- Persistent async mailboxes
- Checkpoint-based workflow continuity

What gastown does not own (and Prescient OS must):
- Keyword / FTS / semantic search over documents
- Ranking and relevance
- Document indexing + chunking
- Evidence chaining, citation graphs
- Query interpretation

The two systems are **orthogonal and composable.** Gastown tells agents *where to run and what to pick up*. Prescient OS tells agents *what they need to read to answer*.

---

## 5. Takeaways for Prescient OS

### Consume vs. copy vs. ignore

- **Consume (if practical):** SessionStart injection pattern, beads-as-state-table, mail-over-beads, identity + prefix routing, all-writes-on-main discipline.
- **Copy (reimplement the idea):** hooked work survives restart → retrieval phase checkpoints survive restart; role directives → retrieval-role directives per document type.
- **Ignore:** molecules/formulas (predetermined step-wise workflows don't match iterative retrieval), refinery/Bors merge queue (code-specific), convoy TUI, polecat namepool (`toast`, `nux`), 3-tier Deacon/Boot/Overseer escalation ladder, witness's specific stuck-polecat detection.

### Patterns worth stealing, concretely

**1. Session-start context injection as the primary context channel.** Stop making agents search. When a research agent starts, push the phase checkpoint + top-ranked documents into context automatically. This is the beads `prime` pattern applied to documents.

**2. Beads rows as phase state, not local files.** A research phase is a row in Dolt with `issue_type=research_phase`, `hook_bead` pointing to itself, `metadata` JSON holding the query + found doc IDs + next phase pointer. On restart, `gt prime` re-injects it. Zero local checkpoint management.

**3. Mail-as-handoff between retrieval phases.** Phase 1 completes → `gt mail send synthesis/bob -s "phase 1 complete" --stdin <<<"docs: [...]; task: summarize"`. Phase 2 agent sees it on next session. No temp files, no coordination code.

**4. Identity + prefix routing.** `researcher/case-law/alice`, `researcher/statutes/bob`, `researcher/synthesis/charlie`. Bead prefix encodes which DB. No hardcoded paths in agent code.

**5. Status-is-truth health monitoring.** Rather than heartbeats, query for phases with stale `updated_at`. If a retrieval phase has been "in_progress" for 5+ minutes without update, escalate or restart. Cheap, durable, reuses existing state.

### Where gastown breaks at knowledge-engine scale

- **Beads are not a document store.** Gastown assumes O(100s) of beads per rig. We'll have O(millions) of documents. Store doc references + metadata in beads; keep full text in an FTS index.
- **`gt prime` injects everything.** Fine for a handful of role directives + one hooked bead. If we naively inject "all relevant docs," we blow the context window. **Injection must be ranked and capped** — which is exactly the retrieval problem. Gastown's injection model implicitly assumes an unlimited context budget.
- **No ranking primitives.** Priority P0–P4 + alphabetical. For retrieval, we need BM25/field-boosted/phrase-aware ranking. That layer is ours to build; gastown doesn't offer anything here.
- **Iterative research > step-wise molecules.** Molecules are fine for "run test → fix → run test." Research loops iterate on findings, revise plans mid-execution, and branch. Wisps can checkpoint linear progress, but they don't model plan revision.

---

## 6. Crossover table: gastown primitive → Prescient OS analog

| Gastown | Prescient OS analog |
|---|---|
| Hook (hooked bead) | Research phase checkpoint row |
| Mail (message bead) | Inter-phase handoff: docs found + next-phase task |
| Agent identity (`gastown/polecats/toast`) | Researcher identity (`researcher/case-law/alice`) |
| Beads prefix routing | Evidence-index prefix routing (`qa-*`, `case-*`, `stat-*`) |
| Convoy | Research batch (bundled queries + synthesis phase) |
| Role directives | Per-document-type retrieval directives |
| `gt prime` injection | Session-start: inject phase + top-N ranked docs |
| Witness stuck detection | Stale-phase detection via `updated_at` |

---

## 7. Things to skip

- **Molecules / formulas.** Retrieval is iterative, not step-wise.
- **Refinery / merge queue.** Code-specific.
- **Witness lifecycle minutiae.** Build our own stall detection tuned to retrieval, don't port the polecat-specific logic.
- **TUI + web dashboard.** Different feature surface; build ours separately.
- **3-tier escalation ladder.** Over-engineered for our error paths.
- **Namepool (`toast`, `nux`, etc.).** Cute, not load-bearing.

---

## 8. Key file references

Architecture:
- `docs/design/architecture.md` — two-level beads, agent taxonomy, rig layout
- `docs/design/dolt-storage.md` — schema, all-on-main writes (lines 75-95, 101-122)
- `docs/HOOKS.md:145-252` — SessionStart injection flow
- `docs/concepts/identity.md` — `BD_ACTOR`, attribution
- `.beads/PRIME.md:1-41` — propulsion principle

Code:
- `internal/cmd/hook.go:27-49, 210-431` — hook durability + injection
- `internal/cmd/prime_output.go:25-95` — context render
- `internal/polecat/types.go:8-63` — lifecycle
- `internal/mail/mailbox.go:37-150` — mailbox-over-beads
- `internal/cmd/sling.go:25-110` — dispatch

Rationale:
- `CHANGELOG.md` (2026 entries) — why per-polecat branches were abandoned, why hooks replaced ad-hoc queues

---

## 9. Open questions / followups

- [ ] Prototype a minimal "phase bead" schema: what fields does a Prescient OS research phase need beyond gastown's `hook_bead` + `metadata`?
- [ ] Decide: does Prescient OS share the town's Dolt server, or run its own? (Probably its own — document scale is different.)
- [ ] Define the injection budget contract: at session start, how many tokens are we willing to spend on pre-retrieved docs vs. role directives vs. mail?
- [ ] Evaluate whether gastown's all-writes-on-main discipline works when one of the "writes" is a retrieval-index update. FTS index rebuild is expensive; can't be a per-bead Dolt commit.
