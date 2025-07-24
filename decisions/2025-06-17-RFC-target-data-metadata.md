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

The `target-data.json` file defines a `target_data_metadata` object with top-level properties that describe expectations across target datasets.

### Top-Level Properties

* `observable_unit`: An array of column names whose unique value combinations define the minimum observable unit. Must be task IDs and additionally always include the column defined in the `date_col` property. Unique combinations. If versioned, unique combinations will also take into account the values in the `as_of` column but is never included in the observable unit as it is not a task ID but a versioning column. This property is required.
* `date_col`: The default date column across time-series, oracle-output, and model-output datasets. Expected to be of type `Date`.
* `versioned`: Boolean indicating whether `as_of` versioning is used. If true, datasets must have a date `as_of` column indicating the version of each data point. Defaults to `false`. 

I am also proposing to allow an `as_of` column in `oracle-output` to support traceability if versioning is being used. it will allow us to link individual oracle value observations to the specific version of time-series data it was derived from. I propose we enforce that there should only be a single version of an observation in oracle output data so no filtering on `as_of` date is required to get a single version of available data.

### Target-Type Specific Configuration

* **`time-series`**:

  * `extra_task_ids`: Additional task IDs used for filtering or grouping.
  * `non_task_id_schema`: key-value pairs of non-task id column names and their R-data types, one of (`character`, `double`, `integer`, `logical`, `Date`). The `as_of` column does not need defining here as it is expected to always be a date column.

