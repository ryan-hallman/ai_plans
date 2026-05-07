# Automotive Commercial Branch Governance Design

## Goal

Clarify that the active product track is the automotive workshop knowledge
assistant while preserving a generic knowledge-engine core that can later expand
to other evidence-heavy domains without a major refactor.

The near-term commercial product is automotive-specific. The underlying source,
evidence, retrieval, citation, review, artifact, and answer contracts should stay
domain-neutral unless a later approved spec deliberately changes that boundary.

## Context

Earlier Prescient OS direction documents describe a horizontal business
knowledge engine. The workshop-manual work began as a dogfood corpus for that
engine. The product decision has changed: the automotive assistant is now the
commercial wedge, not merely an internal validation corpus.

That change should not turn the core into a car-only codebase. Automotive should
own the user surface, domain terminology, vehicle scope, repair/manual workflows,
and commercial roadmap. The KE core should continue to model generic evidence and
answer quality.

## Branch Charter

The active product branch should be treated as the production track for the
automotive workshop assistant. A clearer branch name such as
`prod/automotive-workshop` or `prod/workshop-ke` is preferred when branch
administration is updated.

Work on this branch should optimize for:

- reliable automotive repair answers
- page, figure, table, and source-level provenance
- vehicle, variant, manual, and parts-catalog scope discipline
- reviewable claims and conservative answer behavior
- commercial usability for automotive repair and restoration workflows
- generic KE contracts that remain usable outside automotive

This branch should not accept unrelated platform expansion, old MVP parity work,
or general workflow breadth unless that work directly supports the automotive
product or a KE primitive used by it.

## Product Positioning

The current commercial product is an automotive workshop knowledge assistant.
Its first promise is:

> Ask scoped repair, manual, and parts questions and receive conservative,
> inspectable answers grounded in cited source evidence.

The product may become broader later, but the active product surface should be
automotive-specific enough to be useful and sellable.

## Architecture Boundary

Use an automotive vertical over a domain-neutral KE core.

Core modules may define concepts such as:

- `Source`
- `SourceUnit`
- `SourceStructure`
- `EvidenceSpan`
- `AnswerClaim`
- `Citation`
- `KnowledgeScope`
- `TerminologyMapping`
- retrieval plans and retrieval results
- review states and feedback promotion

Automotive modules may define concepts such as:

- `VehicleProfile`
- `WorkshopManual`
- `PartsCatalog`
- `Fitment`
- `RepairProcedure`
- repair terminology profiles
- shop-oriented UI and MCP adapters

The dependency rule is:

```text
automotive product code -> generic KE core
generic KE core -/-> automotive product code
```

Generic code must not import automotive modules. If a core module needs a
vehicle-specific field to work, the boundary is wrong or the field should be
modeled as generic scope/applicability metadata.

## Feature Admission Test

Every new feature on the production automotive branch must satisfy at least one
of these tests:

1. It improves the automotive workshop product directly.
2. It improves a generic KE primitive used by the automotive product.
3. It improves reliability, provenance, evaluation, deployment, or reviewability
   for the above.

Features that fail this test should be deferred, moved to a separate exploratory
branch, or tracked as parking-lot work. They should not land on the production
automotive branch.

## Guardrails

Allowed:

- automotive UI, terminology, seeded vehicle profiles, manual catalogs, parts
  catalogs, fitment scope, and repair-oriented evals
- generic evidence, retrieval, citation, claim, review, and answer contracts
- domain adapters that map automotive inputs into generic KE contracts

Not allowed without a new approved spec:

- old sponsor/operator workflows as a primary product concern
- broad onboarding or platform-shell work not needed by the automotive product
- car-only concepts inside generic KE modules
- purchase-ready parts recommendations without provenance, fitment warnings, and
  review boundaries
- ungrounded numeric or procedural answers
- feature work justified only by future non-automotive expansion

## Documentation Authority

Root agent guidance should point to this spec when deciding whether a proposed
change belongs on the active branch. Older horizontal-KE documents remain useful
for architectural primitives, but this spec supersedes them for product focus and
branch admission.

When documents conflict:

1. Follow this automotive commercial branch governance for product scope.
2. Follow the KE-first specs for domain-neutral core architecture.
3. Flag the conflict before implementing work that crosses the boundary.

## Persistent Agent Memory

Project memory should summarize the current scope as:

> Prescient OS is currently commercializing an automotive workshop knowledge
> product. Build the product surface for automotive users, while keeping the KE
> core domain-neutral so later vertical expansion does not require a major
> refactor.

This memory should replace any stale memory that describes the active product as
horizontal-only.

## Non-Goals

This governance change does not:

- rename the git branch by itself
- configure remote branch protection
- remove old specs or archived plans
- refactor existing code packages
- define pricing, packaging, or go-to-market details

Those can be handled as follow-on issues once the branch charter is accepted.

