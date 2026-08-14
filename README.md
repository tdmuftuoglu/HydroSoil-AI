# HydroSoil-AI

Prediction of soil saturated hydraulic conductivity (Ksat) from the Soil Water
Infiltration Global (SWIG) database, with an evaluation design that respects the
hierarchical structure of the compilation.

This repository contains the complete analysis notebook, the input database, and
every table, figure and model artefact reported in the accompanying manuscript.

---

## Why the evaluation design matters

SWIG is a compilation of measurements contributed by many independent research
groups. Records arrive in blocks: each block is one study, carried out at one or a
few sites, with one instrument, one operator team and one measurement protocol.
Records inside a block share far more than their soil physics.

If such data are split into training and test partitions at random, records from
the same block land on both sides of the split. A flexible model can then
recognise the block from its covariate signature and reproduce that block's mean
response. The resulting test score measures interpolation inside known studies,
not prediction for new ones.

This pipeline quantifies that difference directly. The same model, the same
features and the same preprocessing are evaluated under four fold-assignment
rules:

| Protocol | Fold assignment | Test R² | RMSE (log₁₀) |
|---|---|---|---|
| P1 | Random row-wise | +0.585 | 0.720 |
| P2 | Grouped by source dataset | −0.641 | 1.248 |
| P3 | Grouped by country | −0.729 | 1.277 |
| P4 | Grouped by 5° geographic block | −0.821 | 1.276 |

The standard deviation of the target is 1.122 log₁₀ units. An RMSE at or above
that value means the model is no more informative than predicting the sample mean.
Averaged over fifteen algorithms, the optimism introduced by row-wise splitting is
0.71 in R².

---

## Running the analysis

Open `HydroSoilAIv2.ipynb` in Google Colab or Jupyter and run all cells. The
notebook prompts for the SWIG workbook, or picks it up automatically if
`SWIG database.xlsx` is present in the working directory.

Required packages are `numpy`, `scipy`, `pandas`, `scikit-learn`, `xgboost`,
`lightgbm`, `shap`, `matplotlib`, `openpyxl` and `joblib`. The notebook installs
`xgboost`, `lightgbm` and `shap` automatically if they are absent, so it runs in a
fresh Colab session without preparation.

Runtime is roughly 30–60 minutes. Every expensive loop writes its result to a
checkpoint file as soon as it completes, so an interrupted run can be resumed by
running the notebook again; finished work is skipped.

Set `FAST_MODE = True` in the configuration cell for a two-minute smoke test with
reduced fold counts and search budget. Results from a fast run are not suitable
for reporting.

---

## Input

`SWIG database.xlsx` is included here for convenience. It was compiled and
published by Rahmati et al. (2018), *Earth System Science Data* 10, 1237–1263, and
is redistributed unmodified under its original CC-BY licence. The MIT licence of
this repository applies to the code, not to the database.

The notebook does not hard-code the file name or its internal layout. Sheet names
are matched by the substrings "meta" and "loc", the header row is detected
automatically, and column names are matched against a table of aliases, so
variants such as `BD` for bulk density or `Ks` for conductivity are recognised.
Optional columns that are absent are created empty; if the geometric mean particle
diameter and its standard deviation are missing they are derived from the texture
fractions following Shirazi and Boersma (1984).

---

## Pipeline

| Phase | Content |
|---|---|
| 1 | Locate and read the database, tolerant of file, sheet and column naming differences |
| 2 | Define the analysis sample and report every exclusion in a flow table |
| 3 | Construct the physics-guided features, with explicit equations and units |
| 4 | Set up the cross-validation machinery, the model panel and the metrics |
| 5 | Compare four validation protocols; decompose the result into level bias and within-study ranking skill |
| 6 | Benchmark fifteen algorithms under two protocols |
| 7 | Feature-set ablation against a null model and an unfitted Kozeny–Carman baseline |
| 8 | Leave-one-instrument-out, leave-one-country-out, variance components |
| 9 | Out-of-fold permutation importance and training-set SHAP |
| 10 | Deployable four-input model, conformal prediction intervals, applicability domain |
| 11 | All figures |
| 12 | Export tables, analysis dataset, model artefact and run metadata |

### Methodological rules observed throughout

- Imputation and scaling are fitted on the training partition of each outer fold
  and applied to the held-out partition. No preprocessing sees the evaluation
  data.
- Hyper-parameters are selected in a group-aware inner loop nested inside each
  outer fold, so the reported score carries no model-selection optimism.
- Every stochastic component is seeded, including the estimator used inside the
  imputer.
- Feature importance is computed on held-out folds. Training-set attributions are
  reported separately and only for contrast.
- The deployable model is evaluated with exactly the inputs its interface
  requests.

---

## Repository contents

| File | Content |
|---|---|
| `HydroSoilAIv2.ipynb` | the complete analysis pipeline |
| `SWIG database.xlsx` | input database (Rahmati et al. 2018), redistributed unmodified |
| `results_tables.xlsx` | every result table, one sheet each |
| `analysis_dataset.csv` | analysis sample with grouping labels and constructed features |
| `protocols.csv`, `algorithms.csv`, `ablation.csv`, `oof.pkl` | intermediate results and out-of-fold predictions |
| `ksat_screening_model.joblib` | deployable four-input screening model |
| `run_metadata.json` | random seed, fold counts, search budget, sample sizes, library versions |
| `Fig1_sample.png` … `Fig8_screening_model.png` | the eight figures at 300 dpi |

The serialised model contains the imputer, the estimator, the scaler, the
conformal calibration constant, the applicability-domain threshold and the
measured performance, so that the deployed pipeline and the evaluated pipeline are
the same object. It expects clay, sand, organic carbon and bulk density, in that
order, and predicts log₁₀ Ksat in cm h⁻¹.

---

## Analysis sample

| Step | Records remaining |
|---|---|
| Infiltration records in SWIG | 5023 |
| Records with a reported Ksat value | 1895 |
| Ksat > 0 (log-transformable) | 1893 |
| Bulk density ≤ 2.65 g cm⁻³ | 1892 |
| At least one core soil property reported | 1883 |

The final sample spans 45 source datasets in 26 countries, measured with 11
different instruments. Predictor missingness is substantial: organic carbon
30.5 %, texture 15.2 %, bulk density 5.2 %.

---

## Scope of the screening model

Under dataset-grouped cross-validation the four-input model reaches R² = −0.122,
RMSE = 1.087 log₁₀ units, a median multiplicative error of a factor of 3.9, and
74 % of predictions within one order of magnitude. Conformal intervals achieve
81 % empirical coverage against a nominal 90 %, with a median width spanning a
factor of about 151 in Ksat.

These figures support use of the model as an order-of-magnitude screening estimate
and as an aid to prioritising where measurements should be made. They do not
support use as a substitute for a permeameter or infiltrometer measurement, and no
such claim should be made on the basis of this model.

---

## Reproducibility

One fixed seed (20250812) throughout. The reference run used Python 3.12.13,
numpy 2.0.2, pandas 2.2.2 and scikit-learn 1.6.1, with five outer folds, three
inner folds and eight randomised-search candidates per inner loop. These settings
are recorded in `run_metadata.json`.

Every number, table and figure in the accompanying manuscript is generated by this
notebook from the database file included here. No result is transcribed by hand.

---

## Citation

If you use this software, please cite the accompanying manuscript and the source
database:

> Rahmati, M., et al. (2018). Development and analysis of the Soil Water
> Infiltration Global database. *Earth System Science Data*, 10(3), 1237–1263.

---

## License

Code: MIT (see `LICENSE`). Database: CC-BY, © Rahmati et al. (2018).
