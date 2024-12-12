# Project Poster: The Hubverse Dashboard

- Date: 2024-12-12

- Owner: Evan L. Ray

- Team: Matthew Cornell, Zhian N. Kamvar, Evan L. Ray

- Status: in progress

## ❓ Problem space

### What are we doing?

Developing a theme-able and low-maintenance dashboard for hub admins that
will publish a Predtimechart vizualization, Evaluations, and any associated
markdown contents when each round closes.

Because hubs can be complex, we are restricting this to hubs that align with
predtimechart's ability for vizualization: forecast hubs that have step-ahead
quantile predictions.

### Why are we doing this?

We are supporting timely communication of modeling hub results to
decision-makers.

Many, but not all hub administrators are proficient with GitHub or static sites.
For example, when the idea of a dashboard site was brought up to the community,
one adminstrator mentioned that this would be a huge help, but her team had no
capacity for maintaining another piece of infrastructure.

Similar websites exist for established efforts:

- https://respicast.ecdc.europa.eu/forecasts/
- https://covid19forecasthub.org/

However, the machinery to build these websites and visualizations was not
trivial. They required knowledge of R Markdown, JavaScript, and GitHub Actions
and that was just for the website.

These sites are run in conjunction with active periods for modeling hubs and it
should be possible for a hub admin to "set it and forget it"---that is, they 
should be able to add the information they need to add and not have to touch it
for another several momnths.

### What are we _not_ trying to do?

We are _not_ trying to make a generalizable dashboard that can accomodate all
hub types.

### How do we judge success?

A successful outcome is a deployable website that can:

 - display an up-to-date predtimechart visualization with data from a specified hub
 - dispaly an evaluation report of model outcomes from the same specified hub
 - include custom markdown pages to describe the purpose of the hub
 - be customizable wrt to theme
 - automatically be built when a modeling round closes
 - be built on demand
 - does not require knowledge of anything more than Markdown or YAML
 - render equations
 - be independent from a hub

### What are possible solutions?

There are four pieces that are required for success: 

1. website generator,
2. predtimechart vis,
3. evaluations dashboard, and 
4. the orchestration of these tools for continuous deployment

Each of the elements should be independent.

#### 1. website generator

 - Ship a static site generator (Hugo, Jekyll)
 - Ship a _theme_ for a static site generator
 - Readthedocs
 - Quarto
 - R Markdown website
 - Docker container
 - streamlit

#### 2. predtimechart vis

Installable python app

#### 3. evaluations dashboard

- R markdown document, quarto document, or similar that pre-computes scores and all tables/figures used to display them.
- Two-part process:
  1. R or python script runs at round close to compute scores and save them e.g. in an S3 bucket or on GitHub
  2. Javascript app running in client side browser fetches data as needed and generates plots/etc
- Web app with server-side logic to render tables and/or figures, e.g. using Shiny or Streamlit

#### 4. orchestration

 - GitHub workflows distributed to hub adminstrators (similar to [hubverse-actions](https://github.com/hubverse-org/actions))
 - GitHub App that controls centralized GitHub workflows (similar to [the r-universe](https://github.com/r-universe-org/control-room))

## ✅ Validation

### What do we already know?

 - website: this is a small site that will be generated several times per week, so build speed is not essential
 - website: there are over 400 static site generators, and they all interpret markdown a little differently
 - website: quarto has the closest markdown syntax to GitHub
 - website: readthedocs is mostly plug-and-play (no deployment workflow), but the structure is not simple.
 - website: all static site generator templates are complex in some way
 - webiste: docker can provide an abstraction for any static site generator we want
 - website: quarto documents can be easily themed

 - evaluations: R markdown documents to precompute scores and tables generate very large HTML files that take a long time to load.
 - evaluations: If we use a static website, we are unable to use a service like shiny or streamlit

 - orchestration: GitHub provides unlimited free workflow runs for all public repositories
 - orchestration: (example) the hubverse distributed workflows sporadically require updates from the hub administrators
   - this requires extra communication directly with administrators
 - orchestration: (example) the r-universe is an opt-in process that requires little to no effort from package maintainers
 - orchestration: a GitHub app can be written with probot and hosted on glitch.io for free (up to 1000 hours per month)

### What do we need to answer?

TODO: evaluations-related.

## 👍 Ready to make it


### Proposed solution

#### Website

We have built a Docker container that builds a website based off of our custom
template and is configured by a custom `site.yml` configuration file, which is
a simplified form of a `_quarto.yml` file. During build time, it inserts the
predtimechart vis and uses Quarto on the backend.

 - [Docker container: hub-dash-site-deployer](https://github.com/hubverse-org/hub-dash-site-deployer)
 - [Website template: hub-dashboard-template](https://github.com/hubverse-org/hub-dashboard-template)

#### Predtimechart vis

We have built a python app that generates predtimechart forecast and target data
from a hub and a `predtimechart-config.yml` file. This is installable as a CLI
app. 

 - [Predtimechart vis: hub-dashboard-predtimechart](https://github.com/hubverse-org/hub-dashboard-predtimechart)

#### Evaluations

TODO

#### Orchestration

We have built a centralized set of GitHub workflows that will build the website
and the vis data separately and store them in separate branches of the dashboard
repository. Dashboard repositories can use this by installing the GitHub App on
to the dashboard repository, giving the app permission to write to the repository.
We can store the app credentials in the central repository and use them to 
generate the site and push back to the dashboard repository. 

The websites and/or data are built in two ways:

1. on a schedule (via the central repository)
2. on demand (via webhook to the app that triggers the central repository)

 - [Centralized workflows: hub-dashboard-control-room](https://github.com/hubverse-org/hub-dashboard-control-room)
 - [App: hubDashboard](https://github.com/apps/hubDashbaord)
 - [Glitch hosted instance of hubDashboard](https://glitch.com/~crystal-glimmer-path)

### Visualize the solution

TODO: port over diagrams
