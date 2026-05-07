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

The linker connects workshop procedures or source units to catalog graph nodes
using:

- explicit part, assembly, table, or part-number mentions
- terminology mappings from the active retrieval profile
- subsystem/table title matches
- component and assembly adjacency
- source scope and applicability metadata

Every link must expose whether it is explicit or inferred. Inference is allowed
because useful procedure parts lists often require it, but inferred links must
carry rationale, confidence factors, and citations.

Implementation should be milestone-gated. The durable model includes inferred
links, but v0 should prove catalog extraction and explicit linking before any
inferred parts are included in default answers.

## Milestone Sequence

### Milestone 1: Catalog Extraction V0

Extract a namespaced parts graph from the Ferrari spare parts catalog only.

Included:

- catalog table metadata
- row-level part records
- callout references
- part numbers
- quantities
- Italian and English descriptions when parseable
- source notes such as `Not as spare part` and `Possible fitting`
- reverse numerical index records
- row-level confidence factors

Excluded:

- procedure linking
- inferred parts
- vendor shopping
- cross-catalog dedupe

This milestone is complete only when representative Ferrari tables parse above
the thresholds in the Verification section.

### Milestone 2: Explicit Source-Unit Linking V0

Link workshop source units to catalog rows only when the source unit explicitly
mentions a part number, catalog table, assembly/table title, or exact part
description that can be matched to one catalog row or table within the active
namespace.

V0 does not require first-class procedure artifacts. It may link from page,
section, or source-unit ids. Procedure artifacts can consume the same links later
when procedure extraction exists.

Excluded:

- `strong_inferred` and `weak_inferred` output
- generated freeform rationale
- purchase-ready parts lists

### Milestone 3: Inferred Linking V1

Add `strong_inferred` and `weak_inferred` candidates after explicit linking has
ground truth and catalog extraction is stable. Inference requires deterministic
or rubric-backed rationale, confidence factors, and eval cases that pin the
boundary between tiers.

### Milestone 4: Review Surface V1

Expose explicit, inferred, and `needs_review` candidates in the answer/planning
surface with citations, fitment warnings, and source notes. The surface may help
the user inspect a parts list, but still does not perform vendor shopping.

This milestone is complete only when the rendered output schema is covered by
eval cases and at least one reviewer has used the surface to accept, reject, or
annotate at least 10 candidate parts-list decisions.

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

## Cross-Namespace Identity Policy

V1 does not deduplicate artifacts across namespaces. A Ferrari 360 Modena row and
a Ferrari 360 Spider row remain separate scoped records even if they share a part
number. This keeps fitment, source edition, and catalog-note differences visible.

Shared part numbers, matching manufacturer part ids, or matching canonical
concept candidates may create equivalence observations:

```yaml
equivalence_observation:
  basis: same_manufacturer_part_number
  refs:
    - vehicle_repair:ferrari:360_modena:spare_parts_catalogue_2000:part:183100
    - vehicle_repair:ferrari:360_spider:spare_parts_catalogue_2001:part:183100
  trust_state: extracted
```

Those observations are useful for search and review, but they do not merge the
underlying scoped records in v1.

Trust states reuse the artifact-substrate vocabulary:

| State | Meaning |
| --- | --- |
| `extracted` | Emitted by extraction or linking and not yet reviewed. |
| `reviewed` | Accepted or corrected by a human reviewer for the scoped context. |
| `disputed` | Challenged by a reviewer or contradictory evidence. |
| `superseded` | Replaced by a newer source, extraction run, or reviewed decision. |
| `stale` | Referenced source rows, fingerprints, or terminology versions changed and the observation needs re-derivation. |

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

## Catalog Extraction Strategy

Catalog extraction is the first load-bearing risk. V0 should be
deterministic-first with bounded model assistance only after candidate structure
has been extracted.

For the Ferrari catalog, extraction should use page text and layout before image
interpretation:

1. Detect table boundaries from repeated `Tavola`, `TAV.`, and footer patterns.
2. Split diagram pages from tabular parts-list pages.
3. Parse parts-list rows from layout text using the known column pattern:
   `Rif`, `Dis.N/Part.N`, `Q.ty`, Italian description, English description.
4. Attach multiline notes to the preceding row when indentation and language
   pairing support that relationship.
5. Parse the numerical index separately and cross-check table/ref locations.
6. Use an LLM only to classify already-extracted note text or resolve ambiguous
   multiline row continuation, not to invent rows or part numbers.

OCR-derived text is in scope. OCR errors in part numbers are catastrophic, so
part-number records from low-confidence OCR or malformed text extraction should
be rejected or sent to `needs_review` unless they agree with an independent
catalog signal such as a reverse index, repeated row occurrence, or
manufacturer-specific part-number format check.

Field robustness should be tracked separately:

| Field | Expected robustness | Notes |
| --- | --- | --- |
| `part_number` | high | Usually numeric and column-aligned. |
| `table_ref` | high | Repeated in title/footer and index. |
| `callout_ref` | high | Numeric or `-`; may repeat for alternates. |
| `quantity` | medium | May be blank, `-`, or variant-dependent. |
| `description_en` | medium | Multi-line descriptions can wrap. |
| `description_it` | medium | Useful for bilingual terminology but not required for English answers. |
| `notes` | fragile | Includes fitment, availability, color, and oversize constraints. |

