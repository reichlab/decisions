# 2025-02-19 Output type conversion functionality

## Context

We have discussed the utility of being able to convert predictions of one output type to another, and there are scattered attempts to implement this functionality in various states of completeness, but we have yet to unify these attempts or fully commit to a concrete plan to implement this output type conversion.

Specifically, we are interested in looking into implementation of conversions from samples to other output types. The sample output type is only partially supported by hubverse infrastructure, meaning that converting it to another output type would allow for users to utilize more of our tooling.

### Aims

- Update our overall plan for how output type conversion will be implemented to be congruent with the current state of the hubverse
- Make a detailed plan of how to code up conversions from samples to other output types

### Anti-Aims

- Implementation of conversions with an initial output type other than "sample" will not be explored in detail or prioritized with this decision
- This proposal does not cover how to determine if a particular conversion from initial output type A to terminal output type B should be implemented (though some ideas are spec'ed out in the supplemental vignette)
- Any proposed names/terminology in the supplemental vignette are subject to change

## Decision

We will begin by implementing output type conversions from initial output type "sample" to terminal output types "quantile" and "median" and "mean", then "PMF" and "CDF". This prioritizes the simple conversions first, with each subsequent conversion increasing in complexity. Although output types like "quantile" and "median" could be implemented very simply using the `quantile()` function (and likewise "mean" with the `mean()` function), we are interested in building the first pass at this function with an eye ahead to expanding support to the case of output type IDs whose associated values depend on task ID variables. For example, FluSight's weekly incident hospitalization rate category target (PMF output type) has output type IDs (categories) whose definition depend on location and horizon; i.e., the definition of a "high increase" in the rate of weekly incident hospitalizations looks different in New York compared to Alaska, just as "high increase" is defined differently for a horizon of 1 week versus 4 weeks, even in the same state or territory.

### Other Options Considered

We considered starting with a case that included output type IDs which depend on task ID variables, but we prefer to implement support incrementally and speeding up development by tackling the simpler cases first. (Our initial thoughts about output type conversion involved implementing many conversions at once, but for similar reasons as stated previously we decided to stick to conversions from samples first.)

## Status

Proposed

## Consequences

Pros:

- Implements several high priority output type conversions relatively quickly
- Allows hubs that collect samples to make use of more hubverse infrastructure that does not yet support (or fully support) the sample output type
- Solidifies of roadmap for development of output type conversion functionality

Cons:
- Some more complex conversions may be put off until later

## Projects

- None
