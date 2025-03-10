# Project Poster: Python client for hubData

- Date: 2025-03-07
- Owner: Matt Cornell
- Team: Matt Cornell, Becky Sweger
- Status: draft

## ❓ Problem space

The Hubverse team maintains [hubData](https://hubverse-org.github.io/hubData/index.html), an R client for accessing hub data locally and on the cloud (S3 buckets). Some of hubData's features are:

- an interface for accessing and querying hub model-output data
- an interface for accessing and querying hub target-data (coming soon)
- a function that returns the model-output arrow schema for a hub (based on its config files)
- conversion of Zoltar forecasts to Hubverse format
- creation of model-output file submission templates

The team has long considered—but never prioritized—building a Python client for hub data. This project poster is a concrete proposal for starting that work.

### What are we doing?

Creating a Hubverse-maintained Python client for interacting with hub data. The proposal is to start small and create a Python package focused on the *hub data functions that are required for our current internal work*.

### Why are we doing this?

There are three main drivers for creating a Hubverse Python client at this time, especially because we anticipate a smaller team in the coming months.

#### hub dashboard support

Based on the results of a recent prioritization process, the Hubverse team is focusing on our [Hub Dashboard project](https://github.com/reichlab/decisions/blob/main/project-posters/hub-dashboard/hub-dashboard.md). That work includes Python functionality that maps to functionality in the R hubData package:

- hub-dashboard-predtime-chart [parses hub config files](https://github.com/hubverse-org/hub-dashboard-predtimechart/blob/main/src/hub_predtimechart/hub_config.py#L15)
- there are plans to update hub-dashboard-predtime-chart with code that can read Hubverse-formatted target data files

Some of the above code is specific to the dashboard work, but some of it is more generic hub data and config access that could benefit other hubverse users if we package it separately.

#### addressing hub schema issues

Hubverse Python code currently has no equivalent of `hubData::create_hub_schema()`. As a result, the Python package that processes model-output files and stores them on S3 doesn't apply an "official" hub schema when reading model-output submissions.

In the past, this has lead to errors when using hubData and other tools to read model-output data from S3. We've patched some of most common issues (*e.g.*, forcing `location` and `output_type_id` to string), but ideally we'd use an explicit schema, generated via a hub's config, when transforming hub data for S3.

#### supporting Hubverse users

In the absence of a Hubverse Python client means that the team is helping Python users by providing sample code on an ad-hoc basis. This practice is not idea, nor is it sustainable:

- As the sample code evolves, there are outdated versions of it scattered around
- Because we have no official Python client, the team may find itself supporting Python users who have brought their own tooling

### What are we _not_ trying to do?

* This iteration of a Python hubData client will not target feature parity with the `hubData` R client. We will focus on implementing features that are needed by existing Python tools in the Hubverse.
* This iteration of Python hubData will not be released to PyPI. We will dogfood it internally.

### How do we judge success?

- There is a Python client for accessing hub data that meets Hubverse development standards
- Existing Hubverse Python projects that interact with hub data and schemas use this new Python client (e.g., hubverse-tranform)
### What are possible solutions?

A high-level list of potential ways to meet the project's goals.

## ✅ Validation

### What do we already know?

- We are not using a config-derived Hubverse schemas when writing model-output parquet files to S3
- The in-process Hubverse dashboard work:
	- reads Hubverse schemas and will
	- will need to read Hubverse target data
### What do we need to answer?

A list of information required before starting the project.
We need to find the answers.

- ??

## 👍 Ready to make it

### Proposed solution

- Python package that:
	- produces an Arrow schema for a hub
	- can read a hub's target data

### Scale and scope

- Two team members (for continuity and code review)