# 2025-06-17 RFC Time-series target data schema metadata

## Context

While moving forward with validation tests for target data, we have identified the need for a structured metadata schema that can be used to validate time-series target data schema deterministically. In particular, our allowance of inclusion of non-task ID columns in time-series data has so far required the opening of target time-series data to extract the default schema for such non-task ID columns when creating a time-series schema. This has uncovered a number of challenges/limitations:
1. What schema to compare incoming files to. Currently we have settled on the schema of the first file in a multi-file dataset but this could in rare circumstances become issue (i.e. post-hoc changes to the first file). It also continues to require  opening the dataset to extract the schema, which itself involves an additional call to cloud hubs and introduces performance consideration: https://github.com/hubverse-org/hubData/issues/87#issuecomment-2901055075 
2. Problems arising from hive partitioning on columns that also are contained in the data e.g. the nowcast hub which is partitioned on a date type column. The parquet files contain the correct data type (date) for that column but the default dataset behaviour when opening datasets without an explicit schema is to cast all partition variables as character. This causes a conflict during the attempt to open the dataset resulting in the dataset not being able to be opened without an explicit schema defined. Because of this, such datasets cannot currently be opened with our current `hubData` functions https://github.com/hubverse-org/hubData/issues/89 

Creating new metadata fields that admins can use to define the schema for such non-task ID columns will allow the definition of a time-series dataset schema deterministically and would alleviate the issues described above.

