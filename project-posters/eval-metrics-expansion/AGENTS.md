# AGENTS.md — Eval Metrics Expansion Project

This file provides context for anyone (human or AI agent) arriving at this project poster for the first time.

## What this project is

A 1–3 month development sprint to expand the hubverse forecast evaluation ecosystem in two areas:
1. Better end-user and developer experience in the existing dashboard
2. Two new metric capabilities: log-scale scoring and the variogram score

The full plan is in **`eval-metrics-expansion.md`** in this directory.

## The hubverse evaluation pipeline (4 repos)

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

Documentation:
- User guide: https://docs.hubverse.io/en/latest/user-guide/dashboards.html#predevals-evaluation-optional
- Developer guide (infrastructure): https://docs.hubverse.io/en/latest/developer/dashboard-predevals.html
- predevals repo: https://github.com/hubverse-org/predevals
- hubPredEvalsData repo: https://github.com/hubverse-org/hubPredEvalsData
- hubEvals repo: https://github.com/Infectious-Disease-Modeling-Hubs/hubEvals
- hubPredEvalsData-docker repo: https://github.com/hubverse-org/hubPredEvalsData-docker

## Current state (as of 2026-02-26)

- **hubEvals v0.1.0** supports: quantile, mean, median, pmf output types; log/sqrt scale transforms already implemented in `score_model_out()`
- **hubEvals PR #103** (branch `ak/sample-scoring/94`) is open and nearly ready to merge — adds `sample` output type with CRPS, bias, DSS, energy score, and dynamic compound metric list (will pick up variogram score automatically once scoringutils#1114 lands)
- **hubPredEvalsData schema v1.0.1** — no support yet for scale transforms or sample output type
- **predevals** — functional but has multiple known UX gaps (see issues)
- **scoringutils** — variogram score being added upstream in issue #1114

## Sprint structure

The sprint is broken into 5 mini-sprints ordered by implementation complexity:

| Sprint | Scope | Repos touched | Approx. duration |
|--------|-------|--------------|-----------------|
| A | UI-only polish (bugs, ergonomics, metric docs) | predevals only | ~2 weeks |
| B | Config-driven enhancements (decimal places, sort, target names) | hubPredEvalsData schema + predevals | ~3 weeks |
| C | Scale transformation pipeline | hubPredEvalsData schema + predevals | ~4 weeks |
| D | Variogram score / sample scoring | hubEvals (PR #103) + hubPredEvalsData + predevals | ~3–4 weeks |
| E | Developer docs (extend existing guide) | hubDocs | ~2 weeks, can overlap |

Sprint A should go first; Sprint C depends on Sprint A's table ergonomics fix (#49). Sprints B, C, D share the hubPredEvalsData schema so should release sequentially (B → C → D). Sprint E can begin in parallel with Sprint B but cannot fully close until Sprint D is complete (the worked example depends on Sprints C + D as prior art).

## Key design decisions made

- **Scale metrics treated as separate metrics**: When `append=true` in transform config, log-scaled and natural-scale metrics appear as distinct columns/dropdown items (e.g., "wis (natural)" and "wis (log)"), not as a filter/toggle. This requires the frozen-column table fix (Sprint A #49) to land first.
- **Variogram score is compound (across locations)**: The variogram score is computed jointly across locations via `compound_taskid_set`. It is NOT disaggregatable by location. hubPredEvalsData should enforce this with a validation error.
- **Scoringutils metrics are automatically available**: Any metric that scoringutils returns as a default for a given output type flows through automatically via `get_standard_metrics()`. No schema/code changes needed beyond adding the output type — just config.

## Open questions (as of 2026-02-26)

- When does scoringutils#1114 (variogram score) land?
- How many hubs currently use sample-format forecasts?
- Should `disaggregate_by: location` + variogram score be a silent skip, warning, or error in hubPredEvalsData?
- Is hubPredEvalsData#21 (target metadata) resolved? Unblocks predevals#44.
- ~~Issue #4 (default sort column): schema change or JS-only?~~ **Resolved**: `predevals-config.yml` schema change, included in Sprint B v1.0.2.

## Key files to know

| File | Purpose |
|------|---------|
| `hubEvals/R/score_model_out.R` | Core scoring function; transform + joint_across params live here |
| `hubEvals/R/validate.R` | Output type validation |
| `hubPredEvalsData/R/utils-metrics.R` | `get_standard_metrics()` maps output types → metric names |
| `hubPredEvalsData/R/generate_eval_data.R` | Top-level orchestration; calls hubEvals |
| `hubPredEvalsData/R/config.R` | Config parsing + validation |
| `hubPredEvalsData/inst/schema/` | JSON schema versions for predevals-config.yml |
| `predevals/src/predevals.js` | Main JS source; reads CSV files, renders UI |
