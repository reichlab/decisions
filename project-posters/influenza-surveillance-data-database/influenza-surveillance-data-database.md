# Project Poster: Influenza surveillance data database

- Date: [date posted is created]

- Owner: Li

- Team: Li, Nick

- Status: Draft

## ❓ Problem space

The goal of this section is to explain why solving this problem matters
by answering the prompts below. Ideally, the answers will be informed
by input from Lab leadership, the Hubverse, users, and/or funders.

### What are we doing?

Collecting many sources of influenza surveillance data in a single place and standardizing it into a single format that can be used for training forecasting models.

### Why are we doing this?

We know that forecasting models generally perform better when they have more (high-quality and informative) data to train on, and we would like to leverage the historical data we have but are not currently using. For example, [Flusion]() made use of several sets of influenza surveillance data and demonstrated some of the best individual model performance for the recent few FluSight seasons.

### What are we _not_ trying to do?

For now, we are only interested in influenza surveillance data (though we may later consider similar projects for other infectious diseases).

### How do we judge success?

A successful outcome is something that:
- is open source (no private data)
- is relatively easy to use and accessible through R
- allows users to call/query any number of named sources of influenza surveillance data with filtering options
- returns the output in a standardized data frame format that can be quickly plugged in as training data for a model

### What are possible solutions?

The surveillance data may be *stored* in one of several ways, including: 1) a data package (likely in R), 2) a relational database (likely in SQL), or 3) in S3 bucket (likely using AWS).

Regardless of storage methodology, we likely will want a few small functions (in R) to support querying and filtering the data.

## ✅ Validation

This section is for listing assumptions that should be validated before
kicking off a project. The answers to the prompts below may change over
time, as the team gets more information.

### What do we already know?

- We would like to collect influenza surveillance data from both the US and other countries around the world
- Some possible surveillance data leads include: NHSN data, NSSP data, ILInet, FLUSurv; Flusion's sources; CMU Delphi's signal database; ECDC's data
- We may borrow some ideas from the hubverse on target data, while also not restricting ourselves if we decide additional columns or other formats serve us better

### What do we need to answer?

We need to determine the storage solution for this influenza surveillance data before moving forward by weighing the pros and cons of each. Some questions include:
- How much data do we expect to store?
- Do we expect to update the data? How often? Just from the currently included sources? Adding new sources?
- Do we want to store any raw formats of data or metadata?
- Who are our users? What level of complexity do we want to introduce in querying the data?
- Do we want to/have the capability to maintain a server?
- Do we want to make suggestions of scaling parameters?

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