In addition to the above considerations, an outstanding piece of information required by hubs involves knowledge of the columns which make up the observable unit in both time-series and oracle-output data (see for example the [discussion about validations in a previous RFC](https://github.com/reichlab/decisions/blob/main/decisions/2025-02-27-rfc-time-series-target-data.md#validations). The observable unit consists of the columns in a target dataset whose values uniquely identify a single observation in the dataset at a specific point in time. Knowledge of this information is important in ensuring there are no duplicate observations in any of the target datasets.


### Aims

Decide on metadata structure and the associated schema and validation rules of properties that will enable a deterministic creation of schema for time-series target data. We will also consider the requested property to describe the task IDs that make up an observational unit.

### Anti-Aims

We will not discuss any other aspect of schema updating that does not relate to time-series target data schema. 

## Decision

We will accept a new config file named `target-data.json` file that will be used to define the schema for time-series target data. This can be achieved by a combination of:
- listing any columns in the target data type that correspond to task ID variables (whose data types are already defined via `tasks.json`)
- listing any non-task ID columns along with their data types that are present in the time-series data. 



The [**target-data-schema.json**](./2025-06-17-RFC-target-data-metadata/target-data-schema.json) file describing the properties of the a `target-data.json` config file will be as follows:


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
                "time-series": {
                    "type": "object",
                    "properties": {
                        "observable_unit": {
                            "description": "Names of columns whose unique value combinations define an observable unit in time-series data. Each combination of values must be unique. If multiple values are available for the same time point they should be unique across `as_of` data versions. The majority are expected to correspond to task ID names but may include other columns as well (e.g. the general `date` column).",
                            "type": "array",
                            "uniqueItems": true,
                            "items": {
                                "type": "string"
                            }
                        },
                        "extra_task_ids": {
                            "description": "Names of task IDs that are not part of the observable unit but are present in the time-series data. These task IDs may be used for additional context or filtering.",
                            "type": ["array", "null"],
                            "uniqueItems": true,
                            "items": {
                                "type": "string"
                            }
                        },
                        "non_task_id_schema": {
                            "type": "object",
                            "uniqueItems": true,
                            "examples": [
                                {
                                    "as_of": "Date",
                                    "location_name": "character"
                                },{
                                    "as_of": "Date",
                                    "date": "Date"
                                }
                            ],
                            "additionalProperties": {
                                "type": "string",
                                "enum": ["character", "double", "integer","logical", "Date"],
                                "description": "Key-value pairs of non-task ID column names and data types found in time-series data. Include any columns in the time-series data that does not correspond exactly to a task ID. If an `as_of` column is included, it should be specified here as well."
                            }
                        }
                    },
                    "required": ["observable_unit"],
                    "additionalProperties": false
                },
                "oracle-output": {
                    "type": "object",
                    "properties": {
                        "observable_unit": {
                            "description": "Names of task IDs whose unique value combinations define an observable unit in oracle output data. Each combination of values must be unique once combined with output type IDs.",
                            "type": "array",
                            "uniqueItems": true,
                            "items": {
                                "type": "string"
                            }
                        },
                        "extra_task_ids": {
                            "description": "Names of task IDs that are not part of the observable unit but are present in the oracle-output data. These task IDs may be used for additional context or filtering (e.g. some derived task IDs).",
                            "type": ["array", "null"],
                            "uniqueItems": true,
                            "items": {
                                "type": "string"
                            }
                        },
                    },
                    "required": ["observable_unit"],
                    "additionalProperties": false
                }
            },
            "required": ["time-series", "oracle-output"],
            "additionalProperties": false
        }
    }
}
```

To summarise, the `target-data.json` file will contain a `target_data_metadata` object with two main properties: `time-series` and `oracle-output`. Each property will define metadata about the respective target data type.

- **`time-series`**: This object contains three main main properties:
    - `observable_unit`: An array of columns whose unique value combinations define an observable unit in time-series data. Each combination of values must be unique across `as_of` data versions. In time-series data, this typically includes task IDs that uniquely identify a specific observation at a given point in time but might include columns that do not directly correspond to task ID names (e.g. a more general `date` column).
    - `extra_task_ids`: An optional array of task IDs that are not part of the observable unit but are present in the time-series data. These task IDs may be used for additional context or filtering.
    - `non_task_id_schema`: An object that defines the schema for non-task ID columns in the time-series data. Each property in this object represents a non-task ID column name, and its value is the data type of that column. The data types represent R data types can be one of the following: `character`, `double`, `integer`, `logical`, or `Date`. This will allow for the inclusion of additional columns in the time-series data that do not correspond directly to task IDs. Note that the standard column `observation` DOES NOT need defining/including as will always be cast as `double`. It is also never part of the observable unit.
- **`oracle-output`**: This object contains two main properties:
    - `observable_unit`: An array of task ID names whose unique value combinations define an observable unit in oracle output data. Each combination of values must be unique once combined with output type IDs. Note that for `oracle-output` target data, the observable unit always consists of task IDs. `output_type` and `output_type_id` if present is always part of the observable unit in practice but does need defining in this property.
    - `extra_task_ids`: An optional array of task IDs that are not part of the observable unit but are present in the oracle-output data. These task IDs may be used for additional context or filtering (e.g. some derived task IDs).
    - Note again that the standard `oracle_value` column does not be defined anywhere and will be cast to `double` by default.


### Validations

In addition to basic JSON validation against the schema described above, the following additional dynamic validations will be performed during `validate_config()`:

- `time-series`: 
    - Any columns listed in the `observable_unit` that are not task IDs must have a data type defined in the `non_task_id_schema` property.
    - If the hub has a target column specified in target_metadata, the `observable_unit` must include that column.
    - The `extra_task_ids` property, if present, must not contain any columns that are already part of the `observable_unit`.
    - The `non_task_id_schema` property must not contain any task IDs.
- `oracle-output`: 
    - All columns listed in the `observable_unit` must be task IDs.
    - The `extra_task_ids` property, if present, must not contain any columns that are already part of the `observable_unit`.
    
### Example `target-data.json` config files

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

The proposed example [`target-data.json`](./2025-06-17-RFC-target-data-metadata/flusight-target-data.json) file for the Flusight hub would look like this:

```json
{
    "target_data_metadata": {
        "time-series": {
            "observable_unit": [
                "target",
                "target_end_date",
                "location"
            ],
            "non_task_id_schema": {
                "as_of": "Date",
                "location_name": "character",
                "weekly_rate": "double"
            }
        },
        "oracle-output": {
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
_Note the `oracle-output` dataset contains an `as_of` column which is currently not allowed_


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

The proposed example [`target-data.json`](./2025-06-17-RFC-target-data-metadata/nowcast-target-data.json) file for the Variant Nowcast Hub hub would look like this:

```json
{
    "target_data_metadata": {
        "time-series": {
            "observable_unit": [
                "location",
                "clade",
                "target_date",
                "nowcast_date"

            ],
            "non_task_id_schema": {"as_of": "Date"}
        },
        "oracle-output": {
            "observable_unit": [
                "location",
                "clade",
                "target_date",
                "nowcast_date"
            ]
        }

    }
}
```
_Note the `oracle-output` dataset contains an `as_of` column which is currently not allowed_

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
