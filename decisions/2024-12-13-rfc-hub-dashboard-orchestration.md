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

The admins of at least one hubverse project have expressed a desire to not manage
any more workflows.

### Aims

 - define one or more potential methods for generating a hubverse dashboard
   website using all or a subset of the three main components
 - define places where the outputs for each of the three components should live
 - provide a solution that does not significantly complicate the workflows of
   hub admins
 - provide a solution that does not add significant maintenance burden for the
   hubverse core team
 - Provide a solution that is secure. For instance, a solution without proper
   attention to security could allow bad actors to insert code that would
   insert malicious javascript in the dashboard websites.

### Anti-Aims

 - define a single solution that we will use from now on.

## Decision

### Centralized workflow

The centralized workflow is a set of workflows that lives in a single repository
that can be used by either solution.

 - We will use [re-useable GitHub
   workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows#limitations)
   to orchestrate each of the build process---these workflows can be reused in
   any repository.
 - We will generate artifacts from each workflow and store them as separate
   orphan branches in the dashboard repository.
 - We will create a template repository for hub admins to use as the basis for
   their website.

### GitHub App: No-code solution for Hub Admins

 - We will build an experimental GitHub App that allows hub admins
   to optionally register their repository to build the dashboard using our
   centralized workflows.
 - We will host the App's webhook code on a free service like glitch.io

To satisfy the aim of avoiding the addition of a siginficant maintenance burden
on hub administrators, this potential solution uses a GitHub App, which requires
no workflow setup on behalf of the administrators.

- PRO: does not require a new workflow to maintain for hubverse adminstrators
- PRO: allows for a richer set of interactions via GitHub's webhooks API (i.e.
  issue comment commands)
- PRO: new optional features are trivially added
- CON: workflow time is run on our GitHub Account
- NEUTRAL: GitHub provides open source repositories with unlimited run time
- CON: requires core team maintenance of additional JavaScript code
- CON: requires an additional service like glitch.io or AWS Lambda
- CON: requires additional management of tokens
- NEUTRAL: uses a GitHub App

### A template workflow that calls out to the centrally-maintained re-usable workflows

 - We will build out a method in which each hub dashboard repo has
   relatively-lightweight workflows that call centrally-maintained reusable
   workflows.
 - We will save these workflows in a template repository.

Similar to current Hubverse-developed GitHub actions, but putting most of
the heavy lifting in centrally-maintained reusable workflows. Hub admins copy
a (relatively small) workflow to their dashboard repo and update it as needed.
Configured to run on a cron schedule and manually as needed. This satisfies the
aim of avoiding a significant maintenance burden for the core hubverse team.

- PRO: Doesn't require a security token that has write access to a control
   room-type repository
- PRO: Doesn't require API/web service
- CON: Increased maintenance burden for hub admins
- CON: Potential increased support burden for hub devs
- CON: New optional features may require workflow modifications by hub devs
- NEUTRAL: Does not use GitHub app

### Other Options Considered

#### Centralized workflows invoked by individual hub repos via PAT

Hub dashboard repos have a small boilerplate workflow that invokes the control
room workflows that perform dashboard tasks. Hub admins are provided a
personal GitHub Personal Access token with write access to the control room
repo. Hubverse organization owners are required to maintain and rotate the
token.

- PRO: Centralized workflows are easier for hub devs to maintain
- PRO: Doesn't require API/web service
- PRO: Less workflow boilerplate for hub admins (but not zero)
- CON: Decentralized secrets management---sharing a PAT for the central repo
   is a security risk for that repository
- NEUTRAL: Does not use GitHub app

#### Centralized workflows invoked by individual hub repos via GitHub App token

Hub dashboard repos have a small boilerplate workflow that invokes the control
room workflows that perform dashboard tasks. Hub admins store the GitHub app
token as repo secret and use it when invoking the control room workflows.

- PRO: Centralized workflows are easier for hub devs to maintain
- PRO: Doesn't require API/web service
- PRO: Less workflow boilerplate for hub admins (but not zero)
- CON: Sharing the GitHub App token is a security risk for any repository that
   installs that GitHub App
- NEUTRAL: Uses a GitHub app

#### Centralized workflows automatically copied to individual dashboard repos

Dashboard workflows are maintained in the control room app, but some kind of
CI/build process automatically adds (or updates) them to individual hub
dashboard repos.

- PRO: Benefits of centralized repo + sidesteps need for hub dashboard repos to
   have write access to control room app
- PRO: Doesn't require API/web service
- CON: In practice, keeping things synchronized between individual repos and a
   central repo is difficult because it requires careful attention to the
   permissions of GitHub tokens.
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

ACCEPTED

## Consequences

 - We will maintain a central repository for these workflows. This will not
   change based on our decision.
 - We will spin up a GitHub App and host it on an external service.
 - We must come to a decision on the appropriate method at a future date.


## Projects

 - [The hubverse dashboard](../project-posters/hub-dashboard/hub-dashboard.md)
