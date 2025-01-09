# 2025-01-08 Reichlab Software Release Process

## Context

The Reich Lab currently has no formalized workflow and release process for
publishing software (for purposes of this RFC, _publishing software_ means
publishing on a package index like PyPI or CRAN).

The Lab does have one application distributed via a package index
([`pymmwr` on PyPI](https://pypi.org/project/pymmwr/)). The Lab does not
actively maintain pymmwr (the last release was in 2018). The application also
predates modern security practices such as publishing to PyPI via GitHub actions
and trusted publishers.

Therefore, this RFC will not attempt to use pymmwr as a
baseline for proposing a software release process circa 2025.

### Aims

- Define a feature development and package release process for Reich Lab
software distributed via a package index (_e.g._, CRAN, npm, PyPI).

### Anti-Aims

- This proposal is not relevant for internal packages or packages distributed
 via GitHub.
- This proposal concerns the mechanics and development process for published
packages; it does not cover how to determine if pursuing publication is
desirable.

## Decision

We will adopt [The Hubverse release process](https://hubverse-org.github.io/hubDevs/articles/release-process.html).

The above link contains implementation details specific to R packages, but
we can apply the recommendations for
[versioning](https://hubverse-org.github.io/hubDevs/articles/release-process.html#versioning)
and
[branch workflows](https://hubverse-org.github.io/hubDevs/articles/release-process.html#after)
to any programming language.

Publication and release will be automated to the extent possible, regardless of
language, and will follow the practices and security recommendations of the
relevant package index.

### Other Options Considered

No other options considered. The age of existing, published Reich Lab packages
make them poor candidates for using as a baseline when standardizing a release
process. Because many people in the Lab also develop Hubverse software, this
proposal strongly recommends following the same procedures.

## Status

Proposed

## Consequences

Most Reich Lab software development would not be impacted at this time. If we
decide to distribute an internal software package via a package index, this RFC
would take effect. In other words, Reich Lab developers would adopt the
versioning and release process as part of preparing for publication.

Pros:

- Although the Lab doesn't publish a lot of software, having a formal,
documented process may make it less daunting to do so (or at the very least,
ensure that people won't need to create their own process every time).
- The recommended publication process gives the team control over the timing
and content of software releases.
- Attaching ourselves to the Hubverse process means that the Reich Lab doesn't
have to maintain its own documentation.

Cons:

- There's a learning curve (though the decision to publish software will add
complexity regardless).

## Projects

Publishing [Cladetime](../project-posters/cladetime.md) on PyPI is the impetus
for this RFC, but the proposal itself isn't tied to a specific project.
