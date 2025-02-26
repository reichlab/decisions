# 2025-02-13 GitHub Action Security

## Context

The Hubverse project board has a [long-standing issue](https://github.com/hubverse-org/hubverse-infrastructure/issues/51)
about ensuring that our GitHub actions follow good security practices. This RFC
stems from that issue but is both broader and more
nuanced in scope:

- We want all GitHub actions to follow good security practices (not just
those that interact with AWS).
- We need a definition of "good security practices" and how to implement them.

This is a good time to consider Hubverse security, as we expect the size of the team to change in upcoming months.

Most of the proposals below are based on
[Github's security recommendations for actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions).

### How we use GitHub actions

Because the Hubverse assumes that hubs are hosted on GitHub, using actions
provides us with a free and convenient way to automate tasks.

For example, the Hubverse (and individual hubs such as the Reich Lab's
[variant-nowcast-hub](https://github.com/reichlab/variant-nowcast-hub)) uses
GitHub actions for the following:

- Continuous integration (*e.g.*, testing code)
- Continuous deployment (*e.g.*, pushing updated sites to ReadTheDocs, Netlify,
  or GitHub Pages)
- A user interface for hub admins and contributors (*e.g.*, validating
  hub submissions, building hub dashboards)
- A runner for scheduled tasks (*e.g.*, executing data pipelines)

### Factors to consider

- Hubverse actions use GitHub public runners (no self-hosted runners).
- Hubverse software is open source, and hub data is open access.
- A subset of our GitHub actions are intended for use by Hubverse users:
    - [hubverse-aws-upload](https://github.com/hubverse-org/hubverse-actions/blob/main/hubverse-aws-upload/hubverse-aws-upload.yaml)
    - [cache-hubval-deps](https://github.com/hubverse-org/hubverse-actions/blob/main/cache-hubval-deps/cache-hubval-deps.yaml)
    - [validate-config](https://github.com/hubverse-org/hubverse-actions/blob/main/validate-config/validate-config.yaml)
    - [validate-submission](https://github.com/hubverse-org/hubverse-actions/blob/main/validate-submission/validate-submission.yaml)
- Hubverse actions are sometimes run on forked repos.
- No workflows in the Hubverse currently use the `pull-request-target` trigger.

### Actions we use

While there are some security practices that apply to all actions
(*e.g.*, setting permissions), there are many that are specific to third-party
actions. For reference, these are the third-party actions we use as of
February 2025:

| Action | Publisher | Verified | Purpose | Job Permissions/Secrets | Semantic Versioning | Feature Branch or workflow_dispatch
|--------|---------|-----------|-----------| ---------------------- | ------------------- | -------------------
| [astral-sh/setup-uv](https://github.com/marketplace/actions/astral-sh-setup-uv) | astral | ✅ | setup uv | no | ✅ | ✅
| [aws-actions/configure-aws-credentials](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions) | aws | ✅ | get AWS creds | OIDC, id-token: write | ✅ | ⛔️
| [carpentries/actions/comment-diff](https://github.com/carpentries/actions/tree/main/comment-diff) | carpentries | ⛔️ | comment on PRs with diff | pull-request: write | ✅ | ✅
| [codecov/codecov-action](https://github.com/marketplace/actions/codecov) | codecov | ✅ | coverage reports | CODECOV_TOKEN | ✅ | ✅
| [docker/build-push-action](https://github.com/docker/build-push-action) | docker | ✅ | build and push Docker images | packages: write | ✅ | ✅
| [docker/login-action](https://github.com/marketplace/actions/docker-login) | docker | ✅ | login to Docker Hub | github_token | ✅ | ✅
| [docker/metadata-action](https://github.com/marketplace/actions/docker-metadata-action) | docker | ✅ | get metadata for Docker images | ? | ✅ | ✅
| [JamesIves/github-pages-deploy-action](https://github.com/marketplace/actions/deploy-to-github-pages) | JamesIves | ⛔️ | deploy to github pages | contents: write, no secrets | ✅ | gh-pages branch
| [limitusus/json-syntax-check](https://github.com/marketplace/actions/json-syntax-check) | limitusus | ⛔️ | check JSON syntax | contents: read? | ✅ | ✅
| [nwtgck/actions-netlify](https://github.com/marketplace/actions/netlify-actions) | nwtgck | ⛔️ | deploy to Netlify for doc preview | github token, contents: write, pull-requests: write | ✅ | ✅
| [pulumi/actions (preview)](https://github.com/marketplace/actions/pulumi-cli-action) | pulumi | ✅ | preview and deploy infrastructure changes | pull-requests: write, pulumi access token | ✅ | ✅
| [pulumi/actions (up)](https://github.com/marketplace/actions/pulumi-cli-action) | pulumi | ✅ | preview and deploy infrastructure changes | pull-requests: write, pulumi access token | ✅ | ⛔️
| [pypa/gh-action-pypi-publish](https://github.com/marketplace/actions/pypi-publish) | pypa | ✅ | publish to PyPI | OIDC, id-token: write | ✅ | ⛔️ (version tag)
| [quarto-dev/quarto-actions/setup](https://github.com/quarto-dev/quarto-actions/tree/main/setup) | quarto | ⛔️ | setup Quarto | ? (looks like no write access needed) | ✅ | ✅
| [r-lib/actions/setup-r](https://github.com/r-lib/actions/tree/v2-branch/setup-r) | tidyverse? | ⛔️ | setup R on runner | no | ✅ | ✅
| [r-lib/actions/setup-r-dependencies](https://github.com/r-lib/actions/tree/v2-branch/setup-r-dependencies) | tidyverse? | ⛔️ | install R dependencies | no | ✅ | ✅
| [r-lib/actions/setup-tinytex](https://github.com/r-lib/actions/tree/v2-branch/setup-tinytex) | tidyverse? | ⛔️ | install TinyTex | no | ✅ | ✅
| [r-lib/actions/setup-pandoc](https://github.com/r-lib/actions/tree/v2-branch/setup-pandoc) | tidyverse? | ⛔️ | install pandoc | no | ✅ | ✅
| [r-lib/actions/pr-push](https://github.com/r-lib/actions/tree/v2-branch/pr-push) | tidyverse? | ⛔️ | push changes to PR branch | github_token | ✅ | ⛔️
| [r-lib/actions/pr-fetch](https://github.com/r-lib/actions/tree/v2-branch/pr-fetch) | tidyverse? | ⛔️ | checkout PR | github_token | ✅ | ⛔️
| [r-lib/actions/check-r-package](https://github.com/r-lib/actions/tree/v2-branch/check-r-package)| tidyverse? | ⛔️ | run rcmdcheck | ? | ✅ | ✅
| [sigstore/gh-action-sigstore-python](https://github.com/marketplace/actions/gh-action-sigstore-python) | sigstore | ✅ | signs Python artifacts for GitHub  releases | id-token: write | ✅ | ⛔️ (version tag)

### Aims

- Provide concrete guidelines for how the Hubverse should secure GitHub actions.
- Strike a balance between improving security and ease of implementation.
- Automate security settings and checks to the extent possible.

### Anti-Aims

- This RFC will not suggest mitigations for every single GiHub action security risk. It's intended to get a
  consensus on a security posture that makes sense for the Hubverse (_i.e._, a small team with many members who
  volunteer their time).
- This RFC does not propose to block our standard workflow when potential security issues are detected
  (with the exception of scans that detect security keys in the code base).
- Specific changes to existing actions (*e.g.*, switching to Github's deploy pages action) are out of scope, as the focus
  is on more general practices.

## Decision

The proposal below requires some up-front, but straightforward investment into the settings of the hubverse-org
GitHub organization and our critical repositories. Recommendations for individual GitHub actions can be implemented
as we touch them.

**TL;DR:**

- Apply organization-wide security [settings](#organization-level) to all existing and new repos in hubverse-org.
- Ensure that all changes to our GitHub workflows are reviewed by at least one member of the active hubverse
  developers team.
- Over time, update existing actions to follow some of GitHub's [recommended practices](#action-level).

### organization level

This section outlines some hubverse-org settings we can change to improve security. Despite GitHub's
bad UI and constant nagging about upgrading to "advanced security," it *is* possible to turn on their recommended
security settings for all current and future repos.

- Apply GitHub's recommended security settings to all existing repos in hubverse-org:
    - [GitHub docs for applying default security configurations](https://docs.github.com/en/enterprise-cloud@latest/code-security/securing-your-organization/enabling-security-features-in-your-organization/applying-the-github-recommended-security-configuration-in-your-organization)
    - Edit the GitHub default security config, updating the policy section to apply the config to new repositories
    - The default security recommendations are managed by GitHub (they include things like secrets scanning,
      code scanning, and dependabot alerts). Org admins can view the specifics.
- Update the hubverse-org action permissions to allow actions created by GitHub, actions created by verified publishers,
  and actions that are currently in use by the Hubverse.

    [example (from a different org)](./2025-02-13-github-action-security/action-permissions.png)

- We will create a new team in the hubverse-io org and add active Hubverse developers (referred to as `hubverse-core`
  in this document):
    - We will assign this new team to the [all-repository admin role](https://docs.github.com/en/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization).
    - We'll use this new team to assign CODEOWNERs to Hubverse github actions (see [below](#repository-level)).

### repository level

The [hubverse-io GitHub organization](https://github.com/hubverse-org) has too many repositories to update all at once.
This RFC proposes to focus on the following in the short term:

    - all hubverse packages (things we publish for Hubverse users)
    - hubDocs (at this time, serves as the Hubverse website)
    - hub-dashboard-control-room (publishes registered hubverse dashboards)
    - hub-dashboard-template (templates for new hubverse dashboards)
    - hubverse-actions (templates we provide for Hubverse users)
    - hubverse-developer-actions (template we provide for Hubverse devs)
    - hubverse-infrastructure (manages Hubverse AWS resources)
    - hubverse-transform (manages Hubverse S3 synchronization)

For the above repos, add a
[CODEOWNERS](https://docs.github.com/en/enterprise-cloud@latest/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
file to the `.github` directory.

CODEOWNERS will specify that changes to GitHub workflows must be approved
by at least member of the new `hubverse-core` team. Below is an example from a
reichlab repo:

```sh
# require review for core team member for updates to GitHub workflows
/.github/CODEOWNERS @reichlab/repo-writers
/.github/workflows/ @reichlab/repo-writers
```

Depending on who we add to `hubverse-core` this won't be a big change to our current workflow, since we already
review every PR. However, it will make it easier to control who can approve changes to GitHub workflows in the future.

### action level

Below are security practices we will follow when creating new GitHub actions. Existing actions should be updated
as we touch them (with the exception of our template actions in `hubverse-actions` and `hubverse-developer-actions`,
which should be updated as soon as possible).

- GitHub workflows will explicitly specify permissions granted to `GITHUB_TOKEN`. If a workflow has multiple jobs,
  apply the lowest level of access to the workflow and override it for individual jobs if necessary.
  [More information about `GITHUB_TOKEN` permissions](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/controlling-permissions-for-github_token).
  For example, code check workflows usually only need read access to repository contents:

    ```yaml
    name: Code Checks
    permissions:
    contents: read
    ```

- At a minimum, third-party actions from unverified publishers (see #actions-we-use) will be
  [pinned by commit hash instead of by tag](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#using-third-party-actions).
The exception is [actions from `r-lib`](https://github.com/r-lib/actions), which are widely used in the hubverse and
maintained by a large team we trust. For example:

    ```yaml
    steps:
    - uses: limitusus/json-syntax-check@v77d5756026b93886eaa3dc6ca1c4b17dd19dc703  # v2.0.2
    ```

- We will prefer the following methods when authenticating to external cloud services (*e.g.*, AWS, Netlify), ordered
  here from most to least preferred:
    - [OIDC tokens](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect#benefits-of-using-oidc)
    - organization/team-level tokens provided by an external cloud service,
    stored as repository secrets (for example Pulumi offers
    [organization access tokens](https://www.pulumi.com/docs/pulumi-cloud/access-management/access-tokens/#organization-access-tokens)
    to Enterprise users)
    - individual tokens stored as repository secrets.

- Where possible, we will use separate jobs for steps that require different
  GITHUB_TOKEN permissions. For example, a workflow that builds a website and
  then deploys it to GitHub pages should contain the build in one job (with
  `content: read permission`) and the deploy in another job (with
  `packages: write` permission).

  **Note:** In this scenario, the build job would upload its output as a
  github artifact, and the deploy job would download that artifact and push it
  to GitHub pages.

### Other Options Considered

- An alternate way to secure unverified, third-party GitHub actions is using forked versions that we can update as
  needed. This method ensures that we control the updates and would allow us to continue pinning version tags instead
  of switching to hash commits.
  Creating forks gives us another maintenance chore, so the proposal prefers pinning commit hashes with the
  assumption that Dependabot will alert us to updates.

- An abbreviated version of this proposal would be to apply the
  [organization-level settings](#organization-level) and address the issues flagged by the code and dependency
  scanners as they arise.
  We would still want to apply best practices in our individual actions, so in practice the item we'd omit in this
  case is the CODEOWNERS files. Because adding that file to our repos is easy, it's included in the
  proposed changes.

- For additional protection wrt write-access GitHub actions that have a
  `workflow_dispatch` trigger or that run against feature branches, we could
  force a manual approval step before they run (by using
  [GitHub environments](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment).
  This is a reasonable precaution for actions that meaningfully change an
  artifact (*e.g.*, push a Docker image or update a website)), but it's a lot of
  friction to add for a small team.

## Status

proposed

## Consequences

- More people will have admin access to Hubverse repositories:
  - that means more access to secrets and workflows
  - it also means more hands to maintain the repositories
- Admin repository access will be consistently applied within hubverse-io.
- Controlling repo admin access via team instead of adding individuals ensures easier onboarding/offboarding.
- The burden of ensuring that we follow the action-level security practices falls on members of the `hubverse-core`
  team.
- There will be a lot of noise immediately after we apply the organization-level security settings, because the initial
  scans will flag every existing issue simultaneously.
- The baseline level of security for Hubverse GitHub actions will improve.

## Projects

n/a
