# 2025-01-13 Reich Lab Model Materials Storage

## Context

The Reich Lab currently has no formalized structure for organizing how forecasting model materials (e.g. general modeling code, weekly model runs, evaluations and experiments) are stored.

Currently, the Lab usually has one or more repositories dedicated to storing each of the three categories of materials described above for each model. Flusion currently uses (1) [a Python package repo with general model implementations](https://github.com/reichlab/idmodels), (2) a folder in [operational-models](https://github.com/reichlab/operational-models) (and a [fork of FluSight-forecast-hub](https://github.com/elray1/FluSight-forecast-hub)), and (3) [a separate experimental repo](https://github.com/reichlab/flusion-experiments/tree/main) (that also stores an archive of weekly model outputs/plots). This seems like a reasonable organizational structure to emulate, with a few adjustments.

Following this structure is not required for models that are not being shared publicly.

### Aims

- Define a structure to follow when organizing for Reich Lab models that will be shared publicly (e.g. submitted to modeling hubs)

### Anti-Aims

- Models not being shared publicly are not required to follow the organizational structure proposed by this decision
- Archived models (i.e., models that are no longer submitting forecasts) are not required to change their current organizational structure
- This proposal only concerns the organizational structure for materials related to publicly shared models; it does not cover how to determine if a model is worth being shared publicly
- This proposal does not concern where research paper-related materials should live or how they should be organized
- This proposal does not enforce a naming convention for models or the repositories their materials are stored in (though some suggestions are given in the examples)

## Decision

We will utilize four (4) main code repositories to store our publicly-shared modeling materials, as described by the following subsections.

### 1 (R or Python) Package Repository

We will use a standard code package repository ([R package](https://r-pkgs.org/Structure.html#sec-source-package), [Python package](https://packaging.python.org/en/latest/tutorials/packaging-projects/)) to store the modeling code that implements a general method.

### 2 `operational-models` repository

We will use [`operational-models`](https://github.com/reichlab/operational-models) to store scripts containing code to run a model once, for routine weekly submissions (including the creation of some plots) by calling the model's package from the Package Repository. The `operational-models` repository will contain such scripts for ALL of the Lab's models submitting to a forecast hub, separated into model-specific directories following the naming convention `disease_modelname`.

```
reichlab/operational-models
├─ README.md
├─ flu_flusion/
│  ├─ .python-version
│  ├─ 0_ar6_pooled.py
│  ├─ 1_gbqr.py
│  ├─ 2_flusion_ensemble.py
│  ├─ 3_plot.py
│  ├─ README.md
│  ├─ X_subset.R
│  ├─ main.py
│  ├─ renv.lock
│  └─ requirements.txt
└─ disease_modelname/
   ├─ main.R
   ├─ README.md
   └─ renv.lock
```

Depending on the complexity of the model, there may be one or more scripts used in concert to generate the results for a single weekly model run. However, `main` (in this example, `main.py` or `main.R`) will be the main script that calls upon the others to obtain the model output submission file and plots to be sent to Slack for review.

### 3 Target Modeling Hub Repository (fork)

We will use the target modeling hub repository to store the model outputs we submit to that hub via committing them to our fork and submitting a pull request. The target modeling hub repository will follow the [standard structure laid out by the hubverse](https://hubverse.io/en/latest/user-guide/hub-structure.html).

No other materials related to the model will be stored here. Additionally, this fork may be on the GitHub account of the model's main developer, rather than the Lab's account.

### 4 Evaluations and Experiments Repository

We will use this repository to store materials that fall under the following purposes: (1) archived submission files or (2) evaluations (retrospective or otherwise) and experiments.

Archived submission files may include the actual model outputs we submit to the target hub repo (with any invalid or particularly poor predictions removed), the raw version of those model outputs (with no predictions removed), and the associated plots of the raw model outputs (which are also sent to Slack for review). At a minimum, the actual submitted model outputs should be saved within this repository. It is up to the model developer whether to save the other categories of archived submission files (though it may be beneficial to save raw model outputs if many forecasts are removed prior to submission).

Evaluations and experiments include code to run a model repeatedly (i.e., several past weeks) using the package from the Package Repository as part of a retrospective evaluation; model outputs for multiple retrospective model runs; code to systematically evaluate model outputs; and results of running code to evaluate model outputs, e.g. plots and scores. Note that the code from retrospective model runs is meant to *generate submissions for multiple rounds simultaneously*, which is the key difference from the scripts stored in the `operational-models` repo even if there are similarities between the two code bases.

The archived submission files and results of retrospective model runs may be stored in toy modeling hubs in individual directories within the repository. These should be formatted like the target modeling hub and contain relevant auxiliary data, config files, and (optionally) target data. Evaluation or experiment code can be stored in its own directory.

```
reichlab/modelname-experiments
├─ README.md
├─ LICENSE
├─ code/
│  ├─ eda/
│  ├─ eval/
│  │  └─ eval.R
│  ├─ team-model_name/
│  │  └─ team-model_name.R
│  └─ README.md
├─ retrospective-hub/
│  ├─ auxiliary-data/
│  ├─ hub-config/
│  ├─ model-artifacts/
│  ├─ model-output/
│  └─ plots/
└─ submissions-hub/
   ├─ auxiliary-data/
   ├─ hub-config/
   ├─ model-output/
   │  └─ team-model_name/
   │     ├─ YYYY-MM-DD-team-model_name-raw.csv
   │     └─ YYYY-MM-DD-team-model_name.csv
   └─ plots/
```

If the model in question is an ensemble, additional directories containing code, model output, or plots of the component models may be added to the above example file structure.

### Other Options Considered

Archived submission files could be stored in a separate repository from the evaluations and experiments materials. We prefer keeping them in a single, combined repository given that the evaluations or experiments code may be used on the archived submission data.

Operational code (such as model requirements, parameters, run incantations, and various scripts) for single model runs could alternatively be stored in the evaluations and experiments materials repository. This would be a shift from our current practices. Also note that we do not want to maintain identical code in multiple places.

## Status

Accepted

## Consequences

Pros:
- A standard organizational structure will make it easier to find where each particular part of a model's material lives within the Lab's GitHub organization
- Developers will know what types of modeling materials should live within each type of repository and when to create a new one
- The number of extraneous repositories will hopefully be reduced
- This will encourage better organizational practice across the development team

Cons:
- Existing model code for currently submitting models may need to be reorganized, which will take some upfront work

## Projects

- None
