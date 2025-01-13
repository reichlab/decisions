# 2025-01-06 Target data formats and organisation

## Context

Target data are the observed data being modeled as the prediction target in a collaborative modeling exercise. We have already descibed the expectation for the contents of such in the [Target (observed) data section](https://hubverse.io/en/latest/user-guide/target-data.html#target-observed-data) of our documentation but we have yet to advise explicitly on file formats, file organisation and expected file names. 



### Aims

 - Define the file formats and file names that are acceptable for target data. 
 - By setting a convention for target data, we can then move on to:
    - Document expectations for hub admins in a new section in [Target (observed) data ](https://hubverse.io/en/latest/user-guide/target-data.html#target-observed-data).
    - Define any missing hubverse functionality for accessing and validating target data and potentially helpers for writing out target data in the required format.

### Anti-Aims

 - We continue to consider functionality for processing target data into desired formats as out of scope for the hubverse and expect hub administrators to provide target data in the required format.

## Decision


### Target data location

Target data should be stored in a directory named `target-data` 

There two expected formats for target data are:
- `time-series` format
- `oracle output` format


### File formats and names

Target data should stored in `parquet` format. Column data types in the `oracle-output` file(s) should be consistent with the schema defined in the config file, with the `oracle-value` column having the same data type as the `value` column in model output files.

Single files for target data are allowed and should be named `time-series.parquet` or `oracle-output.parquet` respectively.

### Partitioned target data 

To enable splitting up potentially large target data files, we will allow for partitioned target data. In this case, the target data should be stored in a directory named either `time-series` or `oracle-output` respectively. 

The directory should contain the partitioned target data files and follow hive partitioning naming conventions.

In Apache Hive, the filename format of partitioned data depends on the partition column names and their values. The files corresponding to each partition are stored in subdirectories, and the directory names encode the partition column names and their values, e.g. `<partition_column_1>=<value_1>/<partition_column_2>=<value_2>/.../<data_files>`. This means hive style partitioned data subdirectories are self describing and can be easily read by partition-aware data readers.

Here's an example of an oracle data in the `orale-output` directory partitioned by `target_end_date`:

```
├── target_end_date=2023-06-03
│   └── part-0.parquet
├── target_end_date=2023-06-10
│   └── part-0.parquet
└── target_end_date=2023-06-17
    └── part-0.parquet
```

_see more details on file names in the [Date file names](#data-file-names)_

Such partitioned data can be easily created using [`arrow::write_dataset()`](https://arrow.apache.org/docs/r/reference/write_dataset.html) in R or [`pyarrow.dataset.write_dataset`](https://arrow.apache.org/docs/python/generated/pyarrow.dataset.write_dataset.html) in Python. 

Hive partitioning is the default partitioning scheme used by `arrow::write_dataset()` and `pyarrow.dataset.write_dataset` so no additional configuration beyond the columns to partition by is required when writing partitioned datasets using this functions. However, hub administrators are free to use any tool of their choice to partition the data, so long as it follows hive partitioning conventions if any data are encoded in sub-directory names.

Proposing this convention is designed to allow us to **access target data as arrow datasets using `arrow::open_dataset()` in R**.  It also allows for the possibility of using `pyarrow.dataset.dataset()` in Python, should we decide to use `arrow` in `hubDataPy`.

In return the benefit of using this approach are:

- we do not need to be prescriptive about the partitioning scheme used as long as hive partitioning is used if any data are encoded in sub-directory names. This allows hub admins to partition the data in a way that makes sense for their use case but for hubverse to be able to use the `arrow::open_dataset()` function to access the data without additional need for defining the partitioning scheme. (see [supplementary materials](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/ak/rfc/target-data-formats/decisions/2025-01-06-rfc-target-data-formats/supplementary-materials.html) for demonstration).

- `arrow::open_dataset()` supports `parquet` formats.

- new data can easily be added to the target data. (see [supplementary material](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/ak/rfc/target-data-formats/decisions/2025-01-06-rfc-target-data-formats/supplementary-materials.html)). 

- partitioned datasets allow more efficient access by partition-aware data readers (for example, arrow, polars, DuckDB), which can filter the data on read if only a subset of it is required.

- `arrow::open_dataset()` works for single files also so we can use the same function to read both single and partitioned target data.

Large un-partitioned data could be stored locally in a `target-data-raw` directory (which is not committed?) and then written and partitioned into the `target-data` directory (once it has been validated and correct schema is set? see below).

### Functionality for writing, reading and validating target data

As aforementioned, we propose that we access target data as arrow datasets using `arrow::open_dataset()`. 

We can then develop functionality to:

- read target data using `arrow::open_dataset()` in R. https://github.com/hubverse-org/hubUtils/issues/196. This functionality is also available in Python using `pyarrow.dataset.dataset()`, should `arrow` be used in `hubDataPy`.

- functionality for validating target data to ensure all columns required for downstream functionality are present. https://github.com/hubverse-org/hubUtils/issues/197

- functionality for ensuring target data conforms to the correct schema, **before writing out**. This is important as join functions fail if corresponding columns are not of the same data type. We could modify [hubData::coerce_to_hub_schema](https://hubverse-org.github.io/hubData/reference/coerce_to_hub_schema.html) to coerce oracle output target data to the correct schema (substituting the datatype for `value` for the `oracle_value` column data type). (Issue needed once proposal accepted). 

### Exploration of applying schema to target data

I performed an exploration of applying schema to target data using `arrow::open_dataset()` in the [supplementary materials](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/ak/rfc/target-data-formats/decisions/2025-01-06-rfc-target-data-formats/supplementary-materials.html#applying-a-schema).

In summary, 
- applying a schema to partitioned target data is **straightforward** using `arrow::open_dataset()` and determining the schema from the config file **for parquet files.**

### Splitting into multiple files that retained all columns

I also explored the potential for splitting data into multiple files that retained all columns using `arrow::write_dataset()` in the [supplementary materials](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/ak/rfc/target-data-formats/decisions/2025-01-06-rfc-target-data-formats/supplementary-materials.html#partitioning-that-contains-all-data-in-files)

Because reducing redundancy is a key aspect of partitioning, this is not possible if files are partitioned by a data column into sub-directories.

However, it is possible to just split files within a top level directory. This allows for all columns to be present in resulting data files and still allow data to be accessible as a dataset. See [supplementary materials](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/ak/rfc/target-data-formats/decisions/2025-01-06-rfc-target-data-formats/supplementary-materials.html#partitioning-that-contains-all-data-in-files) for details.


### Data file names

I also wanted to demonstrate that individual file names do not actually need to follow a specific filename convention. The default `part-{i}.ext` convention used in `write_dataset()` can be overridden by providing a different pattern to argument `basename_template`. 

More importantly, filenames in the node directory do not actually affect the opening of the dataset as no metadata are expected to be stored in arrow dataset file names themselves ( metadata are only stored in directory names). `open_dataset()` continues to be able to open the files as a dataset successfully even if they deviate from the `part-{i}.ext` convention.

As such, initial files can be created and additional files added to a target dataset using whatever file name convention we like and using other tooling also. See [supplementary materials](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/ak/rfc/target-data-formats/decisions/2025-01-06-rfc-target-data-formats/supplementary-materials.html#data-file-names) for details.

The situation described in this section might be closer to the actual use case of hub admins who might be using other tools to create target data files.

## Other Options Considered

The possibility of non-hive partitioned target data was considered but it was assessed that this would require to much additional configuration to successfully read partitioned data so to begin with, hive partitioning will be sufficient for our needs.

The possibility of non-hive (i.e. [Directory partitioned](https://arrow.apache.org/docs/r/reference/Partitioning.html)) target data was considered but this would require too much additional configuration (i.e. explicit knowledge of the column name for each partition level) to successfully read Directory partitioned data.

The possibility of storing target data in `csv` format was also considered but in the exploration of applying a schema on partitioned `csv` target data is **not as straightforward** as with `parquet` data:
  - there are API differences in applying a schema through `open_dataset()` between the two formats and `csv` is more complex than for `parquet` files.
  - for `csv`s partition column datatypes need to be specified explicitly through the `partitioning` argument therefore requiring knowledge of the partition column names. These can however be detected by the directory names of partitioned data but that requires an additional step.
  - There have generally been issues (see <https://github.com/apache/arrow/issues/34589> and <https://github.com/apache/arrow/issues/34640>) with applying schema to partitioning columns in CSV datasets. Although they have been closed as resolved,
I [re-commented on one of the issues](https://github.com/apache/arrow/issues/34640#issuecomment-2577363842) as I don't think that is the case. Will be interesting to hear their response.

## Status

Proposed

## Consequences

The main consequence of this decision is that it introduces an arrow dependency on any package that needs to read target data. As such, thought will be required as to where these functions live as ideally, we do not want to burden `hubUtils` with an `arrow` dependency.

It also means we are currently restricting the format to `parquet` which requires more specialist tools to write out than `csv` and also requires ensuring all files share the same schema. This is a restriction that hub admins will need to be aware of and will require the provision of detailed documentation and helper functions to ensure target data is in the correct format on our end.

## Projects

 - a list of links to project posters affected by this decision
