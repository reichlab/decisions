# Refactor Summary: `mc/idmodels-iddata-refactor`

## Changes by Repository

### iddata (v0.1.0 → 2.0.0) — 14 commits

The monolithic `DiseaseDataLoader` god-object was decomposed into a proper class hierarchy:

- **`DataSource` ABC** (`sources/base.py`) with four concrete subclasses — `NHSNDataSource`, `NSSPDataSource`, `ILINetDataSource`, `FluSurvNetDataSource` — each owning its own loading and filtering logic
- **`AncillaryData` ABC** (`ancillary/base.py`) separates supplementary data (covariates) from surveillance targets; `PopulationData` is its first implementation
- **`enums.py`** — canonical `Disease`, `SourceType`, `AggLevel` enums now live here and are imported by idmodels
- **`constants.py`** — `S3_DATA_RAW_URL`, `PANDEMIC_SEASONS`, etc. extracted from inline hardcoding
- **`s3.py`** — versioned S3 file lookup logic extracted into a standalone function
- **Power transform, normalization, and source-specific numeric offsets removed** from `DiseaseDataLoader.load()` and `DataSource.load()` methods — all moved to idmodels; `DataSource.load()` now returns data in original measurement units
- **`drop_pandemic_seasons` moved** from individual `DataSource` constructors to `DiseaseDataLoader.load()`, since zeroing out pandemic seasons is a dataset-level concern applied uniformly across all sources; a warning is now raised when `drop_pandemic_seasons=False` with NHSN or FluSurvNet burden adjustment, since complete pandemic season data is unavailable in those cases regardless
- **FluSurvNet burden adjustment join fixed** — changed from inner join to left join so pandemic seasons (which lack CDC burden estimates) produce `NaN` rather than being silently dropped from the output

### idmodels (v1.3.1 → 2.0.0) — 19 commits

Three new abstraction layers introduced:

- **`transforms.py`** — `Transform` ABC with `SourceScaleTransform`, `FourthRootTransform`, `IdentityTransform`, `CenterScaleTransform`, and `ComposedTransform`; forward `apply()` and `invert()` are paired methods on the same object, replacing previously scattered inverse logic
- **`features.py`** — decomposed `create_features_and_targets()` into composable `Feature` subclasses (`LagFeature`, `HolidayFeature`, `OneHotEncodingFeature`, `TaylorFeature`, `RollingMeanFeature`, etc.) chained by a `FeaturePipeline`; `preprocess.py` deleted
- **`IDModel` ABC** (`model.py`) — both `GBQRModel` and `SARIXModel` now inherit from it; shared orchestration (load → transform → featurize → fit → invert → save) lives in the base class; subclasses implement only `_build_sources()`, `_build_feature_pipeline()`, `_fit_and_predict()`
- **`ModelConfig` ABC** (dataclass) with `SARIXModelConfig`, `SARIXFourierModelConfig`, `GBQRModelConfig` subclasses; `RunConfig` dataclass for run parameters; `PowerTransform` and `PoolingStrategy` enums added
- **`SourceScaleTransform` added** — applies source-specific floor and scale constants (`(inc + floor) * scale`) before the power transform, with explicit inversion after; source constants (`NHSN_FLOOR`, `ILINET_FLOOR/SCALE`, `FLUSURVNET_FLOOR/SCALE`) now live in `idmodels/constants.py`; this completes the full transform chain: `SourceScaleTransform → FourthRootTransform → CenterScaleTransform`
- `sarix.py` and `gbqr.py` significantly slimmed; `xmas_spike` feature removed from SARIX

### operational-models — 4 commits

- **Deleted** four `0_<model>.py` subprocess scripts; `main.py` in each model now imports and calls idmodels directly in-process
- **`--short_run` flag** added across all models for fast testing with reduced MCMC iterations and quantile levels
- **`Dockerfile.dev` + `build-dev.sh`** added for local development against branch tips (mounts idmodels/iddata from the local filesystem instead of pinned git hashes)

---

## Addressed

