# Project Poster: hubVis2 - hubVis redesign

- Date: 2026-07-23

- Owner: Lucie Contamin

- Status: draft

## ❓ Problem space

### What are we doing?

To improve usability, and make it easier to develop and maintain, we 
propose to freeze `hubVis` as-is (after a final version) and build a new
package: `hubVis2` with multiple `geom_` ggplot2 style and plotly style 
functions. Each function will be designed for a specific model projection
output type (samples, quantiles, or median) and a specific plot output
(spaghetti plot, ribbon, point + line).

The current `hubVis` package will have a last small release with information
pointing to the new `hubVis2` package.

### Why are we doing this?

The hubVis package contains currently only one function with approx. 25
parameters that can generate different kinds of time series plots (
spaghetti plot of samples output type, ribbon of quantiles, etc.). The
function is difficult to use and maintain, and some parameters are quite complex. 
For example, the same call can render a different plot depending on which output 
types are present in the inputted data. 

The goal is to provide a new package with functions that are easier to use 
and maintain, based on the ggplot2 and plotly ecosystems and on the forthcoming
subclass system associated with `model_out_tbl`.

### What are we _not_ trying to do?

We do not plan to:

- keep `plot_step_ahead_model_output()` from hubVis as a backward-compatible
  wrapper. `hubVis` will stay frozen and will not be updated anymore.
- build a high-level, all-in-one function as we have for hubVis. hubVis2 will
  have multiple simple functions that can be used together in a pipe instead.
- unify the "static" and "interactive" version of the plots in the same 
  function (for example, a ribbon function that can output a plotly or
  ggplot object depending on a parameter). Instead, the "static" version
  will use ggplot2 functions (`geom_`) with a `+` pipe style, and the 
  "interactive" version will use plotly functions with a "|>" pipe style.
- supports additional output type. We will keep supporting the same
  output type as previously supported with hubVis: median, quantile, sample


### How do we judge success?

- hubVis2 generates the same kind of plots as hubVis, in term of aesthetics 
  and functionality. No core functionality should disappear. 
- hubVis2 output layers can be combined with ggplot2/plotly functions
- The package has clear documentation and examples to show how to use 
  and/or transition to the new design
- Existing `hubVis` users can continue to use `plot_step_ahead_model_output()`, 
  but will not have any bug fixes or updates on that function. The function will
  continue to be available ONLY in the `hubVis` package, no backward-compatibily
  wrappy is expected in `hubVis2`. Clear documentation and messaging should warn 
  the users.
- `hubverse` meta-package should transition to point at hubVis2 in a two steps 
  process. First, the hubverse package can install both `hubVis` and `hubVis2` 
  with a message stating that we are moving from a one function system in `hubVis` 
  to a multi-function system in `hubVis2` package and encourage users to switch 
  to the new version as it should be easier to use. Once `hubVis2` has all the 
  expected functionality and is stable for some time (no major bug issue and a stable
  version), we can remove `hubVis` from the meta-package. 
- The package has good test coverage on all the steps, including internal calculation 
  (like sample to quantile conversion (the conversion in itself is tested in another
  package, but the behavior of our stat function should still be tested as it also
  format the data, see more informatin below.)) 
  and plotting results (using `vdiffr` for static plots, structural tests for 
  interactive plots).


### What are possible solutions?

##### hubVis

`hubVis` will be frozen after one small last version release. The last update
will include an `.onAttach()` startup message pointing to `hubVis2` (once ready),
a README banner and documentation update, a NEWS entry, and a soft-deprecation 
notice. 

##### Internal calculation

- Delegate the conversion of output type to the existing 
  `hubUtils::convert_output_type()` (to keep the current hubVis 
  capacity to plot quantile ribbons from sample output type)
- Shared internal computation layer (`compute_hub_interval()`,
  `compute_hub_median()`) for both static and interactive
  functions.

##### Static plotting

- We will use custom ggplot2 `geom` and `stat` functions to give full 
  compatibility with the rest of the ggplot2 package:
  	- Four geoms: `geom_hub_interval()`, `geom_hub_median()`, 
  	  `geom_hub_sample()`, `geom_hub_target()`.
  	- Custom `Stat` functions: `stat_hub_interval()` to pivot long quantiles 
    rows into ymin/ymax pairs for each requested interval, and if necessary 
    convert samples rows into quantiles (using hubUtils). `stat_hub_median()`
    to extracts ("median" or `0.5` "quantile" output type) or calculates the 
    median from "sample" output type (using hubUtils).
  	- `Stat` and `geom` constructor functions, along with their underlying 
  	  ggproto classes, will be exported as public objects.
- delegate ribbon plotting to `ggdist::GeomLineribbon` as it already
  solves some issues: nested and ordered ribbon rendering, legend behavior,
  for example. `geom_ribbon` from ggplot2 plots single filled band, whereas
  `ggdist` allows multiple ribbons. We plan to first try using
  `ggdist::GeomLineribbon` to plot multiple intervals. If it does not work
  (unexpected issues) we can fall back to create a custom geoms based 
  on `geom_ribbon()`.

##### Interactive plotting

- We will create separate, hand-built `plotly_*()` functions, one per `geom`,
  mirroring the design of the `geom` logic. Each `plotly_*()` function 
  will follow plotly's own native `data = NULL`, `inherit = TRUE` convention, 
  so it inherits data from the initial plot call by default.
- Plotly faceting function does not follow the same logic as ggplot2. 
  It will need to be declared first, directly after the plot is initialized.

##### Other ideas

