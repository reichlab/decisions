# 2025-07-07 Standard Name for Non-Compound Task ID Set Variables

## Context

The set of task ID variables not included in the compound task ID set are used in several hubverse package functions ([`hubEnsembles` helper `validate_compound_taskid_set()`](https://github.com/hubverse-org/hubEnsembles/blob/main/R/validate_ensemble_inputs.R#L164) and [`hubValidations` function `check_tbl_spl_non_compound_tid()](https://github.com/hubverse-org/hubValidations/blob/main/R/check_tbl_spl_non_compound_tid.R#L19)). This concept can be thought of as the set of task IDs that are allowed to vary within a single sample index value. Given its appearance across several packages, it's worth deciding on a formal, standard name.

### Aims

  - Determine a formal, standardized name for task ID variables not part of the compound task ID set
  - If the final name is different from "non-compound task ID set", update any references to this group of variables

### Anti-Aims

  - Anything other than deciding on a formal name and (if necessary) updating any references to this group of variables

### Examples of compound task ID sets

Assume that a hub has the following task id variables:
- reference_date
- horizon
- target_date
- location
- target

**Example 1:**
- compound_taskid_set = [“reference_date”, “location”, “target”]
- Other task ids = [“horizon”, “target_date”]
- We collect trajectories across (horizon, target_date), within each reference_date/location/target combination

**Example 2:**
- Compound_taskid_set = [“reference_date”, “target”, “horizon”, “target_end_date”]
- We collect joint distributions across locations, separately within each reference_date/target/horizon/target_date combination

**Example 3:**
- Compound_taskid_set = [“reference_date”, “target_date”]
- We should have an error/this is Bad: the derived task id horizon is derived from reference_date and target_date, so it should live in there.

## Decision

This section describes our response to these forces. It is stated in full sentences, with active voice. "We will ..."

### Other Options Considered

Possible names include:
- non_compound_task_ids (current)
- task_id_vars_with_dep
- joint_task_ids or jointly modeled task ids

## Status

Proposed

## Consequences

Gives a formal and standard name to this set of variables. If we decide it should differ from "non-compound task ID set", some functions in `hubValidations` will need to be renamed, which constitute breaking changes.

## Projects

 - a list of links to project posters affected by this decision