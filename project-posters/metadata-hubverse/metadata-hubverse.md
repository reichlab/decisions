# Project Poster: Metadata in the Hubverse

- Date: 2026-06-26

- Owner: Harry Hochheiser and Anna Krystalli (drivers)

- Team: TBD

- Stakeholder: Nicholas Reich

- Status: draft

## ❓ Problem space

### What are we doing?

Building tooling that captures, stores and progressively enriches descriptive metadata for the data products a hub publishes. The aim is that anyone who accesses hub data understands every column they encounter, and that a hub's claim to follow the FAIR principles rests on machine-readable metadata rather than convention.

The approach follows [`dataspice`](https://github.com/ropensci/dataspice):

1. **Extract what already exists.** Column names and types, and the definitions of standard columns, are derivable from the hub config and hubverse conventions.
2. **Make completing the rest easy.** Admins are asked only for what cannot be inferred: descriptions of custom columns and a few dataset-level fields.
3. **Start minimal and build up in phases.** Phase 1 settles the structure so later phases extend it rather than rewrite it.

### What a data user actually encounters

A user accessing hub data downloads tabular data products whose columns are **mostly task ID columns** plus a small number of fixed columns:

| Data product | Columns |
|--------------|---------|
| `model-output` | several **task ID** columns (e.g. `target`, `location`, `horizon`, `reference_date`, `scenario_id`, `age_group`) + `output_type` + `output_type_id` + `value` (+ `model_id` when collected) |
| `time-series` target data | task ID columns (including a date) + `observation` (+ optional `as_of`, + non-task-id columns like `location_name`) |
| `oracle-output` target data | a subset of task ID columns (+ `output_type`/`output_type_id` for pmf/cdf) + `oracle_value` (+ optional `as_of`) |

The metadata that makes these reusable is a **per-column data dictionary**: what each column is, its type, units and permissible values. In [schema.org/Dataset](https://schema.org/Dataset) terms this is the `variableMeasured` array, with each column a `PropertyValue`, each data product a `Dataset` and the hub a `DataCatalog`. schema.org is the shared base. Where it lacks a field (for example a column's data type) we borrow one from another established standard, which is routine practice. The `target` column is one important column, described today by `target_metadata`; it is not the whole dataset.

### Why are we doing this?

- **Data integrity and reuse by a wide audience.** A hub is configured by a few admins but serves many more people: modeling teams, public health decision-makers, researchers and downstream analysts, most of whom were not involved in its setup and may arrive long after it. Each of them needs to understand, from the metadata alone, what each column means, what the observed quantity is, and how it was generated.
- **Honest FAIR claims.** Today a hub's FAIR story is: data are openly available from a GitHub repo (for some hubs also an S3 mirror) in a standardized hubverse format, with config files describing the modeling tasks. That is a real baseline for Findable and Accessible. Beyond it there is no assembled, machine-readable dataset-level record, no per-column data dictionary surfaced to users, and no persistent identifiers.
- **Closing concrete usability gaps.** Harry's analysis names three: the language describing **models**, the language describing **targets** (and, more broadly, the columns), and the **permanence of results** (stable identifiers for a hub's data).
- **Actively requested by hub admins and the community.** In [hubverse discussion #40](https://github.com/orgs/hubverse-org/discussions/40) several maintainers (Harry, Anna, Sam Abbott, Kath Sherratt, CyGei and others) ask for exactly this: machine-readable (JSON-LD) hub metadata, structured metadata and best-practice reporting standards, standardized target naming and ontologies so targets are comparable across hubs, and per-model **model cards** (HuggingFace-style) describing how each model works.
- **Metadata at the point of access.** Descriptions, units and provenance are most valuable where people meet the data, not buried in a file. As with a web service where metadata is reviewed before data is pulled, anyone coming to a hub should be able to review the data dictionary and dataset description first.

### What are we _not_ trying to do?

- Not leading with ontologies or formal standards. The MVP is the pragmatic foundation; vocabularies and ontology mapping come later (see roadmap).
- Not requiring hubs to re-enter information that already exists in their config files or that the hubverse already defines for standard columns.
- **(Hard anti-aim) No schema change that requires downstream software adaptation.** The hub config [`schemas`](https://github.com/hubverse-org/schemas) are depended on by the whole software stack, so we will not change the meaning or structure of existing config properties (for example, what a task ID property holds). Purely *additive*, backward-compatible config metadata that downstream software can safely ignore is acceptable, and in some cases desirable (see [Task ID and output type metadata in the hub config](#task-id-and-output-type-metadata-in-the-hub-config)).
- Not building a hosted metadata portal or catalogue in Phase 1.

### How do we judge success?

- Before or when accessing any hub data product (through `hubData`/the Python client or a viewer), a user can review a clear data dictionary for every column, plus a dataset-level description of what the data is, who maintains it, how to cite it, and its license.
- The large majority of that metadata is auto-generated from config and hubverse conventions; admins supply only descriptions of custom columns and a few dataset-level fields, with low friction.
- Metadata is emitted as JSON-LD using the schema.org/Dataset vocabulary (the keys mapped to schema.org terms), plus an auto-generated human-readable form.
- Phase 1 field naming and structure follow a recognized standard, so adding value-level vocabulary links and identifiers later is an extension, not a rewrite.

### What are possible solutions?

The proposed approach is the three-phase roadmap in [Ready to make it](#-ready-to-make-it). Each phase is independently useful and releasable.

## ✅ Validation

### What do we already know?

Much of the metadata is already present or derivable; the work is to assemble, complete and surface it. Mapped to what a user encounters:

| Metadata a user needs | Available now? | Source |
|-----------------------|----------------|--------|
| Column **names** for all data products | Yes, every column is listed | `tasks.json` task IDs; `target-data.json` |
| Column **data types** | Authoritative for standard task IDs and the representation columns; declared for non-task-id target columns; for custom task IDs only **inferred** today (not authoritative, can be ambiguous), declared once `hub-config/hub-metadata.json` exists | tasks schema; `target-data.json` `non_task_id_schema`; `hubData::create_hub_schema()` infers custom types from values; `hub-metadata.json` `task_ids` |
| Descriptions of **standard task ID columns** (e.g. `origin_date`, `horizon`, `location`, `target_end_date`) | Yes, the hubverse maintains standard names and definitions | `description` of each standard task ID in the tasks schema (machine-readable, versioned); hubDocs repeats the same definitions |
| Descriptions of the **representation columns** (`output_type`, `output_type_id`, `value`, `observation`, `oracle_value`) | Yes, fixed hubverse semantics | hubDocs model-output and target-data pages |
| **Target** column values | Yes, partially | `tasks.json` `target_metadata` (`target_id`, `target_name`, `target_units`, `description`, `target_type`) |
| Descriptions of **custom / hub-specific columns** and their value sets | Not today. Proposed home: `hub-config/hub-metadata.json`, entries scaffolded by tooling and completed by the admin | `hub-metadata.json` `task_ids` (Phase 1) |
| Definitions of **output type categories** (named `output_type_id` values of `pmf` and `cdf` output types, e.g. `large_increase`) | Not today. Hubs define them in READMEs and websites only. Proposed home: `hub-config/hub-metadata.json` | `hub-metadata.json` `output_types` (Phase 1) |
| **Dataset-level** identity: name, maintainer, contact, citation, repository, file formats | Yes | `admin.json` |
| **Dataset-level** free-text description | No, must be supplied | admin scaffolding |
| **Data license** | Partly: there is no hub-level license, but per-model output licenses exist in model-metadata, and `target-data` licensing comes from the upstream observed-data source | model-metadata (per-model); admin supplies the hub-level license |
| **Temporal coverage** | Derivable: extent from the date columns, resolution from `time_unit` | derived |
| **Spatial coverage** | Partly: the set of location codes present is derivable, but a geographic extent (named places, bounding box) needs the codes resolved (FIPS/GeoNames) | derived (codes); Phase 2 (geography) |
| **Model** descriptions | Yes, in mature hubs | per-model `model-metadata` yaml (hub-authored schema, varies between hubs) |

- **The standard-column definitions are the key extraction win.** The hubverse defines standard task ID columns and fixes the representation-column semantics, so a built-in dictionary can auto-describe most columns of most data products. Custom task ID columns are the gap: no schema-defined type or description. Named pmf and cdf categories have no definition anywhere in the config. `hub-config/hub-metadata.json` (proposed below) gives a hub one place to declare all of these.
- **Model metadata is hub-authored and varies between hubs.** We provide a template `model-hub-metadata-schema.json`, but each hub sets its own (FluSight adapted the template; see [FluSight model-metadata](https://github.com/cdcepi/FluSight-forecast-hub/tree/main/model-metadata)). The metadata layer must read each hub's own schema; the template is the lever for cross-hub consistency, not a guarantee. FluSight's schema already has an optional ORCID field and a `license` enum, so such conventions are MVP-level adoptions, not Phase 2 work.
- **`dataspice` is a proven model** for the extract-then-scaffold workflow and for the staged progression from a simple standard to a richer one.
- **Harry's sketch maps the destination**: a FAIR metadata standard in JSON-schema and JSON-LD, with schema.org and Dublin Core as the spine and domain ontologies for specific fields (the Phase 3 list). [`LinkML`](https://linkml.io/), which he suggests as an authoring layer, is adopted for the record standard (see [Where LinkML fits](#where-linkml-fits)). Source: [Opportunities/challenges for Hubverse metadata](https://docs.google.com/document/d/1ewJ8JFBtuUb62Qchd_tO0rWQSYUseP7U4j80O9sKUjA/edit).

### What do we need to answer?

- **How to carry the Step 2 minimum variable spec in schema.org, and the dataset-level field set.** schema.org [`PropertyValue`](https://schema.org/PropertyValue) covers name, description and `unitText` but has no slot for `dataType` or per-value definitions, so those are borrowed from another standard (e.g. [CSVW](https://www.w3.org/TR/tabular-data-primer/)'s column `datatype`). Each target's units and statistical type from `target_metadata` describe what the `value` column holds in rows with that target, not the target value itself, so the record needs a slot on `value` that is conditional on `target`. `output_type_id` needs one conditional on target and output type. The dataset-level field set still needs pinning down. See the [schema.org datasets guide](https://schema.org/docs/data-and-datasets.html).
- **How complete is the built-in standard-column dictionary**, and how do we detect when a hub's column is "standard" vs. custom (exact name match, or something more flexible)?
- **Which of Harry's field-to-standard mappings are cheap enough for Phase 1?** Some need no ontology machinery, just a convention on an existing field (ORCID, DOI, ISO 8601 dates). Heavier mappings (MIDAS ontology for targets, EDAM for formats, GeoNames for locations) are Phase 2+. Decided against Harry's analysis, not guessed.
- **Persistent identifiers for results** (Harry's third gap, Nick's key question): what mechanism (e.g. DataCite/Zenodo DOIs) and at what granularity (hub, data product, version)? Likely Phase 2, but the MVP structure must not preclude it.
- **How to handle per-hub variation in the model-metadata schema**, and whether to encourage a recommended core set the metadata layer can rely on across hubs.
- **How a hub describes an output type category whose meaning differs between targets or rounds.** Extraction derives which categories occur under each target and output type, and in which rounds, so a `hub-metadata.json` entry keyed by output type and category only supplies the meaning. That fails when one label means different things under different targets (`large_increase` at different thresholds) or changes between rounds. A `scheme` has the same problem: one pmf can hold epiweek codes for a peak-week target while another holds categories for a rate-change target. `validate_config()` can detect a label used under more than one target, or a target whose category set changes between rounds, and ask the admin whether the meanings match. How the file then holds a target- or round-specific description is still to be decided, before `hub-metadata-schema.json` is released.
- **Where the metadata record lives in a hub.** Candidate: a top-level `metadata/` directory holding the JSON-LD and the generated page. The cloud sync would need to mirror it (the synced directories are currently a fixed list).

## 👍 Ready to make it

### Proposed solution

A three-phase roadmap. **Phase 1 is the pragmatic foundation**, delivered in four stacked steps: extract a dataset-level record, then a data dictionary, from what already exists; make completing the rest easy; then render the result. Standards, vocabularies, identifiers and ontologies come in Phases 2 and 3, apart from the cheap conventions in Step 4.

Three design principles run throughout:

> **Any metadata is better than none.** Start small, cover the bare minimum, then build up. Every phase leaves a hub more discoverable and better described than before.

> **Decide early for later.** Phase 1 settles a schema.org-based structure and naming, borrowing fields from Dublin Core, CSVW and others where needed, so later work extends rather than rewrites.

> **We build the tooling, not a mandate.** For every step we provide automation, shared crosswalks or suggestions to confirm, so the effort on hub admins and modelers stays minimal. Where manual input is unavoidable, we say whose responsibility it is: admin, modeler, or us.

This is a **cross-cutting descriptive-metadata layer** over the existing config files and data products, not a single new file. The one addition to the hub config is `hub-config/hub-metadata.json`, described next.

### Task ID and output type metadata in the hub config

**Decision:** custom task ID metadata and output type category definitions are captured in the config up front, in a new optional file `hub-config/hub-metadata.json` with its own schema, `hub-metadata-schema.json`, in [`hubverse-org/schemas`](https://github.com/hubverse-org/schemas). The file is additive: nothing in `tasks.json` changes, and software that does not read the file behaves as it does today, so it stays within the hard anti-aim.

Initially the file holds two sections. `task_ids` is keyed by task ID name, standard or custom. Each entry can carry a description, a declared type, an identifier for the concept the column represents, and the code scheme its values use, plus a description and identifier per value where useful. `output_types` is keyed by output type and mirrors the `output_type` / `output_type_id` structure of `tasks.json`. Each entry can carry a description and an identifier for the output type, plus an `output_type_id` entry: what the ID holds for that output type, the code scheme its values use, and a description and identifier per value. These meanings are hub-level facts, which is why the file sits beside `tasks.json` rather than inside its per-round structure. It is not frozen: a hub can change a definition, and the change is recorded rather than prevented.

> [!NOTE]
> Stability is recorded, not enforced. Today `tasks.json` is validated round by round and `hubData::create_hub_schema()` widens a task ID's type across rounds silently, so a change between rounds is neither blocked nor visible. With `hub-metadata.json`, `validate_config()` reports a round whose values no longer fit the declared type, so the admin updates the declaration deliberately rather than by accident; only a declaration the configured values can never satisfy is an error. The existing `validate-config` action runs these checks on every config pull request. The record carries a version and modification date and is regenerated at each hub release, so a user can see what a column meant at any release, and the Phase 2 persistent identifiers per version make those snapshots citable.

The file does more than record descriptions:

- **Declared types are authoritative.** Today `hubData::create_hub_schema()` infers a custom task ID's type from the R type of its configured values, which cannot tell, for example, a numeric code used as a label from an integer quantity. A declared type is used instead of the inferred one, and validation checks submitted columns against it. Without the file, inference continues as today.
- **The Phase 2 value-linking hook is captured at source.** `term` and `scheme` are the slots the Phase 2 crosswalks fill, suggested by tooling and confirmed by the admin once, and the JSON-LD emission reads `term` as `propertyID`. They are recorded once in the config rather than reconstructed at every extraction.
- **Extraction reads instead of asking.** The Phase 1 Step 2 scaffold writes an entry for each custom task ID, and for each pmf or cdf output type with named categories, into this file. The admin completes it there, and extraction then needs no further input for those entries.

**Schema sketch.** Follows the other config schemas: draft 2020-12, a required `schema_version`, `additionalProperties: false` throughout, and an `additional_metadata` escape hatch at the top level. Entries are defined under `$defs`: `task_ids` and `output_types` share the per-value entry, and a later section for non-task-ID columns can reuse the column entry. The `output_type_id` entry mirrors the column entry without `type`, which `tasks.json` already fixes. jsonvalidate resolves the reference; the hubAdmin helpers for this file read the definition from `$defs` rather than by property path.

| Property | Required | Holds |
|----------|----------|-------|
| `schema_version` | yes | URL of the schema version the file is written for |
| `task_ids` | no | Object keyed by task ID name. Every key must name a task ID in `tasks.json` |
| `task_ids.<name>.description` | no | What the column is, for the data dictionary |
| `task_ids.<name>.type` | for a custom task ID's entry | One of `character`, `double`, `integer`, `logical`, `Date` (the `non_task_id_schema` enum). For a standard task ID, must be a type the tasks schema permits |
| `task_ids.<name>.term` | no | `{ uri, label }`: the ontology or vocabulary identifier for what the column is; emitted as `propertyID` |
| `task_ids.<name>.scheme` | no | `{ uri, label }`: the code list the values come from (FIPS state codes, ISO 3166), when there is one; selects the crosswalk |
| `task_ids.<name>.values` | no | Object keyed by value (written as a string). Each holds `description` and `term` for that value, where `term` identifies what the value denotes. Keys must be configured values of the task ID |
| `output_types` | no | Object keyed by output type. Every key must name an output type in `tasks.json` |
| `output_types.<name>.description` | no | What the output type is, for the data dictionary |
| `output_types.<name>.term` | no | `{ uri, label }`: the vocabulary identifier for the output type (for example a statistics ontology term for probability mass function) |
| `output_types.<name>.output_type_id.description` | no | What the ID holds for this output type (for `sample`, what one index represents) |
| `output_types.<name>.output_type_id.scheme` | no | `{ uri, label }`: the code list the values come from (MMWR epidemiological weeks), when there is one; selects the crosswalk |
| `output_types.<name>.output_type_id.values` | no | Object keyed by `output_type_id` value. Each holds `description` and `term` for that value. Keys must be configured `output_type_id` values of that output type |
| `additional_metadata` | no | Free-form, as in the other config files |

<details>
<summary>Schema skeleton (property descriptions omitted)</summary>

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://raw.githubusercontent.com/hubverse-org/schemas/main/v7.0.0/hub-metadata-schema.json",
  "type": "object",
  "properties": {
    "schema_version": { "type": "string", "format": "uri" },
    "task_ids": {
      "type": "object",
      "additionalProperties": { "$ref": "#/$defs/column" }
    },
    "output_types": {
      "type": "object",
      "additionalProperties": { "$ref": "#/$defs/output_type" }
    },
    "additional_metadata": { "type": "object", "additionalProperties": true }
  },
  "required": ["schema_version"],
  "additionalProperties": false,
  "$defs": {
    "column": {
      "type": "object",
      "properties": {
        "description": { "type": "string" },
        "type": { "enum": ["character", "double", "integer", "logical", "Date"] },
        "term": { "$ref": "#/$defs/term" },
        "scheme": { "$ref": "#/$defs/term" },
        "values": {
          "type": "object",
          "additionalProperties": { "$ref": "#/$defs/value" }
        }
      },
      "additionalProperties": false
    },
    "output_type": {
      "type": "object",
      "properties": {
        "description": { "type": "string" },
        "term": { "$ref": "#/$defs/term" },
        "output_type_id": { "$ref": "#/$defs/output_type_id" }
      },
      "additionalProperties": false
    },
    "output_type_id": {
      "type": "object",
      "properties": {
        "description": { "type": "string" },
        "scheme": { "$ref": "#/$defs/term" },
        "values": {
          "type": "object",
          "additionalProperties": { "$ref": "#/$defs/value" }
        }
      },
      "additionalProperties": false
    },
    "value": {
      "type": "object",
      "properties": {
        "description": { "type": "string" },
        "term": { "$ref": "#/$defs/term" }
      },
      "additionalProperties": false
    },
    "term": {
      "type": "object",
      "properties": {
        "uri": { "type": "string", "format": "uri" },
        "label": { "type": "string" }
      },
      "required": ["uri"],
      "additionalProperties": false
    }
  }
}
```

</details>

**Standard task IDs.** The built-in dictionary supplies their description and the tasks schema constrains their type, so an entry is optional. An entry may narrow the description to the hub's usage (for example `location` as two-digit FIPS codes), add `term` and `scheme`, and describe hub-specific values. Where the tasks schema permits more than one type (`horizon` and `scenario_id` may be integer or string), a declared `type` settles which, as it does for a custom task ID. Values a hubverse crosswalk already covers need no per-value entries. The hub names the code list once in the column's `scheme` (FIPS state codes for `location`), and the crosswalk supplies each value's name and link at extraction. The scheme is suggested from the shape of the values and confirmed by the admin. Per-value entries are for values only the hub can explain. The `target` task ID keeps its value descriptions in `target_metadata`; `hub-metadata.json` adds only `term` for target values.

**Output types.** The set of output types is fixed by the hubverse, so the built-in dictionary supplies a description of each, and from Phase 2 a `term`. What the record needs for `output_type_id` differs by output type:

| Output type | `output_type_id` holds | In the record | From the hub |
|-------------|------------------------|---------------|--------------|
| `mean`, `median` | `NA` | Marked as unused | Nothing |
| `quantile` | A probability level | One entry per configured level, generated (the 0.025 level is the 2.5% quantile), units from the target | Nothing |
| `cdf` | A value of the target: a numeric threshold, a date, or an ordinal category | One entry per configured value. Numeric and date values are generated from the target's scale. Ordinal categories take their order from `tasks.json`, which lists them from low to high | A description of each named category |
| `pmf` | A category of a discrete outcome, as a string, which may be a numeric value of the target | One entry per configured category. Numeric values are generated from the target's scale. Ordinal categories take their order from `tasks.json` | A description of each named category, or a `scheme` when the values are codes (the epiweek `EW202240`) |
| `sample` | A sample index | No per-value entries, since `tasks.json` does not list the indices. A description of what one index represents, generated from `compound_taskid_set` (one trajectory across horizons, for each location and reference date) and the sample counts | Optionally, a narrower description |

**Validation.** JSON schema validation covers the structure. `hubAdmin::validate_config()` adds the checks that depend on `tasks.json`: every `task_ids` key names a configured task ID, every `values` key is a configured value of that task ID, every `output_types` key names a configured output type and every `output_type_id.values` key a configured value of it, `type` is present in a custom task ID's entry, and a declared type can hold the configured values and, for a standard task ID, is one the tasks schema permits. A round whose values no longer fit the declared type, or a custom task ID with no entry, is reported but does not fail validation.

**Example.** A hub with one custom task ID, `pathogen`, a narrowed description of the standard `location`, and one pmf category. URIs are illustrative.

```json
{
  "schema_version": "https://raw.githubusercontent.com/hubverse-org/schemas/main/v7.0.0/hub-metadata-schema.json",
  "task_ids": {
    "pathogen": {
      "description": "Pathogen the target quantity refers to.",
      "type": "character",
      "values": {
        "flu": {
          "description": "Influenza",
          "term": { "uri": "http://purl.obolibrary.org/obo/NCBITaxon_11308", "label": "Influenza A virus" }
        },
        "covid": {
          "description": "COVID-19",
          "term": { "uri": "http://purl.obolibrary.org/obo/NCBITaxon_2697049", "label": "SARS-CoV-2" }
        }
      }
    },
    "location": {
      "description": "US states and territories as two-digit FIPS codes, plus \"US\" for the national level.",
      "scheme": { "uri": "https://www.census.gov/library/reference/code-lists/ansi.html", "label": "ANSI/FIPS state codes" }
    }
  },
  "output_types": {
    "pmf": {
      "output_type_id": {
        "values": {
          "large_increase": {
            "description": "The weekly rate rises by more than the hub's large-change threshold for the horizon."
          }
        }
      }
    }
  }
}
```

### Where LinkML fits

The tooling has three parts. [LinkML](https://linkml.io/) is used in the middle one only, at build time, and never touches the hub config.

- **Inputs: the hub config.** `admin.json`, `tasks.json`, `target-data.json`, the model-metadata files and `hub-metadata.json`, validated by hand-written JSON Schema in `hubverse-org/schemas` as today. `hub-metadata.json` is one more input the extractor reads.
- **The record standard, authored in LinkML.** The record is the machine-readable JSON-LD output of Phase 1, the file a catalogue or a client reads. One YAML file, the **record definition**, states its classes and slots (hub as `DataCatalog`, data product as `Dataset`, column as `PropertyValue` with the borrowed `dataType` and per-value slots) and the standard term each slot maps to (`schema:description`, `csvw:datatype`, `dcterms:license`). Note that LinkML calls this file a schema; this poster says record definition to keep it apart from the config schemas and from forecasting models. From that one file LinkML generates the JSON Schema that validates the record, the JSON-LD `@context` that links the record's keys to those terms, and the documentation of the standard. The generated files are committed and used as static files. Without LinkML, those three artifacts are hand-written and kept in sync by hand.
- **Extraction and emission, in `hubAdmin`.** `extract_metadata()` maps config into instances of the record. `write_metadata()` validates against the generated JSON Schema and attaches the generated context. R never depends on LinkML at runtime; generation runs in CI when the record definition changes.

The record definition is authored in Phase 1, since the record's structure and context are Phase 1 deliverables. Phase 2 adds slot-level mappings and `meaning` URIs on enum values to the same file, as hubs' `term` and `scheme` entries flow through. Phase 3 adds OWL and SHACL projections. `hub-metadata.json` and the record definition never reference each other: the hub records a column's `term`, the extractor places it in the record's `propertyID` slot, and the context types that slot as a URI.

![Hub config feeds extract_metadata(); the record definition generates the JSON Schema and context that write_metadata() uses to emit the record](tooling.svg)

A slice of the record definition, with the per-value and conditional slots left out until the open question on the variable spec is settled. Each slot names the standard term it maps to; `gen-json-schema` turns this into the validator and `gen-jsonld-context` into the context that maps `name` to `schema:name`, `data_type` to `csvw:datatype` and so on.

```yaml
id: https://hubverse.io/metadata-record
name: hubverse-metadata-record
prefixes:
  linkml: https://w3id.org/linkml/
  hub: https://hubverse.io/metadata-record/
  schema: http://schema.org/
  csvw: http://www.w3.org/ns/csvw#
default_prefix: hub
default_range: string
imports:
  - linkml:types

classes:
  Dataset:
    class_uri: schema:Dataset
    slots: [name, description, license, variables]
  Variable:
    class_uri: schema:PropertyValue
    slots: [name, description, data_type, unit, term]

slots:
  name:
    slot_uri: schema:name
    required: true
  description:
    slot_uri: schema:description
  license:
    slot_uri: schema:license
  variables:
    slot_uri: schema:variableMeasured
    range: Variable
    multivalued: true
  data_type:
    slot_uri: csvw:datatype
    range: DataType
  unit:
    slot_uri: schema:unitText
  term:
    slot_uri: schema:propertyID
    range: uriorcurie

enums:
  DataType:
    permissible_values:
      character:
      double:
      integer:
      logical:
      Date:
```

The `@context` is a lookup table that ships with the record and gives each key its meaning as a URI, so a reader sees `name` as `schema:name` and `data_type` as `csvw:datatype` rather than as labels we chose. LinkML generates it from the terms declared in the record definition.

---

### Phase 1: minimum viable metadata, extracted and completed (MVP)

*Goal: every hub ships a schema.org metadata record and a per-column data dictionary, mostly auto-generated, with the rest easy to complete.*

Phase 1 is delivered in `hubAdmin`, with the config schema in `hubverse-org/schemas`. Step 1 is deliberately the smallest useful thing.

![Most of the record is read automatically from admin.json, the built-in dictionary, target-data.json and tasks.json; the admin supplies the description, license and keywords and completes hub-metadata.json entries for custom task IDs and output type categories](phase1-sources.svg)

**Step 1: dataset-level record, extracted and completed.** The lowest-hanging fruit: the fields a data user or a catalogue needs to find, cite and reuse a hub, most of which `admin.json` already holds.

- `extract_metadata(hub_path)` reads `admin.json` (name, maintainer, contact, repository URL, citation, file formats) and the list of data products a hub publishes, and returns a pre-filled record, stamped with a version and modification date: the hub as a schema.org `DataCatalog`, each data product (`model-output`, `time-series`, `oracle-output`) as a `Dataset` inheriting the hub-level fields.
- The gaps extraction cannot fill are exactly three: a free-text description, the license and keywords. The record carries them as empty fields rather than leaving them out, so the admin sees exactly what to fill in, and `write_metadata()` reports any still empty.
- Emits the record as JSON-LD, validated against the JSON Schema and carrying the `@context` generated from the record definition; values stay literals. Where the file lives in the hub is an open question (see [What do we need to answer?](#what-do-we-need-to-answer)).
- *Done when:* two function calls on any existing hub yield a record that passes the [schema.org validator](https://validator.schema.org/), and the admin has typed only the description, license and keywords.

**Step 2: per-column data dictionary.** Adds `variableMeasured` to each data product's `Dataset` from Step 1.

- **minimum per variable**: `name`, `description`, `dataType`, `units` (numeric variables), and a `description` for each permissible value of categorical (string/factor) variables.
- the `hub-metadata-schema.json` release in `hubverse-org/schemas` comes first, with its `hubAdmin::validate_config()` checks and `hubData::create_hub_schema()` reading declared types
- pull every column name from `tasks.json` and `target-data.json` (auto), with authoritative types for standard task IDs, the representation columns, declared non-task-id target columns, and custom task IDs declared in `hub-metadata.json`
- auto-describe standard task ID columns and the representation columns (`output_type`, `output_type_id`, `value`, `observation`, `oracle_value`) from a built-in hubverse dictionary
- pull `target` column descriptions from `target_metadata`
- derive the `output_type_id` entries from `tasks.json`: which values occur under each target and output type, in which rounds, and the order of ordinal categories; generated descriptions of quantile levels and of numeric or date cdf and pmf values; and the structure of samples from `compound_taskid_set` and the sample counts. The record's `output_type_id` entry is conditional on target and output type, as `value` is on target. `hub-metadata.json` supplies only category descriptions, schemes and terms
- scaffold a `hub-metadata.json` entry for each custom task ID column, with `type` pre-filled from today's inference for the admin to confirm and `description` left empty to write, and the same for hub-specific value sets and named pmf and cdf categories
- *Done when:* a hub whose columns are all standard gets a complete dictionary with no admin input, and a hub with a custom task ID (FluSight's `reference_date`, which is not in the schema's standard set) is asked only to complete the scaffolded `hub-metadata.json` entries for those columns and any named pmf or cdf categories.

**Step 3: human-readable output.** A data dictionary and landing page generated from the Step 2 JSON-LD and served from the hub repo (as `dataspice` builds a site from its JSON-LD), so the metadata is visible where people meet the data. Embedding the JSON-LD in the page also makes the data products discoverable through general tools such as Google Dataset Search.

- *Done when:* a hub's repo serves the generated page, and the page passes Google's [Rich Results Test](https://search.google.com/test/rich-results) for `Dataset`.

**Step 4: cheap conventions.** Small additions that fall out of Steps 1 to 3.

- Identifier conventions that need no ontology machinery and are already precedented: ORCID and the `license` enum, with DOI for citations and ISO 8601 dates as candidates.

**Deliverable:** for any hub, a schema.org JSON-LD record with a per-column data dictionary for each data product, mostly auto-generated, plus a human-readable page served from the hub repo, and the `hub-metadata-schema.json` config schema with its validation.

---

### Phase 2: mature and enrich, linking values, vocabularies and identifiers

*Goal: link field values to external entities, vocabularies and ontologies, and add persistent identifiers, with tooling that does the heavy lifting (shared crosswalks, suggested mappings, automated minting) so the burden on hubs stays low.*

- **Enrich the dataset-level record** beyond what Step 1 asks the admin for. Suggest the `model-output` license from the per-model licenses in model-metadata (e.g. the most restrictive, flagging incompatible combinations) for admin confirmation, with `target-data` licensing recorded separately from its upstream source. Derive temporal coverage from the date columns and `time_unit`, and the geographic extent from the location codes through the crosswalk below. Render each model's metadata as a human-readable **model card**, the community-requested, HuggingFace-style artifact, reading each hub's own model-metadata schema.
- **Link field values to entities and ontologies.** Take the Phase 1 JSON-LD further by resolving literal values to external identifiers and concepts (via `@id` / typed values), so a location code, a target, a contributor or a license becomes a linked entity rather than a plain string. This is the deeper FAIR-Interoperable step and makes data comparable across hubs. `term` and `scheme` entries a hub has recorded in `hub-metadata.json` are read directly.
- **We provide the common crosswalks.** Ship and maintain the mappings for standard columns and values (FIPS↔GeoNames for `location`, EDAM for file formats, OWL-time for timezones, a target→concept map, output types→statistics vocabulary terms), applied wherever a column's `scheme` names a code list we cover. Admins only confirm the suggested scheme and map genuinely hub-specific values. Hubs can still add or override with their own, and confirmed mappings for hub-specific values are written to `hub-metadata.json` so every extraction reuses them. (Deep domain ontologies such as MIDAS are Phase 3.)
- **Persistent identifiers for results** (the permanence gap): provide a workflow that mints and registers DOIs (e.g. Zenodo/DataCite, triggered from the hub) so the hub owner opts in rather than minting by hand. Mechanism and granularity still open.
- **Provenance (lineage)** with [PROV-O](https://www.w3.org/TR/prov-o/). For **target data**, the target-data maintainer declares the upstream source once and tooling records it and the `as_of` versions. For **model output**, lineage to a model's `data_inputs` (the upstream data the modeler used) is the modeler's responsibility and lives in their model-metadata. We cannot derive it for them, but we can provide the modeler-facing analog of the admin crosswalks: documentation plus an interactive lookup of common data sources and their identifiers, so modelers select structured references rather than writing free text.
- **FAIR self-assessment**: a report of what each hub has vs. the standard, with a maturity score that shows admins exactly which gaps to fill and nudges them from bare minimum toward rich.
- **Surface metadata across access surfaces**: retrievable through the access tools (`hubData`, the Python client) so anyone connecting to a hub on S3 or GitHub can review the data dictionary and dataset description before collecting data; a data-dictionary page on the dashboards; and, optionally, inline in the viewers (predtimechart, predevals, hubVis) alongside validation status.

How a declared scheme turns codes into linked places, the mechanism behind the value-linking and crosswalk bullets:

![hub-metadata.json declares FIPS for location; the crosswalk it selects turns each code into a GeoNames place in the record](phase2-crosswalk.svg)

**Deliverable:** a vocabulary- and ontology-linked metadata standard (values resolved to external entities), persistent identifiers, a per-hub FAIR maturity report, and metadata surfaced across access surfaces (access tools, dashboard data-dictionary page, and viewers).

---

### Phase 3: long-distance, domain ontologies and ecosystem

*Goal: formal interoperability within and across hubs, and integration with the wider data ecosystem.*

- **Domain ontology mapping**: MIDAS ontology for targets, NCBI taxon for pathogens/vectors, SNOMED/ICD10 for outcomes, Software License Ontology for licenses, NIH RePORTER for funding.
- **Publish the record definition as an ontology.** LinkML can project the same definition to OWL, so the hubverse's column and target definitions become terms other ontologies and tools can reference, and to SHACL, which validates records once they are treated as linked data rather than JSON.
- **Link model metadata to model abstracts** and tackle the hardest case, ontologizing methods (where MIDAS work is ongoing).
- **Cross-hub catalogue, search and comparison**; certifiable FAIR scoring; integration with external repositories/portals with rich metadata presentation and citation.

Why the domain terms matter: two hubs that name a target differently become comparable once both carry the same term (identifier illustrative):

![Two hubs with different target names share one MIDAS term, so a cross-hub query finds both](phase3-crosshub.svg)

**Deliverable:** domain-ontology crosswalks, the record definition published as an ontology, and cross-hub discovery and comparison.

---

### Visualize the solution: literals become linked entities

A slice of a data product's metadata record. **Phase 1** emits schema.org-structured JSON-LD with literal values (the keys are linked to schema.org terms; the values are plain strings). **Phase 2** resolves those literals to linked entities (a license URI, an ORCID, ontology terms). Illustrative and simplified; URIs are examples, and the per-value structure under `target` is a placeholder for the slots the open question on the minimum variable spec has to settle.

**Phase 1** (schema.org keys linked, values are literals). Every task ID column is described once, and each of its values once beneath it. `target` is an ordinary character task ID. The units and statistical type in `target_metadata` describe what the `value` column holds in rows with that target, so they sit on `value`, conditional on the target:

```json
{
  "@context": "https://schema.org",
  "@type": "Dataset",
  "name": "FluSight model output",
  "license": "CC-BY-4.0",
  "creator": { "@type": "Person", "name": "Jane Modeler" },
  "variableMeasured": [
    { "@type": "PropertyValue", "name": "location",
      "description": "US state, as a two-digit FIPS code",
      "values": [
        { "name": "06", "description": "California" },
        { "name": "US", "description": "United States" }
      ] },
    { "@type": "PropertyValue", "name": "target",
      "description": "Quantity the row's value refers to",
      "values": [
        { "name": "wk inc flu hosp",
          "description": "Weekly incident influenza hospitalizations" },
        { "name": "wk inc flu prop ed visits",
          "description": "Weekly proportion of emergency department visits due to influenza" }
      ] },
    { "@type": "PropertyValue", "name": "value",
      "description": "Predicted value; units and type depend on target",
      "byTarget": [
        { "target": "wk inc flu hosp", "unitText": "count" },
        { "target": "wk inc flu prop ed visits", "unitText": "proportion" }
      ] }
  ]
}
```

**Phase 2** links the values (changes shown as a diff, red removed / green added):

```diff
   "@type": "Dataset",
   "name": "FluSight model output",
-  "license": "CC-BY-4.0",
+  "license": { "@id": "https://creativecommons.org/licenses/by/4.0/" },
-  "creator": { "@type": "Person", "name": "Jane Modeler" },
+  "creator": { "@type": "Person", "name": "Jane Modeler",
+               "@id": "https://orcid.org/0000-0002-1825-0097" },
   "variableMeasured": [
     { "@type": "PropertyValue", "name": "location",
       "description": "US state, as a two-digit FIPS code",
       "values": [
-        { "name": "06", "description": "California" },
+        { "name": "06", "description": "California",
+          "@id": "https://www.geonames.org/5332921" },
         ...
       ] },
     { "@type": "PropertyValue", "name": "target",
       "description": "Quantity the row's value refers to",
       "values": [
         { "name": "wk inc flu hosp",
-          "description": "Weekly incident influenza hospitalizations" },
+          "description": "Weekly incident influenza hospitalizations",
+          "propertyID": "midas:IncidentHospitalizations" },
         ...
       ] },
     ...
   ]
 }
```

---

### Relationship to existing work

This project adds a descriptive-metadata layer on top of the existing config schemas and data-product conventions. It adds one optional config file, `hub-config/hub-metadata.json`, and changes none of the existing schemas (see the hard anti-aim). It operationalizes Harry's [FAIR metadata sketch](https://docs.google.com/document/d/1ewJ8JFBtuUb62Qchd_tO0rWQSYUseP7U4j80O9sKUjA/edit) on a `dataspice`-style staged path.

Reference material: [`dataspice`](https://github.com/ropensci/dataspice); [Anna's metadata teaching materials](https://acce-rrresearch-2026.netlify.app/) (`rrresearch-acce-rrse`, metadata chapters `04a` to `04c`); the hubverse [schemas](https://github.com/hubverse-org/schemas), [model-output](https://docs.hubverse.io/en/latest/user-guide/model-output.html), [target-data](https://docs.hubverse.io/en/latest/user-guide/target-data.html) and [hub-config](https://docs.hubverse.io/en/latest/user-guide/hub-config.html) docs; [schema.org/Dataset](https://schema.org/Dataset) and [datasets guide](https://schema.org/docs/data-and-datasets.html); [Dublin Core](https://www.dublincore.org/); [LinkML](https://linkml.io/); [hubverse discussion #40](https://github.com/orgs/hubverse-org/discussions/40) (community requests, incl. model cards).

### Scale and scope

- **Required team size:** TBD.
- **Required delivery date:** none; phased and incremental by design. Phase 1 is the priority and is independently shippable.
- **Sequencing:** Phases are ordered, and so are the four Phase 1 steps: dataset-level record, data dictionary, human-readable page, conventions. Each step is releasable on its own. Phases 2 and 3 enrich the Phase 1 record.
