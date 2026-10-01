# Project Poster: Output-type subclasses of `model_out_tbl`

- Date: 2026-06-19

- Owner: Anna Krystalli

- Team: Anna Krystalli, Nicholas Reich, Li Shandross, Lucie Contamin (input from Sam Abbott and Nikos Bosse, scoringutils)

- Status: draft

## ❓ Problem space

### What are we doing?

Introduce output-type subclasses of the hubverse `model_out_tbl`: `quantile`, `cdf`, `pmf`, `sample`, `mean` and `median`. Each subclass holds model output of a single output type. Its class vector extends the existing `model_out_tbl` class, so everything that works on a `model_out_tbl` keeps working:

```r
c("quantile", "model_out_tbl", "tbl_df", "tbl", "data.frame")
```

A subclass is constructed from a `model_out_tbl` by a coercion function, one per output type: `as_model_out_quantile()`, `as_model_out_pmf()`, `as_model_out_sample()` and so on. The constructor keeps the rows of that output type and validates them against what the output type requires (see Validation of a typed object below).

Two output types have an additional statistical type the output type alone does not express: a pmf is nominal or ordinal, and a sample is marginal or joint. The statistical type determines the valid metrics, ensembling methods and plots, and is declared in the hub config rather than carried by the data. The typed object records the statistical type as a second subclass, placed before the output type in the class vector, so that methods can dispatch on both:

```r
c("ordinal", "pmf", "model_out_tbl", "tbl_df", "tbl", "data.frame")
c("joint", "sample", "model_out_tbl", "tbl_df", "tbl", "data.frame")
```

The subclasses are shared hubverse infrastructure. Scoring in hubEvals, ensembling in hubEnsembles and plotting in hubVis2 all have to treat each output type differently, and today each package does so with its own `switch(output_type, ...)` branches. With typed objects, each package instead supplies methods on the subclasses, and a new output type or statistical type is added as a class plus methods.

