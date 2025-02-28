# Time-series target data proposal

## General idea/Aims/Requirements

* Prescribe a file-location and \-format for time-series target data that can be validated and used to support dashboard visualization and potentially could be used for other purposes, such as modeling.
* The format for time-series target data will apply to targets that are defined as “step-ahead” targets in the target-metadata and where the target\_type metadata field is one of “continuous” “discrete” “binary” or “compositional” (i.e. the observation is numeric). \[Note: other data, including time-series data, could be stored in target-data but all files whose name or path starts with “time-series\*” (or possibly “oracle-output\*”, depending on how we end up setting that up) will be subject to validation.\]
* We define a time-series target-data standard for single files and hive-style partitions where the top-level partition is based on the as\_of date. In any situation where data are versioned using an “as\_of” partition or column, the full version must be made available, no support for “diffs” from previous versions will be supported.
* All time-series target data files must either be .csv or .parquet. No mixing of file types will be allowed.
* Validations are enforced based on properties of the target that can be read from the tasks.json file.
* A target can be represented in a time-series file as long as the definition of that target-id is consistent across all rounds/tasks. For example, as long as the names of the task-id variables are the same across all rounds, then it is easy to understand which columns should be present in a time-series file. If those column names change, then it becomes more complicated to support.

## Summary of the proposal

- One time-series.csv (or .parquet) file or partitioned data set contains all of the time-series target data that can be validated for numeric variables in the hub.
- This file has the following columns:
  - “as\_of”: optional, an ISO date that says when the data were available
  - \[a subset of task-id variables\]
    - these variables define the unit of observation for each observed value
    - one task-id variable must be an ISO date, corresponding to the date of the observed event
    - initially, they just need to be task-id variables, and we will later implement a metadata field/validation about which task-id variables these are.
    - they must be readable as the data type for the task-id variable in the schema
  - “observation”: a numeric observed value for the unit


## Some background and an example

- What are target keys and how could they be used for this specification?
  - Within each task group, there is a target\_metadata array with one entry per target that is addressed in that task group.
  - Within each of those entries, there is a block that has the form
    - "Target\_keys": { \<name of task id variable\> : \<task id variable value\> }
  - Example for a hub that tracks covid cases, hospitalizations, and deaths, recorded via the “target” task id variable:
    EXAMPLE 1: multiple targets
    target\_metadata: \[
        {
            “Target\_id”: “cases”,
             …,
            “Target\_keys”: {“target”: “cases”},
            …
        },
        {
            “Target\_id”: “hospitalizations”,
             …,
            “Target\_keys”: {“target”: “hospitalizations”},
            …
        },
        {
            “target\_id” : “deaths”,
             …,
            “Target\_keys”: {“target”: “deaths”},
            …
        }
    \]

Then the time-series.csv would look like this

| as_of | target | date | location | observation |
| :---- | :---- | :---- | :---- | :---- |
|  | cases |  |  |  |
|  | cases |  |  |  |
|  | … |  |  |  |
|  | deaths |  |  |  |
|  | deaths |  |  |  |
|  | … |  |  |  |

EXAMPLE 2: one target, with target\_keys: null
target\_metadata: \[
    {
        “target\_id”: “cases”,
         …,
        “target\_keys”: null
        …
    }
\]

## Validations

A file hub/target-data/time-series.csv (or .parquet, referred to below as .csv for simplicity) is valid if

1. all model-task blocks in the rounds property of the tasks.json file with the named target-id in target-metadata have
   1. is\_step\_ahead: true
   2. target\_type metadata field is one of “continuous” “discrete” “binary” or “compositional”
   3. the same task-id variable names
   4. the same task-id variables specified as what define the observable unit (Note: we can only perform this validation if we have a system for indicating which task-id variables define the observable unit. We will not implement this validation to start, but may add it later when schemas are updated.).
2. The column names of time-series.csv are
   1. “observation”
   2. The task-id column names that define the observable unit
      1. in our initial implementation, these are defined by the hub admin and not validatable other than that they are task id variable names listed in the hub's `tasks.json` file. At a future time, we could implement a metadata field to capture these so it is validatable.
   3. (optionally) “as_of”
