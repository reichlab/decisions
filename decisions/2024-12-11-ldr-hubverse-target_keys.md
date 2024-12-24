# 2024-12-11 target keys should contain single value

## Context

This was discussed in the 2024-12-11 hubverse developer meeting.

In `target_metadata`, we allow for `target_keys` to be multiple fields. E.g.,
if a hub has 4 targets given by "incident hospitalizations", "cumulative
hospitalizations", "incident cases", "cumulative cases", these could be
represented by two variables called `inc_cum` \in `{"incident", "cumulative"}`
and signal \in `{"hospitalizations", "cases"}`.

Downstream analysis will benefit from having one column to use as a partitioning
key.

No hubs in public repositories use multiple `target_keys`; If any do, the
switch from to one target key should be fairly simple.

### Aims

 - simplification of hub configuration and downstream parsing.

### Anti-Aims

 - disrupt operations for currently running hubs.

## Decision

We will communicate this decision over the hubverse mailing list, slack, and the
GitHub discussions repository.

If we find any hubs that use multiple target keys, we will provide support for
them to transition to the updated schema.

We will update our schemas to specify that `target_keys` should contain exactly
one property if it is not `null`.

We will increment the version of the schemas to v4.0.1.

We will update the documentation to refelect this decision.

### Other Options Considered

 - Option: Not restricting the number of target keys
   - PRO: current state of affairs, allows potential for nuanced targets
   - CON: not currently supported in our downstream tools; would require extra
     coding effort to support
   - NEUTRAL: no hubs currently use multiple target keys

## Status

accepted (via meeting)

## Consequences

- POSITIVE: coding for downstream tools will be simpler and easier to maintain
- NEGATIVE: any hubs that use multiple target keys will need to remain at the
  schema version they currently use or migrate
- NEUTRAL: schemas will be updated
- NEUTRAL: documentation will be updated

## Projects

 - The hubverse (no project poster as of 2024-12-24)
