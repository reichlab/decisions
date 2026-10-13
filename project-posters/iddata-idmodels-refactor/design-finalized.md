# Refactoring Design v4: iddata / idmodels / operational-models

## Overview

This document specifies interfaces and signatures for the refactored codebase. It is intended to be read and agreed upon before any code is written.

The repos depend on each other in this order:
```
operational-models → idmodels → iddata
```

---

## 1. iddata

### 1.1 New module structure

```
src/iddata/
├── __init__.py
├── enums.py              # new: Disease, AggLevel, and SourceType enums
├── constants.py          # new: S3_DATA_RAW_URL, PANDEMIC_SEASONS
├── s3.py                 # new: S3 file-selection helpers
├── sources/
│   ├── __init__.py
│   ├── base.py           # new: DataSource ABC
│   ├── nhsn.py           # new: NHSNDataSource
│   ├── nssp.py           # new: NSSPDataSource
│   ├── ilinet.py         # new: ILINetDataSource
│   └── flusurvnet.py     # new: FluSurvNetDataSource
├── ancillary/
│   ├── __init__.py
│   ├── base.py           # new: AncillaryData ABC
│   └── population.py     # new: PopulationData
├── loader.py             # updated: DiseaseDataLoader (thin orchestrator); handles drop_pandemic_seasons uniformly across all sources
└── utils.py              # updated: added load_fips_mappings(), add_season_columns() helpers
```

### 1.2 `enums.py` (new)

`Disease` is moved here from `idmodels/config.py`. `SourceType` is also moved
here (previously in `idmodels/config.py`), so that `DataSource` subclasses can
refer to it without importing from `idmodels` (which would create a circular
dependency). `idmodels` re-exports `SourceType`; callers should import
`Disease` directly from `iddata.enums`.

```python
class Disease(str, Enum):
    FLU   = "flu"
    COVID = "covid"
    RSV   = "rsv"


class AggLevel(str, Enum):
    """Aggregation level of a data source row."""
    NATIONAL = "national"
    STATE    = "state"
    HSA      = "hsa"
    SITE     = "site"
    REGION   = "region"


class SourceType(str, Enum):
    NHSN       = "nhsn"
    NSSP       = "nssp"
    FLUSURVNET = "flusurvnet"
    ILINET     = "ilinet"
```

### 1.3 `constants.py`

```python
# S3 storage
S3_BUCKET = "infectious-disease-data"
S3_DATA_RAW_URL = "https://infectious-disease-data.s3.amazonaws.com/data-raw/"

# Seasons excluded due to pandemic disruptions.
# Used with pandas .isin(); a tuple is sufficient.
PANDEMIC_SEASONS: tuple[str, ...] = ("2008/09", "2009/10", "2020/21", "2021/22")
```

Note: source-specific floor/scale constants (`NHSN_FLOOR`, `ILINET_FLOOR/SCALE`,
`FLUSURVNET_FLOOR/SCALE`) and in-season week bounds (10, 45) are modeling constants
that belong in `idmodels/constants.py`, not here.

### 1.4 `s3.py`

Extracts the repeated S3 file-selection pattern (currently duplicated in
`load_nhsn_from_hhs`, `load_nhsn_from_nhsn`, `load_nssp_from_cdc`):

```python
def get_versioned_file_path(glob_pattern: str, as_of: datetime.date) -> str:
    """
    Return the relative S3 path of the most recent file matching glob_pattern
    that is dated on or before as_of.

    Parameters
    ----------
    glob_pattern : str
        Full S3 glob, e.g.
        "infectious-disease-data/data-raw/influenza-nhsn/nhsn-????-??-??.csv"
    as_of : datetime.date
        Reference date. Returns the latest available file at or before this date.

    Returns
    -------
    str
        Relative path after "data-raw/", e.g. "influenza-nhsn/nhsn-2024-10-05.csv"

    Raises
    ------
    FileNotFoundError
        If no file exists at or before as_of.
    """
```

### 1.5 `DataSource` ABC (`sources/base.py`)

`source_name` now returns `SourceType` rather than `str`. Because `SourceType`
values match the string values written to the `source` DataFrame column
(`SourceType.NHSN.value == "nhsn"`), concrete implementations populate the
`source` column via `.value`. Passing a `SourceType` where a `str` is expected
(or vice versa) is now a type error.

```python
class DataSource(ABC):
    """
    Abstract base class for a single disease surveillance data source.

    Each concrete subclass encapsulates:
      - Which S3 files to read
      - Source-specific column mapping and cleaning
      - Any aggregation specific to that source (e.g., HSA → state for NSSP)

    All implementations return a DataFrame with the standard schema:
        location     (str):      FIPS code or other identifier
        agg_level    (AggLevel): aggregation level of the row
        wk_end_date  (datetime): Saturday end-of-week date
        season       (str):      e.g., "2023/24"
        season_week  (int):      weeks since week 30 of the prior year (1-based)
        inc          (float):    incidence in source-specific units
        source       (str):      source name (equals source_name.value)

    Testing note
    ------------
    Concrete subclasses should be unit-testable with small synthetic DataFrames
    (5–20 rows, 1–2 locations) and no live S3 or network access. Mock s3.py
    helpers at the module level to inject fixture data.
    """

    @property
    @abstractmethod
    def source_name(self) -> SourceType:
        """Returns the SourceType for this data source."""
        ...

    @abstractmethod
    def load(self, as_of: datetime.date | None = None) -> pd.DataFrame:
        """
        Load data available as of `as_of`.

        Parameters
        ----------
        as_of : datetime.date | None
            Reference date for versioned sources (NHSN, NSSP). Pass None or
            omit for sources that do not support versioned snapshots (ILINet,
            FluSurvNet), which always return current data. Versioned sources
            raise ValueError if as_of is None.

        Returns a DataFrame with the standard schema above.
        """
        ...
```

