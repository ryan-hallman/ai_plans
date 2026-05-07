# Parts Catalog Procedure Linking Design

## Goal

Design a manufacturer-neutral way for Prescient OS to extract parts catalog
structure and link it to repair procedures while preserving provenance,
namespace scope, and the difference between explicit evidence and inference.

The first validation corpus is the Ferrari 360 spare parts catalog:

```text
/home/rhallman/Projects/manuals/05 - Spare Parts Catalogue.pdf
```

That catalog is a useful starting point because it has a regular structure:
`Tavola` diagrams, `Rif` callouts, part numbers, quantities, descriptions,
ordering notes, and a reverse numerical index. The design must not assume other
manufacturers will use the same layout.

## Why This Matters

The current workshop-manual dogfood path can answer from manual pages, but it
does not yet compile durable relationships between procedures and the parts
needed to perform them. Parts catalogs add a second evidence layer:

```text
procedure source
  -> component or operation
  -> catalog assembly/table
  -> part candidates
  -> quantity, fitment, and ordering constraints
```

This lets the KE answer questions such as "what parts are likely needed for
this procedure?" without pretending that a generated list is purchase-ready.
The feature is also a strong test of KE-first principles: structured evidence,
source authority, scoped terminology, reviewability, and stable contracts.

Vendor shopping is intentionally a later milestone. The current design should
produce normalized, provenance-backed part candidates that a future shopping
layer can consume, but it should not search vendors, compare prices, build carts,
or recommend purchases.

## Non-Goals

This design does not:

- build vendor search, pricing, carts, or purchase recommendations
- make a flat global parts ontology
- assume every catalog has Ferrari-style exploded diagrams or reverse indexes
- require perfect procedure artifacts before linking can begin
- generate unreviewed purchase-ready parts lists
- solve vehicle fitment or chassis/engine applicability globally
- make the workshop dogfood slice an automotive-only product

## Chosen Approach

Build an evidence-tiered parts linker on top of a namespaced parts catalog graph.

The catalog extractor builds a normalized artifact from manufacturer-specific
source structure. For Ferrari, the source profile understands `Tavola`, `Rif`,
`Dis.N/Part.N`, `Q.ty`, Italian and English descriptions, notes, and the
numerical index. Other catalog profiles can map different structures into the
same normalized graph.

The procedure linker connects workshop procedures to catalog graph nodes using:

- explicit part, assembly, table, or part-number mentions
- terminology mappings from the active retrieval profile
- subsystem/table title matches
- component and assembly adjacency
- source scope and applicability metadata

Every link must expose whether it is explicit or inferred. Inference is allowed
because useful procedure parts lists often require it, but inferred links must
carry rationale, confidence factors, and citations.

## Entity Namespacing

Parts, assemblies, procedures, and catalog nodes are not globally unique. They
must be scoped to the relevant domain, manufacturer, product family, model,
variant, year range, and source when that information is available.

Example namespace:

```yaml
namespace:
  domain: vehicle_repair
  manufacturer: Ferrari
  product_family: 360
  model: 360 Modena
  variant: null
  year_range: [1999, 2005]
  source_id: source-ferrari-360-spare-parts-2000
```

Readable ids may include the namespace:

```text
vehicle_repair:ferrari:360_modena:spare_parts_catalogue_2000:table:21
vehicle_repair:ferrari:360_modena:wsm_1999:procedure:gearbox_flange_grommet_replacement
```

Entities should have two identities:

1. **Scoped identity**: the exact part, assembly, or procedure within a
   namespace. This is the only identity safe for parts-list output.
2. **Canonical concept candidate**: an optional broader concept such as
   `brake_booster`, `water_pump`, or `gasket`. This helps terminology and
   cross-source matching but is not sufficient for ordering.

The same pattern should support later parts-based domains, such as industrial
equipment, medical devices, appliances, or facilities equipment, without changing
the core relationship model.

## Parts Catalog Artifact

`parts_catalog_artifact` is the maintained current-state projection of a catalog
source.

Core shape:

```text
catalog source
  -> catalog section/table/plate
  -> assembly or subsystem
  -> diagram reference/callout
  -> part
  -> quantity
  -> variant, fitment, and ordering notes
```

For the Ferrari catalog, representative nodes are:

```text
source-ferrari-360-spare-parts-2000
  -> table:tavola_32
  -> assembly:brake_booster_system
  -> callout:rif_13
  -> part:<part number>
  -> quantity:4
  -> description:gasket
```

The artifact should preserve source-specific notes because they affect safe
parts-list generation:

- `Not as spare part`
- `With complete assembly`
- `Possible fitting`
- left/right side variants
- color or trim suffix rules
- oversize or thickness options
- chassis and engine applicability requirements

The reverse numerical index should be extracted where available and used as a
validation signal. For Ferrari, the index maps part numbers back to table and
position. If the extracted table rows disagree with the index, the artifact
should keep both observations and mark the conflict or low confidence.

## Procedure Parts Link

`procedure_parts_link` connects a procedure structure to catalog graph nodes.
It can point to a table, assembly, callout, part, or set of candidate nodes.

Example:

