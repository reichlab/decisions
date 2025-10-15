# 2025-06-17 RFC Time-series target data schema metadata

## Context

As we expand validation tests for target data, we've identified the need for a structured metadata schema to support **deterministic schema validation** for time-series datasets.

Currently, the inclusion of non–task ID columns in time-series data requires opening the actual dataset to infer these columns and their types—an approach that introduces several limitations:

1. **Unstable reference schema**: We currently default to using the schema from the first file in a multi-file dataset. This is fragile, especially if the first file is updated post hoc. It also introduces performance costs since the dataset must be opened to extract the schema, which can involve cloud access calls ([relevant issue](https://github.com/hubverse-org/hubData/issues/87#issuecomment-2901055075)).

2. **Partition column conflicts**: Some hubs (e.g. the nowcast hub) are partitioned on variables that are also present in the dataset itself—such as a `date` column. While the Parquet files correctly store this column as a `Date`, Arrow defaults to reading all partition variables as `character` unless an explicit schema is provided. This causes a conflict during dataset loading and prevents it from being opened without a defined schema ([see issue](https://github.com/hubverse-org/hubData/issues/89)).

To address these problems, we propose introducing new metadata fields that allow hub admins to explicitly define the schema for all non–task ID columns. This will enable deterministic construction of a time-series schema without inspecting dataset contents.

In addition, hubs need a way to define the **observable unit** of a dataset—the set of columns whose values uniquely identify a single observation at a point in time. This is essential for verifying data integrity and preventing duplicates, especially in versioned or oracle-output data ([see RFC discussion](https://github.com/reichlab/decisions/blob/main/decisions/2025-02-27-rfc-time-series-target-data.md#validations)).

## Aims

* Define a metadata structure, validation schema, and rules for deterministic schema creation for time-series target data.
* Support explicit definition of task IDs and other columns that form the observable unit.

## Anti-Aims

* This RFC does not address schema management outside of target data.

## Decision

We will adopt a new configuration file named `target-data.json` to define the schema for time-series target data. This file will:

* List columns that correspond to task IDs (whose data types are already defined via `tasks.json`);
* List non–task ID columns along with their data types.

## Summary of `target-data.json` Structure

The `target-data.json` file contains top-level properties that describe expectations across target datasets, with the ability to override these defaults for specific dataset types.

### Configuration Hierarchy

Properties can be set at two levels:

1. **Global (top-level)**: Default values that apply to all target dataset types
2. **Dataset-specific** (`time-series`, `oracle-output`): Override global defaults when needed

When a property is not specified at the dataset level (or is set to `null`), the global value is used.

### Top-Level Properties

* `observable_unit`: An array of column names whose unique value combinations define the minimum observable unit across all target datasets. Must only include the `date_col`, `target_col` (if present), and any other task ID columns. If versioned, unique combinations will also take into account the values in the `as_of` column, but `as_of` is never included in the observable unit itself as it is a versioning column, not a task ID. This property is required.

* `date_col`: The date column name used across time-series, oracle-output, and model-output datasets. This column stores the date on which observed data actually occurred. Expected to be of type `Date`. This property is required.

* `versioned`: Boolean indicating whether all target type datasets use `as_of` versioning by default. If `true`, datasets are expected to have a date `as_of` column indicating the version of each data point. Defaults to `false`. Can be overridden at the dataset level.

### Reserved Columns

The following columns have special meanings and predefined behavior:

* `as_of`: A date column used for versioning. When present, it indicates the version or snapshot date of each data point. Its presence is controlled by the `versioned` property. Always expected to be of type `Date` and does not need to be defined in `non_task_id_schema`.
* `output_type` and `output_type_id`: Columns used to store output type information in oracle-output data when `has_output_type_ids` is `true`.

### Target-Type Specific Configuration

#### `time-series`

* `non_task_id_schema`: Optional. Key-value pairs of non-task ID column names and their R data types, one of (`character`, `double`, `integer`, `logical`, `Date`). Include any columns in the time-series data that do not correspond exactly to a task ID. The `as_of` column does not need to be defined here as it is a reserved column.

* `observable_unit`: Optional. Names of columns whose unique value combinations define the minimum observable unit for time-series data. Use to override the global `observable_unit` when time-series requires a different set of columns. If not specified or set to `null`, uses the global `observable_unit`.

* `versioned`: Optional. Boolean indicating whether time-series data are versioned using `as_of` dates. Use to override the global `versioned` setting. If not specified, inherits from the global `versioned` property.

#### `oracle-output`

* `has_output_type_ids`: Boolean. Must be `true` if `pmf` or `cdf` output types exist. Can be `false` otherwise. If `true`, the dataset must include `output_type` and `output_type_id` columns. Defaults to `false`.

* `observable_unit`: Optional. Names of task IDs whose unique value combinations define an observable unit in oracle-output data. Each combination of values must be unique once combined with output type IDs if present. Use to override the global `observable_unit` in situations where some output types require additional task ID values to map onto target data (e.g., when `pmf` output type [functionally requires horizon](https://github.com/reichlab/flusight-dashboard/issues/20#issuecomment-2815550603)). If not specified or set to `null`, uses the global `observable_unit`.

* `versioned`: Optional. Boolean indicating whether oracle-output data are versioned using `as_of` dates. Use to override the global `versioned` setting. If not specified, inherits from the global `versioned` property. Note that oracle-output data is expected to have only a single version of each unique combination of observable unit values, in contrast to time-series which is allowed to have multiple versions. This is to minimize confusion and reduce the risk of downloading multiple observed values and scoring on each of them.

### `target-data-schema.json`

```json
{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "https://raw.githubusercontent.com/hubverse-org/schemas/main/v6.0.0/target-data-schema.json",
    "title": "Schema for Modeling Hub target data definitions",
    "description": "This is the schema of the target-data.json configuration file that defines metadata about target data used to visualise and evaluate modeling hub model outputs.",
    "type": "object",
    "properties": {
        "schema_version": {
            "description": "URL to a version of the Modeling Hub schema target-data-schema.json file (see https://github.com/hubverse-org/schemas). Used to declare the schema version a 'target-data.json' file is written for and for config file validation. The URL provided should be the URL to the raw content of the schema file on GitHub.",
            "examples": [
                "https://raw.githubusercontent.com/hubverse-org/schemas/main/v6.0.0/target-data-schema.json"
            ],
            "type": "string",
            "format": "uri"
        },
        "observable_unit": {
            "description": "Names of columns whose unique value combinations define the minimum observable unit across all target type data. Each combination of values must be unique (and in time-series data also unique across `as_of` data versions if applicable). The majority are expected to correspond to task ID names but may include other columns as well (e.g., the `date_col` column).",
            "type": "array",
            "uniqueItems": true,
            "items": {
                "type": "string"
            }
        },
        "date_col": {
            "description": "Name of the date column across hub data (time-series, oracle-output and ideally model-output). This is the column that stores the date on which observed data actually occurred.",
            "type": "string"
        },
        "versioned": {
            "description": "Indicates whether all target type datasets are versioned using `as_of` dates by default. If true, both time-series and oracle-output data are expected to have a date `as_of` column that indicates the version of each data point. Can be overridden at the dataset level.",
            "type": "boolean",
            "default": false
        },
        "time-series": {
            "type": "object",
            "properties": {
                "non_task_id_schema": {
                    "type": "object",
                    "description": "Key-value pairs of non-task ID column names and data types found in time-series data. Include any columns in the time-series data that do not correspond exactly to a task ID. The `as_of` column does not need to be defined here as it is a reserved column.",
                    "examples": [
                        {
                            "location_name": "character"
                        },
                        {
                            "population": "integer"
                        }
                    ],
                    "additionalProperties": {
                        "type": "string",
                        "enum": [
                            "character",
                            "double",
                            "integer",
                            "logical",
                            "Date"
                        ]
                    }
                },
                "observable_unit": {
                    "description": "Names of columns whose unique value combinations define the minimum observable unit for time-series data. Each combination of values must be unique across `as_of` data versions if applicable. The majority are expected to correspond to task ID names but may include other columns as well (e.g., the `date_col` column). If not specified or null, uses the global `observable_unit`.",
                    "type": [
                        "array",
                        "null"
                    ],
                    "uniqueItems": true,
                    "items": {
                        "type": "string"
                    },
                    "default": null
                },
                "versioned": {
                    "description": "Indicates whether time-series data are versioned using `as_of` dates. If true, the data is expected to have a date `as_of` column that indicates the version of each data point. If not specified, inherits from the global `versioned` setting.",
                    "type": "boolean"
                }
            },
            "additionalProperties": false
        },
        "oracle-output": {
            "type": "object",
            "properties": {
                "has_output_type_ids": {
                    "type": "boolean",
                    "description": "Indicates whether the oracle-output data have an `output_type` and `output_type_id` column. These columns are necessary if hub includes `pmf` and `cdf` output types but optional otherwise.",
                    "default": false
                },
                "observable_unit": {
                    "description": "Names of task IDs whose unique value combinations define an observable unit in oracle-output data. Each combination of values must be unique once combined with output type IDs if present. Use to override the global `observable_unit` in situations where some output types require additional task ID values to map onto target data. If not specified or null, uses the global `observable_unit`.",
                    "type": [
                        "array",
                        "null"
                    ],
                    "uniqueItems": true,
                    "items": {
                        "type": "string"
                    },
                    "default": null
                },
                "versioned": {
                    "description": "Indicates whether oracle-output data are versioned using `as_of` dates. If true, the data is expected to have a date `as_of` column that indicates the version of each data point. If not specified, inherits from the global `versioned` setting.",
                    "type": "boolean"
                }
            },
            "additionalProperties": false
        },
        "additional_metadata": {
            "description": "Optional property in which any type of custom metadata can be stored.",
            "type": "object",
            "additionalProperties": true
        }
    },
    "required": [
        "schema_version",
        "observable_unit",
        "date_col"
    ],
    "additionalProperties": false
}
```

## Validations

In addition to JSON Schema validation, the following dynamic checks will be applied:

* Global `observable_unit` must only include task ID columns, the `date_col`, and the `target_col` (unless `target_keys` are `NULL`, which implies a single global target and no `target` column).

* `time-series`:
  * If specified, dataset-level `observable_unit` must only include task ID columns, the `date_col`, and the `target_col` (unless `target_keys` are `NULL`).
  * `non_task_id_schema` must not define task ID columns or reserved columns (`as_of`, `output_type`, `output_type_id`).
  * Rows must be unique across the effective `observable_unit` (global or overridden) **including `as_of` column** if versioning is enabled.

* `oracle-output`:
  * If specified, dataset-level `observable_unit` must only include task ID columns, the `date_col`, and the `target_col` (unless `target_keys` are `NULL`).
  * Rows must be unique across the effective `observable_unit` (global or overridden) **excluding `as_of` column** if versioning is enabled (i.e., only one version per observable unit is allowed).

### Example `target-data.json` config files

#### Variant Nowcast hub

The Variant Nowcast hub (<https://github.com/reichlab/variant-nowcast-hub/>) requires the following schema for each relevant dataset:

##### `model-output`

This schema is defined by the `tasks.json` config and can therefore be used to define task ID columns in target data.

```
── <hub_connection/FileSystemDataset> ──

• hub_name: "SARS-CoV-2 Variant Nowcast Hub"
• hub_path: covid-variant-nowcast-hub/
• file_format: "parquet(187/187)"
• checks: FALSE
• file_system: "S3FileSystem"
• model_output_dir: "model-output/"
• config_admin: hub-config/admin.json
• config_tasks: hub-config/tasks.json

── Connection schema 
hub_connection with 187 Parquet files
8 columns
nowcast_date: date32[day]
target_date: date32[day]
location: string
clade: string
output_type: string
output_type_id: string
value: double
model_id: string
```

##### `time-series`

_(Note: opened manually by providing an explicit schema for partitions to work around partition column conflicts)_

```
FileSystemDataset with 473 Parquet files
6 columns
target_date: date32[day]
location: string
clade: string
observation: int64
nowcast_date: date32[day]
as_of: date32[day]
```

##### `oracle-output`

_(Note: opened manually by providing an explicit schema for partitions to work around partition column conflicts)_

```
FileSystemDataset with 41 Parquet files
6 columns
location: string
target_date: date32[day]
clade: string
oracle_value: int64
nowcast_date: date32[day]
as_of: date32[day]
```

**Proposed `target-data.json` for Variant Nowcast Hub:**

For this hub the configuration is straightforward:

1. The observable unit is the same across both time-series and oracle-output, so it's set once at the global level.
2. Both datasets are versioned using an `as_of` column, so `versioned` is set to `true` globally.
3. There are no `output_type` columns in the oracle-output, so the default of `false` for `has_output_type_ids` is appropriate.
4. No dataset-specific overrides are needed.

```json
{
    "schema_version": "https://raw.githubusercontent.com/hubverse-org/schemas/main/v6.0.0/target-data-schema.json",
    "observable_unit": [
        "location",
        "clade",
        "target_date",
        "nowcast_date"
    ],
    "date_col": "target_date",
    "versioned": true
}
```

#### Flusight hub

The Flusight hub (<https://github.com/cdcepi/FluSight-forecast-hub>) requires the following schema for each relevant dataset:

##### `model-output`

This schema is defined by the `tasks.json` config and can therefore be used to define task ID columns in target data.

```
── <hub_connection/FileSystemDataset> ──

• hub_name: "US CDC FluSight"
• hub_path: cdcepi-flusight-forecast-hub/
• file_format: "parquet(2454/2454)"
• checks: FALSE
• file_system: "S3FileSystem"
• model_output_dir: "model-output/"
• config_admin: hub-config/admin.json
• config_tasks: hub-config/tasks.json

── Connection schema 
hub_connection with 2454 Parquet files
9 columns
reference_date: date32[day]
target: string
horizon: int32
location: string
target_end_date: date32[day]
output_type: string
output_type_id: string
value: double
model_id: string
```

##### `time-series`

The time-series data contains additional non-task ID columns `location_name` and `weekly_rate`.

```
target_timeseries with 1 csv file
7 columns
as_of: date32[day]
target: string
target_end_date: date32[day]
location: string
location_name: string
observation: double
weekly_rate: double
```

##### `oracle-output`

The oracle-output in this hub has an additional `horizon` column that is not present in the time-series data. This is because it contains [a `pmf` output type which functionally requires the horizon to be known](https://github.com/reichlab/flusight-dashboard/issues/20#issuecomment-2815550603).

```
target_oracle_output with 1 csv file
8 columns
as_of: date32[day]
target: string
target_end_date: date32[day]
location: string
horizon: int32
output_type: string
output_type_id: string
oracle_value: double
```

**Proposed `target-data.json` for Flusight hub:**

For this hub, additional configuration is needed:

1. The global `observable_unit` defines the default for both datasets.
2. Both datasets are versioned, so `versioned` is set to `true` globally.
3. The `time-series` object defines the non-task ID columns present in time-series data.
4. The `oracle-output` object:
   - Overrides the `observable_unit` to include the additional `horizon` column required for proper mapping of `pmf` output types.
   - Sets `has_output_type_ids` to `true` since the oracle-output contains `output_type` and `output_type_id` columns.

```json
{
    "schema_version": "https://raw.githubusercontent.com/hubverse-org/schemas/main/v6.0.0/target-data-schema.json",
    "observable_unit": [
        "target",
        "target_end_date",
        "location"
    ],
    "date_col": "target_end_date",
    "versioned": true,
    "time-series": {
        "non_task_id_schema": {
            "location_name": "character",
            "weekly_rate": "double"
        }
    },
    "oracle-output": {
        "has_output_type_ids": true,
        "observable_unit": [
            "target",
            "target_end_date",
            "location",
            "horizon"
        ]
    }
}
```

### Other Options Considered

We considered including this metadata in the `tasks.json` config file but rejected this option because:
1. It does not strictly relate to defining modeling tasks
2. The `tasks.json` files are already often quite large and complex, so adding to that complexity seems unnecessary

We also considered requiring all properties to be specified at the dataset level (no global defaults), but this would create unnecessary verbosity for hubs where both datasets share the same configuration.

## Status

APPROVED

## Consequences

This section describes the resulting context after applying the decision. All consequences should be listed here, not just the "positive" ones. A particular decision may have positive, negative, and neutral consequences, but all of them affect the team and project in the future.

## Projects

- A list of links to project posters affected by this decision
