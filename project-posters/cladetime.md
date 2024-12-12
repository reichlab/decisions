# Project Poster: Cladetime

## 📋 Overview

### Onwership and status

- Date: 2024-12-11

- Owner: Evan

- Team: Becky, Ben, Evan

- Status: in progress

## ❓ Problem space

### What are we doing?

Developing a reproducible way to use historic phylogenetic
[reference trees](https://docs.nextstrain.org/projects/nextclade/en/stable/user/terminology.html#reference-tree-concept)
to categorize Genbank-based SARS-CoV-2 genome data into clades.

### How does this project fit into your broader strategy?

Specifically, we need Cladetime to support the new
[variant-nowcast-hub](https://github.com/reichlab/variant-nowcast-hub)
created by the Reich Lab and the CDC.

More broadly, the ability to classify SARS-CoV-2 sequences into clades using
prior reference trees will be required by other hubs that solicit clade-related
COVID-19 predictions.

### Why are we doing this?

Reich Lab and the CDC have recently opened the
[variant-nowcast-hub](https://github.com/reichlab/variant-nowcast-hub), where
modelers predict the prevalence of SARS-CoV-2 clades.

For each of the hub's modeling rounds, we will need to generate target data by:

1. Retrieving current SARS-CoV-2 sequences 90 days after the round's
opening date
2. Assigning those sequences to clades using the reference tree that was
effective as of the round opening date

In other words, perform clade assignments by applying a historic
reference tree to a current set of sequences.

The clade assignment functionality provided by
[Nextclade's CLI](https://docs.nextstrain.org/projects/nextclade/en/stable/user/nextclade-cli/reference.html#nextclade-run)
does not support the use case of looking up past reference trees.

### How do we judge success?

A successful outcome is software that:

- is open source
- is installable via GitHub (at a minimum; preferablly installable from
a package manager like cran or pip)
- allows users to retrieve SARS-CoV-2 sequence and sequence metadata for a
specific point in time and filter it based on collection date
- allows users to specify a SARS-CoV-2 reference tree that was active at a
specific point in time
- provides an interface to the Nextclade CLI clade assignment feature that
accepts the sequence and reference tree information from the prior two
items and returns the corresponding clade assignments

### What are possible solutions?

We plan to use the aforementioned Nextclade CLI for doing clade assignments.
The inputs required are reference sequence, reference tree, and sequences.

All possible solutions will get the reference tree and reference sequence
via the version of the Nextclade SARS-CoV-2
[dataset](https://docs.nextstrain.org/projects/nextclade/en/stable/user/datasets.html)
that corresponds to the desired reference tree date.

Thus, possible solutions focus on determing the best data source for the
sequence data and sequence metadata.

Possible solutions:

- Get sequences via
[NCBI API](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/api/rest-api/#post-/virus/genome/download),
reference tree from a Nextclade dataset (with a version that corresponds
to the historic tree we want), and reference sequence returned as part of the
reference tree JSON. Use `released_since` and `updated_since` API params to download
a targeted subset of sequences.

- Use Nextstrain's
[remote input files](https://docs.nextstrain.org/projects/ncov/en/latest/reference/remote_inputs.html)
to get sequences and sequence metadata. Get the reference tree and reference
sequence via the version of the Nextclade SARS-CoV-2
[dataset](https://docs.nextstrain.org/projects/nextclade/en/stable/user/datasets.html)
that corresponds to the date of the reference tree we want to use for clade
assignment.

## ✅ Validation

### What do we already know?

- The variant nowcast hub has existing code that uses Nextstrain's remote input
files to get sequence metadata (specifically, `metadata.tsv.zst`) to generate
the list of clades to model for each round)
- Nextstrain supports accessing their Genbank-based remote input files via
publicly-accessible, versioned S3 buckets.
- The above metadata contains sequence clades as assigned using the tree that
was active on the file's publication date
- Starting in August, 2024, Nextstrain began publishing metadata that
explicitly states which Nextclade dataset version (_i.e._, reference tree)
was used for each of their pipeline runs

### What do we need to answer?

This is a backfilled project poster, so we've already tracked down our
answers (see above).

## 👍 Ready to make it

### Proposed solution

A Python package that will do custom clade assignments:

- using Nextstrain's remote input files as sources for Genbank SARS-CoV-2
sequence data and metadata
- using the reference tree and reference sequence included in the Nextclade
dataset version that corresponds to the date of the desired tree
- feeding the above inputs into Nextclade's clade assignment CLI