### 1.6 Concrete `DataSource` implementations

`source_name` is now a class-level `SourceType` attribute rather than a string
literal.

#### `NHSNDataSource`

```python
class NHSNDataSource(DataSource):
    source_name = SourceType.NHSN

    def __init__(
        self,
        disease: Disease,               # which disease to load
        rates: bool = True,             # return per-100k rates (True) or raw counts (False)
    ):
        ...

    def load(self, as_of: datetime.date | None = None) -> pd.DataFrame:
        """
        Load NHSN hospitalization data.

        Raises ValueError if as_of is None.
        Routes to the HHS archive for as_of < 2024-11-15, or the NHSN source
        for later dates. Returns inc in per-100k rates if rates=True.

        Note: the HHS archive path (as_of < 2024-11-15) only supports
        disease=Disease.FLU. Passing any other disease value on that path
        raises NotImplementedError.
        """
        ...
```

#### `NSSPDataSource`

```python
class NSSPDataSource(DataSource):
    source_name = SourceType.NSSP

    def __init__(
        self,
        disease: Disease,                         # which disease to load
        agg_level: AggLevel = AggLevel.STATE,     # STATE or HSA; controls output granularity
    ):
        """
        Parameters
        ----------
        agg_level : AggLevel
            The granularity at which data is returned. Different model
            configurations may request different levels from the same source
            (e.g., SARIXModel uses STATE, a future model may use HSA).
            This is a stable property of the source instance, not a per-load
            parameter.
        """
        ...

    def load(self, as_of: datetime.date | None = None) -> pd.DataFrame:
        """
        Load NSSP emergency department visit data.

        Raises ValueError if as_of is None.
        inc is in percentage units (0–100). Aggregates from HSA to state
        level when agg_level=AggLevel.STATE.
        """
        ...
```

#### `ILINetDataSource`

```python
class ILINetDataSource(DataSource):
    source_name = SourceType.ILINET

    def __init__(
        self,
        scale_to_positive: bool = True,           # scale ILI% by test-positivity rate
        agg_level: AggLevel = AggLevel.STATE,     # STATE, REGION, or NATIONAL
    ):
        """
        Parameters
        ----------
        agg_level : AggLevel
            The granularity at which data is returned. ILINet supports
            STATE, REGION, and NATIONAL.
        """
        ...

    def load(self, as_of: datetime.date | None = None) -> pd.DataFrame:
        """
        Load ILINet influenza-like illness data.

        as_of is accepted for interface consistency but always ignored —
        ILINet does not support versioned historical snapshots. The current
        data is always returned.
        """
        ...
```

#### `FluSurvNetDataSource`

```python
class FluSurvNetDataSource(DataSource):
    source_name = SourceType.FLUSURVNET

    def __init__(
        self,
        burden_adj: bool = True,                  # apply burden adjustment to hospitalization rates
        agg_level: AggLevel = AggLevel.STATE,     # STATE, SITE, or NATIONAL
    ):
        """
        Parameters
        ----------
        agg_level : AggLevel
            The granularity at which data is returned. FluSurv-NET supports
            STATE, SITE, and NATIONAL.
        """
        ...

    def load(self, as_of: datetime.date | None = None) -> pd.DataFrame:
        """
        Load FluSurv-NET hospitalization surveillance data.

        as_of is accepted for interface consistency but always ignored —
        FluSurv-NET does not support versioned historical snapshots.

        Note: burden adjustment uses a left join on season so pandemic seasons
        (which lack CDC burden estimates) produce NaN inc rather than being
        silently dropped from the output.
        """
        ...
```

### 1.7 `AncillaryData` ABC and `PopulationData` (`ancillary/`)

Population data (US Census) is used by models for `log_pop` features and for
converting NHSN rates to counts. It is not a surveillance data source, would
never serve as a training target, and its data format varies by implementation.
For these reasons it does not extend `DataSource`; instead a separate
`AncillaryData` base class is introduced.

#### `AncillaryData` ABC (`ancillary/base.py`)

```python
class AncillaryData(ABC):
    """
    Base class for supplementary data used by models but never as training targets.

    Unlike DataSource subclasses:
      - AncillaryData has no standard schema; format is implementation-defined.
      - Output format varies by implementation.
      - Implementations are not expected to be versioned (no as_of parameter).
    """

    @abstractmethod
    def load(self) -> pd.DataFrame:
        """
        Load and return the ancillary data.

        Returns a DataFrame whose columns are implementation-defined.
        DiseaseDataLoader.load() merges this into the surveillance DataFrame
        by location (left join).
        """
        ...
```

#### `PopulationData` (`ancillary/population.py`)

```python
class PopulationData(AncillaryData):
    """
    US Census population by location.

    Returns a DataFrame with columns:
        location  (str):   FIPS code or national identifier
        pop       (float): state or national population
        log_pop   (float): log(pop)

    Used by DiseaseDataLoader.load() to add pop and log_pop columns to
    the merged surveillance DataFrame. pop is also used by IDModel to
    convert NHSN per-100k rates to counts during inverse transform.
    """

    def load(self) -> pd.DataFrame:
        """Load population data from S3 and return as a DataFrame."""
        ...
```

### 1.8 `DiseaseDataLoader` (thin orchestrator)

`DiseaseDataLoader.load()` accepts an optional `ancillary` list. When provided,
each item is loaded and left-joined into the result DataFrame by location. If
`ancillary` is omitted, the output will not contain `pop` or `log_pop` columns.