Extraction confidence is row-level in v0, with optional field-level diagnostics.
Do not expose decorative numeric confidence until the factors are defined and
used. Store confidence factors with polarity:

```yaml
confidence_factors:
  supports:
    - required_fields_parsed
    - row_shape_matched_expected_profile
    - quantity_parsed_as_valid_value_or_known_marker
    - table_ref_agreed_with_reverse_index
  weakens:
    - note_continuation_required_model_assistance
    - duplicate_callout_or_duplicate_part_number_required_review
    - source_page_layout_was_malformed
```

If a numeric score is added later, it must be derived from these factors and
documented with threshold behavior.

## Procedure Parts Link

`procedure_parts_link` connects a procedure structure to catalog graph nodes.
It can point to a table, assembly, callout, part, or set of candidate nodes.
In v0, `procedure_ref` may be a source unit or section reference rather than a
durable `procedure_artifact`.

M2 explicit example:

```yaml
procedure_parts_link:
  procedure_ref: vehicle_repair:ferrari:360_modena:wsm_1999:source_unit:brake-system-service
  catalog_ref: vehicle_repair:ferrari:360_modena:spare_parts_catalogue_2000:part:183100
  relationship_basis: explicit
  confidence_factors:
    supports:
      - exact_part_number_match
      - namespace_scope_matches
      - reverse_index_agrees
    weakens: []
  rationale_facts:
    - source: procedure
      fact: source unit explicitly mentions part number 183100
    - source: catalog_row
      fact: part 183100 appears in table 33, callout 12
  provenance:
    - source_id: source-ferrari-360-wsm
      unit_id: page-or-section-id
    - source_id: source-ferrari-360-spare-parts-2000
      unit_id: table-33-page-or-range
```

M3+ inferred links use the same shape with `relationship_basis:
strong_inferred` or `weak_inferred` and rationale facts describing the
terminology, namespace, and catalog-context evidence.

`relationship_basis` is required:

| Basis | Meaning | Default answer behavior |
| --- | --- | --- |
| `explicit` | Procedure or catalog text names the part, part number, assembly, or table directly. | Include by default. |
| `strong_inferred` | Procedure component maps cleanly to a catalog assembly or table through deterministic terminology, title, and namespace support. | Include by default with inference label after v1 gates pass. |
| `weak_inferred` | Likely supporting parts or consumables inferred from same-table or adjacent-assembly context but not named by the procedure. | Show as suggested. |
| `uncertain` | Candidate needs review because scope, fitment, or mapping is ambiguous. | Place in `needs_review`. |

The answer surface must never flatten these tiers into a single undifferentiated
parts list.

## Tier Rules And Rationale

Evidence tier classification should be deterministic in v0 and rubric-backed in
v1.

`explicit` applies when one of these is true:

- the source unit mentions a manufacturer part number that matches exactly one
  catalog row in the active namespace
- the source unit mentions a catalog table/plate identifier
- the source unit mentions an exact catalog assembly title in the same namespace
- the source unit contains an exact part description and the namespace/table
  context resolves it to one candidate

`strong_inferred` applies when all of these are true:

- the source unit names a component, operation, or assembly term
- the term normalizes through an approved terminology mapping or exact title
  overlap
- the active namespace resolves to one catalog table or assembly family
- no fitment, side, color, oversize, or availability note forces review

Fragile fields such as notes cannot be primary evidence for upgrading a link to
`strong_inferred`. They may only add warnings, fitment notes, or downgrade a
candidate to `uncertain` or `needs_review`.

`weak_inferred` applies when the part is plausible from same-table or
same-assembly context, but the procedure does not name it and deterministic
evidence does not isolate it as required. Common examples are fasteners, washers,
clips, seals, and gaskets adjacent to a named component.

`uncertain` applies when multiple namespaces, variants, sides, catalog tables, or
availability notes could change the answer.

Rationale should be stored as structured facts, not freeform prose. Display text
can be generated from those facts, but the stored record should remain
reproducible when sources are reprocessed.

## Link Policy

The linker emits candidates into three buckets:

```text
include_by_default
  explicit links
  strong inferred links with clear catalog/procedure support

show_as_suggested
  weak inferred links
  consumables or adjacent parts likely needed but not named

needs_review
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

## Terminology Integration

The linker may reuse terminology layers from `vehicle_repair_v1`, but it consumes
more than query-expansion aliases. Link-time terminology records should expose:

- `alias`
- `canonical_term`
- `applicability_layers`
- `term_kind`, such as component, assembly, operation, fastener, consumable, or
  tool
- optional `catalog_match_hints`, such as table-title term, part-description
  term, or note term

Aliases without a `term_kind` can help retrieve evidence, but they should not by
themselves create a `strong_inferred` link. Strong inference requires scoped
component or assembly terminology plus catalog namespace support.

Title matching should normalize language, case, punctuation, and paired
translations before comparison. For bilingual catalogs, the linker should compare
English and source-language title spans separately rather than treating the
combined bilingual heading as one opaque string.

## Architecture And Data Flow

The first implementation should extend the existing workshop/manual dogfood
foundation rather than creating a separate automotive subsystem.

```text
raw sources
  -> source units/pages
  -> catalog structure extraction
  -> namespaced parts catalog artifact
  -> source-unit or procedure-structure linking
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
  explicit:
    - item_id: parts-list-item-1
      namespace:
        domain: vehicle_repair
        manufacturer: Ferrari
        product_family: 360
        model: 360 Modena
        variant: null
        source_id: source-ferrari-360-spare-parts-2000
      part_number: "183100"
      catalog_label: Ferrari S.p.A.
      description: Brakes hose
      quantity:
        value: 4
        basis: catalog_row
      catalog_refs:
        - table_id: tavola_33
          table_title: Brake system
          callout_ref: "12"
          source_unit_id: catalog-page-or-table-unit
      relationship_basis: explicit
      confidence_factors:
        supports:
          - exact_part_number_match
          - namespace_scope_matches
          - reverse_index_agrees
        weakens: []
      rationale_facts:
        - source: procedure
          fact: source unit explicitly mentions part number 183100
      fitment_notes: []
      warnings: []
      provenance:
        - source_id: source-ferrari-360-wsm
          unit_id: procedure-source-unit
        - source_id: source-ferrari-360-spare-parts-2000
          unit_id: catalog-page-or-table-unit
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
catalog labels, aliases, fitment notes, and provenance. It should not influence
catalog extraction or procedure linking.

Each item in `inferred` and `needs_review` uses the same record shape as
`explicit`. The `relationship_basis`, `confidence_factors`, `rationale_facts`,
`fitment_notes`, and `warnings` fields explain why the item is not explicit or
why it requires review.

Field definitions:

| Field | Definition |
| --- | --- |
| `item_id` | Output-local stable identifier for the generated parts-list item. It is stable within one output payload only unless the item is promoted to a durable artifact or link id. |
| `catalog_label` | Manufacturer or catalog publisher label as printed or normalized from source metadata. It may differ from `namespace.manufacturer`; omit it when it adds no information. |
| `quantity.basis` | One of `catalog_row`, `procedure_text`, `kit_contents`, `inferred`, or `unknown`. Use `unknown` when quantity cannot be trusted for ordering. |

## Error Handling

Uncertainty is part of the output model.

Catalog parse uncertainty:

- Keep the raw row citation.
- Mark extraction confidence low through visible confidence factors.
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
  `needs_review` unless the procedure or user scope resolves the variant.

## Incremental Re-Extraction

V1 should favor re-derivation over partial mutation. Every extraction run should
record:

- extractor/profile version
- source id and content fingerprint
- source-unit fingerprints
- terminology/profile versions used for linking
- output artifact ids and link ids

When a catalog edition, source fingerprint, parser profile, or terminology layer
changes, affected catalog artifacts and procedure links should be marked stale
and re-derived. Existing reviewed decisions should not be overwritten silently;
they should be carried forward only when their referenced scoped rows still
exist and their source fingerprints match, otherwise they should return to
review.

Equivalence observations follow the same lifecycle: if any referenced scoped row
is removed, renamed, or re-derived with a changed source fingerprint, the
equivalence observation becomes `stale` and must be re-derived or reviewed.

## Verification

Verification should combine focused parser tests and eval cases:

Catalog extraction gates:

- Hand-code ground truth for at least 20 Ferrari catalog tables across
  mechanical, body, electrical, and variant-heavy examples.
- Achieve at least 95% row precision and 90% row recall on the ground-truth
  tables before using extracted rows for linking.
- Achieve at least 98% precision for part numbers on those tables.
- Preserve source notes for at least 90% of rows that have notes in ground truth.
- Verify that extracted table/ref positions agree with the reverse numerical
  index for at least 100 sampled index entries when an index exists.

Linking gates:

- Build at least 10 explicit-link eval cases before enabling procedure parts
  output.
- Require 100% precision for `explicit` part-number links in the eval set.
- Do not enable `strong_inferred` default inclusion until at least 20 inferred
  link eval cases distinguish `strong_inferred`, `weak_inferred`, and
  `uncertain`.
- Ensure inferred parts are never labeled explicit.
- Ensure shopping fields and vendor behavior are absent from this milestone.

Eval cases should live under `eval/` with the workshop eval assets unless the
implementation plan chooses a more specific fixture path. Each case should name
the source unit, expected catalog refs, expected relationship basis, and required
provenance refs. Human-reviewed corrections should be eligible for promotion
into this eval set.

Manufacturer-neutral checks:

- Keep Ferrari-specific parsing in a source profile, not in the core artifact
  model.
- Validate generic row-shape consistency, required-field completeness,
  part-number format, quantity parse status, duplicate callout detection, and
  callout-to-row coverage where the source exposes callouts.

The first measurable slice is:

```text
Can Prescient parse a representative Ferrari parts graph with measured accuracy
and produce reviewable explicit links from workshop source units to catalog rows?
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