We also considered an alternative architecture, built on custom S3 classes 
and a `plot_step()` function declaring static vs. interactive output first, 
see hubVis [issue #71](https://github.com/hubverse-org/hubVis/issues/71). 
We compared it in depth (with Claude) across users, programmers, and 
maintainers and chose the ggplot2/plotly extension approach for its better 
long-term maintainability and native ecosystem interoperability.


## ✅ Validation

### What do we already know?

##### Problem 

The problem is documented directly in the hubVis GitHub repository issues.

##### Data layer

We can use different `hubUtils` functions in our process:
- `hubUtils::convert_output_type()`: for output type conversion
- `hubUtils::as_model_out_tbl()`/`validate_model_out_tbl()`: to force 
  (if necessary) and validate inputted model projection data.

##### Static rendering

- ggplot2 released a major version (4.0) that impacts how geom 
  constructors are built. So we will need to restrict hubVis2 dependencies 
  to ggplot2 >= 4.0. 
- `ggdist` can be used for ribbon plotting as it allows multiple ordered
  nested ribbons. The package already requires ggplot2 4.0, so no version 
  conflict should appear.

##### Interactive rendering

- plotly R has no faceting equivalent to ggplot2's, so a custom function will
  be built.
- `ggplotly()` cannot really render custom ggplot2 geoms without extra work so 
  we will not use it. 

### What do we need to answer?

##### Class

A data class convention is currently in planning in the hubverse that would
help to identify an input data frame as containing sample, quantile, or 
median output type model projection data.
The current plan follows the ggplot2 logic and allows different data inputs 
in different layers, or lets layers inherit the same data, so a plot 
could have a ribbon layer with quantile output type data as input and add 
a line layer with median data. Having the output type information in the 
class might simplify the internal process.
It has not been decided yet if an object can have mixed output type within
a single classed object. That decision might impact hubVis2.

##### Output type conversion

- Interval calculation will error, not warn and drop, when a group
  is missing the quantile levels required to compute the requested intervals,
  for quantile output type. 

##### Loss of some functionality

- The current hubVis `plot_step_ahead_model_output()` function has an automatic
  process such that when plotting 6 models or more, the ribbon plot will be reduced 
  to show only the biggest interval, for example `.95`. This functionality
  is not included currently in the hubVis2 as it might be difficult to implement
  with the faceting and passing the information through multiple layers/functions. 
  `hubVis2` interval plotting function can returns a warning to the user about
  overplotting and let the users update which interval(s) to plot.
- No package-default coloring. We don't plan to have the `colour/fill` set to 
  `model_id` by default. The users must explicitly write it. It's closer to
  the ggplot2/plotly behavior.


## 👍 Ready to make it

### Proposed solution

Redesign hubVis into a hubVis2 package with four ggplot2 extension functions with
matching plotly functions. 

### Visualize the solution

| Concept | Static (ggplot2) | Interactive (plotly) |
|---|---|---|
| Quantiles | `geom_hub_interval()`, drawing delegated to `ggdist::GeomLineribbon` | `plotly_hub_interval()` |
| Median | `geom_hub_median()`, line or point via a `geom=` override | `plotly_hub_median()` |
| Samples | `geom_hub_sample()`, auto-grouped and x-sorted | `plotly_hub_sample()`, one trace per model |
| Observed/target data | `geom_hub_target()` | `plotly_hub_target()` |
| Faceting | use ggplot2 `facet_` functions | `plotly_hub_facet()`, declared first in the pipe |

#### ggplot 2 functions

```r
geom_hub_interval(mapping = NULL,
                  data = NULL,
                  position = "identity",
                  ...,
                  widths = c(0.5, 0.8, 0.95),
                  source = c("auto", "quantile", "sample"), # control how intervals are derived
                  na.rm = FALSE,
                  show.legend = NA,
                  inherit.aes = TRUE)


```



#### Plotly functions

```r
plotly_hub_interval(p,
                     data = NULL,
                     ...,
                     widths = c(0.5, 0.8, 0.95),
                     source = c("auto", "quantile", "sample"), # 
                     inherit = TRUE)

plotly_hub_median(p,
                   data = NULL,
                   ...,
                   type = c("line", "point"),
                   source = c("auto", "quantile", "sample"),
                   inherit = TRUE)

plotly_hub_sample(p,
                   data = NULL,
                   ...,
                   inherit = TRUE)

plotly_hub_target(p,
                   data = NULL,
                   ...,
                   inherit = TRUE)

plotly_hub_facet(p,
                  facet_by,
                  data = NULL,
                  nrow = NULL,
                  ncol = NULL,
                  shareX = TRUE,
                  shareY = FALSE,
                  ...,
                  inherit = TRUE)
```


#### Example of calls

```r
# Static — note the separate data= argument, different contract than model_out_tbl
ggplot(model_out_tbl, aes(x = target_end_date, colour = model_id)) +
  geom_hub_interval() +
  geom_hub_target(data = target_data, mapping = aes(x = date, y = observation))

# Force sample-derived intervals even if quantile rows are present
ggplot(model_out_tbl, aes(x = target_end_date)) +
  geom_hub_interval(stat = stat_hub_interval(source = "sample"))

# Interactive — target_data overrides the inherited model_out_tbl
plot_ly(data = model_out_tbl) |>
  plotly_hub_interval() |>
  plotly_hub_target(data = target_data)

```

### Scale and scope

- **Team:** Lucie Contamin (owner), review by hubverse team
- **Phasing:**
	0. Freeze hubVis (version bump, soft-deprecation notice)
	1. Build the core internal computation layer
	2. Build the Geom/Stat components
	3. Build the plotly components
	4. Write documentation
	5. Update hubVis and hubverse meta-package to point to hubVis2