3. The [data types](https://hubverse.io/en/latest/user-guide/tasks.html#details) within each column are readable as a consistent data type.
   1. For “observation” column: readable as numeric
   2. For “as_of” column: ISO date
   3. For other task-id columns: the same data type as the task id variable defined in the tasks.json file.
4. Values in each task-id column are valid values per the model task definition
   1. Note that some hubs may have different allowable values for different rounds/tasks that have the same target-id. This is ok, and the options here are (1) to have the union of all possible optional/required values be valid values here, or (2) do something more specific based on round-specific values.
      1. fine-grained validation might also be possible here based on minimum/maximum or unique specified values in the output\_type configuration.
5. One task-id column in the time-series.csv file must be interpretable as a date, corresponding to the date of the event or observation.

## Example 1: FluSight Hub

This hub’s [tasks.json](https://github.com/cdcepi/FluSight-forecast-hub/blob/main/hub-config/tasks.json) has this structure:

* One round, defined by “reference\_date”
* four model\_tasks
  * flu hosp rate change
    * target\_type \= ordinal → observations are labels, not numeric
  * wk inc flu hosp
    * target\_type \= continuous
    * output\_type \= quantile, sample
    * task-id vars
      * reference\_date
      * target
      * horizon
      * location
      * target\_end\_date
    * obs-unit-vars
      * target\_end\_date \= date
      * location
  * peak inc flu hosp → NOT step ahead
  * peak week inc flu hosp → NOT step ahead

So then valid target data would look like

- target-data/time-series.csv (or .parquet)
  - “as_of” (optional)
  - “target\_end\_date”
  - “location”
  - “target” (this is optional since only one target would be included, but might be good to be explicit)
  - “observation”: continuous data

## Example 2: Variant Nowcast Hub

This hub’s [tasks.json](https://github.com/reichlab/variant-nowcast-hub/blob/main/hub-config/tasks.json) has this structure:

* One round defined each week, with only difference being in the options for “clade” each round.
* Note that this hub has “target\_keys”: null, so there is no explicit “target” column in the model\_output file. It is the only hub that we know of so far that does not have a “target” column.
* one model\_task per round
  * clade prop
    * target\_type \= compositional
    * output\_type \= mean, sample
    * task-id vars
      * nowcast\_date
      * target\_date
      * location
      * clade
    * obs-unit-vars
      * target\_date \= date
      * location
      * clade

So then valid target data would look like

- target-data/time-series\_clade-prop.csv (or .parquet)
  - “as_of” (optional)
  - “date” (or “target\_end\_date”, depending on whether we can implement a column mapping)
  - “location”
  - “clade”
  - “observation”: continuous data between 0 and 1 inclusive

## Alternatives considered

### Only supporting a single file, no partitions

The argument for this is that partitions for target data are premature optimization.

- We can think of a single example where target data was large enough to possibly warrant a partition (county-level, daily-scale counts for the original US covid-19 forecast hub). In that example, one version of time-series target data for one target had 3m rows of data. The vast majority of other hubs have much smaller target data files.
- In general, target data files are smaller than model output files because they only need one row for each observation, whereas model output files may have dozens or hundreds of rows for each modeling task. And modeling tasks in different rounds may correspond to the same observation.

However, further discussion showed that to make a single file viable would require using a “diff” strategy to store only updated values. This leads to substantial accessibility issues, where additional processing would be needed to read even a single version of the time-series data.

### Supporting multiple single files, one per target

It was agreed that this is the least accessible option. It would always require multiple bespoke reads of individual files, rather than taking advantage of the hive style partition structure or of just storing all the data in one place.

### Adding a new schema property to define the observed unit variables

The idea here would be to add a new schema property that defines which task-id variables correspond to the variables that define the observational unit. This mapping could also allow for renaming certain variables, e.g. task-id variable of “target\_end\_date” could just be “date” which would be more natural in the context of a time-series target data file.

This is something that we would like to do at some point, but doesn’t feel essential to making this work in the short term.

### Allowing for additional partition structure

It has been suggested that allowing for partitioning time-series target data by target, and/or allowing for multiple single files to store data might also be reasonable. One possible justification for this is operational: some targets come from different data sources and are updated at different times.

This is also something that we could consider explicitly guaranteeing support for across all hubverse tools at a future date. It is possible that from a technical perspective, any file structure supported by an `arrow::open_dataset()` read operation will allow for validation and reading in of target data with the current implementation. However, we decided for accessibility purposes we would start with only explicit guaranteed support (not just for reading data in, but also for the dashboard tools) for the formats described above (single file, and hive-style partition on `as_of` column).

Some arguments against adding additional structure are

- partitions are unfamiliar to some and are harder to interact with (nested folders with single files). adding a second layer would make this even harder.
- it might add development/documentation complexity and we are trying to avoid that in the short-term.
- code for the visualization dashboard currently is custom python code and adding support for multiple different structures might add complexity.

## Using "version" as the column name instead of "as_of"

We are following the Delphi group's standards about using "version" to refer to data that may not contain all of the observations (like "diffs) and "as_of" to refer to a complete dataset. More detail in [their vignette on archives of data](https://cmu-delphi.github.io/epiprocess/articles/archive.html).

## Status
Approved, with minor modifications that have been made to this version.
This proposal has been discussed extensively, including in [this PR](https://github.com/reichlab/decisions/pull/18), and in numerous synchronous meetings. It is the product of many of these discussions.
