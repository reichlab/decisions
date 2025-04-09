# 2025-03-09 Standardize Hubverse model-output directory name

## Context

The Hubverse currently allows hub admins to customize the name of their hub's
`model-output` directory. This customization occurs in the `model_output_dir`
section of the `admin.json` configuration file.

This ability to customize is inconsistent with the approach we took when
recently defining Hubverse target data standards, which state that a directory
named `target-data` is required.

The Hubverse developer team has had recent discussions about following the
target data approach and standardizing the name of Hubverse model output
directories to `model-output`.

### Aims

The goals of standardizing the model output directory name are:

- Making it easier for downstream users to navigate the Hubverse and to
  understand where data are located.
- Simplifying internal Hubverse code by removing the need to check `admin.json`
  before reading model output files.
- Avoid breaking two Hubverse code bases that assume model output files are
  in a directory called `model-output`.

### Anti-Aims

- This is technically a breaking schema change, so we want to avoid disrupting
  existing hubs, particularly those that are currently active.
  After some research, we have determined that all known (public) hubs are using
  the default name of `model-output`.
  [This spreadsheet](https://docs.google.com/spreadsheets/d/1c8Lo07FeylmOFXd1ud_hXapoyoj_SqCKWcwFPZVVt10/edit?gid=0#gid=0)
  has more information.

- We do not want to introduce time-consuming work to the Hubverse developers.
  Some research into the impact of this change determined that hub validations
  would not be impacted, so there's no immediate extra work beyond the schema
  revision.

## Decision

The next revision of the Hubverse schema will remove the `model_output_dir`
component from the `admin.json` configuration file.

### Other Options Considered

We considered retaining the status quo and leaving the model output directory
name configurable. However, we decided that making this change now, when
it doesn't impact existing hubs, is ideal timing.

## Status

Proposed

## Consequences

- This update required a breaking schema change (we don't expect it to impact
  existing hubs).
- When the Hubverse is ready to stop supporting schema versions 5.0 and lower,
  we can remove the `hubUtils::get_hub_model_outuput_dir()` function and
  update that function vall from `hubValidations`.

## Projects

- None
