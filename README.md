# HydroSoil-AI: Physics-Informed Machine Learning Framework for Global Soil Hydraulic Conductivity Prediction

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Python](https://img.shields.io/badge/python-3.8%2B-blue)
![Status](https://img.shields.io/badge/status-active-success)

## Overview

**HydroSoil-AI** is a comprehensive Python framework designed to predict Soil Saturated Hydraulic Conductivity Ksat using a Physics-Informed Machine Learning (PIML) approach. Unlike traditional empirical approaches that rely solely on statistical correlations, this framework explicitly integrates domain-specific physical constraints—specifically **Adams' Compaction Theory** and **Shin's Pore Geometry Models**—directly into the feature engineering pipeline.

The system automates the transition from raw soil data to a deployed predictive model through a rigorous, multi-stage computational workflow. This repository contains the source code, processed datasets, and the deployment pipeline associated with the manuscript.

## Key Features

* **Physics-Informed Feature Engineering:** Automated calculation of theoretical porosity, hydraulic diameter, and compaction indices to correct geometric deficiencies in standard datasets.
* **Stochastic Imputation:** Utilizes Iterative MICE (Multivariate Imputation by Chained Equations) with Extra Trees to preserve the covariance structure of soil properties.
* **Grandmaster Benchmark:** Automatically evaluates 15 machine learning algorithms (including XGBoost, LightGBM, and Stacking Ensembles) to identify the optimal predictor.
* **Explainable AI (XAI):** Integrated SHAP analysis to validate that the model relies on physical drivers (e.g., Macroporosity) rather than statistical artifacts.
* **Operational Deployment:** Includes serialization tools to export the trained model for standalone Windows applications.

## Computational Architecture & Methodology

The software framework consists of four sequential modules designed to handle data heterogeneity, enforce physical consistency, and optimize predictive accuracy.

### 1. Advanced Data Ingestion and Imputation
The framework handles missing values in heterogeneous soil datasets using **Multivariate Imputation by Chained Equations (MICE)**.
* **Estimator:** An **Extra Trees Regressor** is utilized as the kernel estimator within the iterative imputation process.
* **Covariance Preservation:** This stochastic approach preserves the multi-dimensional covariance structure between soil texture, organic carbon, and bulk density, ensuring that imputed values remain physically consistent with observed properties.
* **Preprocessing:** The module automatically filters physically impossible values (e.g., Bulk Density > 2.65 g/cm³) and applies a log-transformation to the target variable ($K_{sat}$) to stabilize variance.

### 2. The Physics Engine (Feature Engineering)
A dedicated computation module automates the injection of domain knowledge by deriving physics-based parameters from raw inputs:
* **Pore Geometry:** Calculates the *Specific Surface Area Proxy ($S_s$)* and *Hydraulic Diameter ($D_h$)* to model the effective flow path size based on Shin (2021).
* **Compaction Constraints:** Computes *Adams’ Theoretical Bulk Density* based on organic matter-mineral interactions to differentiate between structural compaction and organic stabilization (Adams, 1973).
* **Permeability Factors:** Derives *Shin’s Permeability Factor* and *Quadratic Porosity* to capture non-linear flow responses.

### 3. The "Grandmaster" Benchmarking Tournament
The framework executes a comparative benchmark analysis of 15 distinct machine learning algorithms to identify the optimal predictor.
* **Algorithms:** The repository includes implementations for Linear Baselines (Ridge, Lasso, ElasticNet), Non-Linear Estimators (SVR, Gaussian Processes, MLP), and Ensemble Methods (Random Forest, Extra Trees, XGBoost, LightGBM, CatBoost).
* **Evaluation Metrics:** Models are evaluated based on Test $R^2$ and the **Generalization Gap** (R^2_{train} - R^2_{test}).
* **Model Selection:** The system automatically flags models with a generalization gap exceeding 0.20 as "Overfit" and selects the model maximizing generalization capability as the "Champion."

### 4. Operational Deployment and Explainability
* **XAI Integration:** The framework integrates **SHAP (SHapley Additive exPlanations)** to generate global feature importance plots, validating physical consistency.
* **Serialization:** The trained champion model, along with its specific scaling parameters and encoders, is serialized into a binary format (`.pkl`) using the `joblib` library. This allows for the compilation of standalone executables for field application without requiring a Python environment.

## Repository Structure

* `main.py`: The core script execution pipeline (Data Ingestion -> Physics Engine -> Benchmarking -> Export).
* `HydroSoilAI_Processed_Data.xlsx`: The processed dataset containing the calculated physics features (Output of Phase 3).
* `requirements.txt`: List of dependencies (pandas, scikit-learn, xgboost, shap, etc.).

## Installation & Usage

1.  **Clone the repository:**
    ```bash
    git clone [https://github.com/tdmuftuoglu/HydroSoil-AI.git](https://github.com/tdmuftuoglu/HydroSoil-AI.git)
    cd HydroSoil-AI
    ```

2.  **Install dependencies:**
    ```bash
    pip install -r requirements.txt
    ```

3.  **Run the framework:**
    ```bash
    python main.py
    ```
    *Follow the on-screen prompts to upload the SWIG database file.*

## Citation

If you use this code or the methodology in your research, please cite the associated manuscript.


## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
