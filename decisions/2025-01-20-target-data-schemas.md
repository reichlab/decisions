# 2025-01-20 Target Data Time-series File Contents Proposal

## Context

- Many hubs will want to provide target data for modelers, analysts, and 
  hub admins (to support, e.g. visualization and evaluation).
- Many hubs have a script that is manually run to obtain new target time series
  data from raw data sources.
- As highlighted in
  [hubverse-org/hubDocs#230](https://github.com/hubverse-org/hubDocs/issues/230),
  the current standards or recommendations for the contents of target time series
  data files are not as specific as they could be.
- As described in [RFC
  2025-01-06-target-data-formats](2025-01-06-rfc-target-data-formats.md), we have
  established a structure for target data file organization.
- There exists substantial metadata about targets and modeling tasks in the
  hubverse `tasks.config` file already. 
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
  from hubs about what data was available when is difficult.
- Knowledge of the statistical type of data (ordinal, nominal, etc) is necessary
  for visualizations (i.e. a line chart is often inappropriate with nominal
  categories along the independent axis).

[hubPredEvalsData]: https://github.com/elray1/flusight-dashboard/blob/d98e01e132c5705a72ed374fe6168e0888103714/create-oracle-data.R
[hub-dashboard-predtimechart]: https://github.com/hubverse-org/hub-dashboard-predtimechart/blob/1dc5f3e431e13d3a40f9d8fed5bcc7c74ce776e8/src/hub_predtimechart/app/generate_target_json_files.py#L101

### Aims

 - define a strategy that will standardize column names for target time series data.
 - The guidelines here will apply to targets that have `"is_step_ahead": true` in 
   their `model_tasks` > `target_metadata` array in a hub config file. 
 - The guidelines here will apply to hubs that are configured such that the
   task-id variables do not change across rounds.
 - In general the data should conform to the following principles:
   - well-defined: the time series data should have clear mappings to the task IDs in the hub 
   - clear: these data should be easy for a hub administrator to write and store
   - general: anyone with access to a hub should be able to extract and operate
     on the timeseries data without requiring them to write bespoke code for a given hub.
   - language-agnostic: the format of the metadata should not prefer one
     programming language over another.
 - define target time series data column name formats that enable standard interfacing with 
   hubverse viz tooling (hubVis + dashboard packages).

### Anti-Aims

 - enforce the use of specific terms for all target time series columns
 - define a standard for target-data that will be universally suitable for all
   modeling scenarios.
 - define a standard for target data that will generalize to all kinds of hubs.
 - provide any methods or recommendations for producing time series data from
   raw data sources
 - provide any methods or recommendations for oracle output data generation from
   target time series data

## Decision

### Defining scope to time-series or step-ahead targets
- The idea is that a time-series file will include only time series data for
  step-ahead targets. For example, the time series data would not include 
  values of seasonal targets that summarize the raw numeric time series data
  over many weeks. This is not a step-ahead target.

### Observable units
- We introduce the concept of a unique "observable unit" distinct from a unique 
  "modeling task". The distinction here is that two modeling tasks can point 
  to the same observed unit. For example, a two week ahead prediction made at 
  time $t$ and a one week ahead prediction made at time $t+1$ are predicting the same
  observable unit. 
- We establish the standard that a time-series file should have the subset of
  task-id columns that uniquely identify a single observable unit.
  - An example from a relatively simple forecast hub might be that the task
    id variables are:
    - `reference_date`
    - `target_end_date`
    - `horizon` 
    - `location`
  - In this setting, the set of `target_end_date`, `horizon`, and `location` 
    is enough to uniquely identify a modeling task.
  - However, you only need `target_end_date` and `location` to identify an 
    observable unit, so these two columns might be a natural choice for
    task-id column names for the time-series file.
  - Although it is out of scope for this proposal, we suggest that at a future
    schema iteration, including the ability to specify which task-ids variables
    are part of the observable unit would be nice.

### Time-series target data file structure

#### Time-series target data file names
- Although this was also discussed in earlier RFC, time-series files should 
  have the `"target_id"` from the `"target_metadata"` array in the filename.
  As, for example, `time-series_[target-id].csv`.

#### Hubs with one step-ahead target
- The task-id variables that are part of the observational unit are required to
  be present as columns in the time-series file.
- A simple setting is that there is just **one step-ahead target** in the hub configuration. This might be because there is just one round definition (with `"round_id_from_variable": true`) and there is only one step-ahead target in the model-tasks array, similar to FluSight. Or there might be multiple similar rounds with the same step-ahead target defined in each one (e.g., Variant Nowcast Hub).
  - In settings with one step-ahead target, the subset of columns that define the observable unit must be present in the time-series target data file.
  
#### Hubs with more than one step-ahead target

### Versioning
- We will establish the standard that target time-series data can include
  versions of data through the inclusion of an `version` column which is 
  expected to be in a ISO date format. Note that we are not providing
  flexibility in this column name or data format. However it is optional to
  include this `version` column. Note that this is based on the versioned
  data formats implemented by the [`epiprocess` R package](https://cmu-delphi.github.io/epiprocess/articles/epi_archive.html).
  
### Validation
- Validations would then be:
  - Validation of column names
    - there must be a `observation` column
    - there may be a `version` column
    - all other columns present need to be valid task-id variable names
    - there should be at least one task-id variable column.
  - Validation of data types
    - data types in the time-series target data file must match those specified
      in the hub schema file
- When versions of data are collected, updates of observations need not be
  included in updated versions unless they have changed. That is to say that
  downstream tooling will do something like: "group by all task-id variables
  and then find the value from the most recent `version`". 

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

This might help support hubs that have a more complex evaluation set up.

- PRO: might not disrupt validation too much, and would accommodate other hubs
- CON: it might break the pred/evals downstream tooling


#### New targets.json config file in hubs validated against new targets-schema.json

This would be a file that lives in `hub-config/` and would specify the
relationship between column names in time-series files and model-output.

As an example of how this could be useful:
Right now, both `hub-dashboard-predtimechart` and `hubPredEvalsData` require
individual config files that define everything the tool needs to know, which
includes some duplicated information. Any tool that's built on top of this will also
require info duplication because it relies on knowledge of the task ID columns
and what they mean in relation to each other.

Note that target data configurations are different from the model output
configrations because downstream tools need to know more information about the
columns of target data (sort order, display text, data type, dependent or
independent axis, hierarchy, etc.) in order to reliably do something with them.

- PRO: json schemas can be validated
- PRO: Could provide versions of target-data formats if things change
- PRO: existing hubs could be grandfathered in via this schema.
- CON: will require a new schema version
- CON: json config files are not particularly easy to write for non-coders
- CON: introduces more structure to a place where we aren't yet sure if it is needed.


## Status

accepted (via PR review).

## Consequences

This will create a clear promise of target-data format for downstream tools.

However, some hubs that already have systems for target-data may as a result
now be "out of compliance" with the new standards. Either they will not
be able to use the tools or they would have to spend time to bring their data 
into compliance. 

Relatedly, we may want to update some of the other hubverse functions that were
written prior to this fixed standard for target time series data, e.g. 
`hubVis::plot_step_ahead_model_output()` has separate arguments to specify the
names of the date column for the target data and model output.

## Projects

- [hubverse](https://hubverse.io/) (which does not have a project poster at the moment)
