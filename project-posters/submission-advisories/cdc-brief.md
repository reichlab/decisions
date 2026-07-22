# CDC brief: potential submission validation warnings

> Motivating requirements from the CDC FluSight team following a meeting on
> evaluating results from the FluSight challenge during the 2025-26 season for the
> [submission-advisories](submission-advisories.md) poster. Note that this would
> not be implemented until next season.

---

**Potential Submission Validation Warnings Considerations**

- **Maximum:**
  - Quantile values for % for NSSP forecast percentages (e.g., not higher than
    0.05% + highest observed value in any state in recent history)
  - Quantile values for weekly hospital admissions should not be higher than e.g.,
    30% of the state population size
  - (could also consider a warning specifically for median values)

- **Prediction Interval Width or Maximum:**
  - Prediction interval width in relation to median, to be discussed: potential
    option to warn for undesirably large prediction intervals

- **Calibration:**
  - Distance from last observed reported value and -1, 0, and maybe 1-week horizons
    should be less than x admissions or y% with different tolerance thresholds for
    each horizon. Chosen values should consider previously observed ranges of back
    fill, especially for the -1 horizon.

- **General Score/Performance Warning:**
  - Poor performing models in general
  - ex: model was ranked in the bottom 10 performing models in the last 2 weeks
  - Might involve uploading csv with bottom 10 every week, open to ideas

The CDC also raised open questions to discuss: what warning messages should look
like and how much detail they should carry; whether some submission characteristics
should escalate beyond a warning and fail validation (especially with the new
emphasis on a % target); and whether warnings could be tracked across weeks to
proactively flag persistently unreasonable forecasts (noted as likely subjective and
lower priority).
