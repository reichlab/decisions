# Project Poster: Metadata in the Hubverse

- Date: 2026-06-26

- Owner: Harry Hochheiser and Anna Krystalli (drivers)

- Team: TBD

- Stakeholder: Nicholas Reich

- Status: draft

## ❓ Problem space

### What are we doing?

Building functionality to **capture, store, and progressively enrich descriptive metadata** for the data products a hub publishes, so that anyone accessing hub data understands every column they encounter, and a hub's claim to adhere to FAIR principles is backed by real, machine-readable metadata rather than convention.

The approach, drawn from [`dataspice`](https://github.com/ropensci/dataspice), is:

1. **Extract what already exists.** Column names and types, and the definitions of standard columns, are already derivable from the hub config and hubverse conventions. Auto-capture as much as possible.
2. **Make completing the minimum easy.** Guide hub admins through only what cannot be inferred (descriptions of custom columns, dataset-level descriptions, provenance), with as little friction as possible.
3. **Start minimal, build up in deliberate phases**, but make early decisions that placemark the path to richer standards, vocabularies and identifiers.

### What a data user actually encounters

A user accessing hub data downloads tabular data products whose columns are **mostly task ID columns** plus a small number of fixed columns:

| Data product | Columns |
|--------------|---------|
| `model-output` | several **task ID** columns (e.g. `target`, `location`, `horizon`, `reference_date`, `scenario_id`, `age_group`) + `output_type` + `output_type_id` + `value` (+ `model_id` when collected) |
| `time-series` target data | task ID columns (including a date) + `observation` (+ optional `as_of`, + non-task-id columns like `location_name`) |
| `oracle-output` target data | a subset of task ID columns (+ `output_type`/`output_type_id` for pmf/cdf) + `oracle_value` (+ optional `as_of`) |

The metadata that makes these reusable is a **per-column data dictionary**: what each column is, its type, units, and permissible values. In [schema.org/Dataset](https://schema.org/Dataset) terms this is the `variableMeasured` array (each column a `PropertyValue`), with each data product a `Dataset` and the hub a `DataCatalog`. The `target` column is one (important) column, described today by `target_metadata`; it is not the whole dataset.

### Why are we doing this?

- **Data integrity and reuse.** Hubs outlive the people who set them up. Someone downloading the data needs to understand what every column means, what the observed quantity is, and how it was generated, from the metadata alone.
- **Honest FAIR claims.** Today a hub's FAIR story is essentially: data are openly available from a GitHub repo (and for some hubs an S3 mirror) in a standardized hubverse format, with config files describing the modeling tasks. That is a real baseline for Findable and Accessible. Beyond that, there is no assembled, machine-readable dataset-level metadata record, no per-column data dictionary surfaced to users, and no persistent identifiers.
- **Closing concrete usability gaps.** Harry's analysis names three: the language describing **models**, the language describing **targets** (and, more broadly, the columns), and the **permanence of results** (stable identifiers for a hub's data).

### What are we _not_ trying to do?

- Not leading with ontologies or formal standards. The MVP is the pragmatic foundation; vocabularies and ontology mapping come later (see roadmap).
- Not requiring hubs to re-enter information that already exists in their config files or that the hubverse already defines for standard columns.
- Not replacing the `target-data.json` validation schema. That is structural metadata for data integrity, and it is a useful input here, not the focus.
- Not building a hosted metadata portal or catalogue in Phase 1.

### How do we judge success?

- A user downloading any hub data product gets a clear data dictionary for every column, plus a dataset-level description of what the data is, who maintains it, how to cite it, and its license.
- The large majority of that metadata is auto-generated from config and hubverse conventions; admins supply only descriptions of custom columns and a few dataset-level fields, with low friction.
- Metadata is emitted in both a machine-readable form (aligned to schema.org/Dataset) and an auto-generated human-readable form.
- Early field naming and structure align to a recognized standard direction, so moving to JSON-LD, vocabularies and identifiers later is an extension, not a rewrite.

### What are possible solutions?

The proposed approach is the three-phase roadmap in [Ready to make it](#-ready-to-make-it). Each phase is independently useful and releasable.

## ✅ Validation

### What do we already know?

Much of the metadata is already present or derivable; the work is to assemble, complete and surface it. Mapped to what a user encounters:

| Metadata a user needs | Available now? | Source |
|-----------------------|----------------|--------|
| Column **names** for all data products | Yes, every column is listed | `tasks.json` task IDs; `target-data.json` |
| Column **data types** | Authoritative for standard task IDs (those with hubverse schema entries) and the representation columns; declared for non-task-id target columns; only **inferred** (not authoritative, can be ambiguous) for custom task IDs | tasks-schema for standard task IDs; `target-data.json` `non_task_id_schema` for declared columns; `hubData::create_hub_schema()` infers custom types from values |
| Descriptions of **standard task ID columns** (e.g. `reference_date`, `horizon`, `location`, `target_end_date`) | Yes, the hubverse maintains standard names and definitions | hubDocs task ID variable usage guidance, applicable wherever a hub uses standard names |
| Descriptions of the **representation columns** (`output_type`, `output_type_id`, `value`, `observation`, `oracle_value`) | Yes, fixed hubverse semantics | hubDocs model-output and target-data pages |
| **Target** column values | Yes, partially | `tasks.json` `target_metadata` (`target_id`, `target_name`, `target_units`, `description`, `target_type`) |
| Descriptions of **custom / hub-specific columns** and their value sets | No, must be supplied | admin scaffolding (Phase 1) |
| **Dataset-level** description, maintainer, contact, license, citation, repository, coverage | Partially | `admin.json`; per-model `license` etc. in model-metadata |
| **Model** descriptions | Yes, in mature hubs | per-model `model-metadata` yaml (hub-authored schema, varies between hubs) |

- **The standard-column definitions are the key extraction win.** Because the hubverse defines standard task ID columns and fixes the semantics of the representation columns, a built-in dictionary can auto-describe (name, type and definition) most columns of most data products. Custom task ID columns are the gap: they have no schema-defined type or description, so they need scaffolding for both a confirmed type and a description.
- **Model metadata is hub-authored and varies between hubs.** We provide a template `model-metadata-schema.json`, but each hub sets its own (FluSight adapted the template; see [FluSight model-metadata](https://github.com/cdcepi/FluSight-forecast-hub/tree/main/model-metadata)). The metadata layer must read each hub's own schema; the template is the lever for cross-hub consistency, not a guarantee. FluSight's schema already captures contributors (with optional ORCID), a `license` enum, data inputs, short and long methods, and ensemble flags, and the real files are well populated.
- **Some "easy wins" are already in production.** FluSight's optional ORCID field (with a regex pattern) and its `license` enum (a controlled vocabulary, no ontology machinery) show these are realistic MVP-level adoptions, not Phase 2 work. ORCIDs are often left blank despite the field existing.
- **`dataspice` is a proven model** for the extract-then-scaffold workflow and for the staged progression from a simple standard to a richer one. Anna's teaching materials map this path (`rrresearch-acce-rrse`, chapters `04a`–`04c`): *any metadata is better than none; start small, cover the bare minimum, then build up.*
- **`hubAdmin` already has a `create_*()` family** of functions for programmatic config creation, the natural place to hang scaffolding helpers.
- **Harry's sketch maps the destination**: a FAIR metadata standard in JSON-schema (foundation exists) and JSON-LD, backed by vocabularies (schema.org and Dublin Core as the spine; GeoNames, EDAM, OWL-time, MIDAS ontology, NCBI taxon, SNOMED/ICD10, Software License Ontology, PROV-O for specific fields). `linkml.org` is noted as a possible authoring layer. Source: [Opportunities/challenges for Hubverse metadata](https://docs.google.com/document/d/1ewJ8JFBtuUb62Qchd_tO0rWQSYUseP7U4j80O9sKUjA/edit).

### What do we need to answer?

- **The exact MVP data dictionary fields per column** (name, description, type, units, permissible values?) and the dataset-level field set, and how each maps to schema.org `variableMeasured` / `Dataset`.
- **How complete is the built-in standard-column dictionary**, and how do we detect when a hub's column is "standard" vs. custom (exact name match, or something more flexible)?
- **Which of Harry's field-to-standard mappings are cheap enough for the MVP?** Some need no ontology machinery, just a convention on an existing field (ORCID, DOI, ISO 8601 dates), and could land in Phase 1. Heavier mappings (MIDAS ontology for targets, EDAM for formats, GeoNames for locations) are Phase 2+. Decided against Harry's analysis, not guessed.
- **Where does the assembled metadata live?** A generated metadata artifact per data product, completion of existing config in place, or both. Note `additional_metadata` escape hatches already exist in `admin.json`, `tasks.round`, `target_metadata`, and `target-data.json`.
- **Persistent identifiers for results** (Harry's third gap, Nick's key question): what mechanism (e.g. DataCite/Zenodo DOIs) and at what granularity (hub, data product, version)? Likely Phase 2, but the MVP structure must not preclude it.
- **How to handle per-hub variation in the model-metadata schema**, and whether to encourage a recommended core set the metadata layer can rely on across hubs.

## 👍 Ready to make it

### Proposed solution

A three-phase roadmap. **Phase 1 is the pragmatic functional foundation**: auto-generate a data dictionary and dataset-level record from what already exists, and make completing the rest easy. Standards, vocabularies, identifiers and ontologies come in Phases 2 and 3, except for any low-cost identifier conventions Harry's analysis shows are easy.

Two design principles run throughout:

> **Any metadata is better than none. Start small, cover the bare minimum, then build up.** Every phase leaves a hub more discoverable and better described than before.

> **Decide early for later.** Phase 1 field naming and structure align to a recognized standard direction (schema.org + Dublin Core) so that Phase 2's JSON-LD, vocabularies and identifiers are extensions, not rewrites.

This is a **cross-cutting descriptive-metadata layer** over the existing config files and data products, not a single new file.

---

### Phase 1 — Minimum viable metadata: a data dictionary, extracted and completed (MVP)

*Goal: every hub data product ships with a clear per-column data dictionary and a dataset-level description, mostly auto-generated, with the rest easy to complete.*

- **Generate the per-column data dictionary** (the schema.org `variableMeasured`) for each data product (`model-output`, `time-series`, `oracle-output`):
  - pull every column name from `tasks.json` and `target-data.json` (auto), with authoritative types for standard task IDs, the representation columns, and declared non-task-id target columns
  - auto-describe standard task ID columns and the representation columns (`output_type`, `output_type_id`, `value`, `observation`, `oracle_value`) from a built-in hubverse dictionary
  - pull `target` column descriptions from `target_metadata`
  - scaffold custom task ID columns (confirm type, supply description) and hub-specific value sets
- **Assemble the dataset-level record** (title, description, maintainer, contact, license, citation, repository, coverage) from `admin.json` and model-metadata, prompting only for genuine gaps.
- **Make completing the minimum easy**: a `dataspice`-style guided workflow, built on `hubAdmin`'s `create_*()` functions, that pre-fills everything extractable and asks only for what is missing.
- **Emit both forms**: a human-readable data dictionary / dataset landing page, and a machine-readable record aligned to schema.org/Dataset.
- **Fold in easy identifier wins** that need no ontology machinery and are already precedented (ORCID, `license` enum; candidates also DOI for citations, ISO 8601 dates). Decided against Harry's mapping.
- **Placemark the future**: agree the standard direction (schema.org + Dublin Core), align field naming and structure now, and use the existing `additional_metadata` escape hatches.

**Deliverable:** a workflow that produces, for any hub, a per-column data dictionary and dataset-level record for each data product, mostly auto-generated, in human- and machine-readable form.

---

### Phase 2 — Mature and enrich: standards, vocabularies and identifiers

*Goal: express the metadata in a recognized FAIR standard, with controlled vocabularies for column values, persistent identifiers, and machine-readable discoverability.*

- **Formalize the standard** in schema.org + Dublin Core and **emit JSON-LD** from config, making hub data products discoverable through general tools (e.g. Google Dataset Search) and comparable across hubs.
- **Controlled vocabularies and identifiers for column values and higher-cost fields** from Harry's mapping: GeoNames (and/or FIPS) for `location`, MIDAS ontology for `target`, EDAM for file formats, OWL-time for timezones, ORCID/DOI wherever not already adopted in Phase 1.
- **Persistent identifiers for results** (the permanence gap): assign and resolve DOIs (or equivalent) for hub data products at an agreed granularity.
- **Provenance**: link target data to upstream sources and model output to its data inputs (PROV-O).
- **FAIR self-assessment**: report what each hub has vs. the standard as a maturity score that nudges hubs from bare minimum toward rich.

**Deliverable:** a formalized, vocabulary-backed standard with JSON-LD emission, persistent identifiers, and a per-hub FAIR maturity report.

---

### Phase 3 — Long-distance: domain ontologies and ecosystem

*Goal: formal interoperability within and across hubs, and integration with the wider data ecosystem.*

- **Domain ontology mapping**: MIDAS ontology for targets, NCBI taxon for pathogens/vectors, SNOMED/ICD10 for outcomes, Software License Ontology for licenses, NIH RePORTER for funding.
- **Single-source schema authoring** (e.g. `linkml`) projecting to JSON-schema + JSON-LD + docs from one model.
- **Link model metadata to model abstracts** and tackle the hardest case, ontologizing methods (where MIDAS work is ongoing).
- **Cross-hub catalogue, search and comparison**; certifiable FAIR scoring; integration with external repositories/portals with rich metadata presentation and citation.

**Deliverable:** domain-ontology crosswalks, a single-source schema, and cross-hub discovery and comparison.

---

### Relationship to existing work

This project is a descriptive-metadata layer built on top of the existing, validated config schemas and data-product conventions, operationalizing Harry's FAIR sketch on a `dataspice`-style staged path:

- The existing [schemas](https://github.com/hubverse-org/schemas) (`admin-schema.json`, `tasks-schema.json`, model-metadata, `target-data.json`) and the [model-output](https://docs.hubverse.io/en/latest/user-guide/model-output.html) and [target-data](https://docs.hubverse.io/en/latest/user-guide/target-data.html) conventions already define the columns and types and the standard-column semantics. This project assembles, completes and surfaces a descriptive layer (the data dictionary plus dataset-level record) on top of them.
- The approved [`target-data.json` metadata RFC](../../decisions/2025-06-17-RFC-target-data-metadata.md) covers structural/validation metadata for target data and is an input here, not the focus.
- Harry's [Opportunities/challenges for Hubverse metadata](https://docs.google.com/document/d/1ewJ8JFBtuUb62Qchd_tO0rWQSYUseP7U4j80O9sKUjA/edit) defines the destination (standards, vocabularies, identifiers); the roadmap sequences it behind the pragmatic MVP.

Reference material: [`dataspice`](https://github.com/ropensci/dataspice); Anna's metadata teaching materials (`rrresearch-acce-rrse`, chapters `04a`–`04c`); [schema.org/Dataset](https://schema.org/Dataset); [Dublin Core](https://www.dublincore.org/); [hubDocs hub-config guide](https://docs.hubverse.io/en/latest/user-guide/hub-config.html).

### Scale and scope

- **Required team size:** TBD.
- **Required delivery date:** none; phased and incremental by design. Phase 1 is the priority and is independently shippable.
- **Sequencing:** Phases are ordered. Phase 1 establishes the data dictionary, extraction foundation and standard-aligned naming; Phases 2 and 3 enrich it. Low-cost identifier conventions may be pulled into Phase 1 where Harry's analysis shows they are cheap.
