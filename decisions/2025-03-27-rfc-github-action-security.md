# 2025-03-27 GitHub Action Security

## Context

This RFC supersedes the [2025-02-13 GitHub Action Security RFC](./2025-02-13-rfc-github-action-security.md).

Two things have happened since we approved the prior RFC:

### Dependabot testing

Since approving the prior RFC, we learned that GitHub's dependabot does a good job of submitting PRs when a GitHub
action is updated, even when the action is pinned by a commit SHA.

When dependabot is used in conjunction with GitHub actions pinned by commit SHA, we can have the best of both worlds:

- The security benefits of pinning via commit SHA.
- Assurances that we'll get minor GitHub action updates (previously, we relied on sliding tags for those).

### Update to GitHub CodeQL

A [recent update to GitHub CodeQL](https://github.blog/changelog/2025-02-28-improved-code-scanning-coverage-for-github-actions-public-preview/)
moved the check for unpinned third-party dependencies to an optional
[`security-extended` query suite](https://docs.github.com/en/code-security/code-scanning/managing-your-code-scanning-configuration/codeql-query-suites#security-extended-query-suite).

In other words, the default settings for GitHub CodeQL will no longer flag third-party actions that are
not pinned by commit SHA.

## Aims

- Update our prior RFC to reflect new information.

## Anti-Aims

- Specific changes to existing actions (*e.g.*, switching to Github's deploy pages action) are out of scope, as the focus
  is on more general practices.

## Decision

We will add the following security practices to those outlined in the prior RFC:

- Pin all third-party actions—not just unverified actions—by commit SHA.
  The exceptions are workflows in
  [`hubverse-actions`](https://github.com/hubverse-org/hubverse-actions)[^1].
- Add a dependabot configuration to all Hubverse repositories as described in the
  [Hubverse developer docs](https://hubverse.io/en/latest/developer/security.html#dependabot-setup)
- Add the `security-extended` query suite to the default `hubverse-org` security configuration (*i.e.*, new hubs
  will have this setting enabled, but we'll have to update existing hubs separately).

[^1]: Hubverse administrators use these workflows when setting up a new hub. Because we don't have a process
for ensuring that admins configure dependabot, it's safer to pin the actions by tag. That way, hubs will receive
minor updates to actions that use a sliding tag.

### Other Options Considered

Not applicable.

## Status

pending

## Consequences

- The decision to pin more actions by commit SHA will cause a slight increase in the work needed to bring
  repositories into compliance with our new security posture. To make this easier, Zhian created the
  [pinsha package](https://zkamvar.github.io/pinsha/index.html).

## Projects

n/a
