# 2024-12-13 Orchestration of the Dashboard Build Process

## Context

[The hubverse dashboard](../project-posters/hub-dashboard/hub-dashboard.md) is
a project that generates a website with auto-generated dashboards for forecasts
and step-ahead forecast visualizations.

This project has three main components:

1. static site generator
2. forecast visualization generator
3. evaluations dashboard generator

This project requires two data sources:

1. a modeling hub that contains model output data
2. a dashboard repository that defines parameters for the site and the dashboard

The admins of at least one hubverse project has expressed a desire to not manage
any more workflows.


### Aims

 - define a method for generating a hubverse dashboard website using the three
   main components
 - define places where the outputs for each of the three components should live
 - provide a solution that does not significantly complicate the workflows of
   hub admins
 - provide a solution that does not add significant maintenance burden for the
   hubverse core team

### Anti-Aims

 - TBD

## Decision

 - We will use [re-useable GitHub
   workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows#limitations)
   to orchestrate each of the build process---these workflows can be reused in
   any repository.
 - We will generate artifacts from each workflow and store them as separate
   orphan branches in the dashboard repository.
 - We will build an experimental GitHub App that allows hub admins to
   optionally register their repository to build the dashboard using our
   centralized workflows.
 - We will host the App's webhook code on a free service like glitch.io

### Other Options Considered

 - A set of GitHub workflows only
   - **Not chosen because of maintenance burden for hubverse admins** (see 
     [Zhian's comment about distributed workflows](https://github.com/reichlab/decisions/pull/3#discussion_r1884480624))

## Status

proposed

## Consequences

TBD

## Projects

 - [The hubverse dashboard](../project-posters/hub-dashboard/hub-dashboard.md)
