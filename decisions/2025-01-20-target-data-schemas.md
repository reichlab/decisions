# 2025-01-20 Target Data Schema Proposal

## Context

- All hubs need to provide target data for modelers and downstream evaluations.
- Many hubs have a script that is manually run to obtain new target time series
  data from raw data sources.
- Few hubs have oracle output data.
- As highlighted in
  [hubverse-org/hubDocs#230](https://github.com/hubverse-org/hubDocs/issues/230),
  the current standards or recommendations for the contents of target time series
  data files are not as specific as they could be.
- As described in [RFC
  2025-01-06-target-data-formats](2025-01-06-rfc-target-data-formats.md), we have
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
- Feedback from the hubverse developer meeting is that obtaining historical data
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
 - define target data column name formats that enable standard interfacing with 
   hubverse eval tooling (hubEvals + dashboard packages) and hubverse viz
   tooling (hubVis + dashboard packages).

### Anti-Aims

 - enforce the use of specific terms for all target time series columns
 - define a standard for target-data that will always be suitable for use by
   modelers
 - define a standard for target data that will generalize to all kinds of hubs. 
   For example, hubs that have a multi-layer process to determining "truth",
   one setting being the variant hub with changing reference trees that 
   subsequently change the labels/variants assigned to individual cases over time,
   will not necessarily be supported by the proposed set-up as it currently 
   is defined.
 - provide any methods or recommendations for producing time series data from
   raw data sources
 - provide any methods or recommendations for oracle output data generation from
   target time series data

## Decision

- We introduce the concept of a unique "observable unit" distinct from a unique 
  "modeling task". The distinction here is that two modeling tasks can point 
  to the same observed unit. For example a two week ahead prediction made at 
  time t and a one week ahead prediction made at time t+1 are predicting the same
  observable unit. 
- We establish the standard that a time-series file should have the subset of
  task-id columns (assuming a flat file structure, but this should be 
  generalized to accommodate data provided in a partition) that uniquely identify
  a single observable unit.
  - An example from a relatively simple forecast hub might be that the task
    id variables are:
    - `reference_date`
    - `target_end_date`
    - `horizon` 
    - `location`
  - In this setting, the set of `target_end_date`, `horizon`, and `location` 
    is enough to uniquely identify a modeling task (without the derived task
    id of `reference date`)
  - However, you only need `target_end_date` and `location` to identify an 
    observable unit, so these two columns might be a natural choice for
    task-id column names for the time-series file.
- We will establish the standard that target time-series data can include
  versions of data through the inclusion of an `as_of` column which is 
  expected to be in a ISO date format. Note that we are not providing
  flexibility in this column name or data format. However it is optional to
  include this `as_of` column
- Validations would then be:
  - Validation of column names
    - there must be a `observation` column
    - there may be a `as_of` column
    - all other columns present need to be valid task-id variable names
    - there should be at least one task-id variable column.
  - Validation of data types
    - data types in the time-series target data file must match those specified
      in the hub schema file

### Rationale

We decided to prioritize out-of-the-box support for dashboarding tools
(evaluation and viz), rather than trying to make sure the format of these files 
would be appropriate for every possible desire for a modeler. E.g., this drove
the choice to require that the column names in the time-series file match 
task-id variables directly, without needing some cross-walk file between column
names. 

### Other Options Considered

#### More flexibility on specific names for target time series data

This would necessitate a separate file that would document which time-series 
column names map to task-id variables. Given the scope is limited to supporting
tools for eval and viz, this was decided to be unnecessary additional structure
and a low-cost requirement to impose.

#### More flexibility on allowing other columns to be included in the data

This might help support hubs that have a more complex evaluation set up, as
in the variant hub where the reference trees are changing.

- PRO: might not disrupt validation too much, and would accomodate other hubs
- CON: it might break the pred/evals downstream tooling


#### New targets.json config file in hubs validated against new targets-schema.json

This would be a file that lives in `hub-config/` and would specify the
relationship between 

- PRO: json schemas can be validated
- PRO: can version along with other schemas
- CON: will require a new schema version
- CON: json config files are not particularly easy to write for non-coders



## Status

proposed.

## Consequences

This will create a clear promise of target-data format for downstream tools.

## Projects

- [hubverse](https://hubverse.io/) (which does not have a project poster at the moment)
