# 2025-01-03 Additional columns in Variant Nowcast Hub target data

## Context

The [COVID-19 Variant Nowcast Hub](https://github.com/reichlab/variant-nowcast-hub)
requires target data in two formats as described below.

### Time series

The time series format is a count of clades, summarized by collection date and
location. This format is useful for downstream processes that chart target data,
for example. The required colums are:

- location
- target_date
- clade
- observation

Additionally, the files will be partitioned by nowcast_date and sequence_as_of.

[reference](https://github.com/reichlab/variant-nowcast-hub/issues/212#issue-2734124041)

### Oracle output

The oracle output format mirrors the format of the hub's model-output files.
This format will be used to score model submissions. The required columns are:

- location
- target_date
- clade
- oracle_value

Additionally, the files will be partitioned by nowcast_date.

[reference](https://github.com/reichlab/variant-nowcast-hub/issues/215#issue-2736006168)

## Aims

We originally designed the above specifications from the perspective of model
evaluation and programmatic access (a dashboard, for example).

However, we also want target data files that are useful in other contexts, and
using partition keys as the sole way to get important information like
nowcast_date will impede users who:

- download target data files without replicating the file structure in the
repo/S3 bucket
- use a parquet reader that doesn't recognize hive partitioning

## Anti-Aims

- We do not want to generate a third target data format.
- We do not want to create extra steps for people working with downloaded
copies of the variant-nowcast-hub's target data (_e.g._, by storing
repetive information like nowcast_date in parquet metadata).

## Decision

We'll retain the partitioning strategy as originally stated, but will include
partition key values as columns in the target data parquet files. Additionally,
we'll include columns for derived data (_e.g._, tree_as_of) that might not be
intuitive for people unfamiliar with hub operations.

### Revised time series

Additional columns:

- nowcast_date
- sequence_as_of
- tree_as_of

### Revised oracle output

Additional columns:

- nowcast_date

### Other Options Considered

None—this decision mirrors the approach we decided on when converting
model-output submissions to a format for use on S3 (in that case, we add
round_id and model_id columns to the parquet files).

## Consequences

- The target data files will be slightly larger, but we expect minimal
impact due to parquet's ability to encode and compress repetitive columnar data.
- Including all relavent data in the parquet file provides more flexibilty
for implementing alternate partitioning strategies (we don't know all
potential use cases).

## Projects

Variant Nowcast Hub (no project poster).
