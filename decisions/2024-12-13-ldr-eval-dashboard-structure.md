# 2024-12-13 Evals Dashboard Underlying Architecture

## Context

We need to produce a dashboard for model evaluations for both the FluSight and
COVID forecast hubs. The first decision that we need to make is the question
about what technologies we can use to make the dashboard.

We have made a promise that the dashboard would be available in early 2025 to both
groups.

There is uncertainty with respect to the funding situation for the development
and maintenance of tools in the lab due to the results of the 2024 US Elections. 

It is important to choose a toolset for this dashboard that can be maintained
under a future where members are fluent in R, but may not be comfortable with
JavaScript.

### Aims

 - Define the underlying architecture for the hub evals dashboard

### Anti-Aims

 - Define the behaviour of the dashboard 

## Decision

Regardless of our choice of system, we will have an R script that pre-computes
data for the tables and figures and store them on a git branch or s3 bucket.

We will build the dashboard as a front-end javascript application similar to
predtimechart to align with the current status of the hub dashboard project.

### Other Options Considered

1. R markdown document, quarto document, or similar that pre-computes all
   tables/figures used to display them.
   - Note: we’ve tried this before for flusight evals, and were not satisfied
     with the results: when breaking scores down by location, the eval page took
     ~30s to load.
   - Retrieving data based on user selections and re-rendering new content is
     not possible using this option.
   - **Not chosen because of the above limitations of inflexibility**
2. Web app with server-side logic to render tables and/or figures, e.g. using
   Shiny or Streamlit
   - It would likely still make sense to set up scheduled end-of-round
     computation of scores to cache, as score computation can be too
     time-consuming to do on the fly in the context of an interactive app.
   - Streamlit serves ads and shiny.io servers are limited by the number of
     minutes website visitors use.
   - **Not chosen because this requires server-side logic that may not be
     compatible with a static site as the current hub dashboard project builds
     on.**
3. webR or similar: R running in the browser
   - This may work, but when we looked in June 2024 it wasn’t ready for
     prime-time yet.
   - https://shinylive.io/r/examples/ 
   - NOTE: Shifting to using webR from a javascript app at a later date would not
     require changing the rest of the dashboard structure as a static site.
   - **Not chosen because it is still undergoing active development and we may
     be able to switch to it later.**

## Status

proposed

## Consequences

 - We will be able to continue movement on this project
 - Javascript is not a familiar language to many developers in the lab, so this choice creates an additional dependency
   burden
 - Other lab members can be consulted for advice on JavaScript frontend
   applications
 - Server-side dependency management is not necessary

## Projects

 - [2024-12: Evals Dashboard](../project-posters/evals-dashboard/evals-dashboard.md)