```python
class DiseaseDataLoader:
    def load(
        self,
        sources: list[DataSource],
        as_of: datetime.date,
        ancillary: list[AncillaryData] | None = None,
        drop_pandemic_seasons: bool = True,
    ) -> pd.DataFrame:
        """
        Load and merge data from the specified sources, plus any ancillary data.

        Does NOT apply power transforms or center/scale normalization —
        those are modeling decisions belonging to idmodels.

        Parameters
        ----------
        sources : list[DataSource]
            Instantiated DataSource objects to load from.
        as_of : datetime.date
            Reference date passed to each source's load() method.
        ancillary : list[AncillaryData] | None
            Optional supplementary data to merge into the result. Each item's
            load() output is left-joined onto the merged surveillance DataFrame
            by location. Typically [PopulationData()] for models that need
            pop and log_pop columns.
        drop_pandemic_seasons : bool
            If True (default), set inc to NaN for pandemic seasons across all
            sources. Applied uniformly here rather than per-DataSource because
            it is a dataset-level concern. A warning is raised when
            drop_pandemic_seasons=False and NHSN (as_of < 2024-11-15) or
            FluSurvNet with burden_adj=True is present, since complete pandemic
            season data is unavailable in those cases regardless.

        Returns
        -------
        pd.DataFrame
            Merged data with the standard schema, plus any columns contributed
            by ancillary data (e.g., pop, log_pop when PopulationData is included).
        """
        ...
```

---

## 2. idmodels

### 2.1 New module structure

```
src/idmodels/
├── __init__.py
├── config.py             # updated: SourceType re-exported from iddata; ModelConfig, RunConfig, etc.
├── constants.py          # new: NHSN_FLOOR, ILINET_FLOOR/SCALE, FLUSURVNET_FLOOR/SCALE, POWER_TRANSFORM_OFFSET, IN_SEASON_WEEK_MIN/MAX
├── transforms.py         # new: Transform ABC; SourceScaleTransform, FourthRootTransform, IdentityTransform, CenterScaleTransform, ComposedTransform
├── features.py           # new: Feature ABC, concrete features, FeaturePipeline
├── model.py              # new: IDModel ABC
├── sarix.py              # updated: SARIXModel extends IDModel
├── gbqr.py               # updated: GBQRModel extends IDModel
├── spatial_utils.py      # unchanged
└── utils.py              # unchanged
```

### 2.2 `constants.py`

```python
# Source-specific floor and scale constants applied by SourceScaleTransform
# before the power transform. Floor shifts the distribution away from zero;
# scale adjusts the magnitude. Applied as (inc + floor) * scale, inverted after.
NHSN_FLOOR: float = 0.75 ** 4        # ≈ 0.316
ILINET_FLOOR: float = math.exp(-7)   # ≈ 0.0009
ILINET_SCALE: float = 4.0
FLUSURVNET_FLOOR: float = math.exp(-3)  # ≈ 0.050
FLUSURVNET_SCALE: float = 1 / 2.5    # = 0.4

# Small offset applied before power transform to avoid 0^0.25 = 0 issues.
POWER_TRANSFORM_OFFSET: float = 0.01

# Season weeks considered "in-season" for computing scale/center factors.
IN_SEASON_WEEK_MIN: int = 10
IN_SEASON_WEEK_MAX: int = 45
```

### 2.3 `config.py` changes

#### `SourceType` re-export

`SourceType` is no longer defined here; it is imported from `iddata.enums` and
re-exported so existing callers (`from idmodels.config import SourceType`)
continue to work:

```python
from iddata.enums import SourceType  # re-exported for callers
```

#### `Disease` — callers import directly

`Disease` is defined in `iddata.enums` and is not re-exported from
`idmodels.config`. Callers should import it directly:

```python
from iddata.enums import Disease
```

#### `ModelConfig.sources`

`ModelConfig.sources` declares which data sources a model uses as a list of
`SourceType` enum values. This is a static declaration; instantiation of actual
`DataSource` objects happens at run time in `_build_sources()` because some
constructor arguments (e.g., `disease`) are only known from `RunConfig`.

```python
@dataclass
class ModelConfig(ABC):
    model_name: str
    sources: list[SourceType]   # e.g., [SourceType.NHSN, SourceType.NSSP]
    fit_locations_separately: bool
    power_transform: PowerTransform
    # ... rest unchanged
```

Membership checks in model code:
```python
if SourceType.NHSN in self.model_config.sources:
    ...
```

`ModelConfig` has no dependency on `iddata` DataSource classes; it depends only
on `SourceType` (defined in `iddata.enums`, re-exported by `idmodels.config`).
Enum values are serializable and human-readable.

No other changes to the config dataclasses.

### 2.4 `transforms.py`

The transform pipeline replaces the `power_transform` + center/scale logic
currently split across `iddata/loader.py` and the inverse transform logic in
`sarix.py` and `gbqr.py`.

**Design: Stateless (factors stored as DataFrame columns only)**

Each `Transform` object holds no fitted state. `apply()` computes any
data-dependent parameters (e.g., per-location scale factors) from the input
DataFrame and writes them as additional columns, so they travel with the rows
through train/test splitting and are available to `invert()` at prediction time.

#### Testing note

Each concrete `Transform` subclass should be testable with a small synthetic
DataFrame (5–20 rows). `apply()` and `invert()` take and return plain
DataFrames/arrays with no external dependencies. A minimal test checks that
`invert(apply(df)["inc_trans_cs"].values, apply(df)) ≈ df["inc"]` for a range
of inputs.

