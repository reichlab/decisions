# 2025-01-06 Target data formats and organisation

## Context

Target data are the observed data being modeled as the prediction target in a collaborative modeling exercise. We have already descibed the expectation for the contents of such in the [Target (observed) data section](https://hubverse.io/en/latest/user-guide/target-data.html#target-observed-data) of our documentation but we have yet to advise explictly on file formats, file organisation and expected file names. 



### Aims

 - Define the file formats and file names that are acceptable for target data. 
 - By setting a convention for target data, we can then move on to:
    - Document expectations for hub admins in a new section in [Target (observed) data ](https://hubverse.io/en/latest/user-guide/target-data.html#target-observed-data).
    - Define any missing hubverse functionality for accessing and validating target data and potentially helpers for writing out target data in the required format.

### Anti-Aims

 - We continue to consider functionality for processing target data into desired formats as out of scope for the hubverse and expect hub administrators to provide target data in the required format.

## Decision

### File names

There two expected formats for target data:
- `timeseries` format
- `oracle output` format

As such we will expect data associated with each format to be named either `timeseries` or `oracle-output` respectively.

### File formats

Target data files should be either in `csv` or `parquet` format.

### Partitioned target data 

To enable splitting up potentially large target data files, we will allow for partitioned target data. In this case, the target data should be stored in a directory named either `timeseries` or `oracle-output` respectively. 

The directory should contain the partitioned target data files and should be created using [`arrow::write_dataset()`](https://arrow.apache.org/docs/r/reference/write_dataset.html) in R or [`pyarrow.dataset.write_dataset`](https://arrow.apache.org/docs/python/generated/pyarrow.dataset.write_dataset.html) in Python and use **hive partitioning**.

In Apache Hive, the filename format of partitioned data depends on the partition column names and their values. The files corresponding to each partition are stored in subdirectories, and the directory names encode the partition column names and their values, e.g. `<partition_column_1>=<value_1>/<partition_column_2>=<value_2>/.../<data_files>`

Here's an example of an oracle data partitioned by `target_end_date`:

```
target-data/oracle-output/target_end_date=2022-10-22
target-data/oracle-output/target_end_date=2022-10-22/part-0.csv
target-data/oracle-output/target_end_date=2022-10-29
target-data/oracle-output/target_end_date=2022-10-29/part-0.csv
target-data/oracle-output/target_end_date=2022-11-05
target-data/oracle-output/target_end_date=2022-11-05/part-0.csv
```

Hive partitioning is the default partitioning scheme used by `arrow::write_dataset()` and `pyarrow.dataset.write_dataset` so no additional configuration beyond the columns to partition by is required when writing partitioned datasets.

The benefit of using this approach is that: 

- we do not need to be prescriptive about the partitioning scheme used as long as hive partitioning is used. This allows hub admins to partition the data in a way that makes sense for their use case but for hubverse to be able to use open_dataset function to access the data without additional need for defining the partitioning scheme. (see [supplementary materials](https://github.com/reichlab/decisions/blob/ak/rfc/target-data-formats/decisions/2025-01-06-rfc-target-data-formats/2025-01-06-rfc-target-data-formats.html) for demonstration).

- arrow datasets support both `csv` and `parquet` formats and so we can support both formats for partitioned target data.

- new data can easily be added to the target data.(see [supplementary material](https://github.com/reichlab/decisions/blob/ak/rfc/target-data-formats/decisions/2025-01-06-rfc-target-data-formats/2025-01-06-rfc-target-data-formats.html)).

- partitioned datasets allow more efficient access by partition-aware data readers (for example, arrow, polars, DuckDB), which can filter the data on read if only a subset of it is required.

- arrow datasets work for single files also so we can use the same function to read both single and partitioned target data.

Large un-partitioned data could be stored locally in a `target-data-raw` directory (which is not committed?) and then written and partitioned into the `target-data` directory (once it has been validated and correct schema is set? see below).

### Functionality for writing, reading and validating target data

As aforementioned, we propose that arrow datasets are used for accessing target data. 

We can then develop functionality to:

- read target data using `arrow::open_dataset()` in R or `pyarrow.dataset.dataset()` in Python. https://github.com/hubverse-org/hubUtils/issues/196 

- functionality for validating target to ensure all columns required for downstream functionality are present. https://github.com/hubverse-org/hubUtils/issues/197

- functionality for ensuring target data conforms to the correct schema, ideally before writing out (especially for parquet target data). This is important as join functions fail if corresponding columns are not of the same data type. We could modify [hubData::coerce_to_hub_schema](https://hubverse-org.github.io/hubData/reference/coerce_to_hub_schema.html) to coerce oracle output target data to the correct schema (substituting the datatype for `value` for the `oracle_value` column data type). (Issue needed once proposal accepted)

### Other Options Considered

The possibility of non-hive partitioned target data was considered but it was assessed that this would require to much additional configuration to successfully read partitioned data so to begin with, hive partitioning will be sufficient for our needs.

## Status

Proposed

## Consequences

The main consequence of this decision is that it introduces an arrow dependency on any package that needs to read target data. As such, thought will be required as to where these functions live as ideally, we do not want to burden `hubUtils` with an `arrow` dependency.

## Projects

 - a list of links to project posters affected by this decision