```yaml
procedure_parts_link:
  procedure_ref: vehicle_repair:ferrari:360_modena:wsm_1999:procedure:gearbox_flange_grommet_replacement
  catalog_ref: vehicle_repair:ferrari:360_modena:spare_parts_catalogue_2000:table:26
  relationship_basis: strong_inferred
  confidence: 0.74
  rationale:
    - procedure mentions gearbox housing external components
    - catalog table 26 is Gearbox - Covers
    - component terminology overlaps with flange, cover, bearing, and gasket terms
  provenance:
    - source_id: source-ferrari-360-wsm
      unit_id: page-or-section-id
    - source_id: source-ferrari-360-spare-parts-2000
      unit_id: table-26-page-or-range
```

`relationship_basis` is required:

| Basis | Meaning | Default answer behavior |
| --- | --- | --- |
| `explicit` | Procedure or catalog text names the part, part number, assembly, or table directly. | Include by default. |
| `strong_inferred` | Procedure component maps cleanly to a catalog assembly or table. | Include by default with inference label. |
| `weak_inferred` | Likely supporting parts or consumables inferred from assembly context. | Show as suggested. |
| `uncertain` | Candidate needs review because scope, fitment, or mapping is ambiguous. | Hold for review. |

The answer surface must never flatten these tiers into a single undifferentiated
parts list.

## Link Policy

The linker emits candidates into three buckets:

```text
include_by_default
  explicit links
  strong inferred links with clear catalog/procedure support

show_as_suggested
  weak inferred links
  consumables or adjacent parts likely needed but not named

hold_for_review
  uncertain fitment
  ambiguous manufacturer/model scope
  parts marked not available separately
  alternates, oversizes, colors, left/right variants, possible-fitting notes
```

Generated answers should make the basis visible:

```text
Likely supporting parts, inferred from catalog table 32 rather than named in the
procedure:
- screw collar, qty 2
- gasket, qty 4
```

The system should also surface order and fitment warnings when relevant, such as
confirming chassis and engine applicability before ordering or noting that a
component may only be supplied as part of a complete assembly.

## Architecture And Data Flow

The first implementation should extend the existing workshop/manual dogfood
foundation rather than creating a separate automotive subsystem.

```text
raw sources
  -> source units/pages
  -> catalog structure extraction
  -> namespaced parts catalog artifact
  -> procedure structure extraction
  -> evidence-tiered procedure_parts_link candidates
  -> reviewable answer/planning surface
```

The catalog extractor has:

- a generic normalized output model
- source profiles for manufacturer or catalog-specific parsing conventions
- validation hooks, such as reverse-index checks when available
- extraction confidence and confidence factors

The procedure linker has:

- access to source structures, headings, pages, procedure-like sections, bullet
  steps, tools, warnings, and component terms
- terminology mappings from the active `vehicle_repair_v1` profile and scoped
  namespace layers
- evidence-tier classification
- rationale and citation output for every candidate

The output contract should support reviewable planning:

```yaml
required_parts:
  explicit: []
  inferred: []
  needs_review: []
source_links:
  procedure_pages: []
  catalog_tables: []
warnings:
  - Confirm chassis/engine applicability before ordering.
  - Some parts may only be supplied as part of a complete assembly.
```

Vendor shopping remains downstream. It can later consume normalized part numbers,
manufacturer labels, aliases, fitment notes, and provenance. It should not
influence catalog extraction or procedure linking.

## Error Handling

Uncertainty is part of the output model.

Catalog parse uncertainty:

- Keep the raw row citation.
- Mark extraction confidence low.
- Avoid using the row as an included part unless later validated.

Namespace ambiguity:

- If a part or procedure could belong to multiple models, variants, years, or
  sources, require scope clarification before using it in an ordered parts list.

Procedure/catalog mismatch:

- Return candidate tables with rationale instead of inventing a link.
- Mark the candidate `uncertain` unless the evidence supports a stronger tier.

Unavailable part behavior:

- Preserve notes such as `Not as spare part` and `With complete assembly`.
- Do not present unavailable subparts as directly orderable.

Variant-dependent parts:

- Left/right, oversize, color, chassis/engine, or `Possible fitting` cases go to
  `hold_for_review` unless the procedure or user scope resolves the variant.

## Verification

Verification should combine focused parser tests and eval cases:

- Parse representative Ferrari catalog tables into expected rows.
- Verify that extracted table/ref positions agree with the reverse numerical
  index for selected part numbers.
- Confirm catalog notes are preserved and surfaced in output.
- Link known workshop procedures to expected catalog tables with the correct
  evidence tier.
- Ensure inferred parts are never labeled explicit.
- Ensure shopping fields and vendor behavior are absent from this milestone.
- Regression-test manufacturer neutrality by keeping Ferrari-specific parsing in
  a source profile, not in the core artifact model.

The first measurable slice is:

```text
Can Prescient build a namespaced parts graph and produce reviewable explicit or
inferred links from procedures to parts?
```

## Later Milestone: Vendor Shopping

Vendor shopping should be built only after the linker produces trustworthy,
namespaced, provenance-backed part candidates.

The future shopping layer may:

- search vendors by normalized part number and aliases
- compare price, availability, shipping, and vendor confidence
- preserve fitment warnings and source provenance
- require user confirmation before purchase or cart creation

It must not weaken the evidence model. Shopping recommendations should remain
downstream of catalog and procedure trust, not a replacement for it.
