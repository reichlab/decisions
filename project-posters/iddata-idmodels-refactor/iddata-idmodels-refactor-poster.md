# Project Poster: Iddata-Idmodels Refactoring

- Date: 2026-02-09

- Owner: Matt Cornell

- Team: Matt, Li, Thomas

- Status: In progress

## ❓ Problem space

Summarized from [Operational model refactoring ideas #42 GitHub discussion](https://github.com/reichlab/operational-models/discussions/42) and the [iddata idmodels refactoring Google doc](https://docs.google.com/document/d/1GoC7pfFX9RQpNavUh0XhGvheAUI2oK1C9fGrvHaqBKI/edit?tab=t.0)

### What are we doing?

Refactoring the codebases of three related repositories — [iddata](https://github.com/reichlab/iddata/), [idmodels](https://github.com/reichlab/idmodels), and [operational-models](https://github.com/reichlab/operational-models/) — which developed organically and now exhibit code smells and maintenance friction.

### Why are we doing this?

To make it easier to add new models, features, and data sources to our Python modeling pipeline, and to make the code cleaner and easier to maintain.

### What are we _not_ trying to do?

- Add any new models or data sources to the codebase
- Change the code so that the current models cannot be run

### How do we judge success?

Some code smells to tackle:
- Error conditions (model and source validation) are centralized and consistent
- Duplicated code is eliminated (e.g., `_format_as_flusight_output()` across idmodels and sarix)
- The kwargs explosion pattern (`flusurvnet_kwargs`, `nhsn_kwargs`, `ilinet_kwargs`, `nssp_kwargs`) is replaced with a stable, extensible interface
- Adding a new data source or model requires minimal boilerplate and follows a clear pattern
- `ModelConfig` and `RunConfig` have well-defined, non-overlapping responsibilities
- Disease handling and target naming are consistent and validated across the codebase
- Move data transformations (when possible) from iddata to idmodels, as these are modeling decisions _not_ data ones
- Make feature creation for GBQR in idmodels more understandable

### What are possible solutions?

Ways to address the code smells:

**iddata**
- Introduce a `DataSource` ABC with per-source classes (`NHSNDataSource`, `ILINetDataSource`, etc.) in a `sources/` subpackage; replace the kwargs explosion with `sources: list[DataSource]`
- Introduce a separate `AncillaryData` ABC for supplementary data (e.g., population) not suited as a surveillance source
- Move `Disease`, `AggLevel`, and `SourceType` enums into `iddata/enums.py`; re-export `SourceType` from `idmodels` for backwards compatibility
- Extract `constants.py` for S3 constants and pandemic seasons, and `s3.py` for versioned file-selection helpers (eliminating duplicated glob → sort → filter → pick-last pattern)
- Slim `DiseaseDataLoader` to a thin orchestrator that loads and merges sources but does not apply transforms (transforms are a modeling decision)

**idmodels**
- Introduce a `Transform` ABC with composable classes (`SourceScaleTransform`, `FourthRootTransform`, `IdentityTransform`, `CenterScaleTransform`, `ComposedTransform`); move transform logic out of iddata
- Introduce a `Feature` ABC with concrete feature classes and `FeaturePipeline` to replace `preprocess.py`; make feature engineering configurable rather than hardcoded
- Introduce an `IDModel` ABC with a shared `run()` workflow; `SARIXModel` and `GBQRModel` extend it, eliminating duplicated output-formatting logic
- Extract `constants.py` for modeling constants (floors, scales, in-season week bounds)

**operational-models**
- Standardize all models on a direct instantiation pattern with inline config in `main.py`; eliminate intermediate subprocess scripts

## ✅ Validation

This section is for listing assumptions that should be validated before
kicking off a project. The answers to the prompts below may change over
time, as the team gets more information.

### What do we already know?

- The idmodels codebase spans four categories of features for GBQR: target-derived (Taylor polynomials, lags), structural metadata (location, population), external covariates (weather), and supplementary incidence streams (ILINet, NHSN, NSSP) — each with different engineering costs and merge strategies
- The kwargs pattern in `iddata` creates an unstable function signature that grows with every new source
- Duplicated output-formatting logic exists between idmodels and sarix
- `run_config` and `model_config` currently contain misassigned parameters (e.g., `num_warmup`/`num_samples` in `run_config` instead of `model_config`)
- `disease` is used for both target naming and data loading, but lacks a centralized definition or validation
- Category 3 (external covariates, e.g., weather) has the highest per-feature engineering cost and lives in model-specific repos

### What do we need to answer?

- Which transformations, if any, should move from iddata to idmodels? (Needs input from Nick and Evan)
  - Fourth root/identity, centering and scaling
- Should the output table format vary by disease, and if so, how should that be documented and enforced?
  - No
- Should "process response variable" logic live in iddata or idmodels?
  - idmodels
- Should there be a `DiseaseEnum` (or similar) to centralize disease validation and link diseases to valid sources?
  - Yes
- Does Nick plan to merge his model code changes before or after the refactor?
  - SARIX T-distributed innovations PR in idmodels will be merged later
  - Alloy modeling code in sandbox hub will not be integrated back into idmodels directly, just used as a potential guide for this refactor
- Should `load_us_census()` move out of iddata, since it's only used by modeling-based transformations?
  - No, make it separate "metadata" / ancillary data
- Should `as_of` be supported universally across all data sources?
  - Yes

## 👍 Ready to make it

If, after defining the problem space and validating assumptions, the team
decides to move forward with the project, this section is to document
a proposed solution.

Keep the answers to the prompts below brief--this document isn't
inteded to be a detailed project plan.

### Proposed solution

Refactor all three repos in dependency order (iddata → idmodels → operational-models): slim iddata to a thin data-loading orchestrator with per-source classes and shared enums; move transforms and feature engineering into composable class hierarchies in idmodels under a new `IDModel` ABC that provides a shared `run()` workflow; and standardize all operational-models models on a direct instantiation pattern with inline config.

### Visualize the solution

Target module structure:

```
src/iddata/
├── __init__.py
├── enums.py              # new: Disease, AggLevel, SourceType enums
├── constants.py          # new: S3_DATA_RAW_URL, PANDEMIC_SEASONS
├── s3.py                 # new: versioned S3 file-selection helpers
├── loader.py             # updated: DiseaseDataLoader (thin orchestrator)
├── utils.py              # updated
├── sources/
│   ├── __init__.py
│   ├── base.py           # new: DataSource ABC
│   ├── nhsn.py           # new: NHSNDataSource
│   ├── nssp.py           # new: NSSPDataSource
│   ├── ilinet.py         # new: ILINetDataSource
│   └── flusurvnet.py     # new: FluSurvNetDataSource
└── ancillary/
    ├── __init__.py
    ├── base.py           # new: AncillaryData ABC
    └── population.py     # new: PopulationData

src/idmodels/
├── __init__.py
├── config.py             # updated: SourceType re-exported from iddata
├── constants.py          # new: NHSN_FLOOR, ILINET_FLOOR/SCALE, FLUSURVNET_FLOOR/SCALE, etc.
├── transforms.py         # new: Transform ABC and concrete implementations
├── features.py           # new: Feature ABC, concrete features, FeaturePipeline
├── model.py              # new: IDModel ABC with shared run() workflow
├── sarix.py              # updated: SARIXModel extends IDModel
├── gbqr.py               # updated: GBQRModel extends IDModel
├── spatial_utils.py      # unchanged
└── utils.py              # unchanged
```

Summary of feature taxonomy for ML models (currently just GBQR in idmodels):

| Category | Adds | Code location | Cost to add | Examples |
|---|---|---|---|---|
| 1. Target-derived | Features (columns) | idmodels/preprocess.py | Low (config) | Taylor polynomials, lags |
| 2. Structural metadata | Features + parameters | idmodels/gbqr.py, lookup tables | Low (one-time) | location, log_pop, lat/lon |
| 3. External covariates | Features (columns) | mchub_gbqr/data_loader.py | High (new pipeline) | Weather |
| 4. Supplementary incidence | Training rows | iddata/loader.py | Medium (templated) | ILINet, NHSN, NSSP |

### Scale and scope

- Team: Matt, Li, Thomas (+ input from Nick and Evan on transformation/disease questions)
- Implementation order: iddata first, then idmodels and operational-models
- No hard delivery date currently specified
