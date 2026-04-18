# Review: Chat Export and Manual Memo Support Design

## Overall

Mostly right-sized with a clear "this is benchmark support, not KE product" framing that keeps scope honest. Two specific issues — slightly overbuilt in taxonomy/metadata, underbuilt on eval fit.

## What's right-sized

- Sparse normalized schema that doesn't try to model every provider.
- Raw + normalized separation (assuming raw is load-bearing for re-normalization; see below).
- Explicit out-of-scope list (automation, ontology, ACLs, synthesis eval) is disciplined and should stay.
- "Evidence, not knowledge objects" framing defers every premature abstraction.
- Manual memo workflow stays manual.

## Overbuilt — recommend removing or deferring

**1. Organization model with seven named top-level buckets (§ Organization model).** This is product taxonomy, not benchmark support. A benchmark needs memos to have *a* `topic_bucket` field; it doesn't need a blessed taxonomy. Replace with: "memos carry a `topic_bucket` string; we do not predefine the taxonomy for v0."

**2. Enumerated optional session fields (`title`, `model`, `has_tool_calls`, `system_prompt_present`).** Either collapse into a single opaque `provider_metadata` blob or drop. The benchmark needs content + time + session grouping; everything else is speculative.

**3. `content_hash` without a stated use.** If it's for cross-provider dedup or re-export detection, say so. Otherwise drop it until needed.

**4. Two-layer raw/normalized structure is fine, but state why.** If raw exists because normalization may change and you want to reprocess, that's the justification. If it's just backup, say so. As written, raw looks like a product layer when it may only need to be an archive.

## Underbuilt — recommend adding

**1. Eval questions are absent from the spec.** The stated purpose is testing retrieval methodology, but there's no mention of who authors questions against this corpus, how many, or when. A corpus without a question plan is a dataset, not a benchmark. Add a single section: "v0 target of 15–25 questions, authored by the user against the imported corpus, covering specifically: temporal conflicts, pivots, superseded claims, and scope disambiguation."

**2. Ground truth / evidence keys not mentioned.** Given prior agreement that keys are the real bottleneck, the spec should at minimum acknowledge where they come from. One sentence: "the user authors typed evidence keys against imported sessions and memos; key schema is defined separately in the eval harness work."

**3. Conflict/pivot coverage as a design input.** The whole reason to use your own data is messy real pivots. Spec should explicitly say: "imported corpus is expected to contain temporally-inconsistent claims, pivoted decisions, and superseded memos — these are features, and test questions must exercise them." Right now this value is implicit; name it so implementation doesn't accidentally normalize it away.

**4. Privacy/sensitivity boundary.** Your conversation history likely contains third-party names, financial details, and personal content. Spec should say: "this corpus is treated as private, remains local, is not committed to shared repositories, and artifact sharing requires redaction review." Small but important.

**5. Session granularity per provider.** "Session" means different things across Claude, ChatGPT, Codex. One sentence on how a provider export maps to a normalized session (one conversation = one session? one day = one session?) avoids a mid-implementation decision.

**6. Corpus size target.** "v0 expects ~100–500 sessions and ~10–30 memos" lets the importer avoid scaling complexity. Without a target, it might be built for any scale.

## Structural nit

The "Why this is the right boundary" section reads as scope defense. Good intent, but the actual scope discipline is already enforced by the in-scope/out-of-scope lists and the "evidence not knowledge" framing. Could be trimmed or folded into the Purpose section.

## Recommended changes, ranked

1. Add a short section on eval-fit: question count target, conflict/pivot coverage as intentional, ground-truth authoring noted as separate work.
2. Delete or defer the seven-bucket taxonomy.
3. Collapse enumerated optional fields into `provider_metadata`.
4. Add a privacy/local-only statement.
5. State session-granularity rule per provider.
6. Justify or drop `content_hash` and the raw layer's role.

Items 1 and 4 are the load-bearing additions. The rest is trimming.

## Net

Scope is close to right but currently answers "what format do we import?" more completely than "what will we learn from importing it?" The second question is the point.

## Related

- Source spec: `docs/superpowers/specs/2026-04-18-chat-export-and-manual-memo-support-design.md`
- Retrieval-only eval recommendation: `docs/superpowers/findings/2026-04-18-rag-eval-retrieval-only-recommendation.md`
- Retrieval architecture thesis: `docs/superpowers/ideas/2026-04-18-retrieval-thesis.md`
