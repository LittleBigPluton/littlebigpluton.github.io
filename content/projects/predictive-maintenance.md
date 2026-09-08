---
title: "Predictive Maintenance ML Pipeline"
description: "End-to-end machine learning for imbalanced equipment-failure prediction, with rigorous model selection, threshold optimization, uncertainty estimation and error analysis."
---

# Predictive Maintenance ML Pipeline

[← Back to projects](/#projects)

**Context:** Independent Machine Learning Project · 2026  
**Focus:** Imbalanced classification · Model evaluation · Decision thresholds · Reproducible ML engineering

I built an end-to-end machine-learning workflow for predicting industrial equipment failures using the **AI4I 2020 Predictive Maintenance Dataset**.

The project goes beyond fitting a classifier. It treats predictive maintenance as an **imbalanced decision problem**: validating the data, engineering physically meaningful features, comparing multiple model families under stratified cross-validation, selecting thresholds without using the test set, estimating uncertainty, and analysing where the final model succeeds and fails.

[View the full analysis notebook →](https://www.umutgokdemir.com/predictive-maintenance-ml/)  
[View the source on GitHub →](https://github.com/LittleBigPluton/predictive-maintenance-ml)

## Project snapshot

| Metric | Result |
|---|---|
| Dataset | 10,000 equipment observations |
| Models compared | 8 classification approaches |
| Final model | Tuned Random Forest |
| Cross-validated Average Precision | **0.8937** |
| Held-out test Average Precision | **0.876** |
| Test precision | **0.947** |
| Test recall | **0.794** |
| Test F1-score | **0.864** |
| Correctly detected failures | **54 / 68** |
| False alarms | **3** |
| Selected operating threshold | **0.575** |
| Bootstrap AP 95% CI | **0.798–0.942** |

## The challenge

Machine failures are rare relative to normal operation, which makes conventional accuracy a poor primary objective. A model can achieve high accuracy while still missing the failures that matter.

The project therefore focused on four questions:

- How should models be compared under strong class imbalance?
- Can domain-informed feature engineering improve failure detection?
- How should a probability threshold be selected without leaking information from the test set?
- What do the model's errors, subgroup behaviour and feature importances reveal about reliability?

To keep the final evaluation defensible, the held-out test set was used **only after model selection and threshold determination**.

## Data validation and leakage prevention

The workflow begins with explicit checks for missing values, duplicates, feature ranges, target integrity and class distribution.

Identifier columns such as `UDI` and `Product ID` are excluded from modelling. The five failure-mode indicators included in the dataset are also excluded from training because they directly encode the mechanism used to generate the target and would introduce target leakage.

The data is then split using a **stratified train/test split**, while model comparison and tuning are performed only on the training data using stratified cross-validation.

## Domain-informed feature engineering

Three interaction features were derived from the original sensor measurements:

- **Mechanical power** from rotational speed and torque
- **Temperature difference** between process and ambient air
- **Torque × tool wear** as an interaction representing accumulated mechanical strain

These features were chosen to reflect known failure mechanisms rather than adding arbitrary polynomial combinations.

An ablation analysis compares the raw feature set against the raw-plus-engineered representation before the final model is selected.

## Model comparison and selection

I compared eight baseline and imbalance-aware classification approaches:

- Dummy classifier
- Logistic Regression
- Balanced Logistic Regression
- L1-regularized Logistic Regression
- Decision Tree
- Random Forest
- XGBoost
- Class-weighted XGBoost

**Average Precision** was used as the primary model-selection metric because the positive class is rare and ranking likely failures is more informative than raw accuracy.

The tuned Random Forest achieved the strongest cross-validated result with **0.8937 Average Precision**. Hyperparameter tuning provided only a marginal gain over the untuned Random Forest, an important result in itself: the performance came primarily from the modelling pipeline and feature representation rather than aggressive search.

<!--
Copy this figure from:
predictive-maintenance-ml/reports/cli/figures/cross_validation_comparison.png

Suggested website destination:
static/images/projects/predictive-maintenance/cross_validation_comparison.png
-->
![Cross-validation comparison across candidate models](/images/projects/predictive-maintenance/cross_validation_comparison.png)

## Threshold selection as a decision problem

The strongest probability model is not automatically the strongest decision system.

Instead of using the default `0.5` cutoff, I generated **out-of-fold probabilities on the training set** and selected a recall-constrained operating threshold before touching the held-out test data.

The selected threshold was **0.575**.

I also evaluated an illustrative cost-sensitive objective in which missed failures and false alarms carry different costs. Under those assumptions, the cost-minimizing threshold moved to **0.105**, demonstrating how operational priorities can materially change the preferred decision boundary even when the underlying model stays fixed.

<!--
Copy this figure from:
predictive-maintenance-ml/reports/cli/figures/threshold_tradeoff.png

Suggested website destination:
static/images/projects/predictive-maintenance/threshold_tradeoff.png
-->
![Precision and recall across decision thresholds](/images/projects/predictive-maintenance/threshold_tradeoff.png)

## Held-out test performance

After model selection and threshold determination were frozen, the final model was evaluated once on the held-out test set.

It achieved:

- **0.876 Average Precision**
- **0.947 precision**
- **0.794 recall**
- **0.864 F1-score**
- **0.821 F2-score**

The final confusion matrix contained:

- **54** correctly detected failures
- **14** missed failures
- **3** false alarms
- **1,929** correctly identified non-failures

This makes the trade-off concrete: the selected operating point detected most failures while keeping false alarms very low.

<!--
Copy this figure from:
predictive-maintenance-ml/reports/cli/figures/confusion_matrix.png

Suggested website destination:
static/images/projects/predictive-maintenance/confusion_matrix.png
-->
![Held-out test confusion matrix](/images/projects/predictive-maintenance/confusion_matrix.png)

## Uncertainty, interpretability and error analysis

Point estimates alone can overstate confidence, especially when the positive class is small. I therefore used bootstrap resampling to estimate uncertainty around the main held-out metrics.

The **95% bootstrap interval for Average Precision was 0.798–0.942**, with corresponding uncertainty intervals for precision, recall and F1.

For interpretation, I compared native tree-based feature importance with **permutation importance**. Rotational speed emerged as the strongest predictor, while the engineered **temperature difference**, **torque × tool wear** and **power** features also contributed meaningful signal.

I also evaluated performance by product type and failure mode rather than relying only on aggregate metrics. This exposed weaker behaviour for rare Tool Wear Failure and Random Failure cases and showed where additional data would be most valuable.

<!--
Copy this figure from:
predictive-maintenance-ml/reports/cli/figures/permutation_importance.png

Suggested website destination:
static/images/projects/predictive-maintenance/permutation_importance.png
-->
![Permutation feature importance for the final model](/images/projects/predictive-maintenance/permutation_importance.png)

## Engineering the workflow

The project is structured as a reusable Python package rather than a notebook-only analysis.

Dedicated modules cover:

- dataset loading and schema handling,
- data-quality validation,
- feature engineering and preprocessing,
- model construction and hyperparameter tuning,
- threshold selection and model evaluation,
- calibration and cost-sensitive analysis,
- interpretability and subgroup evaluation,
- plotting and report generation,
- command-line workflow orchestration.

The CLI can regenerate the complete analysis, including the fitted model, selected threshold, metadata, figures and CSV reports.

Automated tests cover the reusable implementation, while the notebook provides the analytical narrative and the CLI provides reproducible end-to-end execution.

## Limitations

The **AI4I 2020 dataset is synthetic**, so these results should not be interpreted as production-ready predictive-maintenance performance.

Real industrial systems would introduce additional challenges such as temporal dependencies, machine-specific histories, sensor drift, changing operating conditions and maintenance costs grounded in real operations.

The project therefore demonstrates a rigorous **machine-learning evaluation workflow**, not a claim that this specific model is ready for deployment on physical equipment.

## What I contributed

- Built an end-to-end binary-classification workflow for an imbalanced predictive-maintenance problem.
- Engineered domain-informed interaction features from raw operating measurements.
- Compared eight baseline and imbalance-aware model configurations under stratified cross-validation.
- Tuned Random Forest and XGBoost using Average Precision as the primary selection metric.
- Selected the operating threshold from out-of-fold training predictions rather than the held-out test set.
- Added calibration checks, cost-sensitive threshold analysis and bootstrap confidence intervals.
- Performed permutation/native feature-importance analysis, subgroup evaluation and detailed false-positive/false-negative analysis.
- Packaged the workflow into reusable Python modules with automated tests, CLI execution and reproducible artifact generation.

## What this project demonstrates

This project demonstrates how I approach machine learning as an **evaluation and decision-engineering problem**, not only a model-fitting task.

It combines data validation, leakage prevention, feature engineering, imbalanced classification, cross-validation, hyperparameter tuning, threshold optimization, uncertainty estimation, interpretability, error analysis and reproducible software engineering in one workflow.

**Core technologies:** Python · scikit-learn · XGBoost · pandas · NumPy · Matplotlib · Random Forest · Model Evaluation · Feature Engineering · Imbalanced Classification · pytest

[View the full analysis notebook →](https://www.umutgokdemir.com/predictive-maintenance-ml/)  
[View source on GitHub →](https://github.com/LittleBigPluton/predictive-maintenance-ml)

[← Back to projects](/#projects)