```python
class Transform(ABC):
    @abstractmethod
    def apply(self, df: pd.DataFrame) -> pd.DataFrame:
        """
        Apply the forward transformation.

        For transforms with data-dependent parameters (like CenterScaleTransform),
        compute parameters from df and store them as additional columns so they
        are available to invert() later.
        """
        ...

    @abstractmethod
    def invert(self, values: np.ndarray, context: pd.DataFrame) -> np.ndarray:
        """
        Apply the inverse transformation.

        context is the transformed DataFrame, aligned with values. May contain
        factor columns that apply() wrote (e.g., inc_trans_scale_factor).
        """
        ...


class SourceScaleTransform(Transform):
    """
    Applies (inc + floor) * scale per source before the power transform; inverts after.

    Source-specific constants (NHSN_FLOOR, ILINET_FLOOR/SCALE, FLUSURVNET_FLOOR/SCALE)
    live in idmodels/constants.py. Sources absent from params pass through unchanged
    (floor=0, scale=1).

    params: dict mapping source name string to (floor, scale).
    """
    def __init__(self, params: dict[str, tuple[float, float]]):
        self.params = params

    def apply(self, df: pd.DataFrame) -> pd.DataFrame:
        floors = df["source"].map({k: v[0] for k, v in self.params.items()}).fillna(0.0)
        scales = df["source"].map({k: v[1] for k, v in self.params.items()}).fillna(1.0)
        df["inc"] = (df["inc"] + floors) * scales
        return df

    def invert(self, values: np.ndarray, context: pd.DataFrame) -> np.ndarray:
        floors = context["source"].map({k: v[0] for k, v in self.params.items()}).fillna(0.0).values
        scales = context["source"].map({k: v[1] for k, v in self.params.items()}).fillna(1.0).values
        return values / scales - floors


class FourthRootTransform(Transform):
    """f(x) = (x + offset)^0.25"""

    def __init__(self, offset: float = POWER_TRANSFORM_OFFSET):
        self.offset = offset

    def apply(self, df: pd.DataFrame) -> pd.DataFrame:
        df["inc_trans"] = (df["inc"] + self.offset) ** 0.25
        return df

    def invert(self, values: np.ndarray, context: pd.DataFrame) -> np.ndarray:
        return np.maximum(values, 0.0) ** 4 - self.offset


class IdentityTransform(Transform):
    """f(x) = x + offset (no power transform).

    Used when power_transform=PowerTransform.NONE. Keeps the same
    offset structure as FourthRootTransform for a consistent inverse.
    """

    def __init__(self, offset: float = POWER_TRANSFORM_OFFSET):
        self.offset = offset

    def apply(self, df: pd.DataFrame) -> pd.DataFrame:
        df["inc_trans"] = df["inc"] + self.offset
        return df

    def invert(self, values: np.ndarray, context: pd.DataFrame) -> np.ndarray:
        return np.maximum(values, 0.0) - self.offset


class CenterScaleTransform(Transform):
    """
    Scales by in-season 95th-percentile, then centers by in-season mean,
    per (source, location).

    Computes factors from the input DataFrame and writes them as columns
    so they are available at invert time.

    Output column: inc_trans_cs
    Factor columns written to df: inc_trans_scale_factor, inc_trans_center_factor
    """
    def __init__(
        self,
        in_season_week_min: int = IN_SEASON_WEEK_MIN,  # 10
        in_season_week_max: int = IN_SEASON_WEEK_MAX,  # 45
    ):
        self.in_season_week_min = in_season_week_min
        self.in_season_week_max = in_season_week_max

    def apply(self, df: pd.DataFrame) -> pd.DataFrame:
        """Compute factors from df, write as columns, apply transform."""
        df["inc_trans_scale_factor"] = (
            df.assign(
                inc_trans_in_season=lambda x: np.where(
                    (x["season_week"] < self.in_season_week_min) |
                    (x["season_week"] > self.in_season_week_max),
                    np.nan, x["inc_trans"]
                )
            )
            .groupby(["source", "location"])["inc_trans_in_season"]
            .transform(lambda x: x.quantile(0.95))
        )
        df["inc_trans_cs"] = df["inc_trans"] / (df["inc_trans_scale_factor"] + 0.01)
        df["inc_trans_center_factor"] = (
            df.assign(
                inc_trans_cs_in_season=lambda x: np.where(
                    (x["season_week"] < self.in_season_week_min) |
                    (x["season_week"] > self.in_season_week_max),
                    np.nan, x["inc_trans_cs"]
                )
            )
            .groupby(["source", "location"])["inc_trans_cs_in_season"]
            .transform("mean")
        )
        df["inc_trans_cs"] = df["inc_trans_cs"] - df["inc_trans_center_factor"]
        return df

    def invert(self, values: np.ndarray, context: pd.DataFrame) -> np.ndarray:
        return (
            (values + context["inc_trans_center_factor"].values)
            * (context["inc_trans_scale_factor"].values + 0.01)
        )


class ComposedTransform(Transform):
    """
    Chains multiple transforms in sequence.

    apply() applies each sub-transform in order; each sees the df as modified
    by the previous step.
    invert() applies inverses in reverse order.
    """
    def __init__(self, transforms: list[Transform]):
        self.transforms = transforms

    def apply(self, df: pd.DataFrame) -> pd.DataFrame:
        for t in self.transforms:
            df = t.apply(df)
        return df

    def invert(self, values: np.ndarray, context: pd.DataFrame) -> np.ndarray:
        for t in reversed(self.transforms):
            values = t.invert(values, context)
        return values
```

### 2.5 `features.py`

Replaces `preprocess.py`. The `create_features_and_targets()` and
`create_directional_wave_features()` functions become `Feature` subclasses.

#### Testing note

Each concrete `Feature` subclass should be testable with a small synthetic
DataFrame (5–20 rows, 1–2 locations). `apply()` takes a plain DataFrame and
returns `(DataFrame, list[str])` — no S3 access, network calls, or external
state required. A minimal test checks:

- Expected columns appear in the returned df.
- Expected column names are added to (or removed from) `feat_names`.
- NaN behavior at group boundaries matches the documented behavior.

