# Project Poster: Submission Advisories (non-blocking warnings in hubValidations)

- Date: 2026-06-18

- Owner: Anna Krystalli

- Team: TBC

- Status: draft

## ❓ Problem space

### What are we doing?

Scoping how to add a new kind of validation result to [hubValidations](https://github.com/hubverse-org/hubValidations): a **submission advisory**. This is a check that flags something about a submission that looks implausible or off, and surfaces that flag **prominently** to the submitter, **without failing validation or blocking the submission**.

The motivating request comes from the CDC FluSight team (full brief: [`cdc-brief.md`](cdc-brief.md)). They want sanity checks such as "this quantile is higher than 30% of the state's population" or "this prediction interval is implausibly wide" to warn submitters, while still letting the forecast through.

Advisories will not be limited to model output content: they can equally flag model metadata or file characteristics. The CDC's current ask just happens to be about model output content.

**This poster is about how this plugs into hubValidations, not the statistics.** The point is to investigate the best way to express "advisory severity" within hubValidations, not to pin down the exact thresholds for each check (those will be tuned later, likely per-hub).

### Why are we doing this?

- **Direct CDC FluSight request** for next season (see [`cdc-brief.md`](cdc-brief.md)).
- hubValidations today is effectively **binary**: a check either passes (`check_success`) or fails (and **fails CI**). A separate `check_info` class can emit a non-blocking informational note, but neither non-failing class lets you say *"this is technically valid but looks wrong, please double-check."* The CDC criteria are exactly that middle ground.
- Wiring these checks in with the tools we have would force the wrong outcome: a check can only pass or fail, so a 30%-of-population warning would **turn CI red** and block an otherwise-valid submission.

### What are we _not_ trying to do?

- **Not** designing the exact statistical thresholds for each CDC check (e.g. *is it 30% of population or 25%?*). Thresholds are check-implementation detail and will be configurable.
- **Not** building the cross-week "bottom-10 performing models" tracking system in this project. That needs external score data and state that persists across submissions. It sits outside the per-submission validator and is flagged as a separate problem.
- **Not** changing how existing pass/fail checks behave. Advisories only add to what's there.
- **Not** building PR-comment rendering. Posting validation results into the PR is a separate, pre-existing desire that applies to *all* check results, not just advisories. Out of scope here.

### How do we judge success?

- A hub admin can turn an advisory check on or off in `validations.yml`, with no new configuration mechanism to learn.
- When a submission trips an advisory, the submitter sees it **prominently** in the CI log, and **CI stays green**.
- The advisory result is rich enough to be pretty-printed clearly, so a submitter can see exactly what tripped the check and by how much.
- An advisory can optionally be **escalated to a hard failure** per-hub, for hubs that decide a given criterion should block (the CDC explicitly asks about this).

### What are possible solutions?

See **Architecture options** under "Ready to make it". The main choice is whether an advisory is a new **condition class** stored alongside the existing checks in the validation list, or a **file-level attribute** (an annotation on the result). For either, the most efficient way to dispatch them is to reuse the same config/dispatch machinery the optional checks already use.

---

## ✅ Validation

### What do we already know?

**How check results work today (the gap, from the code).** Every check result is a condition object that inherits `hub_check` (`R/capture_check_cnd.R`). The available severities are:

| Result class | Constructor | Fails CI? | Surfaced how |
|---|---|---|---|
| `check_success` | `capture_check_cnd(check = TRUE, …)` | No | tick, inline |
| `check_info` | `capture_check_info()` | No | `i`, inline |
| `check_failure` | `capture_check_cnd(check = FALSE, error = FALSE)` | **Yes** | `x`, inline |
| `check_error` | `capture_check_cnd(check = FALSE, error = TRUE)` | **Yes** | early-return |
| `check_exec_error` / `check_exec_warn` | exec capture | **Yes** | inline |
| `validation_warning` (validation-level) | `capture_validation_warning()` + `attr(x, "warnings")` | No | **boxed, prominent**, but aggregated at object/PR top |
| `validation_warning` (check-level) | `capture_check_cnd(…, warnings = …)` | No | inline, muted, **opt-in** (`show_check_warnings = TRUE`) |

What fails CI is decided in `R/check_for_errors.R` → `extract_failing_checks()` → `not_pass()` (`R/cnd_utils.R`). Note that `not_pass()` is an **allowlist** (`!inherits(x, "check_success") & !inherits(x, "check_info")`): *only* `check_success` and `check_info` are treated as passing, and **everything else fails**. This is a key design consequence: a naively-added new class would **fail CI by default**; making it non-blocking requires explicitly adding it to the `not_pass()` allowlist.

**Why nothing here fits an advisory:**

- A check that flags a problem emits `check_failure` or `check_error` (via `capture_check_cnd()`), both of which **fail CI**. That holds across the whole check family, the standard `check_*()` checks and the optional `opt_check_*()` ones alike. None of them can return a non-blocking advisory result. Wrong severity.
- The **`validation_warning`** class (`capture_validation_warning()` in `R/capture_check_cnd.R`) is non-blocking (`check_for_errors()` never aborts on it) and can be attached at **two distinct levels**, which between them split the properties an advisory needs but never combine them:
  - **Validation-level**: stored as `attr(x, "warnings")` on a `hub_validations` / `hub_validations_collection` object and printed in a **prominent yellow box** by `print_validation_warnings()` (`R/print.R`). But this box is **aggregated and emitted once at the top of the object**: in the PR flow the warnings are merged up across files in `combine.R` and attached to the whole collection (`validate_pr.R`, the config-modified warning), so they surface at **PR level, not against the offending file or rows**. Current use: "a hub config file was modified" (`check_pr_config_modified()`).
  - **Check-level**: stored in a `warnings` field on an individual check result (passed via `capture_check_cnd(..., warnings = ...)`). It *is* file- and check-local and carries structured fields, and is already used to flag a coarser-than-configured `compound_taskid_set` (`check_tbl_spl_compound_taskid_set.R`). But it is **extra information attached to an existing check, not a check in its own right**: it can only be emitted from inside a host check's logic and **cannot be declared and dispatched as its own check in `validations.yml`.** An advisory needs to *be* a configurable check ("flag any forecast exceeding 30% of state population"), not a side-note riding on whatever check happens to be running.

  In both cases `validation_warning` is **not** a `hub_check`, is **not** dispatched through `validations.yml`, and carries no pass/fail structure. Warnings were designed to **annotate the validation process, or an existing check, not to express a new, independently-configured check**, which is exactly what an advisory is.

So what's missing is an **advisory severity**: a result that behaves like a check (configurable, returns structured detail) but is prominent and non-blocking, sitting between `check_info` and `check_failure`.

**How checks are configured and dispatched (the reuse path).** Optional/custom checks are declared per-round in `hub-config/validations.yml` and run by `execute_custom_checks()` → `exec_cfg_check()` (`R/execute_custom_checks.R`). New check functions are scaffolded with `create_custom_check()` (`R/create_custom_check.R`). This same path drives model-output, model-metadata, and file checks ([custom functions](https://hubverse-org.github.io/hubValidations/articles/deploying-custom-functions.html)), which is why advisories are available for all of those, not just forecast content. An advisory check should plug into this **unchanged**. The new part is only the *result class* it returns and how that class is treated downstream.

### What do we need to answer?

These are the open decisions the work hinges on. Because they need feedback from the broader team and determine how much of hubValidations we'd have to refactor, the formal choices are deferred to a follow-up **RFC**. This poster scopes them so we can first decide **whether the effort is worth it**. The four questions below are the core design decisions, and each is largely independent of the others.

- **Q1. Is an advisory a check result, or an annotation?** This is the framing the team should settle first, because it decides storage. If an advisory is **check-like** (it inspects the data and reports a finding, just at a softer, non-blocking severity), it belongs as a `hub_check` **list element** alongside the other checks (Option A). If it is an **annotation** on the file's validation (a flag attached to the result, not itself a check), it belongs as a **file-level attribute** on the `hub_validations` object (Option B). The two differ in where the implementation effort and awkwardness fall; the **Architecture options** below compare them in detail.
- **Q2. Rendering: how is an advisory surfaced?** Inline alongside the file's other checks (the [sketch](#visualize-the-solution)), in a separate prominent box/subsection, or both (an inline marker plus a per-file box). This *largely* follows from Q1 (a list element renders inline almost for free, an attribute pushes toward a separate box), but it is a real choice the team should make explicitly rather than inherit by accident.
- **Q3. Presence: should an advisory appear when it is *not* triggered?** Unlike a check, which always emits a binary pass/fail result, an advisory only carries information when it fires. **Proposed default: emit only when triggered.** Return `NULL` on pass, which the dispatch (`execute_custom_checks()`) and constructors already `compact()` away, so passing data produces no advisory output and avoids advisory fatigue. Trade-off: no per-advisory "ran and was fine" record; mitigable with a single summary count line if auditability is wanted. Orthogonal to Q1/Q2.
- **Q4. Escalation: can a hub promote an advisory to a hard failure?** The CDC explicitly asks "are there characteristics that should move beyond warning status and fail validation?", so per-hub escalation is in scope. **This interacts strongly with Q1; it is not a separate, independent choice:**
  - In the **list-element** model, escalation is essentially free: an escalated advisory just returns a failing class (`check_failure`) instead of a non-blocking advisory result. Severity is a property of the list element and the existing fail path already handles it.
  - In the **attribute** model, escalation is awkward by construction: an attribute can never fail CI, because `not_pass()` only inspects list elements. To escalate you must either special-case `check_for_errors()` to scan the attribute and abort, or re-emit the escalated advisory as a `check_failure` in the list, which means escalated advisories become list elements anyway, dragging the model back toward Option A for exactly the cases a hub cares most about.
  - So if per-hub escalation matters, it counts **in favour of the list-element model**.

---

## 👍 Ready to make it

### Proposed solution

Introduce two things in hubValidations:

- **A new family of advisory check functions** (likely `advise_*()`, parallel to the existing `check_*()` and `opt_check_*()` families). They inspect a submission (forecast data, model metadata, or file characteristics), are configured per-round in `validations.yml`, and run through the existing `execute_custom_checks()` dispatch path, the same way optional checks do today.
- **A non-blocking advisory result** that these functions return: surfaced prominently to the submitter and carrying structured detail (which rows, values, and locations tripped it) so it can be pretty-printed clearly, but never failing CI.

Where that result is stored, and its exact shape, is the Q1 question this poster scopes and the follow-up RFC decides.

### Architecture options (for the RFC to decide)

**Option A. Advisory as a new `hub_check` subclass `check_advisory` (the "check result" model).** A new condition class parallel to `check_info` / `check_failure`, constructed via the existing `capture_check_cnd()` machinery (or a thin `capture_check_advisory()` wrapper), stored as a list element like any other check.

- Add `check_advisory` to the `not_pass()` allowlist (`R/cnd_utils.R`) so `extract_failing_checks()` ignores it and it never fails CI. (This is the one essential change; without it the new class fails CI by default.)
- `print_check()` in `R/print.R` gets a new prominent bullet (e.g. a yellow `!`), and advisories can additionally be collected into the existing prominent warning box.
- Flows through `combine.R` and the `hub_validations` / collection classes like any check.
- Dispatched via `validations.yml` exactly like `opt_check_*`, with **no dispatch changes**.
- Holds its structured detail in extra named fields on the result object, the same way some existing checks already attach extra data (e.g. which rows tripped the check) to their result.
- *Trade-off:* the cleanest, clearest behaviour; the cost is that the changes land in core files (`check_for_errors`, `print`, `combine`).
- *Effort:* **Small to Medium.** Because a `check_advisory` is just another `hub_check`, it drops into the validation list like any other check, so the constructors, the assignment methods, the `c()` part of `combine`, and config dispatch are all untouched. The actual work is narrow: a one-line `not_pass()` allowlist entry (`R/cnd_utils.R`), a `capture_check_advisory()` constructor, and a bullet/class-list/theme branch in `R/print.R`.

**Option B. Advisory as a file-level attribute (the "annotation" model).** Store triggered advisories in a new `attr(hub_validations, "advisories")` on the per-file object, kept at file level (not aggregated up to the collection), with a dedicated prominent print feature for it (which can reuse `print_validation_warnings()`'s box style). Advisories never enter the check list, so they are structurally incapable of failing CI.

- *Trade-off:* the cleanest conceptual separation (an advisory is an annotation on the file, not a check), and needs no `not_pass()` change. The cost is that it fights the dispatch scaffolding: `execute_custom_checks()` assembles every configured check's return into the *list*, so advisory returns must be **diverted out of the list into the attribute**, which is the inelegant part.
- *Effort:* **Medium.** No fail-path risk, and the print feature is small and self-contained, but the diversion plus an attribute-merge in `combine` and a print call in `check_for_errors` are all real work.

Both converge on the same downstream requirement: **a result the failure-detection path ignores but the print path highlights.** Since the effort is broadly comparable, the choice should be made on conceptual grounds (is an advisory a check or an annotation?) rather than cost. The one substantive tie-breaker is escalation (Q4): it is essentially free for the in-list class and awkward for the attribute, so if per-hub escalation matters, that tilts toward Option A. The RFC makes the final call.

### Visualize the solution

Sketch of the intended submitter experience in the CI log. The two options surface the same advisory in different places.

**Option A: inline, in the check list.**

```
── 2024-11-23-team-model.parquet ───────────
v [valid_round_id]   Round id is valid.
v [valid_tbl_values] Values are valid.
! [advisory_counts_lt_popn]  3 forecasts exceed 30% of state population.
    └ Affected rows: 14, 22, 39 (locations: 06, 36, 48). CI not affected.
✔ All validation checks have been successful.   ← still green
```

**Option B: a separate advisory box, with the check list left clean.**

```
── 2024-11-23-team-model.parquet ───────────
v [valid_round_id]   Round id is valid.
v [valid_tbl_values] Values are valid.
✔ All validation checks have been successful.   ← still green

┌ Advisories ─────────────────────────────────────
│ ! 3 forecasts exceed 30% of state population.
│   Rows 14, 22, 39 (locations 06, 36, 48). 
└─────────────────────────────────────────────────
```