The first consumer is a bridge to [scoringutils](https://epiforecasts.io/scoringutils/), built in hubEvals, which already depends on scoringutils. A typed object converts to the matching scoringutils forecast object in one call, `as_scoringutils_forecast()`, and scored output converts back with `as_hubverse()`. A user can also jump straight from a `model_out_tbl` into a chosen scoringutils workflow, for example `as_forecast_quantile(my_model_out_tbl)`, through `model_out_tbl` methods on the scoringutils constructors.

This poster covers the class system in hubUtils and the scoringutils bridge in hubEvals. Ensembling and plotting methods on the new classes are follow-on work in hubEnsembles and hubVis2.

### Why are we doing this?

- **Dispatch instead of branching.** The hubverse has a single `model_out_tbl` class today, with no output-type subclasses and no formal methods, so every package re-derives output-type handling for itself. A typed object flows from data read through scoring, ensembling and plotting, and each package selects behaviour by dispatch on the same set of classes.
- **The statistical type has to live somewhere.** It is declared in config, not carried by the data, and it decides which methods are valid. A typed object is the natural place to record and dispatch on it.
- **Interoperability with scoringutils.** scoringutils is built around per-output-type classes and is designed for users to write `as_forecast_*.<their_format>()` converters, score, then convert back ([Sam Abbott](https://github.com/hubverse-org/hubEvals/issues/94#issuecomment-3891921846)). Typed hubverse objects let a user compose their own evaluation with the full scoringutils toolkit rather than only what `score_model_out()` exposes.
- **A more modular scoring pipeline, as a byproduct.** `score_model_out()` stays as it is, the one function dashboards and other consumers call. The class layer would let its internals dispatch into scoringutils rather than branch over `output_type`, if and when that refactor is worthwhile, with no change to the public API.

### What are we _not_ trying to do?

- Not changing the public interface or behaviour of `score_model_out()`.
- Not making the class layer or the scoring layer read hub config (`tasks.json`). hubEvals operates on model output extracted from a hub, not on the hub itself ([context](https://github.com/hubverse-org/hubEvals/issues/94#issuecomment-3892376999)). Config-derived values are passed in as arguments.
- Not reimplementing scoringutils classes, validation or metrics. The connectors stay thin wrappers around the scoringutils constructors, which validate their own inputs ([Sam Abbott](https://github.com/hubverse-org/hubEvals/issues/94#issuecomment-3892781599)).
- Not adding a scoringutils dependency to hubUtils. The class system stays dependency-light in hubUtils; everything that touches scoringutils lives in hubEvals.
- Not delivering the ensembling and plotting methods. This project establishes the class layer and the first consumer; the downstream methods follow in their own packages.

### How do we judge success?

- A `model_out_tbl` can be coerced to an output-type subclass that holds only that type's rows and is validated for that type.
- Hubverse packages can select behaviour by dispatch on the subclass, so a new output type or statistical type is added as a class plus methods rather than a new `switch` branch.
- `as_scoringutils_forecast()` routes every typed subclass with a scoringutils equivalent to the correct scoringutils constructor, so a user can score with scoringutils directly without going through `score_model_out()`.
- A round trip (`model_out_tbl` to scoringutils forecast object to scored output to `as_hubverse()`) preserves data fidelity, with test coverage.
- Optional: `score_model_out()` can be re-expressed on the class layer with no change to its public behaviour.

### What are possible solutions?

A class hierarchy layered on the existing base class, with coercion constructors and S3 methods, and the statistical type of a pmf or sample encoded as a second subclass. See "Ready to make it" for the design and the alternative considered.

## ✅ Validation

### What do we already know?

**The base class.** `model_out_tbl` already exists in hubUtils as a subclass of `tbl_df`. Subclasses extend its class vector, so existing `model_out_tbl` methods keep working.

**scoringutils is already S3 and designed to be dispatched into.** `score()` and `as_forecast_*()` are S3 generics ([score.R](https://github.com/epiforecasts/scoringutils/blob/main/R/score.R)). Sam Abbott confirmed that a "mask" over the scoringutils API is feasible: hubverse-named classes that dispatch into the scoringutils generics, mostly as pass-throughs, following the `as_forecast_sample.<format>` pattern used by [epinowcast](https://github.com/epinowcast/epinowcast/blob/main/R/model-validation.R). A few scoringutils touch points (such as [`summarise_scores`](https://github.com/epiforecasts/scoringutils/blob/main/R/summarise_scores.R)) may need minor tweaks for the pattern to work cleanly.

**The transform logic already exists.** hubEvals has one internal transform function per output type (`transform_quantile_model_out()`, `transform_pmf_model_out()`, `transform_point_model_out()`, `transform_sample_model_out()`) plus the marginal/joint sample work from [PR #103](https://github.com/hubverse-org/hubEvals/pull/103). These become the `as_scoringutils_forecast()` methods: one generic dispatching on the subclass replaces the per-type functions and the `switch` that selects between them.

**The hubverse-to-scoringutils crosswalk** (from the [issue #94 plan](https://github.com/hubverse-org/hubEvals/issues/94)):

| Hubverse | scoringutils |
|---|---|
| `model_id` | `model` |
| `value` | `predicted` |
| `oracle_value` | `observed` |
| `output_type_id` (sample) | `sample_id` |
| modeling task (task ID combination) | forecast unit |
| `compound_taskid_set` | inverse of `joint_across` |

**Output types and scoringutils classes do not map one to one, and neither side is consistently finer.** The hubverse is finer on point forecasts: mean and median both become `forecast_point`. scoringutils is finer on pmf and sample forecasts, where the statistical type decides the class. cdf has no scoringutils equivalent.

| Hubverse output type | scoringutils class | What decides the scoringutils class |
|---|---|---|
| quantile | `forecast_quantile` | nothing further |
| mean | `forecast_point` | nothing further; the `output_type` column keeps mean and median apart on the way back |
| median | `forecast_point` | as for mean |
| pmf | `forecast_nominal` or `forecast_ordinal` | whether an `output_type_id_order` is supplied (config) |
| sample | `forecast_sample` or `forecast_sample_multivariate` | the `compound_taskid_set` (config) |
| cdf | none | n/a |

### What do we need to answer?

The questions in the first draft of this poster (constructor semantics for mixed input, what metadata to carry, and the name of the connector) were settled in review. The decisions are recorded under "Ready to make it". Still open:

1. **Coarser compound task ID sets.** A hub's config declares the finest `compound_taskid_set` it accepts, and a coarser submission is valid. When a caller passes a coarser set than the data supports, should the constructor subset to the matching samples with a warning, or error and ask for a valid subset?
2. **A strict validation layer.** Should the constructors offer `strict = TRUE`, running the value checks hubValidations applies on submission, for data that did not come through a hub (see "Validation of a typed object")? It means moving the reusable check utilities from hubValidations into hubUtils. Feedback welcome on whether the opt-in is worth that, or whether value checks on non-hub data are the user's responsibility.
3. **scoringutils touch points.** Which scoringutils functions beyond `summarise_scores` need upstream tweaks for the mask pattern to work cleanly. To be established during implementation with the scoringutils maintainers.

## 👍 Ready to make it

### Proposed solution

The subclasses, constructors and print methods live in hubUtils. The scoringutils bridge lives in hubEvals, as thin wrappers over the scoringutils constructors and generics following the mask pattern. Downstream packages supply methods on the subclasses.

#### Constructors

One coercion function per output type, named `as_model_out_<output type>()`: `as_model_out_quantile()`, `as_model_out_cdf()`, `as_model_out_pmf()`, `as_model_out_sample()`, `as_model_out_mean()`, `as_model_out_median()`. Given a `model_out_tbl`:

- if only that output type is present, the constructor returns the typed object;
- if other output types are present as well, it drops them with a warning;
- if that output type is absent, it errors.

There is no companion that splits a mixed `model_out_tbl` into a list of typed objects. If a helper that maps a single-type operation over every output type present proves useful later, it can be added then.

#### Validation of a typed object

By default a constructor checks only what its methods depend on: that the rows are of one output type, that `output_type_id` has the form the output type needs (numeric for quantile and cdf, categories for pmf), and that the data is consistent with any `output_type_id_order` or `compound_taskid_set` supplied. The consistency check is the one piece nothing upstream has done, since hubValidations only ever checks against the config's values. This keeps the constructors as light as the existing `validate_model_out_tbl()`, which checks columns and column types only.

Checks on the values themselves (quantile levels in [0, 1] and values non-decreasing across levels within a modeling task, pmf probabilities summing to one) are not run by default. Data collected from a hub has already passed them on submission, and scoringutils repeats some of them in its own constructors. A `strict = TRUE` argument would run them for data that did not come through a hub. The checks exist in hubValidations today; the reusable parts would move to hubUtils and hubValidations would call them from there. Whether to include the strict layer is an open question. Hub-specific checks, such as which task ID values a hub accepts, stay with hubValidations in either case.

#### Metadata

A typed object carries only what its methods need. Quantile, cdf, mean and median need nothing beyond the output type. A pmf needs the `output_type_id_order` to be ordinal, and a sample needs the `compound_taskid_set` to be joint. Both are supplied to the constructor as arguments, and the constructor validates the data against them rather than inferring them from the data. No summary metadata (task ID columns, model counts and the like) is cached; the print method computes what it shows.

#### Populating config-derived values

The `output_type_id_order` and the `compound_taskid_set` are properties of a modeling task, not of a hub or a round. Two modeling tasks in one round can differ, and rounds can differ again, routinely so in scenario hubs. hubUtils will expose both values per modeling task, keyed by round and by the columns that tell a round's modeling tasks apart, with a simplified form that returns the single vector when every modeling task agrees. The design and its child issues are tracked in hubUtils [#310](https://github.com/hubverse-org/hubUtils/issues/310). Passing them as constructor arguments follows the precedent of `score_model_out()`'s existing `output_type_id_order` and `compound_taskid_set` arguments, and a caller working with model output not connected to a hub supplies them the same way.

A `model_out_tbl` spanning modeling tasks or rounds with different values is routed to its modeling tasks through those keys and split by value, and each group becomes its own scoringutils object. scoringutils imposes that boundary in any case, since a `forecast_ordinal` carries one level set and a multivariate sample forecast one `joint_across`. Scores are per forecast unit, so the groups bind back together. This covers scenario hubs, where rounds differ, and hubs with several targets of one output type in a round that declare different values.

**Stamping at read time in hubData.** `collect_hub()` already coerces collected data to a `model_out_tbl`, and it has the hub config to hand: a `hub_connection` carries `config_tasks`, and a filtered query keeps the connection it was built from. When the coercion succeeds, `collect_hub()` can take the round IDs present in the data, extract the config-derived values for those rounds only, and attach them to the object as attributes: the per-modeling-task values, the assignment of rows to modeling tasks, and the round ID variable where the rounds the data comes from share one. The subclass constructors read these attributes as defaults, so the caller passes nothing:

```r
hub_con |>
  dplyr::filter(output_type == "pmf", origin_date %in% c("2026-01-05", "2026-01-12")) |>
  collect_hub() |>
  as_model_out_pmf()   # ordinal: output_type_id_order read from the stamped attribute
```

A stamped attribute is a default the caller can override, not the record of truth. Where it is absent, because the data did not come through `collect_hub()`, coercion to `model_out_tbl` failed, or a dplyr operation dropped it, the constructor falls back to its arguments and asks for the value where it is needed. Stamping is a further consumer of the per-modeling-task extraction, not a dependency of it. Note that routing rows to a round relies on a round ID column in the data, so a hub whose rounds do not share a round ID variable, or that sets `round_id_from_variable` to false, cannot be stamped from the data alone.

#### Statistical type as a second subclass

The statistical type is its own class, placed before the output type in the class vector:

```r
c("nominal", "pmf", "model_out_tbl", ...)   # as_model_out_pmf(x)
c("ordinal", "pmf", "model_out_tbl", ...)   # as_model_out_pmf(x, output_type_id_order = c("low", "med", "high"))
c("marginal", "sample", "model_out_tbl", ...)  # as_model_out_sample(x)
c("joint", "sample", "model_out_tbl", ...)     # as_model_out_sample(x, compound_taskid_set = c("location", "target_end_date"))
```

Without an `output_type_id_order` a pmf is nominal; without a `compound_taskid_set` a sample is marginal. No further argument is needed on the scoringutils side: `as_scoringutils_forecast()` dispatches on that class and calls the matching scoringutils constructor.

```r
as_scoringutils_forecast(as_model_out_pmf(x))                             # forecast_nominal
as_scoringutils_forecast(as_model_out_pmf(x, output_type_id_order = ord)) # forecast_ordinal
as_scoringutils_forecast(as_model_out_sample(x))                          # forecast_sample
as_scoringutils_forecast(as_model_out_sample(x, compound_taskid_set = s)) # forecast_sample_multivariate
```

Mean and median run the other way: two hubverse types share one scoringutils class. They sit as sibling subclasses under a shared `point` class, `c("mean", "point", "model_out_tbl", ...)`, so the connector is written once at `point` while a metric can specialise at `mean` or `median`. On the return path nothing is lost, because the `output_type` column travels with the data as part of the scoringutils forecast unit.

S3 dispatches on the first matching class, so a method written at `pmf` applies to both nominal and ordinal objects, and a method at `ordinal` overrides it only where the statistical type matters. The class is added in front of the output type rather than replacing it, so an object constructed without config is a valid `pmf` and becomes an `ordinal` `pmf` once the order is known.

Note that the statistical type is a bare class name (`ordinal`), so dispatch relies on the convention that it only ever appears together with its output type.

#### Alternative considered

**Attributes instead of subclasses (rejected).** Keep one class per output type and carry the statistical type as an object attribute, with methods branching on the attribute internally. Fewer classes, but every method that cares about the statistical type has to branch by hand, and attributes are silently dropped by many dplyr verbs, so the object needs re-validation after any manipulation. That is the failure mode scoringutils' own forecast objects have. Subclasses give pure dispatch and survive manipulation as part of the class vector.

### Visualize the solution

Indicative layering (names illustrative):

```
hubUtils  (general, dependency-light: no scoringutils dependency)
  ├── class defs: model_out_tbl  (base, exists)
  │     └── quantile / cdf / pmf / sample / mean / median  (new subclasses)
  │           + statistical type classes: nominal|ordinal (pmf), marginal|joint (sample)
  │           + shared class: point (parent of mean, median)
  ├── constructors: as_model_out_quantile(), as_model_out_pmf(), ...
  │     (subset to one output type, validate, set class)
  └── print methods

hubEvals  (depends on scoringutils)
  ├── connectors:
  │     as_scoringutils_forecast()   generic on a typed subclass -> scoringutils::as_forecast_*()
  │     as_forecast_*.model_out_tbl  direct jump from a model_out_tbl into a chosen scoringutils fn
  │     as_hubverse()                scored output back to hubverse shape
  └── score_model_out()  public wrapper, behaviour unchanged

later
  ├── hubData: on read (with config), set the statistical type class and config-derived values
  └── hubEnsembles / hubVis2: methods on the same subclasses
```

Constructor and connector sketch:

```r
as_model_out_pmf <- function(model_out_tbl, output_type_id_order = NULL, ...) {
  x <- keep_output_type(model_out_tbl, "pmf")  # warn and drop others; error if none
  # validate: categories within output_type_id_order if supplied;
  #           value checks (probabilities sum to one) only with strict = TRUE
  stat_type <- if (is.null(output_type_id_order)) "nominal" else "ordinal"
  class(x) <- c(stat_type, "pmf", class(model_out_tbl))
  x
}

# entry point 1: hubverse generic, dispatches on the output type or statistical type class
as_scoringutils_forecast <- function(x, ...) UseMethod("as_scoringutils_forecast")

as_scoringutils_forecast.nominal <- function(x, ...) {
  # hand scoringutils a declassed tibble so it dispatches to its own method,
  # not back into as_forecast_nominal.model_out_tbl below
  scoringutils::as_forecast_nominal(tibble::as_tibble(x), ...)
}
as_scoringutils_forecast.ordinal <- function(x, ...) {
  scoringutils::as_forecast_ordinal(tibble::as_tibble(x), ...)
}
as_scoringutils_forecast.point <- function(x, ...) {
  scoringutils::as_forecast_point(tibble::as_tibble(x), ...)
}
# ... one method per output type or statistical type class with a scoringutils equivalent

# entry point 2: model_out_tbl methods on scoringutils' own as_forecast_*() fns.
# The user picks the type by the function called; the method builds the
# subclass, then routes through entry point 1.
as_forecast_quantile.model_out_tbl <- function(data, ...) {
  as_scoringutils_forecast(as_model_out_quantile(data), ...)
}
# ... one model_out_tbl method per scoringutils as_forecast_*() fn
```

### Scale and scope

- **Team:** Anna Krystalli (owner), Nicholas Reich, Li Shandross, Lucie Contamin; coordination with scoringutils maintainers (Sam Abbott, Nikos Bosse), who have signalled support for the mask pattern.
- **Repos touched:** hubUtils, hubEvals; later hubData, hubEnsembles and hubVis2 (see the layering above).
- **Phasing:** (1) subclasses, constructors, validation and print in hubUtils; (2) the connectors in hubEvals with round-trip tests (the issue #113 / Sprint E deliverable); (3) optionally re-express the hubEvals scoring path on the class layer behind the unchanged public API; (4, later) read-time stamping in hubData and methods in hubEnsembles and hubVis2.
- **Dependency:** the per-modeling-task extraction of config-derived values (hubUtils [#310](https://github.com/hubverse-org/hubUtils/issues/310)) gates read-time population in phase 4. Phases 1 and 2 take the values as arguments and do not depend on it.
- This is a change to foundational classes rather than a contained feature, so sequencing against other ecosystem work matters. Phases 1 and 2 are independently useful and releasable.
