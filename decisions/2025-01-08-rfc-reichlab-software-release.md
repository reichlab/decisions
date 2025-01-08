# 2025-01-08 Reichlab Software Release Process

## Context

The Reich Lab currently has no formalized workflow and release process for
publishing software (for purposes of this RFC, _publishing software_ means
publishing on a package index like PyPI or CRAN).

To my knowledge, the Lab does not actively maintain any software packages on a
package index.

However, that will change once Cladetime is published on PyPI.

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

We will adopt the Hubverse release process as
[defined here](https://hubverse-org.github.io/hubDevs/articles/release-process.html).

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

No other options considered. The Reich Lab has no prior art for standardizing
branch workflows and versioning for published packages (that I'm aware of).
Because many people in the Reich Lab also develop Hubverse software, this
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
