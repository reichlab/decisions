# CDC brief: potential validation warnings (verbatim)

> Source: email from the CDC FluSight team following the FluSight evaluation
> meeting, June 2026. Reproduced here as the motivating requirements for the
> [submission-advisories](submission-advisories.md) poster. The CDC notes there
> is **no rush** — this would not be implemented until next season.

---

Hello,

After our FluSight evaluation meeting last Thursday, Rebecca and I put together a
short list of some criteria we may want to consider adding validation warnings for
with submissions. We wouldn't want to implement this until next season, so there is
no rush and we are still thinking about other possibilities.

**Potential Validation Warnings Considerations**

- **Maximum:**
  - Quantile values for % for NSSP forecast percentages (e.g., not higher than
    0.05% + highest observed value in any state in recent history)
  - Quantile values for weekly hospital admissions should not be higher than e.g.,
    30% of the state population size
  - (could also consider a warning specifically for median values)

- **Prediction Interval Width or Maximum:**
  - Prediction interval width in relation to median, to be discussed – potential
    option to warn for undesirably large prediction intervals

- **Calibration:**
  - Distance from last observed reported value and -1, 0, and maybe 1-week horizons
    should be less than x admissions or y% with different tolerance thresholds for
    each horizon. Chosen values should consider previously observed ranges of back
    fill, especially for –1 horizon.

- **General Score/Performance Warning:**
  - Poor performing models in general
  - ex: model was ranked in the bottom 10 performing models in the last 2 weeks
  - Might involve uploading csv with bottom 10 every week… open to ideas

**To consider:**

- What would warning messages look like?
- How much detail would there be?
- Are there any submission characteristics that should move beyond warning status and
  cause the submission to fail validation? Especially with a new emphasis on a %
  target, there may be some new options for quality control.
- Is there a way to track/record/flag warnings collectively over multiple weeks?
  E.g., is there a way to leverage this to proactively limit forecasts that are
  frequently unreasonable? Likely to be highly subjective and may not be worth
  pursuing initially.

Let us know what you think and if you would like to schedule a meeting to discuss
further!
