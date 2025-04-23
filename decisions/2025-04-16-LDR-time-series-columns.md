# 2025-04-16 Time Series Columns Allowed

## Context

- We have a [narrow proposal defining target time series data](2025-02-27-rfc-time-series-target-data.md) that defines a set of columns that a time series target file MUST have.
- The above proposal was created to facilitate processing of target data by dashboard software.
- There is no guidance for additional colums that a time series file MAY have.
- The FluSight hub currently has additional target data columns that add information and transformations of the data such as a human-readable location name and a calculated rate from the target count data
- The hub-dashboard-predevals code currently filters target data based on task ID variables
- The function to read target data in hubData (development version) currently reads in all the columns of target data successfully
- The validation of target data is currently being written.

### Aims

 - provide guidance for any additional columns that MAY be included in target data

### Anti-Aims

 - specify columns that MUST be in target data

## Decision

We will allow additional columns to exist in target data with the following conditions

1. additional column names (i.e. for columns that do not correspond to
   existing task IDs) MUST NOT be identical to existing task ID names, or
   one of "value", "output_type", "output_type_id", or "oracle_value"
2. content of additional columns MUST NOT result in duplication of rows in the
   required task ID columns---that is, the additional columns should not
   represent a dependent subset of another column.

### Other Options Considered

#### Disallowing additional columns

We had considered disallowing additional columns, but decided to allow
additional columns to make sure target data could be used beyond the hubverse
tooling. While we cannot validate these columns, we can still validate the
columns that are required.

#### Allowing for additional columns that result in data duplication

Columns that represent dependent subsets would be problematic for validation
because they would likely result in a transformed value column. This would
create extra work to validate time series data.

## Status

PROPOSED

## Consequences

- Validation of time series data will be restricted to the required task ID columns
- Current implementations do not have to change
- Target data for existing hubs will remain valid

## Projects

hubverse
