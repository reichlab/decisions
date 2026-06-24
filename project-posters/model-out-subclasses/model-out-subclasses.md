# Project Poster: Output-type subclasses of `model_out_tbl`

- Date: 2026-06-19

- Owner: Anna Krystalli

- Team: Anna Krystalli, Nicholas Reich (input from Sam Abbott and Nikos Bosse, scoringutils)

- Status: draft

## ❓ Problem space

### What are we doing?

Introduce a system of output-type-specific subclasses of the hubverse `model_out_tbl`, for example `quantile`, `sample`, `pmf`, `mean`, `median`. Each subclass is constructed from a `model_out_tbl` by an output-type coercion function (`as_quantile()`, `as_sample()`, ...) that:

1. subsets to the rows of a single output type,
2. validates the data for that output type,
3. caches useful metadata (which fields, and how they are stored, is an open question for the team, see "What do we need to answer?"), and
4. supports an informative `print` method built from that metadata.

These typed objects give the hubverse a clean S3 dispatch surface, and the first concrete payoff is a formal, documented bridge to [scoringutils](https://epiforecasts.io/scoringutils/) (built in hubEvals, which already depends on scoringutils). Two complementary entry points:

- **From a typed object:** a single `as_scoringutils_forecast()` generic dispatches on the hubverse subclass and routes to the matching `scoringutils::as_forecast_*()` constructor (an `ordinal` object to `as_forecast_ordinal()`, a `quantile` object to `as_forecast_quantile()`), so a user reaches the right scoringutils class in one call without naming it. This needs an output-type subclass to dispatch on, so it does not apply to a bare `model_out_tbl` (which may hold several output types).
- **Straight from a `model_out_tbl`:** `model_out_tbl` methods added to each scoringutils `as_forecast_*()` function give a direct jump into a chosen scoringutils workflow, e.g. `as_forecast_quantile(my_model_out_tbl)`. The user names the target type by the function they call; that method constructs the corresponding hubverse subclass (subset, validate, attach metadata) and then converts.

An `as_hubverse()` method converts scored output back. That same dispatch surface then drives downstream functionality (ensembling, visualisation) by output type without `switch(output_type, ...)` branching.

This poster covers the class system and the connector layer built on it. Concrete downstream method implementations for ensembling and visualisation are a longer arc, noted here but scoped to follow-on work.

### Why are we doing this?

Two goals of comparable weight drive this, one internal to the hubverse and one outward to scoringutils:

- **A typed class system to drive hubverse functionality by dispatch.** A proper output-type class layer is foundational hubverse infrastructure in its own right. The same subclass can flow from data read through scoring, ensembling, and visualisation, so each package selects methods by dispatch on a shared, typed vocabulary instead of re-deriving output-type handling with `switch(output_type, ...)` blocks. This is valuable independent of scoringutils.
- **Seamless interoperability with scoringutils, and user-built evaluation workflows.** scoringutils is built around per-output-type classes and is designed (per [Sam Abbott](https://github.com/hubverse-org/hubEvals/issues/94#issuecomment-3891921846)) for users to write `as_forecast_*.<their_format>()` converters, score through the scoringutils workflow, then convert results back. Typed hubverse subclasses let a user take a `model_out_tbl`, coerce it straight into the right scoringutils object, and compose their own custom evaluations with the full scoringutils toolkit (including new features as they land), rather than being limited to what a single hubEvals wrapper exposes.
- **The hubverse needs finer output-type distinctions than the column data carries.** Whether a pmf is ordinal or nominal, or whether samples are marginal or joint, determines which metrics and methods are valid. A typed object is the natural place to record and dispatch on these distinctions. (Today the hubverse has only a very limited model-output class system: the single `model_out_tbl` class, no output-type subclasses, no formal dispatch methods.)
- **A more elegant internal pipeline (a welcome byproduct, not the driver).** `score_model_out()` stays exactly as it is, a deliberate, valued part of hubEvals that gives consumers such as dashboards one function to call. A class layer would simply make its internals more modular if and when refactored, dispatching into scoringutils' S3 generics rather than branching over `output_type`, with no change to the public API.

### What are we _not_ trying to do?

- Not changing the public interface or behaviour of `score_model_out()`; it continues to wrap the full workflow.
- Not making the scoring layer depend on hub config (`tasks.json`). hubEvals operates on model outputs extracted from a hub, not on the hub itself ([context](https://github.com/hubverse-org/hubEvals/issues/94#issuecomment-3892376999)). Config-derived metadata is populated upstream or supplied by the caller, not read inside the scoring path.
- Not reimplementing scoringutils classes, validation, or metrics. Subclass constructors stay lightweight around the scoringutils constructors, which already validate ([Sam Abbott](https://github.com/hubverse-org/hubEvals/issues/94#issuecomment-3892781599)).
- Not adding a scoringutils dependency to hubUtils. The class system stays dependency-light in hubUtils; everything that touches scoringutils (the connectors, `as_hubverse()`) lives in hubEvals.
- Not delivering the full downstream method set (ensembling, visualisation dispatch) in this project. We establish the class layer and the connector payoff; downstream methods follow.

### How do we judge success?

- A `model_out_tbl` can be coerced to an output-type subclass that holds only that type's rows, is validated, and prints a useful summary.
- Hubverse functionality can select behaviour by dispatch on the subclass: a new output type (or finer distinction) is added as a class plus methods rather than a new `switch` branch, and the design records where each finer distinction (ordinal/nominal, marginal/joint) is sourced.
- A user can take a hubverse `model_out_tbl`, coerce it into the appropriate scoringutils forecast object, apply scoringutils functionality directly, and bring results back, composing their own evaluation workflow without going through the single hubEvals wrapper.
- A single `as_scoringutils_forecast()` generic dispatches on the hubverse subclass and routes to the correct `scoringutils::as_forecast_*()` constructor, so one call covers any output type.
- A round trip (hubverse model output to scoringutils forecast object and scored output back to a hubverse-shaped result via `as_hubverse()`) preserves data fidelity, with test coverage.
- (Optional, not a requirement for success) The existing `score_model_out()` pathway could be re-expressed on top of the class layer, with no change to its public behaviour, if and when an internal refactor is worthwhile.

### What are possible solutions?

The proposed approach is a class hierarchy layered on the existing base class, with coercion constructors and S3 methods that dispatch into scoringutils, and the finer-than-output-type granularity encoded as composable subclass tokens. See "Ready to make it" for the proposed structure and the alternative considered.

## ✅ Validation

### What do we already know?

**The base class.** `model_out_tbl` already exists in hubUtils as a subclass of `tbl_df`. Subclasses extend its class vector, for example:

```r
c("quantile", "model_out_tbl", "tbl_df", "tbl", "data.frame")
```

so existing `model_out_tbl` methods keep working and output-type specificity is added on top.

**scoringutils is already S3 and wants to be dispatched into.** `score()` and `as_forecast_*()` are S3 generics ([score.R](https://github.com/epiforecasts/scoringutils/blob/main/R/score.R)). Sam Abbott confirmed a "mask" over the scoringutils API is feasible: hubverse-named classes that dispatch into the scoringutils generics, mostly as pass-throughs, following the `as_forecast_sample.<format>` pattern used by [epinowcast](https://github.com/epinowcast/epinowcast/blob/main/R/model-validation.R). A few scoringutils touch points (such as [`summarise_scores`](https://github.com/epiforecasts/scoringutils/blob/main/R/summarise_scores.R)) may need minor tweaks for the pattern to work cleanly.

**The transform logic already exists.** hubEvals has internal transform functions per output type (`transform_quantile_model_out()`, `transform_pmf_model_out()`, `transform_point_model_out()`, `transform_sample_model_out()`) plus the marginal/compound sample work from [PR #103](https://github.com/hubverse-org/hubEvals/pull/103). This is the code the connectors and constructors consolidate, not new logic to invent.

**The hubverse-to-scoringutils crosswalk** (from the [issue #94 plan](https://github.com/hubverse-org/hubEvals/issues/94)):

| Hubverse | scoringutils |
|---|---|
| `model_id` | `model` |
| `value` | `predicted` |
| `oracle_value` | `observed` |
| `output_type_id` (sample) | `sample_id` |
| modeling task (task ID combination) | forecast unit |
| `compound_taskid_set` | inverse of `joint_across` |

**Output types do not map one-to-one to scoringutils classes.** The mismatch runs both ways. Where the hubverse wants a finer distinction than the data carries (pmf, sample), it is config-driven and the extracted data cannot supply it on its own. Where the hubverse is already finer than scoringutils (mean and median both map to `forecast_point`), the distinction is in the data:

| Hubverse output type | Finer distinction | scoringutils class | Source of the distinction |
|---|---|---|---|
| quantile | (none) | `forecast_quantile` | data |
| mean | (vs median) | `forecast_point` | `output_type` column (data) |
| median | (vs mean) | `forecast_point` | `output_type` column (data) |
| pmf | ordinal vs nominal | `forecast_ordinal` / `forecast_nominal` | config: the ordered category set from `output_type_id.required` (the `output_type_id_order`) |
| sample | marginal vs joint | `forecast_sample` / `forecast_sample_multivariate` | `compound_taskid_set` (config; partly inferable from data) |
| cdf | (no direct equivalent) | n/a | convert to quantile |

The output-type-level class is always assignable from the data; the config-derived finer distinction (ordinal/nominal, marginal/joint) generally is not. Mean and median are the mirror case: two data-derivable output types share one scoringutils class (`forecast_point`), so they can sit as sibling subclasses under a shared `point` superclass in the class vector (e.g. `c("mean", "point", "model_out_tbl", ...)`), letting the common connector dispatch at `point` while the metric that makes sense (absolute error for median, squared error for mean) specialises at the leaf. This is the same layered-class-vector mechanism the design uses for the finer distinctions (see Granularity encoding below), with the shared layer below the leaf rather than a refinement token above it.

### What do we need to answer?

**1. Constructor semantics when multiple output types are present.** Should `as_quantile()` silently subset, warn that other types were dropped, or error? Is there a companion that splits a mixed `model_out_tbl` into a list of typed objects, one per output type present?

**2. What metadata should a typed object carry, and how durably? (open for team input)** The constructors can cache summaries in attributes for printing and for driving downstream methods, but *which* fields are genuinely useful is not yet clear and is exactly the kind of thing the team should weigh in on. Illustrative candidates only: task ID columns, output type ID levels (quantile levels, sample IDs, pmf categories), model IDs, number of models, modeling-task counts, and the config-derived refinement facts (`output_type_id_order` for ordinal pmf, `compound_taskid_set` for samples). Separately, the caching *mechanism* is a choice: attributes (fast, but dropped by many dplyr verbs) versus recompute on demand (always correct, slower) versus both with a reconstruction/validation helper. Worth settling early since it affects every method.

**3. Naming and scope of the connector surface.** The connector has two parts: a generic that dispatches on the hubverse subclass and routes to the right `scoringutils::as_forecast_*()` constructor, plus `model_out_tbl` methods on each scoringutils `as_forecast_*()` for the direct-from-raw-data path. Name the generic `as_scoringutils_forecast()` (explicit and unambiguous) or the terser `as_forecast()` (cleaner, but easily confused with scoringutils' own `as_forecast_*` family)? Also confirm `as_hubverse()` for the return path, and which `summarise_scores`-style scoringutils touch points need upstream tweaks for the mask pattern.

## 👍 Ready to make it

### Proposed solution

Define the output-type subclasses of `model_out_tbl`, their coercion constructors (`as_*()`, which subset to one output type, validate, and cache metadata), validators, and print methods in hubUtils, which is general and dependency-light. Build the scoringutils bridge (a single `as_scoringutils_forecast()` generic that routes to the matching `scoringutils::as_forecast_*()`, plus `as_hubverse()` for the return path) in hubEvals, which already depends on scoringutils, as thin wrappers over the scoringutils constructors and generics following the mask pattern. Downstream packages provide method implementations on these classes.

Splitting the layers this way keeps scoringutils out of hubUtils and lets a typed object flow through scoring, ensembling, and visualisation. It also keeps the classes low enough for the later read-time enhancement described under Populating config-derived metadata.

### Granularity encoding: composable subclass tokens

Where the hubverse needs a finer distinction than the output type (ordinal vs nominal pmf, marginal vs joint sample), the refinement is its own class token layered into the class vector, rather than a combined class name or an object attribute:

```r
c("ordinal", "pmf", "model_out_tbl", "tbl_df", "tbl", "data.frame")   # an ordinal pmf
c("joint", "sample", "model_out_tbl", "tbl_df", "tbl", "data.frame")  # a joint sample
```

The tokens are `ordinal`/`nominal` (for `pmf`) and `marginal`/`joint` (for `sample`). Order is most-specific first, so the refinement (`ordinal`) precedes the output type (`pmf`): S3 dispatches first-match-first, which lets a method specialise at `ordinal`/`nominal` where the scoringutils target actually differs while a generic method at `pmf` still applies to both flavours by fall-through. The mean/median/`point` relationship uses the same mechanism with the shared layer below the leaf (see the crosswalk note above). This gives:

- pure S3 dispatch: the right method is selected by class, with no internal branching;
- composable methods: written once at the output-type level and specialised at the token level only where it matters;
- additive refinement: an object read without config is just `c("pmf", "model_out_tbl", ...)`, and the token is prepended once the `output_type_id_order` (pmf) / `compound_taskid_set` (sample) is known, with no renaming, so the base output-type class stays assignable from the data alone;
- no class explosion: tokens compose onto the base types rather than multiplying into combined names.

The caveats: a token is a bare, generic class name (`ordinal`), so dispatch relies on the convention that it only ever co-occurs with its base type; and the config-derived token can only be added where config is available (most naturally at read time in `hubData`, see Populating config-derived metadata below).

#### Alternatives considered

**Attributes instead of subclass tokens (rejected).** Keep one class per output type and carry the finer distinction as an object attribute (e.g. an `output_type_id_order` or `compound_taskid_set` attribute), with methods branching internally on the attribute. It has the appeal of fewer classes and a single place to record metadata, but we rejected it because dispatch is not pure (every method that cares about the distinction has to branch on an attribute by hand) and attributes are fragile under dplyr operations (silently dropped, so the object needs re-validation or reconstruction after manipulation, the same failure mode scoringutils' own forecast objects have). The token model keeps the one property this option was built around, the base output-type class being assignable from data, while giving pure dispatch.

### Populating config-derived metadata

The config-derived facts a typed object needs (the `output_type_id_order` for ordinal pmf, the `compound_taskid_set` for samples) are pulled by config-extraction utilities run against the hub and passed to the constructors as arguments, following the precedent of `score_model_out()`'s existing `output_type_id_order` and `compound_taskid_set` arguments. These utilities are already planned and useful well beyond this work: `hubValidations` extracts the compound task ID set internally (`get_round_compound_task_ids()`), and hubUtils [#283](https://github.com/hubverse-org/hubUtils/issues/283) (`get_output_type_id_order()`) and [#284](https://github.com/hubverse-org/hubUtils/issues/284) (`get_compound_taskid_set()`) propose exposing both from a hub config.

Later enhancements: stamp the granular subclass and metadata at data-read time in `hubData` so the typed object arrives ready to use, and detect from data where possible (sample compound structure is partly inferable; the ordinal category order is not).

One consideration to flag upfront: a hub can have rounds whose metadata differs (e.g. changed pmf categories), and so whose submitted outputs differ. Likely rare but possible, and both the extraction utilities and the class constructors will need to handle it (e.g. a `model_out_tbl` spanning mixed rounds) rather than assume a single config per hub.

### Visualize the solution

Indicative layering (names illustrative):

```
hubUtils  (general, dependency-light: no scoringutils dependency)
  ├── class defs: model_out_tbl  (base, exists)
  │     └── quantile / sample / pmf / mean / median / cdf  (new subclasses)
  │           + refinement tokens prepended where config is known:
  │             ordinal|nominal (pmf), marginal|joint (sample)
  │           + shared superclass where useful: point (parent of mean, median)
  ├── constructors: as_quantile(), as_sample(), ...
  │     (subset to one output type, validate, cache metadata, print method)
  └── validators

hubEvals  (depends on scoringutils)
  ├── connectors:
  │     as_scoringutils_forecast()   generic on a typed subclass -> scoringutils::as_forecast_*()
  │     as_forecast_*.model_out_tbl  direct jump from a raw model_out_tbl into a chosen scoringutils fn
  │     as_hubverse()                scored output back to hubverse shape
  └── score.*()  dispatch on subclass -> as_scoringutils_forecast() -> scoringutils::score() -> as_hubverse()
      score_model_out()  public convenience wrapper, behaviour unchanged

later / optional enhancements
  ├── hubData: on read (with config), stamp the granular subclass + metadata
  │     (output_type_id_order, compound_taskid_set) so typed objects arrive ready
  └── hubEnsembles / hubVis: ensemble.*(), autoplot.*() dispatch on the same subclasses
```

Constructor sketch:

```r
as_quantile <- function(model_out_tbl, ...) {
  x <- dplyr::filter(model_out_tbl, output_type == "quantile")  # subset to one type
  # validate (delegate distributional checks to the scoringutils constructor;
  #           add hubverse-specific structural checks)
  # cache metadata as attributes (candidate fields, to be decided with the team):
  #   task_id_cols, output_type_id levels, model_ids, n modeling tasks
  # set class: c("quantile", class(model_out_tbl))
  x
}

# entry point 1: hubverse generic, dispatches on the output-type subclass and
# routes to the matching scoringutils constructor. Needs a typed object.
as_scoringutils_forecast <- function(x, ...) UseMethod("as_scoringutils_forecast")

as_scoringutils_forecast.quantile <- function(x, ...) {
  # hand scoringutils a declassed tibble so it dispatches to its own method,
  # not back into as_forecast_quantile.model_out_tbl below
  scoringutils::as_forecast_quantile(tibble::as_tibble(x), ...)
}
as_scoringutils_forecast.ordinal <- function(x, ...) {
  scoringutils::as_forecast_ordinal(tibble::as_tibble(x), ...)
}
# ... one method per subclass / refinement token

# entry point 2: model_out_tbl methods on scoringutils' own as_forecast_*() fns,
# for a direct jump from raw hubverse data into a chosen scoringutils workflow.
# The user picks the type via the function called; the method builds the
# corresponding subclass, then routes through entry point 1.
as_forecast_quantile.model_out_tbl <- function(data, ...) {
  as_scoringutils_forecast(as_quantile(data), ...)
}
# ... one model_out_tbl method per scoringutils as_forecast_*() fn
```

### Scale and scope

- **Team:** Anna Krystalli (owner), Nicholas Reich; coordination with scoringutils maintainers (Sam Abbott, Nikos Bosse), who have signalled support for the masking/dispatch pattern.
- **Repos touched:** hubUtils (new subclasses, constructors, validators, print methods; stays dependency-light, no scoringutils); hubEvals (the `as_scoringutils_forecast()` / `as_hubverse()` connectors, and optionally re-expressing scoring on the class layer with no public API change); later hubData (read-time stamping), hubEnsembles and hubVis (downstream dispatch).
- **Phasing:** (1) class definitions + constructors + validation + print in hubUtils; (2) the `as_scoringutils_forecast()` / `as_hubverse()` connectors in hubEvals with round-trip tests (the issue #113 / Sprint E deliverable); (3) optionally re-express the hubEvals scoring path on the class layer behind the unchanged public API; (4, later) read-time stamping in hubData and downstream method dispatch.
- **Dependency:** the config-extraction utilities (hubUtils [#283](https://github.com/hubverse-org/hubUtils/issues/283), [#284](https://github.com/hubverse-org/hubUtils/issues/284)) gate read-time population of the config-derived tokens (phase 4); phases 1 and 2 do not depend on them.
- This is a meaningful refactor of foundational classes rather than a contained feature, so sequencing against other ecosystem work matters; phases 1 and 2 are independently useful and releasable.