```python
class Feature(ABC):
    """
    A single feature-engineering step applied to a DataFrame.

    Each Feature adds columns to df and/or updates the feat_names list.
    Features are composed into a FeaturePipeline.
    """

    @abstractmethod
    def apply(
        self,
        df: pd.DataFrame,
        feat_names: list[str],
    ) -> tuple[pd.DataFrame, list[str]]:
        """
        Augment df with new feature columns.

        Parameters
        ----------
        df : pd.DataFrame
            Input data, grouped by ["source", "location"] and sorted by
            ["source", "location", "wk_end_date"].
        feat_names : list[str]
            Running list of active feature column names.

        Returns
        -------
        tuple of (augmented df, updated feat_names)
        """
        ...


class OneHotEncodingFeature(Feature):
    """One-hot encode categorical columns."""

    def __init__(self, columns: list[str]):
        self.columns = columns

    def apply(
        self, df: pd.DataFrame, feat_names: list[str]
    ) -> tuple[pd.DataFrame, list[str]]:
        ...


class HolidayFeature(Feature):
    """
    Adds delta_xmas: signed distance in season weeks from Christmas week.
    (e.g., delta_xmas = -2 means 2 weeks before Christmas)

    Also adds xmas_spike = max(3 - |delta_xmas|, 0) as a covariate,
    for use by SARIX models.
    """

    def apply(
        self, df: pd.DataFrame, feat_names: list[str]
    ) -> tuple[pd.DataFrame, list[str]]:
        ...
        # new columns: delta_xmas, xmas_spike
        # adds "delta_xmas" to feat_names (xmas_spike is a covariate, not a feature)


class TaylorFeature(Feature):
    """Windowed Taylor polynomial coefficients via timeseriesutils."""

    def __init__(
        self,
        column: str,
        degree: int,
        window_sizes: list[int],
        window_align: str = "trailing",
        fill_edges: bool = False,
    ):
        self.column = column
        self.degree = degree
        self.window_sizes = window_sizes
        self.window_align = window_align
        self.fill_edges = fill_edges

    def apply(
        self, df: pd.DataFrame, feat_names: list[str]
    ) -> tuple[pd.DataFrame, list[str]]:
        ...


class RollingMeanFeature(Feature):
    """
    Rolling mean over specified window sizes.

    Parameters
    ----------
    column : str
        Column to compute rolling means over.
    window_sizes : list[int]
        Window sizes in weeks.
    group_columns : list[str]
        Columns to group by when computing the rolling mean. Defaults to
        ["location"], matching the existing preprocess.py behavior where
        rolling means are computed within each location independently of
        source. Pass ["source", "location"] to roll within source-location
        pairs instead.
    """

    def __init__(
        self,
        column: str,
        window_sizes: list[int],
        group_columns: list[str] | None = None,
    ):
        self.column = column
        self.window_sizes = window_sizes
        self.group_columns = group_columns if group_columns is not None else ["location"]

    def apply(
        self, df: pd.DataFrame, feat_names: list[str]
    ) -> tuple[pd.DataFrame, list[str]]:
        ...


class LagFeature(Feature):
    """
    Create lagged versions of specified columns.

    When columns=None, FeaturePipeline resolves this to all feature columns
    accumulated since the previous LagFeature step (or since the pipeline
    start if no prior LagFeature has run). See FeaturePipeline.apply() for
    the accumulation semantics.

    NaN behavior
    ------------
    Lags are computed within each (source, location) group via pandas
    shift(). The first `lag` rows of each group will be NaN for the
    corresponding lag column (e.g., lag=2 → first 2 rows NaN). This
    matches the existing preprocess.py behavior; downstream code handles
    these NaN rows by dropping them or relying on them falling outside
    the train/test window.
    """

    def __init__(self, columns: list[str] | None, lags: list[int]):
        self.columns = columns  # None means "lag previous segment's accumulated columns"
        self.lags = lags

    def apply(
        self, df: pd.DataFrame, feat_names: list[str]
    ) -> tuple[pd.DataFrame, list[str]]:
        ...


class HorizonTargetFeature(Feature):
    """
    Create forecast target columns for each horizon (1 to max_horizon).

    Uses the "long" layout: the DataFrame is expanded so that each original
    row appears max_horizon times (once per horizon value). New columns:

        inc_trans_cs_target  (float): value of `column` h weeks in the future
                                      (NaN for the last h rows of each group)
        horizon              (int):   forecast horizon (1, 2, ..., max_horizon)
        delta_target         (float): inc_trans_cs_target - inc_trans_cs

    Adds "horizon" to feat_names. inc_trans_cs_target and delta_target are
    training targets, not features, and are not added to feat_names.

    The GBQR model targets delta_target (the change from the current value),
    not the absolute value of inc_trans_cs_target.
    """

    def __init__(self, column: str, max_horizon: int):
        self.column = column
        self.max_horizon = max_horizon

    def apply(
        self, df: pd.DataFrame, feat_names: list[str]
    ) -> tuple[pd.DataFrame, list[str]]:
        ...


class LevelFeatureFilter(Feature):
    """
    Remove absolute-level features from feat_names (does not drop df columns).

    Removes: inc_trans_cs, its lags, Taylor c0 (constant) terms,
    and rolling mean features. Used when incl_level_feats=False.
    """

    def apply(
        self, df: pd.DataFrame, feat_names: list[str]
    ) -> tuple[pd.DataFrame, list[str]]:
        ...


class DirectionalWaveFeature(Feature):
    """
    Spatial wave propagation features: inverse-distance-weighted average of
    neighboring states' incidence in each compass direction.
    """

    def __init__(
        self,
        directions: list[str],        # e.g., ["N", "NE", "E", "SE", "S", "SW", "W", "NW"]
        temporal_lags: list[int],     # e.g., [1, 2]
        max_distance_km: float,       # neighbor distance cutoff
        include_velocity: bool = False,
        include_aggregate: bool = True,
    ):
        self.directions = directions
        self.temporal_lags = temporal_lags
        self.max_distance_km = max_distance_km
        self.include_velocity = include_velocity
        self.include_aggregate = include_aggregate

    def apply(
        self, df: pd.DataFrame, feat_names: list[str]
    ) -> tuple[pd.DataFrame, list[str]]:
        ...


class FeaturePipeline:
    """
    Applies a sequence of Feature steps to a DataFrame.

    Column accumulation for LagFeature(columns=None)
    -------------------------------------------------
    FeaturePipeline tracks which columns have been added by Feature steps
    since the last LagFeature (or since the pipeline start if no LagFeature
    has run yet). When a LagFeature with columns=None is encountered, its
    columns are resolved to this accumulated set. After every LagFeature
    step — whether it used explicit columns or columns=None — the accumulator
    resets to empty.

    This means that if a pipeline contains two LagFeature steps, each
    columns=None resolution sees only columns produced between the two lag
    steps, not columns from before the first lag.

    Note: initial_feat_names columns (e.g., "inc_trans_cs", "season_week")
    are NOT included in the accumulator; they are part of the pipeline's
    starting state, not added by any Feature step. Columns from initial
    feat_names that should be lagged must be specified explicitly in a
    LagFeature(columns=[...]) step.
    """

    def __init__(
        self,
        features: list[Feature],
        initial_feat_names: list[str] | None = None,
    ):
        self.features = features
        self.initial_feat_names = initial_feat_names or []

    def apply(self, df: pd.DataFrame) -> tuple[pd.DataFrame, list[str]]:
        """
        Apply all Feature steps in sequence.

        Returns
        -------
        tuple of (augmented DataFrame, final list of feature names)
        """
        feat_names = list(self.initial_feat_names)
        # Accumulates columns added by Feature steps since the last LagFeature.
        # Resets to [] after every LagFeature step (explicit or columns=None).
        accumulated_new: list[str] = []

        for feature in self.features:
            feat_names_before = list(feat_names)

            if isinstance(feature, LagFeature) and feature.columns is None:
                # Resolve columns=None to all columns accumulated since the
                # last LagFeature step (or since pipeline start).
                feature = LagFeature(columns=list(accumulated_new), lags=feature.lags)

            df, feat_names = feature.apply(df, feat_names)
            new_this_step = [f for f in feat_names if f not in feat_names_before]

            if isinstance(feature, LagFeature):
                # Reset accumulator after every LagFeature so the next
                # columns=None resolution starts fresh.
                accumulated_new = []
            else:
                accumulated_new.extend(new_this_step)

        return df, feat_names
```

