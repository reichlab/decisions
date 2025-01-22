# 2025-01-20 Target Data Schema Proposal

## Context

- All hubs need to provide target data for modelers and downstream evaluations.
- Many hubs have a script that is manually run to obtain new target time series
  data from raw data sources.
- Few hubs have oracle output data.
- As highlighted in
  [hubverse-org/hubDocs#230](https://github.com/hubverse-org/hubDocs/issues/230),
  there is no current standard or recommendations for the contents of target
  data.
- As described in [RFC
  2025-01-06-target-data-formats](2025-01-06-target-data-formats.md), we have
  established a structure for target data file organization.
- Visualizations and evaluations rely on being able to read in target data.
- Current solutions for visualizations and evaluations require explicit
  knowledge of the target data structure in the hub.
  - Example: both the [evaluations dashboard pilot][hubPredevalsData]
    and [forecast dashboard pilot][hub-dashboard-predtimechart] depend on
    specific assumptions of the target data as specified in the
    hub-dashboard-predtimechart README:
    > Target data generation: The app `generate_target_json_files.py` is
    > limited to hubs that store their target data as a `.csv` file in the
    > `target-data` subdirectory. That file is specified via the
    > `target_data_file_name` field in the hub's `predtimechart-config.yml`
    > file. We expect the file has these columns: `date`, `value`, and
    > `location`.
- Feeback from the hubverse developer meeting is that obtaining historical data
  from hubs is difficult.
- Knowledge of the statistical type of data (ordinal, nominal, etc) is necessary
  for visualizations (i.e. a line chart is often inappropriate with nominal
  categories along the independent axis).

[hubPredEvalsData]: https://github.com/elray1/flusight-dashboard/blob/d98e01e132c5705a72ed374fe6168e0888103714/create-oracle-data.R
[hub-dashboard-predtimechart]: https://github.com/hubverse-org/hub-dashboard-predtimechart/blob/1dc5f3e431e13d3a40f9d8fed5bcc7c74ce776e8/src/hub_predtimechart/app/generate_target_json_files.py#L101

### Aims

 - define a strategy that will standardize column names for target data that
   should conform to the following principles:
   - well-defined: the time series data should have clear mappings to the task IDs in the hub 
   - clear: these data should be easy for a hub administrator to write and store
   - general: anyone with access to a hub should be able to extract and operate
     on the timeseries data and oracle output data without requiring them to
     write bespoke code for a given hub.
   - language-agnostic: the format of the metadata should not prefer one
     language (both programming and human) over another.

### Anti-Aims

 - enforce the use of specific terms for target time series columns
 - provide any methods or recommendations for producing time series data from
   raw data sources
 - provide any methods or recommendations for oracle output data generation from
   target time series data

## Decision

- We will establish the standard that target time series data must be
  disaggregated by all observable non-dependent task IDs (i.e. `date` and
  `location`, but not `horizon`) including all optional values.
- We will establish a `targets.json` config file that provides the following
  information:
  - the name of the time series file
  - the name of the oracle outputs file
  - For each column across the files:
    - a boolean to indicate if it corresponds to a task ID
    - the task ID or it corresponds to
    - the data type: (nominal, ordinal, etc)
- We will establish a new `targets-schema.json` to validate the `targets.json`
  file.

### Rationale

TBC

### Other Options Considered

#### Mandate specific names for target time series data

This would mandate that target time series data follow the following column
recommendations... TBC


#### New targets.json config file in hubs validated against new targets-schema.json

This would be a file that lives in `hub-config/` and would specify the
relationship between 

PRO: json schemas can be validated
PRO: can version along with other schemas
CON: will require a new schema version
CON: json config files are not particularly easy to write for non-coders


### 



## Status

A decision may be "proposed" if the project stakeholders haven't agreed with it yet, or "accepted" once it is agreed. If a later ADR changes or reverses a decision, it may be marked as "deprecated" or "superseded" with a reference to its replacement.

## Consequences

This section describes the resulting context, after applying the decision. All consequences should be listed here, not just the "positive" ones. A particular decision may have positive, negative, and neutral consequences, but all of them affect the team and project in the future.

## Projects

- [hubverse](https://hubverse.io/) (which does not have a project poster at the moment)
