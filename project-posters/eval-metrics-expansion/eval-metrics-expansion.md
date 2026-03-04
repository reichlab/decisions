# Project Poster: Expanding Hubverse Evaluation Metrics and Dashboard Support

- Date: 2026-02-26

- Owner: Nicholas Reich

- Status: draft

## ❓ Problem space

### What are we doing?

Extending the hubverse forecast evaluation ecosystem across five goals:

1. **UI polish for end users**: Fix known usability gaps in the predevals dashboard—metric documentation, score direction indicators, table ergonomics, and several priority bugs—without touching R packages or config schemas. Issues: [predevals#13](https://github.com/hubverse-org/predevals/issues/13), [#31](https://github.com/hubverse-org/predevals/issues/31), [#42](https://github.com/hubverse-org/predevals/issues/42), [#41](https://github.com/hubverse-org/predevals/issues/41), [#49](https://github.com/hubverse-org/predevals/issues/49), [#5](https://github.com/hubverse-org/predevals/issues/5). Explore whether column hiding (so users can make the tabler simpler to focus on just desired metrics) is possible using datatables.

2. **Config-driven UI enhancements**: Small additions to the hubPredEvalsData schema enabling per-target decimal precision, human-readable target names, and a configurable default sort column for the evaluation table. Issues: [predevals#48](https://github.com/hubverse-org/predevals/issues/48), [#44](https://github.com/hubverse-org/predevals/issues/44), [#4](https://github.com/hubverse-org/predevals/issues/4).

3. **Scale transformation pipeline**: Wire the already-implemented log/sqrt transform support in `hubEvals::score_model_out()` through the hubPredEvalsData config schema and the predevals UI so hub admins can evaluate forecasts on transformed scales. Issue: [hubPredEvalsData#34](https://github.com/hubverse-org/hubPredEvalsData/issues/34).

4. **Variogram score**: Add `sample` output type support to hubEvals and expose `variogram_score_multivariate()` and `variogram_score_multivariate_point()` (recently added to scoringutils) as a metric evaluating ensemble spatial correlation structure across locations.

5. **Developer documentation**: A hubDocs guide explaining the full metric pipeline for future contributors, plus standardized end-user metric definitions in the dashboard.

### Why are we doing this?

- Increasing evaluation diversity was the most highly ranked priority in a recent survey of hubverse users. This project both surfaces existing metrics, adds new ones, and makes all evaluations more interpretable.
- Many predevals usability gaps have been open for over a year; the dashboard is hard for non-expert users to interpret (no metric definitions, no "lower is better" cues, table ergonomics issues).
- Infectious disease forecasts predict count data that spans orders of magnitude across locations. Log-scale evaluation provides fairer cross-location comparisons.
- The variogram score is a multivariate proper scoring rule capturing spatial correlation—something WIS cannot measure—and scoringutils now supports it natively.
- The [existing developer guide](https://docs.hubverse.io/en/latest/developer/dashboard-predevals.html) covers infrastructure (Docker, build, testing) but not metric development workflow, making it hard for new contributors to add metrics end-to-end.

### What are we _not_ trying to do?

- Not adding new chart types (existing table/heatmap/line plot are sufficient).
- Not changing the existing WIS/AE/interval coverage metrics.
- Not implementing calibration/reliability diagram visualizations in this sprint.
- Not adding the variogram score for quantile-format forecasts (requires ensemble/sample format).

### How do we judge success?

- A first-time user reading the dashboard understands what WIS means, which direction is better, and which models are performing best—without leaving the page.
- A hub admin can configure log-scale evaluation in `predevals-config.yml` and see log-scaled metrics appear as distinct items alongside natural-scale metrics in all dropdowns and table columns.
- A hub submitting sample-format ensemble forecasts can configure and view the variogram score in the dashboard.
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
- scoringutils dev branch has `as_forecast_multivariate_sample()`, `variogram_score_multivariate()`, and `variogram_score_multivariate_point()`.
- hubPredEvalsData schema versioning is established (v0.1.0 → v1.0.0 → v1.0.1); v1.1.0 is the natural target for transform + variogram + config-driven enhancements.
- The predevals JS dashboard builds via webpack into `dist/predevals.bundle.js`; the existing table/heatmap/line plot handle new metric columns without new chart types.
- predevals issues #31, #5, #42, #41, #49 are pure JS/CSS changes with no schema dependencies.

### What do we need to answer?

- **Variogram score in scoringutils**: [scoringutils#1114](https://github.com/epiforecasts/scoringutils/issues/1114) tracks adding `variogram_score_multivariate()` as a default compound metric. Confirm its merge status before closing Sprint D. hubEvals PR #103 is already wired to pick it up dynamically once it lands.
- **How many active hubs submit `sample`-format forecasts?** This determines Sprint D's near-term impact. Although, note that the variogram point score could be used by almost all hubs, with some additional specification about joint across. It is possible that some specification would be needed for any variogram score to specify which dimension to assume the forecasts are joint across.
- **Variogram + disaggregate_by conflict**: If a target has `disaggregate_by: location` and also uses the variogram score (computed *across* locations), should hubPredEvalsData skip disaggregation silently, warn, or error?
- **Issue #44 (human-readable target names)**: Is [hubPredEvalsData#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21) resolved? If not, #44 has an unresolved upstream dependency.
- **Issue #4 (default sort column)**: ~~Should this be configured in `predevals-config.yml` (schema change) or via a standalone predevals options object (JS only)?~~ **Resolved**: configure in `predevals-config.yml` as a schema change; included in Sprint B's v1.0.2 bump.
- **Which scoringutils metrics are already surfaceable?** For quantile forecasts, any metric in `scoringutils::get_metrics(scoringutils::example_quantile)` is already available via the existing `get_standard_metrics()` pathway — no schema or code changes needed, just config. For sample forecasts, once PR #103 merges, CRPS/bias/DSS/energy score become available the same way. New scoringutils defaults propagate automatically.


---

## 👍 Ready to make it

### Proposed solution

Five mini-sprints ordered by implementation complexity: UI-only first, then incremental schema additions, then new metric features. Each sprint is independently releasable.

The core complexity axis is:

> **UI-only** (predevals JS/CSS, no schema changes) → **Config-driven** (small schema additions + JS) → **Full pipeline** (R packages + schema + JS across 4 repos)

---

### Mini-Sprint A — UI-only polish (~2 weeks)
*Scope: predevals JS/CSS only. No schema changes. No R package changes.*

| Issue | Change | Complexity |
|-------|--------|-----------|
| [#31](https://github.com/hubverse-org/predevals/issues/31) 🐛 **priority** | Auto-update metric + disaggregate_by selectors when target changes | JS logic fix |
| [#5](https://github.com/hubverse-org/predevals/issues/5) **priority** | Preserve model selection state when switching plots | JS state management |
| [#49](https://github.com/hubverse-org/predevals/issues/49) **priority** | Freeze model name column; horizontal scroll for other columns | CSS/JS table layout |
| [#42](https://github.com/hubverse-org/predevals/issues/42) **priority** | Add "lower is better" / "closer to nominal is better" cues to axes and table headers | JS + display logic |
| [#13](https://github.com/hubverse-org/predevals/issues/13) | Add metric definitions panel with abbreviations and direction indicators, shown by default and togglable | JS; metric glossary baked into predevals.js |
| [#50](https://github.com/hubverse-org/predevals/issues/50) | Enable column hiding via ColumnControl visibility toggle (one-line config change; ColumnControl already loaded) | JS config (trivial) |
| [#30](https://github.com/hubverse-org/predevals/issues/30) | Refactor code for getting full metrics list (prerequisite for Sprint C's per-scale metric treatment) | JS refactor |
| [#34](https://github.com/hubverse-org/predevals/issues/34) | Rename all `predeval` → `predevals` references in source | JS cleanup (trivial) |
| [#22](https://github.com/hubverse-org/predevals/issues/22) | Add unit tests for existing JS functionality; establishes test harness for TDD in subsequent sprints | JS testing infrastructure |

**Deliverable**: New predevals release. No changes to hubPredEvalsData, hubEvals, or Docker.

**Sprint A — Acceptance criteria:**
- All listed issues resolved and closed
- JS test harness established (#22) with tests covering new functionality
- CI passes; new predevals release tagged

---

### Mini-Sprint B — Config-driven enhancements (~3 weeks)
*Scope: Small hubPredEvalsData schema additions + corresponding predevals JS. Schema bump to v1.0.2.*

| Issue | Change | Where |
|-------|--------|-------|
| [predevals#28](https://github.com/hubverse-org/predevals/issues/28) | Refactor numeric rounding into a shared helper (prerequisite for #48) | predevals JS refactor |
| [predevals#48](https://github.com/hubverse-org/predevals/issues/48) **priority** | Add `decimal_places` per-target in config schema; JS reads and applies it for table display | hubPredEvalsData schema + predevals JS |
| [predevals#27](https://github.com/hubverse-org/predevals/issues/27) | Refactor score-sorting into a helper function (prerequisite for #4) | predevals JS refactor |
| [predevals#4](https://github.com/hubverse-org/predevals/issues/4) **priority** | Add `default_sort_metric` to config/options; JS uses it for initial table sort (e.g., sort models by relative WIS ascending on first load rather than alphabetically) | hubPredEvalsData schema (or predevals options object) + predevals JS |
| [predevals#44](https://github.com/hubverse-org/predevals/issues/44) | Display `target_name` from `tasks.json` instead of `target_id` in dropdowns | hubPredEvalsData (depends on [#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21)) + predevals JS |

> ⚠️ Issue #44 depends on upstream hubPredEvalsData work ([#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21)). Include only if that work is complete.

**Deliverable**: hubPredEvalsData v1.0.2 schema, new predevals release, Docker rebuild.

**Sprint B — Acceptance criteria:**
- All listed issues resolved and closed (#44 only if [hubPredEvalsData#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21) is resolved)
- hubPredEvalsData schema v1.0.2 released; R tests pass
- Docker image rebuilt and integration test passes
- New predevals release tagged

---

### Mini-Sprint C — Scale transformation pipeline (~4 weeks)
*Scope: hubPredEvalsData schema v1.1.0 additions + predevals JS for scale UI. hubEvals unchanged (transforms already implemented).*

The detailed implementation plan for this sprint is in [hubPredEvalsData#34](https://github.com/hubverse-org/hubPredEvalsData/issues/34), which covers the schema design, config validation, R pipeline changes, and predevals JS behavior.

**Summary**: Add `transform_defaults` (top-level) and per-target `transform` override to the hubPredEvalsData config schema. Wire the resolved transform config through to `hubEvals::score_model_out()`. When `append: true`, scores.csv gains a `scale` column and the predevals dashboard treats each (metric × scale) combination as a distinct item in dropdowns and table columns.

**Note**: Sprint A's table ergonomics work ([#49](https://github.com/hubverse-org/predevals/issues/49) fixed column) should land before or alongside this sprint, since adding scale variants increases the number of metric columns.

**Deliverable**: hubPredEvalsData v1.1.0 schema, new predevals release, Docker rebuild.

---

### Mini-Sprint D — Variogram score (~3–4 weeks)
*Scope: Merge near-complete hubEvals PR, then wire sample scoring + variogram through hubPredEvalsData and predevals.*

**hubEvals changes — mostly done:**
[PR #103](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/pull/103) (branch `ak/sample-scoring/94`) is open and awaiting final review. It adds:
- `transform_sample_model_out()` — converts hubverse sample format to scoringutils-compatible objects
- `"sample"` as a valid output type in `validate_output_type()`
- Marginal scoring (CRPS, bias, DSS) and compound/multivariate scoring (energy score) via `compound_taskid_set`
- Dynamically generates compound metric list from scoringutils — so variogram score will be picked up automatically once [scoringutils#1114](https://github.com/epiforecasts/scoringutils/issues/1114) lands and bumps the default metrics
- Requires scoringutils ≥ 2.1.2.9000
- Open issues on the branch: [#99](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/issues/99) (NaN/Inf validation), [#100](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/issues/100) (test updates), [#101](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/issues/101) (compound_taskid_set validation), [#102](https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals/issues/102) (test warnings)

**Action**: Review and merge PR #103; track scoringutils#1114 for variogram score availability.

**hubPredEvalsData changes** (extends v1.1.0 schema from Sprint C):
- Add `compound_taskid_set` optional property to target config
- Add `"variogram_score"` as a recognized metric name for sample output types
- `R/utils-metrics.R` — add `sample = "variogram_score"` case in `get_standard_metrics()`
- `R/generate_eval_data.R` — extract and propagate `compound_taskid_set`; skip location-based disaggregation for sample metrics when `compound_taskid_set = "location"`

**predevals JS changes:**
- No new chart types; variogram score appears as another column in the overall scores table
- Graceful handling of missing metric columns in disaggregated views (may already work)

> ⚠️ **Scope constraint**: The variogram score is computed jointly across locations and is **not** disaggregable by location. Document this in the config schema and validation error messages.

**hubPredEvalsData-docker changes:**
- [docker#6](https://github.com/hubverse-org/hubPredEvalsData-docker/issues/6): Replace ad-hoc oracle data fetching with `hubData` tooling, so the container fetches oracle output through the standard hubverse data access layer rather than direct file paths

**Deliverable**: hubEvals new minor version, hubPredEvalsData v1.1.0 (with Sprint C changes), Docker rebuild (with hubData oracle fetching).

**Sprint D — Acceptance criteria:**
- hubEvals PR #103 merged with issues #99–#102 resolved
- hubPredEvalsData supports `sample` output type with `compound_taskid_set` config; validates against `disaggregate_by` conflicts
- Variogram score appears in predevals dashboard (once scoringutils#1114 lands)
- Docker image uses `hubData` for oracle fetching ([docker#6](https://github.com/hubverse-org/hubPredEvalsData-docker/issues/6))
- `R CMD check` passes for both hubEvals and hubPredEvalsData

---

### Mini-Sprint E — Documentation (~2 weeks, can overlap with D)
*Scope: Extend the [existing developer guide](https://docs.hubverse.io/en/latest/developer/dashboard-predevals.html) + predevals JS (end-user metric definitions, partly overlapping with Sprint A #13).*

The existing guide covers infrastructure well (Docker setup, renv, build process, integration testing). It does **not** cover metric development workflow. Sprint E adds that missing layer as a new section or sibling page within the same developer guide.

**New content to add to hubDocs:**
1. Architecture diagram: hubEvals → hubPredEvalsData → predevals → dashboard (with pointers to existing infrastructure docs)
2. How to add a metric in hubEvals: implement a transform function, update `score_model_out`, update `get_metrics`, write tests
3. How to wire it through hubPredEvalsData: add metric name to schema, `get_standard_metrics`, `get_metric_name_to_output_type`, config validation
4. How predevals reads scores CSVs — when new metrics appear automatically vs. when JS changes are needed
5. Worked example: variogram score end-to-end (referencing Sprints C + D as concrete prior art)

**End-user metric documentation** (builds on Sprint A #13):
- If not fully addressed in Sprint A, finalize a metric glossary embedded in predevals covering: WIS, ae_point, se_point, interval coverage, variogram score
- Hub-level task variable definitions handled separately (via hub model-output README, not in predevals)

**Deliverable**: New or extended hubDocs page; predevals minor release if glossary was deferred from Sprint A.

**Sprint E — Acceptance criteria:**
- A developer unfamiliar with the codebase can follow the guide to add a toy metric end-to-end without reading source across repos
- All metric names in the guide are present in the predevals JS glossary (#13)
- hubDocs CI passes

---

### Files to modify across all sprints

| Repo | Files | Sprint |
|------|-------|--------|
| hubEvals | `R/validate.R`, `R/score_model_out.R`, new `R/transform_sample_model_out.R` | D |
| hubPredEvalsData | `inst/schema/v1.0.2/config_schema.json` (new), `inst/schema/v1.1.0/config_schema.json` (new), `R/config.R`, `R/utils-metrics.R`, `R/generate_eval_data.R` | B, C, D |
| predevals | `src/predevals.js` and related source files | A, B, C, D |
| hubPredEvalsData-docker | Dockerfile / entrypoint | B, C, D |
| hubDocs | New developer + end-user guide page | E |

### Scale and scope

- **Duration**: 1–3 months total across all sprints; Sprints A and E can run in parallel with others
- **Sequencing**:

```
Sprint A (UI polish, ~2w)     ──────────────────────────► release
Sprint B (config enhancements, ~3w)    ─────────────────► release
Sprint C (scale transforms, ~4w)            ────────────► release
Sprint D (variogram score, ~3–4w)                ────────► release
Sprint E (docs, ~2w)               ──────────────────────► release
```

> **Note**: Sprint E can begin in parallel with Sprint B, but cannot fully close until Sprint D is complete. The worked example (item 5 in the content list) depends on Sprints C and D as concrete prior art.

### Key risks

1. **scoringutils#1114 timing**: Variogram score will land in hubEvals automatically once scoringutils#1114 merges, but the timeline is upstream-dependent. The rest of Sprint D (hubPredEvalsData schema, pipeline) can proceed with energy score in the meantime.
2. **hubPredEvalsData#21 dependency**: Sprint B's issue #44 (human-readable target names) cannot land until that upstream issue is resolved.
3. **Schema versioning coordination**: Sprints B, C, and D all modify the hubPredEvalsData schema. Recommend sequential release of B → C → D to avoid conflicting bumps.
4. **Table width**: Treating scaled metrics as separate columns (Sprint C) will widen the evaluation table significantly. Sprint A's frozen-column fix ([#49](https://github.com/hubverse-org/predevals/issues/49)) is a soft prerequisite for Sprint C.