| # | Description | Status | Why It Matters |
|-|---|--------|--------|
| 1 | **Primitive Obsession** — configs were unstructured bags of arbitrary attributes | ✅ Done — all configs are typed dataclasses (`ModelConfig`, `SARIXModelConfig`, `GBQRModelConfig`, `RunConfig`) | Previously, nothing stopped you from misspelling `model_config.sorces` — the error only appeared when the model ran. Now the config objects have a fixed, documented shape: required fields are enforced at construction, optional fields have explicit defaults, and any IDE can tell you exactly what's valid before you ever run the code. |
| 2 | **Magic Strings** — disease names, source names, pooling strategies were free-form strings | ✅ Done — `Disease`, `SourceType`, `AggLevel` in `iddata/enums.py`; `PowerTransform`, `PoolingStrategy` in `idmodels/config.py` | Before, writing `disease = "Flu"` instead of `"flu"` would silently produce wrong results. Now those values are a fixed list — you pick from `Disease.FLU`, `Disease.COVID`, etc. The set of valid choices is visible in one place, and any invalid value is caught immediately. |
| 3 | **Hidden Parameter Lists** — one monolithic loader with hidden kwargs dicts per source | ✅ Done — `DataSource` ABC + `NHSNDataSource`, `NSSPDataSource`, `ILINetDataSource`, `FluSurvNetDataSource`; `DiseaseDataLoader.load()` is a thin orchestrator | The old loader had one giant function where every data source's parameters were buried inside a dictionary — callers had to know the internal names by memory or documentation. Now each source is its own class with explicit, named parameters. Adding a new data source means adding one self-contained class, not modifying a shared function that touches all sources. |
| 4 | **Strategy Pattern for Transforms** — transformation logic was scattered if/elif chains | ✅ Done — `Transform` ABC with `FourthRootTransform`, `IdentityTransform`, `CenterScaleTransform`, `ComposedTransform` in `idmodels/transforms.py` | Previously, the forward transform (applied to raw data) and its inverse (applied to predictions to get back to real units) lived in different places and had to be kept manually in sync. Now each transform is a single object with paired `apply()` and `invert()` methods — they can't get out of sync, and adding a new transform doesn't require hunting down all the places to change. |
| 5 | **Parallel Hierarchies** — model classes and their configs were separate structures with no formal relationship | ✅ Done — `ModelConfig` ABC with typed subclasses; `IDModel` ABC with `GBQRModel`/`SARIXModel` | There used to be no formal relationship between a model and its configuration — the config was just a loose collection of values that the model hoped would be there. Now both have proper inheritance hierarchies: shared behavior lives in base classes, model-specific behavior lives in subclasses, and the compiler enforces that each model has what it needs. |
| 6 | **Lack of Encapsulation** — transform logic and source-specific offsets scattered across files | ✅ Done — power transform/normalization in `idmodels/transforms.py`; source-specific offsets in `SourceScaleTransform` with paired `apply()`/`invert()` | Transformation steps were spread across the data loader, the model, and utility functions — to understand what happened to the numbers between raw data and final predictions, you had to trace through multiple files. Now each transform step is a self-contained object that knows how to both apply and reverse itself, and all normalization lives in the transforms module. |
| 7 | **Conditional Complexity** — growing if/elif chains routed behavior based on string values | ✅ Partially — source-loading chains eliminated via polymorphism; some disease/source conditionals remain in `IDModel._invert_and_scale()` and `_format_output()`, now using enums rather than raw strings | The worst of it — routing to different data sources via `if source == "nhsn": ... elif source == "flusurvnet": ...` — is gone, replaced by each source simply doing its own thing. Some branching on disease type remains (e.g. choosing the output target name), but it now uses typed enum values rather than strings, so at least the valid cases are explicit. |
| 8 | **Feature Envy** — model classes constantly reached into config objects to extract values | ✅ Partially — `IDModel.run()` centralizes orchestration; config access now confined to the base class | The old models read dozens of config attributes throughout a single method — they were really just procedures wrapped in a class. Now the shared orchestration loop (load → transform → featurize → fit → invert → save) lives in one place in the base class, and individual models only implement their specific steps. Config access still happens, but it's consolidated rather than scattered. |

## Not Yet Addressed

| # | | Status | Why It Matters |
|---|---|---|---|
| 9 | **Data Clumps** — `q_levels`/`q_labels` and MCMC settings are always used together but stored separately | ❌ Not addressed — `q_levels` and `q_labels` remain separate flat fields on `RunConfig`; `num_warmup`/`num_samples`/`num_chains` are flat on `SARIXModelConfig` | These groups of values are always set and changed together, but there's nothing enforcing that. In `short_run` mode, `q_levels` and `q_labels` are mutated in two separate lines — if you update one and forget the other, the model runs with mismatched quantile levels and labels silently. A `QuantileConfig` object with auto-derived labels, or an `MCMCConfig` grouping the three sampling parameters, would close this gap. |
| 10 | **Registry Pattern** — each model hard-codes which data source classes it instantiates | ❌ Not implemented — sources instantiated directly in `_build_sources()` | If a new data source is added to iddata, every model's `_build_sources()` method needs to be updated by hand. A registry would let sources announce themselves once and be available to all models automatically. |
| 11 | **Builder Pattern** — every model script constructs the full configuration from scratch | ❌ Not implemented | Every `main.py` repeats the same 23-element quantile list and full location list in multiple files. A builder or factory for common setups (e.g. "standard flu run" or "short test run") would reduce this repetition and make the common case one line instead of twenty. |
| 12 | **Validation Layer** — invalid configurations are only caught when they cause a downstream error | ❌ Not systematically implemented — spot `ValueError`s exist (`_filter_locations`, `NHSNDataSource.load()`) but no comprehensive `ConfigValidator` | A misconfigured reference date or empty location list is only caught when it causes a failure deep inside a model run that may have taken minutes to reach. Upfront validation would catch these immediately with a clear message, before any computation starts. |
