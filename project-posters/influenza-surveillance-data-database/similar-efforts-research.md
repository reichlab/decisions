## CMU Delphi Epidata for real-time disease indicators

[Summary table](https://delphi.cmu.edu/epiportal)
[General Epidata API](https://cmu-delphi.github.io/delphi-epidata/api/client_libraries.html)
[Epidatr R package](https://cmu-delphi.github.io/epidatr)

Database of multiple infectious disease indicators (signals) from public and private partners. Mostly focused in the US, but also includes data from several other countries. Supports versioning. Has/will have soon packages in R, Python, etc for querying data; additional analysis and modeling can be performed with separate, integrated packages.

Data generally try to have the same columns, especially for the same types of signals (e.g. ILI incidents in Europe vs South Korea), but sometimes columns differ, requiring some data processing prior to joining.

The earliest flu source begins in 1997, meaning we might be able to do better with additional historical data? Or we might consider trying to add to their work?

## UMass-Flusion

[Flusion repository](https://github.com/reichlab/flusion)
[Manuscript repository](https://github.com/reichlab/flusion-manuscript/blob/main/manuscript/flusion-manuscript.pdf)

The raw inputs to Flusion are stored as separate CSV files with little to no revisions made from the original data source. The column names are more disparate than those present in the Epidata database. Data is processed using a Python script, then ingested into the models for training. Versioning is used for some sources, stored as separate files for each version. Only US-based data sources are included.