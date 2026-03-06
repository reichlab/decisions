# Project Poster: Expanding Hubverse Evaluation Metrics and Dashboard Support

- Date: 2026-02-26

- Owner: Nicholas Reich

- Status: draft

## ❓ Problem space

### What are we doing?

Extending the hubverse forecast evaluation ecosystem across four goals:

1. **Dashboard UI improvements**: Fix known usability gaps in the predevals dashboard—table ergonomics for wider tables (column hiding, frozen model column), metric documentation, score direction indicators, and several priority bugs—without touching R packages or config schemas. Table readiness issues ([#49](https://github.com/hubverse-org/predevals/issues/49), [#50](https://github.com/hubverse-org/predevals/issues/50)) are critical prerequisites for the scaled-metrics work in goal 2. Additional polish issues: [#13](https://github.com/hubverse-org/predevals/issues/13), [#31](https://github.com/hubverse-org/predevals/issues/31), [#42](https://github.com/hubverse-org/predevals/issues/42), [#41](https://github.com/hubverse-org/predevals/issues/41), [#5](https://github.com/hubverse-org/predevals/issues/5).

2. **Evaluation config schema updates**: Consolidate all hubPredEvalsData schema changes into a single v1.1.0 release, including: config-driven UI enhancements (per-target decimal precision, human-readable target names, configurable default sort column), and the scale transformation pipeline (wiring the already-implemented log/sqrt transform support in `hubEvals::score_model_out()` through the config schema and predevals UI). Issues: [predevals#48](https://github.com/hubverse-org/predevals/issues/48), [#44](https://github.com/hubverse-org/predevals/issues/44), [#4](https://github.com/hubverse-org/predevals/issues/4), [hubPredEvalsData#34](https://github.com/hubverse-org/hubPredEvalsData/issues/34).

3. **Sample-based scoring with compound_taskid_set awareness**: Add `sample` output type support to hubEvals and build pipeline infrastructure for multivariate/compound metrics that require [`compound_taskid_set`](https://docs.hubverse.io/en/latest/user-guide/sample-output-type.html#compound-modeling-tasks) awareness. This enables the variogram score (`variogram_score_multivariate()` and `variogram_score_multivariate_point()`, recently added to scoringutils) as a metric evaluating spatial correlation structure across locations (or horizons, or more generally, across some subset of task-id variables), and lays groundwork for future compound metrics on sample forecasts. For background on this topic, see also [hubDocs PR #439](https://github.com/hubverse-org/hubDocs/pull/439) for incoming updates to compound modeling task documentation.

4. **Developer documentation**: A hubDocs guide explaining the full metric pipeline for future contributors, plus standardized end-user metric definitions in the dashboard.

### Why are we doing this?

- Increasing evaluation diversity was the most highly ranked priority in a recent survey of hubverse users. This project both surfaces existing metrics, adds new ones, and makes all evaluations more interpretable.
- Many predevals usability gaps have been open for over a year; the dashboard is hard for non-expert users to interpret (no metric definitions, no "lower is better" cues, table ergonomics issues).
- Infectious disease forecasts predict count data that spans orders of magnitude across locations. Log-scale evaluation provides fairer cross-location comparisons.
- The variogram score is a multivariate proper scoring rule capturing correlation across linked prediction tasks—something WIS cannot measure—and scoringutils now supports it natively.
- The [existing developer guide](https://docs.hubverse.io/en/latest/developer/dashboard-predevals.html) covers infrastructure (Docker, build, testing) but not metric development workflow, making it hard for new contributors to add metrics end-to-end.

### What are we _not_ trying to do?

- Not adding new chart types (existing table/heatmap/line plot are sufficient).
- Not changing the existing WIS/AE/interval coverage metrics.
- Not implementing calibration/reliability diagram visualizations in this project.
- Not adding the variogram score for quantile-format forecasts (requires sample or mean/median format).

### How do we judge success?

- A first-time user reading the dashboard understands what WIS and other metrics mean, which direction is better, and which models are performing best, all without leaving the page.
- A hub admin can configure log-scale evaluation in `predevals-config.yml` and see log-scaled metrics appear as distinct items alongside natural-scale metrics in all dropdowns and table columns.
- A hub submitting multivariate forecasts can configure compound_taskid_set and view compound metrics (e.g., variogram score) in the dashboard.
- A new developer can follow the hubDocs guide to add a hypothetical metric end-to-end without reading source code across repos.

### What are possible solutions?

See the **mini-sprint breakdown** in "Ready to make it" below. Each sprint is self-contained and releasable.

---

## ✅ Validation

### What do we already know?

**The hubverse evaluation pipeline (4 repos):**

```
hubEvals (R pkg)
  └── core scoring library; wraps scoringutils; exposes score_model_out()
      ↓
hubPredEvalsData (R pkg)
  └── orchestrates scoring for a hub; reads predevals-config.yml;
      outputs scores.csv files organized by target/eval_set/disaggregate_by
      ↓
hubPredEvalsData-docker
  └── Docker container wrapping hubPredEvalsData::generate_eval_data();
      used in hub CI/CD pipelines
      ↓
predevals (JavaScript)
  └── client-side module that reads the scores CSV files and renders
      interactive tables, heatmaps, and line plots
```

Repos:
- [hubEvals](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals)
- [hubPredEvalsData](https://github.com/hubverse-org/hubPredEvalsData)
- [hubPredEvalsData-docker](https://github.com/hubverse-org/hubPredEvalsData-docker)
- [predevals](https://github.com/hubverse-org/predevals)

Documentation:
- [User guide](https://docs.hubverse.io/en/latest/user-guide/dashboards.html#predevals-evaluation-optional)
- [Developer guide (infrastructure)](https://docs.hubverse.io/en/latest/developer/dashboard-predevals.html)

- `hubEvals::score_model_out()` (v0.1.0) already supports `transform`, `transform_append`, and `transform_label`—no hubEvals changes needed for the scale transform pipeline.
- scoringutils dev branch has `as_forecast_multivariate_sample()`, `variogram_score_multivariate()`, and `variogram_score_multivariate_point()`. (to be released by mid-March.)
- hubPredEvalsData schema versioning is established (v0.1.0 → v1.0.0 → v1.0.1); v1.1.0 is the natural target for all config-driven enhancements and scale transform additions in a single release. The schema update checklist from the last version bump is tracked in [hubPredEvalsData#32](https://github.com/hubverse-org/hubPredEvalsData/issues/32).
- The predevals JS dashboard builds via webpack into `dist/predevals.bundle.js`; the existing table/heatmap/line plot handle new metric columns without new chart types.
- predevals issues #31, #5, #42, #41, #49 are pure JS/CSS changes with no schema dependencies.

### What do we need to answer?

- **Variogram score in scoringutils**: [scoringutils#1114](https://github.com/epiforecasts/scoringutils/issues/1114) tracks adding `variogram_score_multivariate()` as a default compound metric. Confirm its merge status before closing Sprint C. hubEvals PR #103 is already wired to pick it up dynamically once it lands.
- **What is the best way to specify the compound-task-id set to allow for multivariate scoring?** To compute a score on a multivariate outcome, such as variogram score or the energy score, we need to identify which task id variables define a compound-task-id. How can these be simply and flexibly specified and validated to enable these metrics? 
- **Compound_taskid_set + disaggregate_by conflict**: If a target has `disaggregate_by: location` and also uses a compound metric computed *across* locations (e.g., variogram score), should hubPredEvalsData skip disaggregation silently, warn, or error? e.g., how should overall aggregation work, or how would we turn off disaggregation by the task-id variable that defines the compound task id set. For example, if you wanted to compute the variogram score across all locations, then you couldn't disaggregate the computation by location.
- **Issue #44 (human-readable target names)**: ~~Is [hubPredEvalsData#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21) resolved? If not, #44 has an unresolved upstream dependency.~~ **Resolved**: Issue #21 is not resolved so there is an upstream dependency.
- **Issue #4 (default sort column)**: ~~Should this be configured in `predevals-config.yml` (schema change) or via a standalone predevals options object (JS only)?~~ **Resolved**: configure in `predevals-config.yml` as a schema change; included in Sprint B's v1.1.0 schema bump.
- **Which scoringutils metrics are already surfaceable?** For quantile forecasts, any metric in `scoringutils::get_metrics(scoringutils::example_quantile)` is already available via the existing `get_standard_metrics()` pathway — no schema or code changes needed, just config. For sample forecasts, once PR #103 merges, CRPS/bias/DSS/energy score become available the same way. New scoringutils defaults propagate automatically.


---

## 👍 Ready to make it

### Proposed solution

Four mini-sprints ordered by implementation complexity: UI-only first, then schema additions, then new metric pipeline features. Each sprint is independently releasable.

The core complexity axis is:

> **UI-only** (predevals JS/CSS, no schema changes) → **Schema-driven** (hubPredEvalsData schema + R pipeline + JS) → **Full pipeline** (R packages + schema + JS across 4 repos)

---

### Mini-Sprint A — Dashboard UI improvements
*Scope: predevals JS/CSS only. No schema changes. No R package changes.*

Sprint A is split into two tiers. **A1** contains table readiness work that is a prerequisite for Sprint B's scaled-metrics columns. **A2** contains important usability improvements that are not on the critical path for scaled metrics.

**A1 — Table readiness (blocking for Sprint B)**

| Issue | Change | Complexity |
|-------|--------|-----------|
| [#49](https://github.com/hubverse-org/predevals/issues/49) **priority** | Freeze model name column; horizontal scroll for other columns | CSS/JS table layout |
| [#50](https://github.com/hubverse-org/predevals/issues/50) | Enable column hiding via ColumnControl visibility toggle (one-line config change; ColumnControl already loaded) | JS config (trivial) |

**A2 — UI polish (non-blocking)**

| Issue | Change | Complexity |
|-------|--------|-----------|
| [#31](https://github.com/hubverse-org/predevals/issues/31) **priority** | Auto-update metric + disaggregate_by selectors when target changes | JS logic fix |
| [#5](https://github.com/hubverse-org/predevals/issues/5) **priority** | Preserve model selection state when switching plots | JS state management |
| [#42](https://github.com/hubverse-org/predevals/issues/42) **priority** | Add "lower is better" / "closer to nominal is better" cues to axes and table headers | JS + display logic |
| [#13](https://github.com/hubverse-org/predevals/issues/13) | Add metric definitions panel with abbreviations and direction indicators, shown by default and togglable | JS; metric glossary baked into predevals.js |
| [#34](https://github.com/hubverse-org/predevals/issues/34) | Rename all `predeval` → `predevals` references in source | JS cleanup (trivial) |
| [#22](https://github.com/hubverse-org/predevals/issues/22) | Add unit tests for existing JS functionality; establishes test harness for TDD in subsequent sprints | JS testing infrastructure |

**Deliverable**: New predevals release(s). No changes to hubPredEvalsData, hubEvals, or Docker.

**Sprint A — Acceptance criteria:**
- A1 issues resolved and closed (required before Sprint B)
- A2 issues resolved and closed
- JS test harness established (#22) with tests covering new functionality
- CI passes; new predevals release tagged

---

### Mini-Sprint B — Evaluation config schema updates (v1.1.0)
*Scope: All hubPredEvalsData schema changes consolidated into a single v1.1.0 release + corresponding predevals JS + R pipeline changes. hubEvals unchanged (transforms already implemented).*

This sprint consolidates config-driven UI enhancements and the scale transformation pipeline into a single schema bump, since schema updates are painful to coordinate across repos. The schema update checklist from the previous version bump is in [hubPredEvalsData#32](https://github.com/hubverse-org/hubPredEvalsData/issues/32).

**Config-driven UI enhancements:**

| Issue | Change | Where |
|-------|--------|-------|
| [predevals#28](https://github.com/hubverse-org/predevals/issues/28) | Refactor numeric rounding into a shared helper (prerequisite for #48) | predevals JS refactor |
| [predevals#48](https://github.com/hubverse-org/predevals/issues/48) **priority** | Add `decimal_places` per-target in config schema; JS reads and applies it for table display | hubPredEvalsData schema + predevals JS |
| [predevals#27](https://github.com/hubverse-org/predevals/issues/27) | Refactor score-sorting into a helper function (prerequisite for #4) | predevals JS refactor |
| [predevals#4](https://github.com/hubverse-org/predevals/issues/4) **priority** | Add `default_sort_metric` to config/options; JS uses it for initial table sort (e.g., sort models by relative WIS ascending on first load rather than alphabetically) | hubPredEvalsData schema + predevals JS |
| [predevals#44](https://github.com/hubverse-org/predevals/issues/44) | Display `target_name` from `tasks.json` instead of `target_id` in dropdowns | hubPredEvalsData (depends on [#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21)) + predevals JS |

> ⚠️ Issue #44 depends on upstream hubPredEvalsData work ([#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21)). Include only if that work is complete.

**Scale transformation pipeline:**

The detailed implementation plan is in [hubPredEvalsData#34](https://github.com/hubverse-org/hubPredEvalsData/issues/34), which covers the schema design, config validation, R pipeline changes, and predevals JS behavior.

| Issue | Change | Where |
|-------|--------|-------|
| [predevals#30](https://github.com/hubverse-org/predevals/issues/30) | Refactor code for getting full metrics list (prerequisite for per-scale metric treatment) | predevals JS refactor |
| [hubPredEvalsData#34](https://github.com/hubverse-org/hubPredEvalsData/issues/34) | Add `transform_defaults` (top-level) and per-target `transform` override to config schema; wire through to `hubEvals::score_model_out()` | hubPredEvalsData schema + R pipeline |
| — | When `append: true`, scores.csv gains a `scale` column; predevals treats each (metric × scale) combination as a distinct item in dropdowns and table columns | predevals JS |

**Deliverable**: hubPredEvalsData v1.1.0 schema, new predevals release, Docker rebuild.

**Sprint B — Acceptance criteria:**
- All config-driven UI issues resolved and closed (#44 only if [hubPredEvalsData#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21) is resolved)
- Scale transformation pipeline functional end-to-end: hub admin can configure log-scale evaluation in `predevals-config.yml` and see log-scaled metrics appear alongside natural-scale metrics
- hubPredEvalsData schema v1.1.0 released; R tests pass; schema update checklist ([#32](https://github.com/hubverse-org/hubPredEvalsData/issues/32)) followed
- Docker image rebuilt and integration test passes
- New predevals release tagged

---

### Mini-Sprint C — Sample-based scoring with compound_taskid_set
*Scope: Merge near-complete hubEvals PR, then wire sample scoring + compound_taskid_set awareness through hubPredEvalsData and predevals.*

This sprint builds pipeline support for sample-format output types that require [`compound_taskid_set`](https://docs.hubverse.io/en/latest/user-guide/sample-output-type.html#compound-modeling-tasks) awareness. The variogram score is the motivating use case, but the infrastructure enables any future compound/multivariate metric for sample forecasts. See also [hubDocs PR #439](https://github.com/hubverse-org/hubDocs/pull/439) for incoming updates to compound modeling task documentation.

**hubEvals changes — mostly done:**
[PR #103](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/pull/103) (branch `ak/sample-scoring/94`) is open and awaiting final review. It adds:
- `transform_sample_model_out()` — converts hubverse sample format to scoringutils-compatible objects
- `"sample"` as a valid output type in `validate_output_type()`
- Marginal scoring (CRPS, bias, DSS) and compound/multivariate scoring (energy score) via `compound_taskid_set`
- Dynamically generates compound metric list from scoringutils — so variogram score will be picked up automatically once [scoringutils#1114](https://github.com/epiforecasts/scoringutils/issues/1114) lands and bumps the default metrics
- Requires scoringutils ≥ 2.1.2.9000
- Open issues on the branch: [#99](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/issues/99) (NaN/Inf validation), [#100](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/issues/100) (test updates), [#101](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/issues/101) (compound_taskid_set validation), [#102](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/issues/102) (test warnings)

**Action**: Review and merge PR #103; track scoringutils#1114 for variogram score availability.

**hubPredEvalsData changes** (extends v1.1.0 schema from Sprint B or adds v1.2.0):
- Add `compound_taskid_set` optional property to target config
- Add `"variogram_score"` as a recognized metric name for sample output types
- `R/utils-metrics.R` — add `sample = "variogram_score"` case in `get_standard_metrics()`
- `R/generate_eval_data.R` — extract and propagate `compound_taskid_set`; skip location-based disaggregation for compound metrics when `compound_taskid_set = "location"`

**predevals JS changes:**
- No new chart types; variogram score appears as another column in the overall scores table
- Graceful handling of missing metric columns in disaggregated views (may already work)

> ⚠️ **Scope constraint**: Compound metrics computed jointly across a dimension (e.g., variogram score across locations) are **not** disaggregable by that dimension. Document this in the config schema and validation error messages.

**hubPredEvalsData-docker changes:**
- [docker#6](https://github.com/hubverse-org/hubPredEvalsData-docker/issues/6): Replace ad-hoc oracle data fetching with `hubData` tooling, so the container fetches oracle output through the standard hubverse data access layer rather than direct file paths

**Deliverable**: hubEvals new minor version, hubPredEvalsData schema update, Docker rebuild (with hubData oracle fetching).

**Sprint C — Acceptance criteria:**
- hubEvals PR #103 merged with issues #99–#102 resolved
- hubPredEvalsData supports `sample` output type with `compound_taskid_set` config; validates against `disaggregate_by` conflicts
- Variogram score appears in predevals dashboard (once scoringutils#1114 lands)
- Docker image uses `hubData` for oracle fetching ([docker#6](https://github.com/hubverse-org/hubPredEvalsData-docker/issues/6))
- `R CMD check` passes for both hubEvals and hubPredEvalsData

---

### Mini-Sprint D — Documentation
*Scope: Extend the [existing developer guide](https://docs.hubverse.io/en/latest/developer/dashboard-predevals.html) + predevals JS (end-user metric definitions, partly overlapping with Sprint A #13). Can overlap with other sprints.*

The existing guide covers infrastructure well (Docker setup, renv, build process, integration testing). It does **not** cover metric development workflow. Sprint D adds that missing layer as a new section or sibling page within the same developer guide.

**New content to add to hubDocs:**
1. Architecture diagram: hubEvals → hubPredEvalsData → predevals → dashboard (with pointers to existing infrastructure docs)
2. How to add a metric in hubEvals: implement a transform function, update `score_model_out`, update `get_metrics`, write tests
3. How to wire it through hubPredEvalsData: add metric name to schema, `get_standard_metrics`, `get_metric_name_to_output_type`, config validation
4. How predevals reads scores CSVs — when new metrics appear automatically vs. when JS changes are needed
5. Worked example: sample-based scoring with compound_taskid_set end-to-end (referencing Sprints B + C as concrete prior art)

**End-user metric documentation** (builds on Sprint A #13):
- If not fully addressed in Sprint A, finalize a metric glossary embedded in predevals covering: WIS, ae_point, se_point, interval coverage, variogram score
- Hub-level task variable definitions handled separately (via hub model-output README, not in predevals)

**Deliverable**: New or extended hubDocs page; predevals minor release if glossary was deferred from Sprint A.

**Sprint D — Acceptance criteria:**
- A developer unfamiliar with the codebase can follow the guide to add a toy metric end-to-end without reading source across repos
- All metric names in the guide are present in the predevals JS glossary (#13)
- hubDocs CI passes

---

### Files to modify across all sprints

| Repo | Files | Sprint |
|------|-------|--------|
| hubEvals | `R/validate.R`, `R/score_model_out.R`, new `R/transform_sample_model_out.R` | C |
| hubPredEvalsData | `inst/schema/v1.1.0/config_schema.json` (new), `R/config.R`, `R/utils-metrics.R`, `R/generate_eval_data.R` | B, C |
| predevals | `src/predevals.js` and related source files | A, B, C |
| hubPredEvalsData-docker | Dockerfile / entrypoint | B, C |
| hubDocs | New developer + end-user guide page | D |

### Scale and scope

- **Sequencing**: Sprint A1 must complete before Sprint B. All other ordering is flexible.

```
Sprint A1 (table readiness)  ──► Sprint B (schema v1.1.0) ──► Sprint C (sample scoring)
Sprint A2 (UI polish)        ──────────────────────────────────────────► release
Sprint D  (docs)             ──────────────────────────────────────────► release
```

> **Note**: Sprint D can begin early, but cannot fully close until Sprint C is complete. The worked example (item 5 in the content list) depends on Sprints B and C as concrete prior art.

### Key risks

1. **scoringutils#1114 timing**: Variogram score will land in hubEvals automatically once scoringutils#1114 merges, but the timeline is upstream-dependent. The rest of Sprint C (hubPredEvalsData pipeline, compound_taskid_set support) can proceed with energy score in the meantime.
2. **hubPredEvalsData#21 dependency**: Sprint B's issue #44 (human-readable target names) cannot land until that upstream issue is resolved.
3. **Table width**: Treating scaled metrics as separate columns (Sprint B) will widen the evaluation table significantly. Sprint A1's frozen-column fix ([#49](https://github.com/hubverse-org/predevals/issues/49)) is a hard prerequisite.
