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

#### A set of GitHub workflows copied to individual hub dashboard repos calling centrally-maintained reusable workflows

Similar to current Hubverse-developed GitHub actions, but putting most of
the heavy lifting in centrally-maintained reusable workflows. Hub admins copy
a (relatively small) workflow to their dashboard repo and update it as needed.
Configured to run on a cron schedule and manually as needed.

- PRO: Doesn't require a security token that has write access to a control
room-type repository
- PRO: Doesn't require API/web service
- CON: Increased maintenance burden for hub admins
- CON: Potential increased support burden for hub devs
- NEUTRAL: Does not use GitHub app

#### Centralized workflows invoked by individual hub repos via PAT

Hub dashboard repos have a small boilerplate workflow that invokes the control
room workflows that perform dashboard tasks. Hub admins are provided a
personal GitHub Personal Access token with write access to the control room repo. Hubverse organization owners are required to maintain and rotate the token.

- PRO: Centralized workflows are easier for hub devs to maintain
- PRO: Doesn't require API/web service
- PRO: Less workflow boilerplate for hub admins (but not zero)
- CON: Decentralized secrets management
- NEUTRAL: Does not use GitHub app

#### Centralized workflows invoked by individual hub repos via GitHub App token

Hub dashboard repos have a small boilerplate workflow that invokes the control
room workflows that perform dashboard tasks. Hub admins store the GitHub app
token as repo secret and use it when invoking the control room workflows.

- PRO: Centralized workflows are easier for hub devs to maintain
- PRO: Doesn't require API/web service
- PRO: Less workflow boilerplate for hub admins (but not zero)
- CON: Sharing the GitHub App token
- NEUTRAL: Uses a GitHub app

#### Centralized workflows automatically copied to individual dashboard repos

Dashboard workflows are maintained in the control room app, but some kind of
CI/build process automatically adds (or updates) them to individual hub
dashboard repos.

- PRO: Benefits of centralized repo + sidesteps need for hub dashboard repos
to have write access to control room app
- PRO: Doesn't require API/web service
- CON: In practice, keeping things synchronized between individual repos and
a central repo is difficult
- NEUTRAL: Does not use GitHub app

#### Secrets vault

Hub dashboard repos have a small boilerplate workflow that invokes the control
room workflows that perform dashboard tasks. The boilerplate workflow retrieves
the GitHub app secret from a secrets value (e.g., Hashicorp Vault or AWS
Secrets Manager).

- PRO: Centralized workflows are easier for hub devs to maintain
- PRO: Doesn't require API/web service
- PRO: Less workflow boilerplate for hub admins (but not zero)
- CON: It replaces the API/web service moving part with a different moving part
(albeit one that's similar to the existing onboarding process for cloud-based
hubs)
- NEUTRAL: Uses a GitHub app
   - **Not chosen because of maintenance burden for hubverse admins** (see 
     [Zhian's comment about distributed workflows](https://github.com/reichlab/decisions/pull/3#discussion_r1884480624))

## Status

proposed

## Consequences

TBD

## Projects

 - [The hubverse dashboard](../project-posters/hub-dashboard/hub-dashboard.md)
