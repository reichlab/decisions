# Project Poster: [project name]

- Date: 2024-12-13

- Owner: Evan L. Ray

- Team: Zhian N. Kamvar, Evan L. Ray

- Status: draft

## ❓ Problem space

2024-11-13: this is summarized from [the planning document Evan created](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/64276d6cf10bffa74d27282f7e6e17054adb813d/project-posters/eval-dashboard/2024-12-13-planning/Evaldashboardplanning.html)

### What are we doing?

Website that can be used to explore forecast performance.
 - Overall performance by model
 - Performance broken down by levels of one task id variable (e.g. by location, by reference date, by horizon, by target date)
 - Scores may be represented using:
     - Tables
     - Line plots: one line per model showing scores over time, scores by location, etc.
     - Heat maps: fill color corresponds to score value, models in rows or columns, task id variable values in columns or rows
 - Key question: how to handle missing values (e.g. a team submits only for California, or teams submit for different subsets of dates).
     - Past work (see next section) has used 2 main strategies:
         - “Relative” scores that use a pairwise tournament approach in which
           each pair of models is compared on the subset of forecast tasks they
           produced in common.  See, e.g., the reichlab flusight eval tool
           linked in next section.
         - In an interactive tool, when a specific set of models is selected,
           subset to the forecast tasks the selected models have in common and
           aggregate scores over those tasks dynamically.  See, e.g., the
           delphi tool linked in next section.
     - The first of these options is substantially easier from a technical perspective, so I (Evan) put in a strong vote for that one.

### Why are we doing this?

TODO: We needs

### What are we _not_ trying to do?

Anti-aim: more than one task id variable is actually an anti-aim for now because it's more complex, but in practice it's what is normally done

### How do we judge success?

A list of outcomes that should be true for the project to be considered a
success. Be sure to focus on outcomes, not implementation details.

### What are possible solutions?

NOTE: A detailed [writeup can be found as an HTML document in this folder](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/64276d6cf10bffa74d27282f7e6e17054adb813d/project-posters/eval-dashboard/2024-12-13-planning/Evaldashboard.html).

#### Score Creation

1. R or python script runs at round close to compute scores and save them e.g. in an S3 bucket or on GitHub (R easier as of Dec 2024; existing R tooling for computing scores).

#### The website itself

1. R markdown document, quarto document, or similar that pre-computes all
   tables/figures used to display them.
   - Note: we’ve tried this before for flusight evals, and were not satisfied
     with the results: when breaking scores down by location, the eval page took
     ~30s to load.
   - Retrieving data based on user selections and re-rendering new content is
     not possible using this option.
   - This motivates the following two options, with dynamic data fetches and
     rendering of web page elements
2. Front-end Javascript app running in client side browser fetches data as
   needed and generates plots/etc: similar structure to predtimechart
3. Web app with server-side logic to render tables and/or figures, e.g. using
   Shiny or Streamlit
   - It would likely still make sense to set up scheduled end-of-round
     computation of scores to cache, as score computation can be too
     time-consuming to do on the fly in the context of an interactive app.
4. webR or similar: R running in the browser
   - This may work, but when we looked in June 2024 it wasn’t ready for
     prime-time yet.
   - https://shinylive.io/r/examples/ 
   - NOTE: Shifting to using webR from a javascript app at a later date would not
     require changing the rest of the dashboard structure as a static site.

## ✅ Validation

### What do we already know?

Prior art in two dashboards: one static, the other server-generated

 - https://reichlab.io/flusight-eval/
 - https://delphi.cmu.edu/forecast-eval/

The static dashboard is built from R Markdown and runs pretty slow (and is quite
large due to embedded elements). 


### What do we need to answer?

From [Output data that we will compute](https://htmlpreview.github.io/?https://github.com/reichlab/decisions/blob/64276d6cf10bffa74d27282f7e6e17054adb813d/project-posters/eval-dashboard/2024-12-13-planning/Evaldashboard.html#h.gdnhn91qg7og):

 - Note, we may also need to say what output types to use to compute scores?
   E.g., for wk inc flu hosp, forecasts are submitted in both quantile and
   sample format.  We might compute WIS and MAE based on the quantile
   forecasts, and energy score based on the sample forecasts?
   - It may be possible to infer the desired output type to use for computing
     scores based on the name of the score; propose to do nothing about this
     until we realize it won’t work.
 - For energy score, do we need to specify what the compound unit is?  This
   gets complicated, and is a bridge we don’t have to cross quite yet.

## 👍 Ready to make it

If, after defining the problem space and validating assumptions, the team
decides to move forward with the project, this section is to document
a proposed solution.

Keep the answers to the prompts below brief--this document isn't
inteded to be a detailed project plan.

### Proposed solution

1-2 sentences about the team's chosen approach for tackling the problem space.

### Visualize the solution

Optional. If there is a mockup, data dictionary, or other artifact that clarifies
the project's implementation, you can put it here (or link to it).

### Scale and scope

A place to define items like:

- Required team size
- Required delivery date (if applicable)