* **`oracle-output`**:

  * `has_output_type_ids`: Boolean. Must be true if `pmf` or `cdf` output types exist. Can be false otherwise. If true, the dataset must include `output_type` and `output_type_id` columns. Defaults to `false`.
  * `observable_unit`: Task IDs whose combination plus any output type IDs if present, uniquely define a row. This can be especially useful for [output types whose values are functionally dependent on other task IDs](https://github.com/reichlab/flusight-dashboard/issues/20#issuecomment-2815550603. 
  
The schema of this configuration file is defined in the following JSON Schema:

### `target-data-schema.json`

```json
{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "https://raw.githubusercontent.com/hubverse-org/schemas/main/v6.0.0/target-data-schema.json",
    "title": "Schema for Modeling Hub target data definitions",
    "description": "This is the schema of the target-data.json configuration file that defines metadata about target data used to visualise and evaluate modeling hub model outputs.",
    "type": "object",
    "description": "Target data metadata.",
    "properties": {
        "target_data_metadata": {
            "type": "object",
            "properties": {
                "observable_unit": {
                            "description": "Names of columns whose unique value combinations define the minimum observable unit in time-series data. Each combination of values must be unique across `as_of` data versions if applicable. The majority are expected to correspond to task ID names but may include other columns as well (e.g. the general `date` column).",
                        "type": "array",
                        "uniqueItems": true,
                        "items": {
                            "type": "string"
                        }
                    },
                "date_col": {
                    "description": "Name of the date column across hub data (time-series, oracle-output and model output). This is the column that stores the date on which observed data actually occured.",
                    "type": ["string", "null"],
                    "default": null
                },
                "versioned": {
                    "description": "Indicates whether target data are versioned using `as_of` dates. If true, the data is expected to have a date `as_of` column that indicates the version of each data point.",
                    "type": "boolean",
                    "default": false
                },
                "time-series": {
                    "type": "object",
                    "properties": {
                        "extra_task_ids": {
                            "description": "Names of task IDs that are not part of the observable unit but are present in the time-series data. These task IDs may be used for additional context or filtering.",
                            "examples": [
                                ["horizon"]
                            ],
                            "type": ["array", "null"],
                            "uniqueItems": true,
                            "items": {
                                "type": "string"
                            }
                        },
                        "non_task_id_schema": {
                            "type": "object",
                            "uniqueItems": true,
                            "description": "Key-value pairs of non-task ID column names and data types found in time-series data. Include any columns in the time-series data that does not correspond exactly to a task ID. If an `as_of` column is included, it should be specified here as well.",
                            "examples": [
                                {
                                    "location_name": "character"
                                },{
                                    "date": "Date"
                                }
                            ],
                            "additionalProperties": {
                                "type": "string",
                                "enum": ["character", "double", "integer","logical", "Date"]
                            }
                        },
                    },
                    "additionalProperties": false
                },
                "oracle-output": {
                    "type": "object",
                    "properties": {
                        "has_output_type_ids": {
                            "type": "boolean",
                            "description": "Indicates whether the oracle output data have an `output_type` and `output_type_id` column. These columns are necessary if hub includes `pmf` and `cdf` output types but optional otherwise.",
                            "default": false},
                        "observable_unit": {
                            "description": "Names of task IDs whose unique value combinations define an observable unit in oracle output data. Each combination of values must be unique once combined with output type IDs. Can be used to override default observable units in situations where some output types require additional task ID value to map onto target data.",
                            "type": "array",
                            "uniqueItems": true,
                            "items": {
                                "type": "string"
                            }
                        },
                    },
                    "additionalProperties": false
                }
            },
            "required": ["observable_unit", "date_col"],
            "additionalProperties": false
        }
    }
}

```


## Validations

In addition to JSON Schema validation, the following dynamic checks will be applied:

* `observable_unit` must only include task ID columns and the `date_col` and the `target_col` unless `target_keys` are `NULL` which implies a single global target and no `target` column.

* `time-series`:

  * `extra_task_ids` must not overlap with `observable_unit`.
  * Rows must be unique across `observable_unit`.
  * `non_task_id_schema` must not define task ID columns.


* `oracle-output`:

  * `observable_unit` must only include task ID columns, the `date_col` and the `target_col` unless `target_keys` are `NULL` which implies a single global target and no `target` column.

    
### Example `target-data.json` config files

#### Variant Nowcast hub

The Variant Nowcast hub (<https://github.com/reichlab/variant-nowcast-hub/>) requires the following schema for each relevant datasets

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

_(Note I've opened this manually by providing an explicit schema for partitions to get around the problems discussed above)_

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

_(Note I've opened this manually by providing an explicit schema for partitions to get around the problems discussed above)_

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

The proposed example `target-data.json` file for the Variant Nowcast Hub hub would look like this:

For this hub the config is quite simple:

1. The observable unit is the same for both dataset so can be set once at the root level
2 There are no `output_type` columns in the `oracle-output` so the default of `false` is used

```json
{
    "target_data_metadata": {
        "observable_unit": [
            "location",
            "clade",
            "target_date",
            "nowcast_date"
        ],
        "versioned": true,
        "date_col": "target_date"
    }
}
```


#### Flusight hub

The Flusight hub (<https://github.com/cdcepi/FluSight-forecast-hub>) requires the following schema for each relevant datasets

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

The timeseries data contains additional non task ID columns `location_name` and `weekly_rate`.  

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

The oracle output in this hub has an additional `horizon` column that is not present in the time-series data. This is because it contains [a `pmf` output type which functionally requires the horizon to be known](https://github.com/reichlab/flusight-dashboard/issues/20#issuecomment-2815550603). 

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

The proposed example `target-data.json` file for the Flusight hub requires some additional configuring. Specifically:

1. a `time-series` object is used to define the non-task ID columns in the time-series data.
2. an `oracle-output` object is used to define the additional `horizon` column in the oracle output data.

```json
{
    "target_data_metadata": {

        "observable_unit": [
            "target",
            "target_end_date",
            "location"
        ],
        "versioned": true,
        "date_col": "target_end_date",
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
}
```



### Other Options Considered

We considered including this metadata in the `tasks.json` config file but rejected this option as:
1. It does not strictly relate to defining modeling tasks
2. The `tasks.json` files are already often quite large and complex files so adding to that complexity seems unnecessary.

## Status

PROPOSED

## Consequences

This section describes the resulting context, after applying the decision. All consequences should be listed here, not just the "positive" ones. A particular decision may have positive, negative, and neutral consequences, but all of them affect the team and project in the future.

## Projects

 - a list of links to project posters affected by this decision
