# Project Poster: Python client for hubData

- Date: 2025-03-07
- Owner: Matt Cornell
- Team: Matt Cornell, Becky Sweger
- Status: draft

## ❓ Problem space

The Hubverse team maintains [hubData](https://hubverse-org.github.io/hubData/index.html), an R client for accessing hub
data locally and on the cloud (S3 buckets). Some of hubData's features are:

- an interface for accessing and querying hub model-output data
- an interface for accessing and querying hub target-data (coming soon)
- a function that returns the model-output arrow schema for a hub (based on its config files)
- conversion of Zoltar forecasts to Hubverse format
- creation of model-output file submission templates

The team has long considered—but never prioritized—building a Python client for hub data. This project poster is a
concrete proposal for starting that work.

### What are we doing?

Creating a Hubverse-maintained Python client for interacting with hub data. The proposal is to start small and create
a Python package focused on the *hub data functions that are required for our current internal work*.

### Why are we doing this?

There are three main drivers for creating a Hubverse Python client at this time, especially because we anticipate
a smaller team in the coming months.

#### hub dashboard support

Based on the results of a recent prioritization process, the Hubverse team is focusing on our
[Hub Dashboard project](https://github.com/reichlab/decisions/blob/main/project-posters/hub-dashboard/hub-dashboard.md).
That work includes Python functionality that maps to functionality in the R hubData package:

- hub-dashboard-predtimechart
  [parses hub config files](https://github.com/hubverse-org/hub-dashboard-predtimechart/blob/main/src/hub_predtimechart/hub_config.py#L15)
  (reads hub config files to generate a predtimechart options object)
- hub-dashboard-predtimechart reads model-ouput files to generate .json data for predtimechart
- there are plans to update hub-dashboard-predtimechart with code that can read Hubverse-formatted target data files

Some of the above code is specific to the dashboard work, but some of it is more generic hub data and config access
that could benefit other hubverse users if we package it separately.

#### address hub schema issues

Hubverse Python code currently has no equivalent of `hubData::create_hub_schema()` (*i.e.*, the `hubData` function that
generates a model-output Arrow schema by reading a `tasks.json`).

As a result, any Python code that reads and writes model-output files (especially in parquet format) does not apply an
"official" hub schema and may result in data that doesn't work with downstream Hubverse tools.

A good example is the `hubverse-transform` repo that converts individual model-output submissions to parquet files for
S3. We've patched some of most common issues (*e.g.*, forcing `location` and `output_type_id` to string), but ideally
we'd use an explicit schema, generated via a hub's config, when transforming hub data for S3.

#### supporting Hubverse users

The absence of a Hubverse Python client means no explicit guidance for Python users. Currently, we've written some
ad-hoc Python-based code samples and added them to hub READMEs (these examples are specific to hubs that use S3).

This practice is not ideal, nor is it sustainable:

- As the sample code evolves, there will be outdated versions of it scattered around. With an official client, we
  have a central place for Python-specific documentation that can evolve as needed.
- Because we have no official Python client, the team may find itself supporting Python users who have brought
  their own tooling.

### What are we *not* trying to do?

- This iteration of a Python hubData client will not target API design or feature parity with the `hubData` R client.
- *We will focus on implementing features that are needed by existing Python tools in the Hubverse*.
- This iteration of Python hubData will not be released to PyPI. We will start by dogfooding it internally and adding
  it to Hubverse documentation when ready.

### How do we judge success?

- There is a Python client for accessing hub data that meets Hubverse development standards (*e.g.*, running tests in CI)
- Existing Hubverse Python projects that interact with hub data and schemas use this new Python client
  (e.g., hubverse-tranform and hub-dashboard-predtimechart)
- Python package will run on MacOS, Linux, and Windows
- Python package has documentation hosted on GitHub pages (same pattern as Hubverse R packages)

### What are possible solutions?

- Develop a Python package to work with Hubverse data (this proposal)
- Stop using Python in the Hubverse. Ruled this out because:
    - We already know there are people who want to consume hub data using Python, even if they're not the primary
      use case for this version of a client.
    - We use an [AWS Lambda function](https://aws.amazon.com/lambda/) to mirror hub data to S3 buckets. There is no
      "out of the box" AWS Lambda R runtime. While it's possible to create one, we chose Python for `hubverse-transform`
      because it's officially supported and because there's a lot of documentation about Python-based Lambda functions.

## ✅ Validation

### What do we already know?

- We are not using a config-derived Hubverse schemas when writing model-output parquet files to S3
- The in-process Hubverse dashboard work already has code for reading Hubverse schemas and model-output data and will
  soon need to read Hubverse-formatted target data.
- We still encounter [edge cases when trying to use Python libraries to access hub data](https://github.com/hubverse-org/hubverse-infrastructure/issues/72).
  Having a Hubverse package would give us a central place to work around those issues and would allow us to give Python
  hub users an alternate should their own tools fail.

### What do we need to answer?

Because we've limited the scope of this Python client to solving internal use cases, there are few assumptions we need
to validate at this time.

- What is the minimum Python version this client would need to support? Does it matter? (if *does* matter, the answer
  to this question might impact the package's internal implementation details)
- Viability of using the new Python client as a wrapper around the existing R code (for example,
  using a tool like [rpy2](https://github.com/rpy2/rpy2)).

## 👍 Ready to make it

### Proposed solution

- Python package with the following features:
    - works with local and S3-based hub data
    - can operate at the hub level but not directly on the model-output folder (*i.e.*, users will need access to an
      entire hub, either locally or on S3)
    - produces an Arrow schema for a specified hub
    - can read a hub's target data and return a lazy-style dataframe
    - can read a hub's model-output data and return a lazy-style dataframe

### Scale and scope

- Two team members (for continuity and code review)
