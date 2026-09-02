# HydroSoil-AI

Prediction of soil saturated hydraulic conductivity (Ksat) from the Soil Water
Infiltration Global (SWIG) database, with an evaluation design that respects the
hierarchical structure of the compilation.

This repository contains the analysis script, the input database, and the tables,
figures and model artefact it produces.

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

| Protocol | Fold assignment | Skill | R² | RMSE (log₁₀) |
|---|---|---|---|---|
| P1 | Random row-wise | +0.514 | +0.512 | 0.873 |
| P2 | Grouped by source dataset | −0.629 | −1.044 | 1.371 |
| P3 | Grouped by country | −0.584 | −1.184 | 1.405 |
| P4 | Grouped by 5° geographic block | −0.345 | −0.715 | 1.313 |

Averaged over fifteen algorithms, the optimism introduced by row-wise splitting is
1.07 in R². The difference between protocols is significant in all nine paired
comparisons.

For the standard pedotransfer inputs of texture, organic carbon and bulk density,
grouped validation gives a skill of −0.005 with a 95 % confidence interval of
0.033 across fifteen folds. For a soil from a study that was not represented in
training, such a model is exactly as good as predicting the mean of the training
data and no better.

## Two ways of measuring performance

The coefficient of determination is computed against the mean of the evaluation
fold. When a whole study is held out and that study sits far from the overall
mean, the denominator shrinks and R² falls even for a model that is not
performing worse, so R² is not comparable across these protocols.

Skill is computed against the mean of the training partition, which is the
prediction a model with no information would actually make. A skill of zero means
the model matches that prediction; a negative value means it is worse. Both are
reported throughout, and skill is the quantity to compare across protocols.

---

## Running the analysis

Open `HydroSoilAI_analysis.py` in Google Colab or Jupyter, or run it from a
terminal. The script prompts for the SWIG workbook, or picks it up automatically
if `SWIG database.xlsx` is present in the working directory.

    python HydroSoilAI_analysis.py
    python HydroSoilAI_analysis.py /path/to/SWIG.xlsx

Required packages are `numpy`, `scipy`, `pandas`, `scikit-learn`, `xgboost`,
`lightgbm`, `shap`, `matplotlib`, `openpyxl` and `joblib`. The script installs
`xgboost`, `lightgbm` and `shap` automatically if they are absent, so it runs in a
fresh Colab session without preparation.

Runtime is roughly 60 to 90 minutes. Every expensive loop writes its result to a
checkpoint file as soon as it completes, so an interrupted run can be resumed by
running the script again; finished work is skipped.

Set `FAST_MODE = True` in the configuration block for a two-minute smoke test with
reduced fold counts and search budget. Results from a fast run are not suitable
for reporting.

---

## Input

`SWIG database.xlsx` is included here for convenience. It was compiled and
published by Rahmati et al. (2018), *Earth System Science Data* 10, 1237–1263, and
is redistributed unmodified under its original CC-BY 4.0 licence. The MIT licence
of this repository applies to the code only, not to the database.

The script does not hard-code the file name or its internal layout. Sheet names
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
| 2 | Define the analysis sample, report every exclusion, and remove exactly duplicated records |
| 3 | Construct the physics-guided features, with explicit equations and units |
| 4 | Set up the cross-validation machinery, the model panel and the metrics |
| 5 | Compare four validation protocols, test the difference fold by fold, and decompose the result into level bias and within-study ranking skill |
| 5b | Measure how much of the row-wise result comes from duplicated records |
| 6 | Benchmark fifteen algorithms under two protocols |
| 7 | Feature-set ablation against a null model and an unfitted Kozeny–Carman baseline |
| 8 | Leave-one-instrument-out, leave-one-country-out, variance components |
| 9 | Out-of-fold permutation importance and training-set SHAP |
| 10 | Deployable four-input model, conformal prediction intervals, applicability domain |
| 11 | Eight figures |
| 12 | Export tables, analysis dataset, model artefact and run metadata |

### Methodological rules observed throughout

- Records that are exact duplicates of another record are removed before any model
  is fitted, because a duplicate falling on both sides of a split is the most
  direct form of leakage available.
- Imputation and scaling are fitted on the training partition of each outer fold
  and applied to the held-out partition. No preprocessing sees the evaluation
  data.
- Hyper-parameters are selected in a group-aware inner loop nested inside each
  outer fold, so the reported score carries no model-selection optimism.
- Skill is measured against the mean of the training partition, not that of the
  evaluation fold.
- The outer cross-validation is repeated with several fold assignments, and the
  difference between protocols is tested fold by fold.
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
| `HydroSoilAI_analysis.py` | the analysis pipeline |
| `HydroSoilAIv2.ipynb` | the same pipeline as a notebook |
| `predict_ksat.py` | command-line tool that serves the four-input screening model |
| `requirements.txt` | package list |
| `SWIG database.xlsx` | input database (Rahmati et al. 2018), redistributed unmodified |
| `results_tables.xlsx` | every result table, one sheet each |
| `analysis_dataset.csv` | analysis sample with grouping labels and constructed features |
| `ablation.csv`, `algorithms.csv` | intermediate checkpoint files |
| `ksat_screening_model.joblib` | deployable four-input screening model |
| `run_metadata.json` | random seed, fold counts, search budget, sample sizes, library versions |
| `Fig1_sample.png` … `Fig8_screening_model.png` | the eight figures produced by the script, at 300 dpi |

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
| Texture fractions summing to 100 ± 5 % | 1892 |
| At least one core soil property reported | 1883 |
| Not an exact duplicate of a retained record | 1058 |

The final sample spans 45 source datasets in 26 countries, measured with eleven
different instruments. The largest single source dataset contributes 135 records,
12.8 % of the sample. Predictor missingness: organic carbon 16.5 %, texture 8.7 %,
bulk density 7.1 %, particle density 78.0 %.

Restoring the 825 duplicated records raises the row-wise R² from 0.291 to 0.448,
so more than a third of the apparent row-wise performance of the compilation comes
from identical records appearing on both sides of the split.

---

## Scope of the screening model

Under dataset-grouped cross-validation the four-input model reaches R² = −0.355,
RMSE = 1.226 log₁₀ units, a median multiplicative error of a factor of 15, and
68 % of predictions within one order of magnitude. Conformal intervals achieve
77 % empirical coverage against a nominal 90 %, with a median width of about three
orders of magnitude in Ksat.

These figures support use of the model as an order-of-magnitude screening estimate
and as an aid to prioritising where measurements should be made. They do not
support use as a substitute for a permeameter or infiltrometer measurement, and no
such claim should be made on the basis of this model.

---

## Reproducibility

One fixed seed (20250812) throughout. The reference run used Python 3.13.15,
numpy 2.1.3, pandas 2.2.3 and scikit-learn 1.6.1, with five outer folds repeated
three times, three inner folds, and eight randomised-search candidates per inner
loop. These settings are recorded in `run_metadata.json`.

Every numerical result in the accompanying manuscript is generated by this
pipeline from the database file included here; no result is transcribed by hand.
The manuscript additionally contains figures that are not produced by the script:
a schematic of the analysis framework, a correlation heat map, a panel comparing
training-set SHAP attribution with out-of-fold importance, and screenshots of the
desktop application.

---

## Citation

If you use this software, please cite the accompanying manuscript and the source
database:

> Rahmati, M., et al. (2018). Development and analysis of the Soil Water
> Infiltration Global database. *Earth System Science Data*, 10(3), 1237–1263.

---

## License

Code: MIT (see `LICENSE`). Database: CC-BY 4.0, © Rahmati et al. (2018).