### 2.6 `model.py` — `IDModel` base class

```python
class IDModel(ABC):
    """
    Abstract base class for infectious disease forecast models.

    Implements the shared run() workflow. Subclasses provide model-specific
    behavior via three abstract methods:
      - _build_sources()           which DataSource objects to load
      - _build_feature_pipeline()  which FeaturePipeline to apply
      - _fit_and_predict()         model fitting and quantile prediction

    _build_transform() has a concrete default implementation that is suitable
    for both SARIXModel and GBQRModel. Subclasses may override it if needed.
    """

    def __init__(self, model_config: ModelConfig):
        self.model_config = model_config

    def run(self, run_config: RunConfig) -> None:
        """
        Load data, generate predictions, and save to file.

        Subclasses should not override this method. Customize behavior
        via the abstract methods below.
        """
        # 1. Load raw data
        sources = self._build_sources(run_config)
        df = DiseaseDataLoader().load(
            sources=sources,
            as_of=run_config.ref_date,
            ancillary=[PopulationData()],
        )

        # 2. Filter to requested locations
        df = self._filter_locations(df, run_config)
        df["unique_id"] = df["agg_level"] + df["location"]

        # 3. Apply transform (power + center/scale)
        transform = self._build_transform()
        df = transform.apply(df)

        # 4. Build and apply feature pipeline
        pipeline = self._build_feature_pipeline(run_config)
        df, feat_names = pipeline.apply(df)

        # 5. Fit model and generate quantile predictions on transformed scale
        preds_df = self._fit_and_predict(df, feat_names, run_config)

        # 6. Inverse transform predictions to original (inc) scale
        preds_df = self._invert_and_scale(preds_df, transform, run_config)

        # 7. Format for hub submission and save
        preds_df = self._format_output(preds_df, run_config)
        save_path = build_save_path(
            root=run_config.output_root,
            run_config=run_config,
            model_config=self.model_config
        )
        preds_df["output_type_id"] = preds_df["output_type_id"].astype(str)
        preds_df.to_csv(save_path, index=False)

    # ------------------------------------------------------------------ #
    # Abstract methods — subclasses must implement                        #
    # ------------------------------------------------------------------ #

    @abstractmethod
    def _build_sources(self, run_config: RunConfig) -> list[DataSource]:
        """
        Instantiate iddata DataSource objects for this model.

        Uses self.model_config.sources (SourceType enum values) and
        run_config (for disease, ref_date) to construct the right objects.
        """
        ...

    @abstractmethod
    def _build_feature_pipeline(self, run_config: RunConfig) -> FeaturePipeline:
        """Return the FeaturePipeline for this model."""
        ...

    @abstractmethod
    def _fit_and_predict(
        self,
        df: pd.DataFrame,
        feat_names: list[str],
        run_config: RunConfig,
    ) -> pd.DataFrame:
        """
        Fit model and generate quantile predictions.

        Parameters
        ----------
        df : pd.DataFrame
            Fully transformed and featurized data. Contains both
            historical rows (for training) and the most recent week
            (test set for predictions).
        feat_names : list[str]
            Names of feature columns to use as model inputs.
        run_config : RunConfig

        Returns
        -------
        pd.DataFrame
            Long-format predictions with columns:
                source, agg_level, location, wk_end_date, pop, horizon,
                inc_trans_cs, inc_trans_center_factor, inc_trans_scale_factor,
                quantile (output_type_id label), value (in transformed space)
        """
        ...

    # ------------------------------------------------------------------ #
    # Method with default implementation — subclasses may override        #
    # ------------------------------------------------------------------ #

    def _build_transform(self) -> Transform:
        """
        Default transform: ComposedTransform([SourceScaleTransform, power_transform, CenterScaleTransform()]).

        Both SARIXModel and GBQRModel use this default unchanged. Subclasses
        may override for a different transform.
        """
        _SOURCE_SCALE_PARAMS = {
            SourceType.NHSN:       (NHSN_FLOOR, 1.0),
            SourceType.ILINET:     (ILINET_FLOOR, ILINET_SCALE),
            SourceType.FLUSURVNET: (FLUSURVNET_FLOOR, FLUSURVNET_SCALE),
        }
        source_scale_params = {
            src.value: params
            for src, params in _SOURCE_SCALE_PARAMS.items()
            if src in self.model_config.sources
        }
        if self.model_config.power_transform == PowerTransform.FOURTH_ROOT:
            power_t: Transform = FourthRootTransform()
        else:
            power_t = IdentityTransform()

        return ComposedTransform([SourceScaleTransform(source_scale_params), power_t, CenterScaleTransform()])

    # ------------------------------------------------------------------ #
    # Shared methods — not overridden by subclasses                       #
    # ------------------------------------------------------------------ #

    def _filter_locations(
        self, df: pd.DataFrame, run_config: RunConfig
    ) -> pd.DataFrame:
        """Filter to states and HSAs requested in run_config."""
        df_states = df.loc[
            (df["location"].isin(run_config.states)) & (df["agg_level"] != "hsa")
        ]
        df_hsas = df.loc[
            (df["location"].isin(run_config.hsas)) & (df["agg_level"] == "hsa")
        ]
        return pd.concat([df_states, df_hsas], join="inner", axis=0)

    def _invert_and_scale(
        self,
        preds_df: pd.DataFrame,
        transform: Transform,
        run_config: RunConfig,
    ) -> pd.DataFrame:
        """
        Apply inverse transform to predictions, then convert to original units.

        For NHSN: multiply per-100k rate by population / 100000 → counts.
        For NSSP: divide percentage by 100 → proportion, clipped to [0, 1].
        """
        preds_df["value"] = transform.invert(
            preds_df["value"].values, context=preds_df
        )
        preds_df["value"] = np.maximum(preds_df["value"], 0.0)

        if SourceType.NHSN in self.model_config.sources:
            preds_df["value"] = preds_df["value"] * preds_df["pop"] / 100000
        elif SourceType.NSSP in self.model_config.sources:
            preds_df["value"] = np.minimum(preds_df["value"] / 100, 1.0)

        return preds_df

    def _format_output(
        self, preds_df: pd.DataFrame, run_config: RunConfig
    ) -> pd.DataFrame:
        """
        Reshape predictions to FluSight hub submission format.

        Output columns:
            location, reference_date, horizon, target_end_date,
            target, output_type, output_type_id, value
        """
        ...
```

