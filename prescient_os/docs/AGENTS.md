# AGENTS.md

Instructions for design documentation under `docs/`.

## Docs Repo Authority

`docs/` is a symlink to the separate docs repository at
`../ai_plans/prescient_os/docs`. The active design authority is that repository
on branch `ke-first-greenfield`.

- Write design docs, specs, findings, plans, and ideas under `docs/`.
- Commit docs changes from the `../ai_plans` working tree, not from this repo.
- Do not `git add -f docs` from `prescient_os`; the symlink is intentionally
  excluded here.
- Archived docs are reference only. If they conflict with `ke-first-greenfield`,
  follow the active branch and call out the mismatch.

## Taxonomy

Use the existing neighboring structure and naming style:

- `docs/superpowers/ideas/` - brainstorming, proposals, theses, and options not
  yet committed to a plan.
- `docs/superpowers/findings/` - investigations, code reviews, and vendor deep
  dives. Use `YYYY-MM-DD-kebab-name.md`.
- `docs/superpowers/specs/` - formal specs for durable design decisions.
- `docs/superpowers/plans/` - implementation plans derived from approved specs.

## Rationale Placement

- Specs hold durable design why: problem framing, chosen approach, rejected
  alternatives, assumptions, and non-goals.
- Plans hold execution why: sequence, task split, dependencies, and risk.
- Beads holds operational why: issue purpose, acceptance criteria, and links to
  specs or plans.

Do not duplicate full design rationale in Beads when the spec already contains
it.
