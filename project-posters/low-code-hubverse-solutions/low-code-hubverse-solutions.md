# Project Poster: Low code hubverse solutions

- Date: 2026-09-25

- Owner: Li

- Team: Anna, Li, ... (?)

- Status: Draft

## ❓ Problem space

We are interested in knowing the specific needs of our users, and we identified private hubs as a subset to prioritize. According to the Paraguay Hub admins, their priorities lie in (further) developing a no-code solution to create usable forecasts that under-resourced public health departments can generate without technical forecasting or coding expertise. Their current solution is the [MicroHub project](https://github.com/sjfox/microhub), which uses hubverse data structures, but they would like greater integration with the hubverse to make use of more of the features.

### What are we doing?

Create more local R functions that the MicroHub can use under the hood to make use of more hubverse functionality. The following are a list of functions we will consider creating, ordered by priority:
- **`create_hub()`**: Creates a hub file structure locally (like the hubTemplate repo, but fully programmatic)
- **`save_model_out_tbl()`**: Saves a `model_out_tbl` with the correct formatting, automatically splits outputs containing multiple models into individual files within directories named using the model ID
- `create_tasks_json()`: A `usethis`-style function to create a tasks.json file by providing the values for each of the fields when prompted
- `create_model_metadata_file()`: Similar to above, but for creating a model metadata file

We may also consider creating a private dashboard that could be shared among colleagues, with a particular focus on the ability to look at multiple seasons worth of forecasts and evaluations.

### Why are we doing this?

We would like to support more users, particularly those with less GitHub or coding experience, and makes strides to (at least partially) decouple from total GitHub dependency.

### What are we _not_ trying to do?

We will not be developing any code directly for the MicroHub project, just hubverse (R) functions that it could call (but could also be used directly by users).

### How do we judge success?

A list of outcomes that should be true for the project to be considered a
success. Be sure to focus on outcomes, not implementation details.
- A `create_hub()` function that creates a local hub file structure
- A `save_model_out_tbl()` function that saves model output in the correct location and format within a hub

Any other functions are likely more of a nice to have, not required.

### What are possible solutions?

We can use [`hubTemplate`](https://github.com/hubverse-org/hubTemplate) as a guide for `create_hub()` and [`trendsEnsemble::save_model_out_tbl()`](https://github.com/reichlab/trendsEnsemble/blob/main/R/save_model_out_tbl.R) as an example for `save_model_out_tbl()`. We could also use some of the `hubAdmin` package's create functions if we want to create a `create_tasks_json()` function

## ✅ Validation

This section is for listing assumptions that should be validated before
kicking off a project. The answers to the prompts below may change over
time, as the team gets more information.

### What do we already know?

- MicroHub uses R under the hood, so we only need to code R versions of the functions
- MicroHub currently saves all the forecasts in a single model output file, which means a `save_model_out_tbl()` function would be useful
- Microhub does not yet use the hubverse modeling hub file structure, only the data standards, which `save_model_out_tpl()` would also need
- We have some existing templates/code that can help us write some of the desired functions (see "What are possible solutions?" section)

### What do we need to answer?

- What package out of `hubUtils`, `hubData`, or `hubAdmin` is the best place for these functions to live?
- Do we want to write more than just the `create_hub()` and `save_model_out_tbl()` functions in this initial push?
- How important is it that hubs using MicroHub have the typical config and metadata files, given that they generally don't have outside modeling team submitting forecasts?

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