### 2.7 Updated `SARIXModel`

```python
class SARIXModel(IDModel):
    """SARIX (Bayesian ARIMA with exogenous covariates) forecast model."""

    def _build_sources(self, run_config: RunConfig) -> list[DataSource]:
        sources_map = {
            SourceType.NHSN: NHSNDataSource(disease=run_config.disease),
            SourceType.NSSP: NSSPDataSource(disease=run_config.disease),
        }
        if not set(self.model_config.sources) <= sources_map.keys():
            raise ValueError("SARIXModel only supports NHSN and NSSP sources.")
        if SourceType.NHSN in self.model_config.sources and \
           SourceType.NSSP in self.model_config.sources:
            raise ValueError("Only one of NHSN or NSSP may be selected.")
        return [sources_map[s] for s in self.model_config.sources]

    def _build_feature_pipeline(self, run_config: RunConfig) -> FeaturePipeline:
        return FeaturePipeline(
            features=[HolidayFeature()],
            initial_feat_names=["inc_trans_cs"] + self.model_config.x,
        )
        # Note: xmas_spike (added by HolidayFeature) is used as a covariate
        # passed to SARIX, not as a feat_name. The _fit_and_predict method
        # reads it directly from df.

    def _fit_and_predict(
        self,
        df: pd.DataFrame,
        feat_names: list[str],
        run_config: RunConfig,
    ) -> pd.DataFrame:
        """Fit SARIX and return quantile predictions in long format."""
        # filter date range, interpolate, reshape for SARIX, fit, extract quantiles
        # (existing logic, substantially unchanged)
        ...

    def _get_extra_sarix_params(self, df: pd.DataFrame) -> dict:
        """Hook for subclasses. Returns {} by default."""
        return {}


class SARIXFourierModel(SARIXModel):
    """Extends SARIXModel with Fourier seasonality terms."""

    def __init__(self, model_config: SARIXFourierModelConfig):
        if not isinstance(model_config, SARIXFourierModelConfig):
            raise TypeError(
                f"SARIXFourierModel requires SARIXFourierModelConfig, "
                f"got {type(model_config).__name__}"
            )
        super().__init__(model_config)

    def _get_extra_sarix_params(self, df: pd.DataFrame) -> dict:
        day_of_year = (
            df.groupby("location")["wk_end_date"]
            .apply(lambda x: x.dt.dayofyear.values)
            .iloc[0]
        )
        return {
            "day_of_year": day_of_year,
            "fourier_K": self.model_config.fourier_K,
            "fourier_pooling": self.model_config.fourier_pooling,
        }
```

