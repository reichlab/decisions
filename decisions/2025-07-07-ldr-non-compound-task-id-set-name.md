# 2025-07-07 Standard Name for Non-Compound Task ID Set Variables

## Context

The set of task ID variables not included in the compound task ID set are used in several internal hubverse functions ([`hubEnsembles` helper `validate_compound_taskid_set()`](https://github.com/hubverse-org/hubEnsembles/blob/main/R/validate_ensemble_inputs.R#L164) and [`hubValidations` function `check_tbl_spl_non_compound_tid()](https://github.com/hubverse-org/hubValidations/blob/main/R/check_tbl_spl_non_compound_tid.R#L19)). This concept can be thought of as the set of task IDs that are allowed to vary within a single sample index value. Given its appearance across several packages, it's worth deciding on a formal, standard name.

Additionally, it has been noted that some of the terminology around the compound task ID set, unique modeling tasks, etc is confusing to the wider community (see [this issue on sample terminology](https://github.com/hubverse-org/hubDocs/issues/169)).

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
- Compound_taskid_set = [“reference_date”, “target”, “horizon”]
- Other task ids = [“location”, “target_date”]
- We collect joint distributions across locations, separately within each reference_date/target/horizon/target_date combination

**Example 3:**
- Compound_taskid_set = [“reference_date”, “target_date”]
- We should have an error/this is Bad: the derived task id horizon is derived from reference_date and target_date, so it should live in there.

## Decision

We will stick with the current name, "non-compound task id set". Given that this name is only used in internal functions, avoids introducing new terminology, and will not introduce breaking changes, we feel it is best to not make any changes.

### Other Options Considered

Possible alternative names include:
- task_id_vars_with_dep
- joint_task_ids or jointly modeled task ids

## Status

Proposed

## Consequences

Gives a formal and standard name to this set of variables. If we decide it should differ from "non-compound task ID set", some functions in `hubValidations` will need to be renamed, which constitute breaking changes.

## Projects

N/A
