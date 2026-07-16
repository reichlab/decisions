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
2. **Make completing the minimum easy.** Guide hub admins through only what cannot be inferred (descriptions of custom columns, a free-text dataset description, license, provenance), with as little friction as possible.
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

- **Data integrity and reuse by a wide audience.** A hub is configured by a small number of admins but is designed to serve many more stakeholders: modeling teams, public health decision-makers, researchers and downstream analysts, most of whom were never involved in its setup (and who may arrive long after it). Every one of them needs to understand what each column means, what the observed quantity is, and how it was generated, from the metadata alone.
- **Honest FAIR claims.** Today a hub's FAIR story is essentially: data are openly available from a GitHub repo (and for some hubs an S3 mirror) in a standardized hubverse format, with config files describing the modeling tasks. That is a real baseline for Findable and Accessible. Beyond that, there is no assembled, machine-readable dataset-level metadata record, no per-column data dictionary surfaced to users, and no persistent identifiers.
- **Closing concrete usability gaps.** Harry's analysis names three: the language describing **models**, the language describing **targets** (and, more broadly, the columns), and the **permanence of results** (stable identifiers for a hub's data).
- **Actively requested by hub admins and the community.** In [hubverse discussion #40](https://github.com/orgs/hubverse-org/discussions/40) several maintainers (Harry, Anna, Sam Abbott, Kath Sherratt, CyGei and others) ask for exactly this: machine-readable (JSON-LD) hub metadata, structured metadata and best-practice reporting standards, standardized target naming and ontologies so targets are comparable across hubs, and per-model **model cards** (HuggingFace-style) describing how each model works.
- **Metadata at the point of access.** Descriptions, units and provenance are most valuable surfaced where people meet the data, not buried in a file. Like a web service where you review metadata before pulling data, anyone coming to a hub should be able to review the data dictionary and dataset description first. The roadmap surfaces it as a generated page in the repo, through the access tools, and on the dashboards.

### What are we _not_ trying to do?

- Not leading with ontologies or formal standards. The MVP is the pragmatic foundation; vocabularies and ontology mapping come later (see roadmap).
- Not requiring hubs to re-enter information that already exists in their config files or that the hubverse already defines for standard columns.
- **(Hard anti-aim) No schema change that requires downstream software adaptation.** The hub config [`schemas`](https://github.com/hubverse-org/schemas) are depended on by the whole software stack, so we will not change the meaning or structure of existing config properties (for example, what a task ID property holds). Purely *additive*, backward-compatible config metadata that downstream software can safely ignore is acceptable, and in some cases desirable (see custom task ID metadata below).
- Not building a hosted metadata portal or catalogue in Phase 1.

### How do we judge success?

- Before or when accessing any hub data product (through `hubData`/the Python client or a viewer), a user can review a clear data dictionary for every column, plus a dataset-level description of what the data is, who maintains it, how to cite it, and its license.
- The large majority of that metadata is auto-generated from config and hubverse conventions; admins supply only descriptions of custom columns and a few dataset-level fields, with low friction.
- Metadata is emitted as JSON-LD using the schema.org/Dataset vocabulary (the keys mapped to schema.org terms), plus an auto-generated human-readable form.
- Early field naming and structure align to a recognized standard direction, so adding value-level vocabulary and ontology links and identifiers later is an extension, not a rewrite.

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
| **Dataset-level** identity: name, maintainer, contact, citation, repository, file formats | Yes | `admin.json` |
| **Dataset-level** free-text description | No, must be supplied | admin scaffolding |
| **Data license** | Partly derivable: there is no hub-level license, but per-model output licenses exist in model-metadata. A combined `model-output` license can be suggested from these (e.g. the most restrictive, flagging incompatible combinations) and confirmed by the admin. `target-data` licensing is separate and comes from the upstream observed-data source | model-metadata (per-model); admin confirms |
| **Temporal coverage** | Derivable: extent from the date columns, resolution from `time_unit` | derived |
| **Spatial coverage** | Partly: the set of location codes present is derivable, but a geographic extent (named places, bounding box) needs the codes resolved (FIPS/GeoNames) | derived (codes); Phase 2 (geography) |
| **Model** descriptions | Yes, in mature hubs | per-model `model-metadata` yaml (hub-authored schema, varies between hubs) |

- **The standard-column definitions are the key extraction win.** The hubverse defines standard task ID columns and fixes the representation-column semantics, so a built-in dictionary can auto-describe most columns of most data products. Custom task ID columns are the gap: no schema-defined type or description, so they need scaffolding for both.
- **Model metadata is hub-authored and varies between hubs.** We provide a template `model-metadata-schema.json`, but each hub sets its own (FluSight adapted the template; see [FluSight model-metadata](https://github.com/cdcepi/FluSight-forecast-hub/tree/main/model-metadata)). The metadata layer must read each hub's own schema; the template is the lever for cross-hub consistency, not a guarantee. FluSight's schema already captures contributors (with optional ORCID), a `license` enum, data inputs, short and long methods, and ensemble flags, and the real files are well populated.
- **Some "easy wins" are already in production.** FluSight's optional ORCID field (with a regex pattern) and its `license` enum (a controlled vocabulary, no ontology machinery) show these are realistic MVP-level adoptions, not Phase 2 work. ORCIDs are often left blank despite the field existing.
- **`dataspice` is a proven model** for the extract-then-scaffold workflow and for the staged progression from a simple standard to a richer one. Anna's teaching materials map this path (`rrresearch-acce-rrse`, chapters `04a`–`04c`): *any metadata is better than none; start small, cover the bare minimum, then build up.*
- **Harry's sketch maps the destination**: a FAIR metadata standard in JSON-schema (foundation exists) and JSON-LD, backed by vocabularies (schema.org and Dublin Core as the spine; GeoNames, EDAM, OWL-time, MIDAS ontology, NCBI taxon, SNOMED/ICD10, Software License Ontology, PROV-O for specific fields). [`LinkML`](https://linkml.io/) is noted as a possible authoring layer. Source: [Opportunities/challenges for Hubverse metadata](https://docs.google.com/document/d/1ewJ8JFBtuUb62Qchd_tO0rWQSYUseP7U4j80O9sKUjA/edit).

### What do we need to answer?

- **How to carry the minimum variable spec in schema.org, and the dataset-level field set.** We propose a minimum per variable of `name`, `description`, `dataType`, `units` (numeric) and a description per value (categorical), mirroring EML (see Phase 1). schema.org [`PropertyValue`](https://schema.org/PropertyValue) covers name/description/`unitText` but has no native slot for `dataType` or per-category definitions, so the standard will need to profile or extend it. The dataset-level field set still needs pinning down. See the [schema.org datasets guide](https://schema.org/docs/data-and-datasets.html).
- **How complete is the built-in standard-column dictionary**, and how do we detect when a hub's column is "standard" vs. custom (exact name match, or something more flexible)?
- **Which of Harry's field-to-standard mappings are cheap enough for the MVP?** Some need no ontology machinery, just a convention on an existing field (ORCID, DOI, ISO 8601 dates), and could land in Phase 1. Heavier mappings (MIDAS ontology for targets, EDAM for formats, GeoNames for locations) are Phase 2+. Decided against Harry's analysis, not guessed.
- **Persistent identifiers for results** (Harry's third gap, Nick's key question): what mechanism (e.g. DataCite/Zenodo DOIs) and at what granularity (hub, data product, version)? Likely Phase 2, but the MVP structure must not preclude it.
- **How to handle per-hub variation in the model-metadata schema**, and whether to encourage a recommended core set the metadata layer can rely on across hubs.
- **Could custom task ID metadata be captured in the config up front?** Rather than only describing custom task IDs at extraction time, admins could supply their `description`, `type` and useful constraints (e.g. `min`/`max`) in the config, which would improve both metadata generation and validation of those columns. To stay within the hard anti-aim, this must go in a separate, additive property (as `target-data.json` does), not inside the existing task ID value properties that downstream software reads.

## 👍 Ready to make it

### Proposed solution

A three-phase roadmap. **Phase 1 is the pragmatic functional foundation**: auto-generate a data dictionary and dataset-level record from what already exists, and make completing the rest easy. Standards, vocabularies, identifiers and ontologies come in Phases 2 and 3, except for any low-cost identifier conventions Harry's analysis shows are easy.

Three design principles run throughout:

> **Any metadata is better than none. Start small, cover the bare minimum, then build up.** Every phase leaves a hub more discoverable and better described than before.

> **Decide early for later.** Phase 1 field naming and structure align to a recognized standard direction (schema.org + Dublin Core) so that Phase 2's value-level vocabulary and ontology links and identifiers are extensions, not rewrites.

> **We build the tooling, not a mandate.** For every step we provide automation, shared crosswalks, or suggestions to confirm, so the effort on hub admins and modelers stays minimal. Where manual input is unavoidable, we say whose responsibility it is (admin, modeler, or us).

This is a **cross-cutting descriptive-metadata layer** over the existing config files and data products, not a single new file.

---

### Phase 1 — Minimum viable metadata: a data dictionary, extracted and completed (MVP)

*Goal: every hub data product ships with a clear per-column data dictionary and a dataset-level description, mostly auto-generated, with the rest easy to complete.*

- **Delivered in `hubAdmin`** as functions that extract the metadata already present across a hub's config (from a `hub_path`) and consolidate it into a single metadata record, the inverse of authoring config (sketch: `extract_metadata()` / `build_metadata()`).
- **Generate the per-column data dictionary** (the schema.org `variableMeasured`) for each data product (`model-output`, `time-series`, `oracle-output`):
  - **minimum per variable**: `name`, `description`, `dataType`, `units` (numeric variables), and a `description` for each permissible value of categorical (string/factor) variables. This mirrors EML's minimum attribute requirements (`attributeDefinition` + `measurementScale`, which forces unit + numberType for numeric and a definition per code for categorical), which also sets up the Phase 3 EML crosswalk.
  - pull every column name from `tasks.json` and `target-data.json` (auto), with authoritative types for standard task IDs, the representation columns, and declared non-task-id target columns
  - auto-describe standard task ID columns and the representation columns (`output_type`, `output_type_id`, `value`, `observation`, `oracle_value`) from a built-in hubverse dictionary
  - pull `target` column descriptions from `target_metadata`
  - scaffold custom task ID columns (confirm type, supply description) and hub-specific value sets
- **Assemble the dataset-level record** from `admin.json` (title, maintainer, contact, citation, repository), derive temporal coverage and the set of location codes from the data (geographic extent deferred to the Phase 2 location vocabulary), and suggest the `model-output` license from the per-model licenses for admin confirmation (see the data license row above). Prompt only for genuine gaps: free-text description, `target-data` source license, provenance.
- **Make completing the minimum easy**: a `dataspice`-style guided workflow in `hubAdmin` that pre-fills everything extractable and asks only for what is missing.
- **Emit both forms**: a machine-readable record as JSON-LD whose `@context` links the keys to schema.org/Dataset terms (field values stay literals at this stage), and a human-readable data dictionary / landing page generated from it and served from the hub repo (as `dataspice` builds a site from its JSON-LD). Serving the schema.org JSON-LD also makes the data products discoverable through general tools like Google Dataset Search. The same generated output can render each model's metadata as a human-readable **model card** (the community-requested, HuggingFace-style artifact).
- **Fold in easy identifier wins** that need no ontology machinery and are already precedented (ORCID, `license` enum; candidates also DOI for citations, ISO 8601 dates). Decided against Harry's mapping.
- **Placemark the future**: agree the standard direction (schema.org + Dublin Core) and align field naming and structure now, so Phase 2's value-level vocabulary and ontology links and identifiers extend rather than rewrite.

**Deliverable:** a workflow that produces, for any hub, a per-column data dictionary and dataset-level record for each data product, mostly auto-generated, in human-readable and schema.org JSON-LD form.

---

### Phase 2 — Mature and enrich: link values, vocabularies and identifiers

*Goal: deepen interoperability beyond the Phase 1 schema.org structure by linking field values to external entities, vocabularies and ontologies, and adding persistent identifiers, with tooling that does the heavy lifting (shared crosswalks, suggested mappings, automated minting) so the burden on hubs stays low.*

- **Link field values to entities and ontologies.** Take the Phase 1 JSON-LD further by resolving literal values to external identifiers and concepts (via `@id` / typed values), so a location code, a target, a contributor or a license becomes a linked entity rather than a plain string. This is the deeper FAIR-Interoperable step and makes data comparable across hubs.
- **We provide the crosswalks, hubs don't.** Ship and maintain the mappings for standard columns and values (FIPS↔GeoNames for `location`, EDAM for file formats, OWL-time for timezones, a target→concept map), applied automatically wherever a hub uses standard codes, so admins only confirm or map genuinely hub-specific values. (Deep domain ontologies such as MIDAS are Phase 3.)
- **Persistent identifiers for results** (the permanence gap): provide a workflow that mints and registers DOIs (e.g. Zenodo/DataCite, triggered from the hub) so the hub owner opts in rather than minting by hand. Mechanism and granularity still open.
- **Provenance (lineage)** with [PROV-O](https://www.w3.org/TR/prov-o/). For **target data**, the target-data maintainer declares the upstream source once and tooling records it and the `as_of` versions. For **model output**, lineage to a model's `data_inputs` (the upstream data the modeler used) is the modeler's responsibility and lives in their model-metadata. We can't derive it for them, but we can help them along, the modeler-facing analog of the admin crosswalks: documentation plus an interactive lookup/registry of common data sources and their identifiers, so modelers select structured references rather than writing free text.
- **FAIR self-assessment**: a report of what each hub has vs. the standard, with a maturity score that shows admins exactly which gaps to fill and nudges them from bare minimum toward rich.
- **Surface metadata across access surfaces** (beyond the human-readable page already served from the repo in Phase 1): retrievable through the access tools (`hubData`, the Python client) so anyone connecting to a hub on S3 or GitHub can review the data dictionary and dataset description before collecting data; a dedicated data-dictionary page on the dashboards; and, optionally, inline in the viewers (predtimechart, predevals, hubVis) alongside validation status.

**Deliverable:** a vocabulary- and ontology-linked metadata standard (values resolved to external entities), persistent identifiers, a per-hub FAIR maturity report, and metadata surfaced across access surfaces (access tools, dashboard data-dictionary page, and viewers).

---

### Phase 3 — Long-distance: domain ontologies and ecosystem

*Goal: formal interoperability within and across hubs, and integration with the wider data ecosystem.*

- **Domain ontology mapping**: MIDAS ontology for targets, NCBI taxon for pathogens/vectors, SNOMED/ICD10 for outcomes, Software License Ontology for licenses, NIH RePORTER for funding.
- **Crosswalk to richer metadata standards** such as [EML](https://eml.ecoinformatics.org/), whose attribute model the minimum variable spec already mirrors (the `dataspice` → EML progression).
- **Single-source schema authoring** (e.g. [`LinkML`](https://linkml.io/)) projecting to JSON-schema + JSON-LD + docs from one model.
- **Link model metadata to model abstracts** and tackle the hardest case, ontologizing methods (where MIDAS work is ongoing).
- **Cross-hub catalogue, search and comparison**; certifiable FAIR scoring; integration with external repositories/portals with rich metadata presentation and citation.

**Deliverable:** domain-ontology crosswalks, a single-source schema, and cross-hub discovery and comparison.

---

### Visualize the solution: literals become linked entities

A slice of a data product's metadata record. **Phase 1** emits schema.org-structured JSON-LD with literal values (the keys are linked to schema.org terms; the values are plain strings). **Phase 2** resolves those literals to linked entities (a license URI, an ORCID, ontology terms). Illustrative and simplified; URIs are examples.

**Phase 1** (schema.org keys linked, values are literals):

```json
{
  "@context": "https://schema.org",
  "@type": "Dataset",
  "name": "FluSight time-series target data",
  "license": "CC-BY-4.0",
  "creator": { "@type": "Person", "name": "Jane Modeler" },
  "variableMeasured": [
    { "@type": "PropertyValue", "name": "location",
      "description": "US jurisdiction (FIPS code)" },
    { "@type": "PropertyValue", "name": "target",
      "description": "Weekly incident influenza hospitalizations",
      "unitText": "count" }
  ]
}
```

**Phase 2** links the values (changes shown as a diff, red removed / green added):

```diff
   "@type": "Dataset",
   "name": "FluSight time-series target data",
-  "license": "CC-BY-4.0",
+  "license": { "@id": "https://creativecommons.org/licenses/by/4.0/" },
-  "creator": { "@type": "Person", "name": "Jane Modeler" },
+  "creator": { "@type": "Person", "name": "Jane Modeler",
+               "@id": "https://orcid.org/0000-0002-1825-0097" },
   "variableMeasured": [
     { "@type": "PropertyValue", "name": "location",
-      "description": "US jurisdiction (FIPS code)" },
+      "description": "US jurisdiction (FIPS code)",
+      "propertyID": "https://www.geonames.org/ (codes resolve to Place entities)" },
     { "@type": "PropertyValue", "name": "target",
       "description": "Weekly incident influenza hospitalizations",
-      "unitText": "count" }
+      "unitText": "count",
+      "propertyID": "https://midasnetwork.us/term/{...} (MIDAS ontology term)" }
   ]
}
```

---

### Relationship to existing work

This project adds a descriptive-metadata layer on top of the existing, validated config schemas and data-product conventions, which already define the columns, types and standard-column semantics. It operationalizes Harry's [FAIR metadata sketch](https://docs.google.com/document/d/1ewJ8JFBtuUb62Qchd_tO0rWQSYUseP7U4j80O9sKUjA/edit) on a `dataspice`-style staged path, and changes none of those schemas (see the hard anti-aim).

Reference material: [`dataspice`](https://github.com/ropensci/dataspice); Anna's metadata teaching materials (`rrresearch-acce-rrse`, chapters `04a`–`04c`); the hubverse [schemas](https://github.com/hubverse-org/schemas), [model-output](https://docs.hubverse.io/en/latest/user-guide/model-output.html), [target-data](https://docs.hubverse.io/en/latest/user-guide/target-data.html) and [hub-config](https://docs.hubverse.io/en/latest/user-guide/hub-config.html) docs; [schema.org/Dataset](https://schema.org/Dataset) and [datasets guide](https://schema.org/docs/data-and-datasets.html); [Dublin Core](https://www.dublincore.org/); [EML](https://eml.ecoinformatics.org/); [LinkML](https://linkml.io/); [Hex dbt integration](https://learn.hex.tech/docs/connect-to-data/data-connections/dbt-integration); [hubverse discussion #40](https://github.com/orgs/hubverse-org/discussions/40) (community requests, incl. model cards).

### Scale and scope

- **Required team size:** TBD.
- **Required delivery date:** none; phased and incremental by design. Phase 1 is the priority and is independently shippable.
- **Sequencing:** Phases are ordered. Phase 1 establishes the data dictionary, extraction foundation and standard-aligned naming; Phases 2 and 3 enrich it. Low-cost identifier conventions may be pulled into Phase 1 where Harry's analysis shows they are cheap.
