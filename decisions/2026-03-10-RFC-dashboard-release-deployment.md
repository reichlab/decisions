# 2026-03-10 Streamlining Dashboard Release and Deployment

## Context

The [hubverse dashboard infrastructure](https://hubverse.io/en/latest/developer/dashboard-local.html) generates static websites with auto-generated forecast and evaluation visualizations for modeling hubs. It consists of three build tools orchestrated via [reusable workflows](https://hubverse.io/en/latest/developer/dashboard-workflows.html) in the [hub-dashboard-control-room](https://github.com/hubverse-org/hub-dashboard-control-room):

| Component | Tool | Language | Packaging | Output |
|-----------|------|----------|-----------|--------|
| Forecast viz data | [hub-dashboard-predtimechart](https://github.com/hubverse-org/hub-dashboard-predtimechart) | Python | pip from GitHub (`$latest` tag) | `ptc/data` branch |
| Eval viz data | [hubPredEvalsData](https://hubverse-org.github.io/hubPredEvalsData) via [Docker wrapper](https://github.com/hubverse-org/hubPredEvalsData-docker) | R | Docker image (`:latest` tag) | `predevals/data` branch |
| Website | [hub-dash-site-builder](https://github.com/hubverse-org/hub-dash-site-builder) | Quarto/BASH | Docker image (`:latest` tag) | `gh-pages` branch |

This infrastructure was built under time pressure by two team members who have since left the project. While functional and [well-documented](https://hubverse.io/en/latest/developer/dashboard-tools.html), the system involves many manual steps and requires deep institutional knowledge to operate safely. The remaining team's primary strength is R, with strong but limited Python skills across few members and JavaScript skills concentrated in a single team member — creating bus factor risks for the Python and JS components.

### Current issues

The team's first schema-change deployment after these members' departure ([hubPredEvalsData#33](https://github.com/hubverse-org/hubPredEvalsData/discussions/33)) exposed several structural problems. These, combined with issues identified through review of the [developer documentation](https://hubverse.io/en/latest/developer/) and team experience, are catalogued below:

| # | Issue | Severity | Source |
|---|-------|----------|--------|
| P1 | **Manual multi-step release sequences across repos.** Releasing a tool change requires a specific sequence of steps across multiple repositories (e.g., merge -> rebuild Docker -> update hub configs -> rebuild). The hub-dashboard-template also needs updating but is independent of the critical deployment path. Misordering the dependent steps breaks dashboards. | High | [staging docs](https://hubverse.io/en/latest/developer/dashboard-staging.html), [hubPredEvalsData#33](https://github.com/hubverse-org/hubPredEvalsData/discussions/33) |
| P2 | **Manual renv.lock update process.** Every release of `hubPredEvalsData` requires manually regenerating the `renv.lock` file in the Docker wrapper repo, a process that was unclear to the team during the first deployment. | High | [hubPredEvalsData#33](https://github.com/hubverse-org/hubPredEvalsData/discussions/33), [predevals docs](https://hubverse.io/en/latest/developer/dashboard-predevals.html) |
| P3 | **No automated end-to-end integration test.** Individual tools have unit tests and the Docker images have comparison tests. The full pipeline can be tested manually via the [local build workflow](https://hubverse.io/en/latest/developer/dashboard-local.html) and [staging process](https://hubverse.io/en/latest/developer/dashboard-staging.html), but both require multiple manual steps including installing specific dev versions of tools to test unreleased changes. There is no automated way to test a specific combination of tool versions through the full pipeline. | Medium | architecture review |
| P4 | **Deployment downtime during interface changes.** When a tool's interface changes, the [staging docs explicitly note](https://hubverse.io/en/latest/developer/dashboard-staging.html#staging-tools-predevals) that "there will be a period of time that the workflows will not work" between releasing the tool and updating the control room. | High | [staging docs](https://hubverse.io/en/latest/developer/dashboard-staging.html) |
| P5 | **Flawed config schema versioning in hubPredEvalsData.** `predevals-config.yml` requires a `schema_version` URL pointing to a specific schema version bundled in the R package. However, this mechanism provides no real value: (a) the tool hardcodes a `minimum_version` that rejects older schemas, so there is no backwards compatibility — breaking changes (e.g., adding required `rounds_idx` in what was labelled a patch bump from v1.0.0 to v1.0.1) simply reject old configs rather than handling them gracefully; (b) for non-breaking changes, admins must still update the `schema_version` URL to opt in to validation of new optional fields, even though the tool would process their config identically; (c) the version is redundant — the tool already knows what version it is. Meanwhile, `predtimechart-config.yml` has no schema versioning at all. The `:latest`/`$latest` auto-upgrade pattern already used by the other dashboard tools (hub-dash-site-builder and hub-dashboard-predtimechart) is actually a strength for non-breaking changes (zero admin effort), but breaking changes force simultaneous config updates across all dashboards. | Medium | [config.R](https://github.com/hubverse-org/hubPredEvalsData/blob/main/R/config.R), architecture review, team insight |
| P6 | **Docker indirection adds a manual release step for hubPredEvalsData.** The R package is wrapped in a Docker image because "installing R packages on GitHub workflows involves several steps." This adds an entire repo (`hubPredEvalsData-docker`) with its own release cycle to the deployment chain. The Docker image also brings dev versions of 5 upstream packages (`hubData`, `hubEvals`, `hubUtils`, `hubPredEvalsData`, `scoringutils`). | Medium | [predevals docs](https://hubverse.io/en/latest/developer/dashboard-predevals.html) |
| P7 | **Circular CI dependencies during schema changes.** During the [hubPredEvalsData#33 deployment](https://github.com/hubverse-org/hubPredEvalsData/discussions/33), the Docker CI pipeline tested against the old image which couldn't process the new schema. The team had to disable the test step to unblock the build, undermining safety checks. | High | [hubPredEvalsData#33](https://github.com/hubverse-org/hubPredEvalsData/discussions/33) |
| P8 | **JavaScript CDN caching adds deployment uncertainty.** The [PredTimeChart](https://github.com/reichlab/predtimechart) and [PredEvals](https://github.com/hubverse-org/predevals) JS modules are served via jsDelivr with version-aliased URLs. The CDN cache can persist up to 7 days, requiring [manual cache purging](https://hubverse.io/en/latest/developer/dashboard-staging.html#staging-javascript) for timely updates. | Low | [staging docs](https://hubverse.io/en/latest/developer/dashboard-staging.html) |
| P9 | **Remote staging is complex.** Staging changes remotely requires creating forks of dashboard repos, branches in the control room, and up to triple-reference updates (generate workflow, push-things workflow, and scripts). | Medium | [staging docs](https://hubverse.io/en/latest/developer/dashboard-staging.html) |
| P10 | **Knowledge concentrated in departed team members.** While the documentation is thorough, the processes are complex enough that documentation alone is insufficient for confident execution by the remaining team. | High | team experience |
| P11 | **hubPredEvalsData scales poorly.** The evaluations tool [currently takes over 20 minutes](https://hubverse.io/en/latest/developer/dashboard-predevals.html) to run for the FluSight hub with 70 models, and this will grow. | Medium | [predevals docs](https://hubverse.io/en/latest/developer/dashboard-predevals.html) |

### What different types of changes require

To illustrate the deployment burden, here are the release chains for representative changes. Key version-pinning points are: `renv.lock` pins R package versions in `hubPredEvalsData-docker`; the control room pins Docker images to `:latest` and `hub-dashboard-predtimechart` to `$latest` (latest GitHub release tag); JS modules are pinned to major version aliases on jsDelivr (e.g., `@v3`). Once a tool is released, all dashboards pick it up automatically on their next build via these pins.

**Non-breaking changes** (new optional config field with sensible default, bug fix, performance improvement):

- **Non-breaking change in hubPredEvalsData** (e.g., new optional scoring metric): implement change in `hubPredEvalsData` -> release `hubPredEvalsData` -> update `renv.lock` in `hubPredEvalsData-docker` -> rebuild and release Docker image (`:latest` tag now points to new version) -> all dashboards automatically pick up the change on next scheduled build. Dashboards only need to update their `predevals-config.yml` if they want to opt in to the new feature.
- **Non-breaking change in hub-dashboard-predtimechart** (e.g., bug fix): implement change -> release new version (`$latest` now resolves to it) -> all dashboards automatically pick up the change on next build. No config updates needed.

**Breaking/schema changes** (required field added, field renamed, structure changed, interface change):

- **Breaking schema change in hubPredEvalsData** (e.g., adding required `rounds_idx` property as in [hubPredEvalsData#33](https://github.com/hubverse-org/hubPredEvalsData/discussions/33)): implement change in `hubPredEvalsData` -> release `hubPredEvalsData` -> update `renv.lock` in `hubPredEvalsData-docker` -> rebuild and release Docker image. At this point, the Docker CI comparison tests cannot pass because the `:latest` image cannot process the new schema, so **tests must be disabled to unblock the build** (P7). Then -> **update `predevals-config.yml` in every evaluation-enabled dashboard** to add the new required field **before** the next scheduled build, or builds fail (`:latest` has already moved and the old config is now invalid) -> rebuild data for affected hubs.
- **Interface change in hub-dash-site-builder** (e.g., changing CLI argument flags): implement change -> merge to main -> build and publish Docker image -> **update control room workflows** to use new CLI syntax -> release tagged version. Dashboards break between Docker release and control room update (P4).

**JavaScript changes**:

- **Change in PredTimeChart or PredEvals JS** (e.g., UI fix): implement change -> release new version within the current major version alias -> purge jsDelivr CDN cache -> wait for propagation (up to 7 days without manual purge). No config or workflow changes needed.

### Upcoming pressure

The [eval-metrics-expansion project](https://github.com/reichlab/decisions/pull/34) plans multiple changes to `hubPredEvalsData` that will each require deployment through this pipeline:

- Sprint C: Schema v1.1.0 changes (`transform_defaults`, per-target `transform` in config)
- Sprint D: Variogram score integration (`compound_taskid_set` support)

Each of these will trigger the full release chain documented in the issues above.

### Aims

 - Reduce the number of manual steps required to release and deploy dashboard tool changes
 - Enable schema/config changes to be rolled out without breaking existing dashboards during the transition window
 - Make the release process accessible to team members who did not build the infrastructure
 - Ensure the upcoming eval-metrics-expansion schema changes can be deployed confidently
 - Maintain the security posture established in [prior decisions](./2024-12-13-rfc-hub-dashboard-orchestration.md)

### Anti-Aims

 - Rewrite the entire dashboard architecture from scratch
 - Change the fundamental control room / reusable workflow pattern (it works well for hub admins)
 - Address JavaScript skill gaps within the team (separate concern)
 - Change how hub admins interact with their dashboard repos

## Decision

We will adopt a phased approach: near-term automation to address the most urgent pain points, followed by medium-term structural improvements to prevent the same class of problems from recurring.

### Phase 1: Near-term automation (low effort, high impact)

_Addresses: P1, P2, P9, P10_

#### Claude Code skills for dashboard operations

We will create a family of Claude Code skills that encode institutional knowledge into guided, executable workflows:

- **`/dashboard-local-build`** — Guides a developer through the [local build workflow](https://hubverse.io/en/latest/developer/dashboard-local.html) for a given hub, the foundational process for understanding and verifying dashboard changes. A practical first skill to implement and a useful first step before staging or releasing. Addresses P10.
- **`/dashboard-release`** — Guides a developer through the full release sequence for any dashboard tool. Includes pre-flight checks (correct branch, tests passing, no uncommitted changes), correct step ordering, and verification at each stage. Addresses P1, P10.
- **`/dashboard-stage`** — Determines whether local or remote staging is appropriate based on the type of change, then guides through the relevant process. Addresses P9, P10.
- **`/dashboard-debug`** — Helps diagnose failed builds and workflow issues by checking workflow run status, inspecting artifacts, and identifying common failure modes. Addresses P10.
- **`/dashboard-config-migrate`** — Assists with updating dashboard config files across hubs after a schema change, including validation that configs conform to the new schema. Addresses P1, P10.

These skills wrap the existing [documented processes](https://hubverse.io/en/latest/developer/) but make them executable and checkable rather than requiring developers to read through extensive documentation and manually track multi-step sequences.

#### Automated renv.lock update workflow

We will add a GitHub Action to `hubPredEvalsData-docker` that automatically generates a pull request with an updated `renv.lock` file when a new version of `hubPredEvalsData` is released. The cross-repo trigger mechanism (e.g., `repository_dispatch` via PAT, GitHub App token, or a cron-based check in the target repo) is TBD. This automates a manual step in the deployment chain (P2).

### Phase 2: Medium-term restructuring (moderate effort, structural improvement)

_Addresses: P3, P5, P6, P7_

#### Replace comparison-based Docker CI with independent validation

The current `hubPredEvalsData-docker` CI tests build a new image and compare its output against the `:latest` image's output. This is a brittle snapshot approach that breaks on any schema change (P7) and is difficult to update expectations for. We will replace it with proper independent validation that tests actual expectations: expected files exist, CSV columns and types are correct, `predevals-options.json` conforms to schema, score values are within expected ranges, etc. The output schema *is* the contract — if the output validates against it, the image is correct. This eliminates the circular dependency entirely.

#### Dedicated test hub and end-to-end smoke test

We will create a minimal dedicated test hub in `hubverse-org` purpose-built for dashboard CI. This hub will contain minimal data (few models, small dataset) so tests run fast, and include configs exercising key features. It will serve as the test target for both Docker image CI and the end-to-end smoke test.

We will also create an end-to-end smoke test workflow (in the control room or a dedicated repo) that tests the full pipeline against this test hub: install tools at specified versions -> generate forecast data -> generate eval data -> build site. This can be run on-demand before releases or automatically as part of the release process. Addresses P3.

The test hub will need updating for breaking schema changes or to exercise new optional features. This will be incorporated into the `/dashboard-release` skill as a guided step, and existing [example hubs](https://github.com/hubverse-org/example-complex-forecast-hub) can serve as additional non-blocking test targets for broader coverage.

#### Rework config schema handling in hubPredEvalsData

Currently, `predevals-config.yml` requires a `schema_version` URL that the tool validates against. In principle, schema versioning could enable backwards compatibility — allowing the tool to handle older config formats gracefully during migration windows. However, in practice this has been useless because we are not handling breaking changes for backwards compatibility: the tool hardcodes a `minimum_version` that simply rejects older schemas outright. This means the `schema_version` field adds admin overhead (requiring URL updates even for non-breaking changes) without providing any of the benefits that schema versioning is designed for.

The tool could instead always validate against its own latest bundled schema — non-breaking changes would pass automatically, breaking changes would fail naturally with clear error messages, and no hardcoded rejection logic would be needed.

I propose we remove the `schema_version` requirement from `predevals-config.yml` in two non-breaking steps:

1. **Make `schema_version` optional**: Release a version of `hubPredEvalsData` that no longer requires the field — the tool validates configs against its own latest bundled schema regardless. The field is accepted but ignored if present. No dashboards break.
2. **Remove `schema_version` from all dashboard configs**: A follow-up PR sweep across dashboard repos to remove the now-inert field, leaving a clean end state.

Going forward:

- **Non-breaking changes** (new optional fields): existing configs pass validation automatically. Admins opt in to new features by adding the new fields — no version updates needed.
- **Breaking changes** (new required fields, renamed fields): old configs naturally fail validation against the new schema, producing a clear error message.
- **Traceability**: the schema version used would be recorded in the generated output (`predevals-options.json`) rather than requiring admins to declare it in their input config.
- **Semantic versioning**: proper semver should be enforced for the config schema going forward (P5).

#### Evaluate eliminating Docker indirection for hubPredEvalsData

We will investigate whether the R package can be installed directly in GitHub Actions without Docker, potentially eliminating the Docker image release step from the deployment chain entirely. The [Docker approach was chosen](https://hubverse.io/en/latest/developer/dashboard-predevals.html) because "the process for installing R packages on GitHub workflows involves several steps and we want this to just work." This tradeoff should be re-evaluated given the deployment pain the Docker indirection causes (P6, P7).

A promising option is using [`r-lib/actions/setup-renv`](https://rstudio.github.io/renv/articles/ci.html), which restores packages from `renv.lock` directly in GitHub Actions with automatic caching:

```yaml
- uses: actions/checkout@v5
- uses: r-lib/actions/setup-r@v2
- uses: r-lib/actions/setup-renv@v2
```

This preserves the reproducibility benefit of the lockfile while dropping the Docker overhead. The `renv.lock` is built against a specific R version, so R version compatibility would need to be managed — but this can be tested as part of the same PR workflow that updates the lockfile, similar to how the Docker image is currently tested before release.

If viable, the `hubPredEvalsData-docker` repository could be retired, removing an entire link from the deployment chain.

### Not addressed by this proposal

**P4 (deployment downtime during interface changes)** is inherent to the `:latest`-tag auto-upgrade pattern — the only mitigation would be version-pinnable tool references, which was considered and rejected (see Other Options Considered). **P8 (CDN caching delays)** and **P11 (hubPredEvalsData scaling)** are lower priority and can be addressed independently.

### Other Options Considered

1. **Version-pinnable tool references and backwards compatibility.** Allow dashboards to pin specific tool versions in their configs, with tools maintaining backwards compatibility for older config schemas. Not chosen because the complexity of maintaining backwards compatibility across the tool chain (different languages, Docker images, config schemas, JS modules) would be very high for a small team. With a small number of known dashboards, coordinated upgrades on breaking changes are manageable if the process is well-automated and guided by skills.

2. **Full rewrite of the orchestration architecture.** Replace the control room pattern with a monorepo or different CI/CD system. Not chosen because the control room pattern and reusable workflows work well for hub admins and the architecture is fundamentally sound — the problems are in the release/deployment process, not the runtime architecture.

3. **Better documentation only.** Improve documentation without adding automation. Not chosen because the team already has [thorough documentation](https://hubverse.io/en/latest/developer/) covering the full dashboard lifecycle. The problem is not lack of knowledge but the number of manual steps and the consequences of missteps.

4. **Centralize all tools into a single repository.** Combine `hub-dashboard-predtimechart`, `hubPredEvalsData`, and `hub-dash-site-builder` into one repo. Not chosen because these tools use different languages (Python, R, Quarto/BASH), have different release cadences, and address independent concerns. Combining them would create a different set of coordination problems.

## Status

Proposed

## Consequences

Positive:

- Reduced deployment risk and time for upcoming eval-metrics-expansion sprints
- Institutional knowledge encoded in executable skills rather than dependent on specific people
- Removing `schema_version` requirement reduces admin overhead for non-breaking changes
- Automated `renv.lock` updates remove a manual step from the deployment chain
- End-to-end testing catches cross-component issues before they reach production

Negative:

- Claude Code skills require maintenance as the underlying processes evolve
- Breaking changes will still require coordinated config updates across all dashboards — this is accepted as manageable given the small number of known dashboards
- Phase 2 changes require careful migration planning for existing dashboards
- Evaluating Docker elimination requires dedicated investigation time and may not be feasible

Neutral:

- The control room pattern and reusable workflow architecture remain unchanged
- Hub admin experience is unaffected — they continue using the same lightweight workflows
- The security posture from prior decisions is maintained

## Projects

 - [Eval Metrics Expansion](https://github.com/reichlab/decisions/pull/34)
 - [The hubverse dashboard](../project-posters/hub-dashboard/hub-dashboard.md)