### 2.8 Updated `GBQRModel`

```python
class GBQRModel(IDModel):
    """Gradient Boosted Quantile Regression forecast model."""

    def _build_sources(self, run_config: RunConfig) -> list[DataSource]:
        source_map = {
            SourceType.NHSN:      NHSNDataSource(disease=run_config.disease),
            SourceType.NSSP:      NSSPDataSource(disease=run_config.disease),
            SourceType.ILINET:    ILINetDataSource(
                scale_to_positive=self.model_config.reporting_adj
            ),
            SourceType.FLUSURVNET: FluSurvNetDataSource(
                burden_adj=self.model_config.reporting_adj
            ),
        }
        if SourceType.NHSN in self.model_config.sources and \
           SourceType.NSSP in self.model_config.sources:
            raise ValueError("Only one of NHSN or NSSP may be selected.")
        return [source_map[s] for s in self.model_config.sources]

    def _build_feature_pipeline(self, run_config: RunConfig) -> FeaturePipeline:
        if run_config.disease in (Disease.FLU, Disease.RSV):
            initial_feats = ["inc_trans_cs", "season_week", "log_pop"]
        else:
            initial_feats = ["inc_trans_cs", "log_pop"]

        features: list[Feature] = []

        if self.model_config.use_directional_waves:
            features.append(DirectionalWaveFeature(
                directions=self.model_config.wave_directions,
                temporal_lags=self.model_config.wave_temporal_lags,
                max_distance_km=self.model_config.wave_max_distance_km,
                include_velocity=self.model_config.wave_include_velocity,
                include_aggregate=self.model_config.wave_include_aggregate,
            ))

        features += [
            OneHotEncodingFeature(columns=["source", "agg_level", "location"]),
            HolidayFeature(),
            # Lag the base column explicitly. inc_trans_cs is in initial_feat_names
            # (not produced by a pipeline step), so it is not captured by a
            # subsequent LagFeature(columns=None).
            LagFeature(columns=["inc_trans_cs"], lags=[1, 2]),
            TaylorFeature(column="inc_trans_cs", degree=2, window_sizes=[4, 6]),
            TaylorFeature(column="inc_trans_cs", degree=1, window_sizes=[3, 5]),
            RollingMeanFeature(column="inc_trans_cs", window_sizes=[2, 4]),
            # columns=None: FeaturePipeline resolves to all columns accumulated
            # since the previous LagFeature — i.e., all Taylor and rolling mean
            # columns above. The accumulator resets after each LagFeature step.
            LagFeature(columns=None, lags=[1, 2]),
            HorizonTargetFeature(column="inc_trans_cs", max_horizon=run_config.max_horizon),
        ]

        if not self.model_config.incl_level_feats:
            features.append(LevelFeatureFilter())

        return FeaturePipeline(features=features, initial_feat_names=initial_feats)

    def _fit_and_predict(
        self,
        df: pd.DataFrame,
        feat_names: list[str],
        run_config: RunConfig,
    ) -> pd.DataFrame:
        """Bag-of-quantile-regression models; returns long-format predictions."""
        # filter to in-season, split train/test, fit LightGBM, return predictions
        # (existing logic, substantially unchanged)
        ...
```

---

## 3. operational-models

Standardize all five models on the **direct instantiation pattern** (currently
used by `flu_ar2`). Eliminate intermediate subprocess scripts (`0_ar6_pooled.py`,
`0_gbqr.py`, `1_gbqr.py`) and move their config inline into `main.py`.

Each `main.py` has the same shape:

```python
from iddata.enums import Disease
from idmodels.config import SourceType, RunConfig
# ...

@click.command()
@click.option("--today_date", type=str, required=False)
@click.option("--short_run", is_flag=True)
def main(today_date: str | None, short_run: bool) -> None:
    reference_date = ...  # next Saturday from today_date

    model_config = <ModelConfig subclass>(
        model_name=...,
        sources=[SourceType.NHSN],
        ...
    )
    run_config = RunConfig(
        disease=Disease.FLU,
        ref_date=reference_date,
        ...
    )

    if short_run:
        run_config.q_levels = [0.025, 0.1, 0.25, 0.5, 0.75, 0.9, 0.975]
        run_config.q_labels = ["0.025", "0.1", "0.25", "0.5", "0.75", "0.9", "0.975"]
        model_config.num_warmup = 100    # SARIX only
        model_config.num_samples = 100   # SARIX only
        model_config.num_bags = 10       # GBQR only

    model = <Model class>(model_config)
    model.run(run_config)

    subprocess.run(["Rscript", "plot.R", str(reference_date)], check=True)
```

`flu_flusion` retains its R ensemble step as a subprocess call after the two
Python models have run. `flu_trends_ensemble` remains R-only and is unchanged.

---

## 4. Open items / decisions needed

All previously open items are resolved. No open items remain.

1. ~~**Transform option**~~ — Resolved: Option B (stateless `apply()`/`invert()`)
   adopted (§2.4).

2. ~~**`PopulationDataSource`**~~ — Resolved: population data is handled via the
   new `AncillaryData` / `PopulationData` design (§1.7, §1.8).

---

I'm having trouble adding more comments to my review, so I'm submitting it as-is with some overall/remaining thoughts:
- A lot of inline comments and function descriptions were removed, perhaps in an attempt to cleanup the code. Personally, I'd like to keep them since the comments improve the human readability of the code
- Claude has removed various lines of code throughout idmodels in its refactor of the codebase, particularly in areas where it made large changes like the directional wave features. I'd like to know if the codebase is still doing the same thing as the original code
- I'd like to chat with Claude to determine if the tests it added for the new features and transforms classes are sufficient