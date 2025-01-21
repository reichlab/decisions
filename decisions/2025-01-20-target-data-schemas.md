# 2025-01-20 Target Data Schema Proposal

## Context

All hubs need to provide target data for modelers and downstream evaluations.

Many hubs have a script that is manually run to obtain new target time series
data from raw data sources.

Few hubs have oracle output data.

There is no current standard for the contents of target data.

As described in [RFC 2025-01-06-target-data-formats]("2025-01-06-target-data-formats.md"),
we have established a structured format for target data files.

Visualizations and evaluations rely on being able to read in target data.

Current solutions for the dashboard require explicit knowledge of the target
data structure in the hub.

Obtaining historical data from hubs is difficult.

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

TBD!

### Other Options Considered

This section describes any other options that were considered, but were not chosen with a brief description of why they were not chosen.

## Status

A decision may be "proposed" if the project stakeholders haven't agreed with it yet, or "accepted" once it is agreed. If a later ADR changes or reverses a decision, it may be marked as "deprecated" or "superseded" with a reference to its replacement.

## Consequences

This section describes the resulting context, after applying the decision. All consequences should be listed here, not just the "positive" ones. A particular decision may have positive, negative, and neutral consequences, but all of them affect the team and project in the future.

## Projects

- [hubverse](https://hubverse.io/) (which does not have a project poster at the moment)
