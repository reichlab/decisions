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

### Development standards

Every issue follows a **two-phase workflow** before any code is written:

#### Phase 1 — Issue refinement

Before an implementer picks up an issue, the issue must be rewritten (if necessary) so that it is:

- **Specific and testable**: describes a concrete, observable outcome rather than a vague intent. For example, not "fix the target-change bug" but "when a new target is selected, the metric dropdown resets to the first valid metric for that target and the disaggregate_by dropdown resets to 'overall'."
- **Single-concern**: one responsibility per issue; split if needed.
- **Unblocked**: all upstream dependencies are resolved.

If the issue text isn't clear enough to write a test from, update it before starting implementation.

#### Phase 2 — Test-Driven Development (TDD)

1. **Write a failing test** that encodes the specific outcome from the refined issue.
2. **Implement the minimum code** to make the test pass.
3. **Refactor** with the test suite green.

**Special cases where the TDD sequence is adapted:**

| Case | Adapted sequence |
|------|-----------------|
| **Refactors** (#27, #28, #30) | Write *characterization tests* against the existing behaviour first → refactor → confirm tests still pass. No observable behaviour should change. |
| **Rename** (#34) | Not TDD. Completion is verified by a grep/search confirming no old string remains. |
| **Documentation** (Sprint E) | Not TDD. Acceptance criterion: a developer unfamiliar with the codebase can follow the guide and add a toy metric in a local dev environment (verified by peer walkthrough). |
| **Docker integration** (docker#6) | Write the integration test against current oracle-fetching behaviour → migrate to hubData → confirm test still passes. |

**Tooling by repo:**

| Repo | Test framework | Notes |
|------|---------------|-------|
| hubEvals (R) | `testthat` via `devtools::test()` | Hand-computed expected values where possible (see PR #103 pattern) |
| hubPredEvalsData (R) | `testthat` via `devtools::test()` | Integration tests using example hub data |
| predevals (JS) | To be established in Sprint A [#22](https://github.com/hubverse-org/predevals/issues/22) | Unit test framework (e.g. Jest or Vitest) chosen during Sprint A setup |
| hubPredEvalsData-docker | Integration tests in GitHub Actions | Compare CSV outputs between image versions (existing pattern) |

**Universal Definition of Done** — every issue is closed only when:

- [ ] The issue was refined to a specific, testable outcome *before* any code was written
- [ ] A failing test encoding that outcome was written *before* the implementation (or the appropriate adapted sequence above was followed)
- [ ] All existing tests continue to pass (`R CMD check` / CI green)
- [ ] The change is documented (inline comments for non-obvious logic; function-level docs for new public API)
- [ ] A PR is reviewed and approved by at least one other team member
- [ ] The relevant GitHub issue is referenced in the PR and closed on merge

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

**Sprint A — Definition of Done:**
- [ ] #22: Test harness chosen and one passing test for an existing function exists; all subsequent issues in this sprint add tests to it
- [ ] #31: Issue refined to specify exact reset behaviour → failing test written → implemented. Test: selecting a new target resets metric and disaggregate_by to the first valid value for that target
- [ ] #5: Issue refined to specify which state is preserved and when → failing test written → implemented. Test: model selection is unchanged after switching between table/heatmap/line views
- [ ] #49: Issue refined to name the exact column and scroll behaviour → failing DOM test written → implemented. Test: first column has `position: sticky` and table body scrolls independently
- [ ] #42: Issue refined per metric (which direction, what text) → failing tests written → implemented. Tests: each metric name has a registered direction; that direction renders correctly in headers and axis labels
- [ ] #13: Issue refined with complete metric glossary entries → failing test written → implemented. Test: glossary panel present on load; toggles; every metric name used in the dashboard has a glossary entry
- [ ] #50: Add `'visibility'` to `columnControl` array; test: each metric column header has a working hide/show toggle; `model_id` column is excluded from hiding
- [ ] #30: Characterization tests written against *existing* metrics-list behaviour → refactored → tests still pass. No behaviour change
- [ ] #34: Grep confirms zero occurrences of `predeval` (without trailing `s`) in `src/`
- [ ] CI passes; new predevals release tagged

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

**Sprint B — Definition of Done:**
- [ ] #28: Characterization tests written against existing rounding behaviour → refactored into shared helper → tests still pass. Must be merged before #48 is started
- [ ] #48: Issue refined to specify which targets need non-default precision and what values are valid → failing tests written (R: schema rejects invalid `decimal_places`; JS: rendered table rounds to configured places) → implemented
- [ ] #27: Characterization tests written against existing sorting behaviour → refactored into helper → tests still pass. Must be merged before #4 is started
- [ ] #4: Issue refined to specify fallback behaviour when metric is absent → failing tests written (R: schema accepts/rejects new field; JS: initial sort matches config, falls back to alphabetical when field absent) → implemented
- [ ] **Before starting #44**: confirm [hubPredEvalsData#21](https://github.com/hubverse-org/hubPredEvalsData/issues/21) is resolved; skip #44 in this sprint if not
- [ ] #44 (if included): Issue refined to specify behaviour when `target_name` is missing from data → failing test written → implemented. Test uses fixture data with and without `target_name`
- [ ] R `testthat` tests cover new schema properties end-to-end through `generate_eval_data()`, written before config.R is modified
- [ ] Docker image rebuilt and integration test passes

---

### Mini-Sprint C — Scale transformation pipeline (~4 weeks)
*Scope: hubPredEvalsData schema v1.1.0 additions + predevals JS for scale UI. hubEvals unchanged (transforms already implemented).*

**hubPredEvalsData changes:**
- Add `transform_defaults` (top-level) and per-target `transform` to `inst/schema/v1.1.0/config_schema.json`
- Allowed transform functions: `log_shift`, `sqrt`, `log1p`, `log`, `log10`, `log2`
- `append: true/false` — when true, scores.csv gains a `scale` column (`"natural"` or transform label)
- Add `validate_config_transforms()` in `R/config.R`
- Wire resolved transform config into `get_scores_for_output_type()` → `hubEvals::score_model_out(transform=..., transform_append=..., ...)`

**predevals JS changes:**
- When the `scale` column is present in scores data, treat each (metric × scale) combination as a distinct metric: e.g., "wis (natural)" and "wis (log)" appear as separate items in dropdowns and as separate columns in tables
- No separate filter/toggle; scales are just more metrics
- Info banner when any transformed metrics are present
- **Note**: Sprint A's table ergonomics work ([#49](https://github.com/hubverse-org/predevals/issues/49) fixed column) should land before or alongside this sprint, since adding scale variants doubles the number of metric columns

**Deliverable**: hubPredEvalsData v1.1.0 schema, new predevals release, Docker rebuild. Resolves [hubPredEvalsData#34](https://github.com/hubverse-org/hubPredEvalsData/issues/34).

**Sprint C — Definition of Done:**
- [ ] Each behaviour below has its failing test written and merged to a test branch *before* the corresponding R or JS implementation is written
- [ ] R: `generate_eval_data()` with `transform_defaults: {function: log_shift, append: true}` produces `scores.csv` with both `scale = "natural"` and `scale = "log_shift"` rows
- [ ] R: config with an invalid transform function name fails `validate_config_transforms()` with a clear error message
- [ ] R: config applying a transform to a `pmf` target fails validation
- [ ] R: per-target `transform: null` correctly overrides `transform_defaults` (hierarchical override)
- [ ] JS: when `scale` column present, metric dropdown contains `"wis (natural)"` and `"wis (log_shift)"` as distinct entries
- [ ] JS: table has one column per (metric × scale) combination
- [ ] JS: info banner visible when transformed metrics are present; absent otherwise
- [ ] Schema v1.1.0 is backward-compatible: all existing example configs validate against it without changes

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
- Add `joint_across` optional property to target config
- Add `"variogram_score"` as a recognized metric name for sample output types
- `R/utils-metrics.R` — add `sample = "variogram_score"` case in `get_standard_metrics()`
- `R/generate_eval_data.R` — extract and propagate `joint_across`; skip location-based disaggregation for sample metrics when `joint_across = "location"`

**predevals JS changes:**
- No new chart types; variogram score appears as another column in the overall scores table
- Graceful handling of missing metric columns in disaggregated views (may already work)

> ⚠️ **Scope constraint**: The variogram score is computed jointly across locations and is **not** disaggregable by location. Document this in the config schema and validation error messages.

**hubPredEvalsData-docker changes:**
- [docker#6](https://github.com/hubverse-org/hubPredEvalsData-docker/issues/6): Replace ad-hoc oracle data fetching with `hubData` tooling, so the container fetches oracle output through the standard hubverse data access layer rather than direct file paths

**Deliverable**: hubEvals new minor version, hubPredEvalsData v1.1.0 (with Sprint C changes), Docker rebuild (with hubData oracle fetching).

**Sprint D — Definition of Done:**
- [ ] Issues #99–#102 each refined and resolved via TDD before PR #103 is merged
- [ ] PR #103: each open issue has a failing test written first; hand-computed expected values for CRPS and energy score are confirmed correct before the implementation is accepted
- [ ] hubEvals: failing test for `score_model_out()` with `output_type = "sample"` returning the expected variogram score is written against energy score first (interim), then updated when scoringutils#1114 lands — written before implementation is touched
- [ ] hubPredEvalsData: issue for sample output type support is refined to specify exact config fields and error conditions → failing R tests written → `utils-metrics.R` and `generate_eval_data.R` implemented. Tests: `get_standard_metrics("sample")` returns expected names; `generate_eval_data()` with `joint_across: location` produces correct `scores.csv`
- [ ] hubPredEvalsData: failing validation test for the `joint_across` + `disaggregate_by` conflict written before `config.R` validation is modified
- [ ] docker#6: integration test written against *current* oracle-fetching behaviour → hubData migration implemented → same test still passes
- [ ] JS: issue refined to specify how missing variogram score column is handled in disaggregated views → failing test written → implemented
- [ ] `R CMD check` passes for both hubEvals and hubPredEvalsData

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

**Sprint E — Definition of Done:**
- [ ] The guide explicitly describes the issue-refinement + TDD workflow expected for each repo, including the special cases (refactors, renames, Docker)
- [ ] The guide is validated by a walkthrough: a developer unfamiliar with the codebase reads it and successfully adds a toy metric in a local dev environment without asking for help
- [ ] All metric names mentioned in the guide are present in the predevals JS glossary (#13)
- [ ] hubDocs CI (link checks, build) passes

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
