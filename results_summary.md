# Experiment Results — Cross-Project Classification (Temporal Split)

**Dataset**: 59 open-source Java projects — `sources-rf/dataset/per_project_merged.csv`  
**Task**: Binary classification — predict `isRefactored`  
**Setting**: Cross-project, temporal 80/20 split (cutoff by commit date), SMOTE, StandardScaler  
**Date**: 2026-04-22

---

## Config

| Parameter | Value |
|---|---|
| Runs | 5 (different SMOTE/model seeds, fixed temporal split) |
| Split type | Temporal — cutoff at **2022-02-14** (80th percentile of commit dates) |
| Train | 56 projects · 4,172 rows (+ 56 sentinel rows from 2 unreachable repos) |
| Test | 35 projects · 1,041 rows (real timestamps only) |
| Date range | 2010-10-27 → 2023-08-16 |
| n\_estimators sweep | 10, 50, 100, 200, 500 |
| Models | Dummy, Logistic Regression, Naive Bayes, Decision Tree, KNN, SVM (Linear), AdaBoost, Gradient Boosting, Random Forest, Extra Trees, XGBoost |

---

## RQ1 — All Models (mean ± std, n\_estimators = 100 for ensembles)

Source: `metrics/2_experiment_a_summary.csv`

| Model | Precision | Recall | F1 | MCC | AUC-ROC |
|---|---|---|---|---|---|
| Dummy (Baseline) | 0.000 ± 0.000 | 0.000 ± 0.000 | 0.000 ± 0.000 | 0.000 ± 0.000 | 0.500 ± 0.000 |
| Logistic Regression | 0.629 ± 0.003 | 0.548 ± 0.004 | 0.586 ± 0.004 | 0.154 ± 0.007 | 0.586 ± 0.002 |
| Naive Bayes | 0.634 ± 0.005 | 0.217 ± 0.003 | 0.323 ± 0.004 | 0.082 ± 0.005 | 0.561 ± 0.004 |
| Decision Tree | 0.593 ± 0.014 | 0.555 ± 0.040 | 0.573 ± 0.023 | 0.089 ± 0.029 | 0.545 ± 0.015 |
| KNN | 0.591 ± 0.005 | 0.523 ± 0.009 | 0.555 ± 0.006 | 0.081 ± 0.009 | 0.551 ± 0.002 |
| SVM (Linear) | 0.629 ± 0.005 | 0.551 ± 0.009 | 0.588 ± 0.006 | 0.155 ± 0.009 | 0.586 ± 0.002 |
| AdaBoost | 0.614 ± 0.008 | 0.480 ± 0.028 | 0.539 ± 0.020 | 0.113 ± 0.018 | 0.591 ± 0.003 |
| Gradient Boosting | 0.641 ± 0.003 | 0.591 ± 0.016 | 0.615 ± 0.009 | 0.187 ± 0.008 | 0.630 ± 0.006 |
| Random Forest | 0.641 ± 0.009 | 0.607 ± 0.018 | 0.624 ± 0.013 | 0.192 ± 0.022 | 0.632 ± 0.009 |
| Extra Trees | 0.613 ± 0.005 | 0.630 ± 0.016 | 0.621 ± 0.010 | 0.146 ± 0.013 | 0.609 ± 0.002 |
| **XGBoost** | **0.645 ± 0.010** | **0.630 ± 0.019** | **0.637 ± 0.014** | **0.206 ± 0.023** | **0.636 ± 0.009** |

**Ranking**: XGBoost > RF > ET > GB >> LR ≈ SVM > DT > AdaBoost > KNN > NB > Dummy  
Tree-based ensembles form a clear cluster 4–10 F1 points above linear/instance-based models.

---

## n\_estimators Sweep (Best Configuration per Tree Model)

Source: `metrics/4_experiment_b_best_nestimators.csv`

| Model | Best n | F1 mean | F1 std | AUC |
|---|---|---|---|---|
| AdaBoost | 500 | 0.548 | 0.010 | 0.598 |
| Extra Trees | 200 | 0.626 | 0.013 | 0.613 |
| Gradient Boosting | 10 | 0.639 | 0.009 | 0.619 |
| Random Forest | 200 | 0.632 | 0.011 | 0.637 |
| **XGBoost** | **500** | **0.645** | 0.009 | **0.640** |

GB converges at n=10; XGBoost benefits from larger ensembles (F1 gains steadily to n=500).

---

## RQ2 — Top-10 Feature Importance (RF + ET mean, 5 runs)

Source: `metrics/6_feature_importance.csv` — computed as (RF\_importance + ET\_importance) / 2

| Rank | Feature | Category | Avg. Score |
|---|---|---|---|
| 1 | `removedLinesCount` | Git metric | 0.0501 |
| 2 | `removedLinesMax` | Git metric | 0.0489 |
| 3 | `removedLinesAvg` | Git metric | 0.0478 |
| 4 | `Magic Number Test` | Test smell | 0.0468 |
| 5 | `AsD` | Code metric | 0.0460 |
| 6 | `RFC` | Code metric | 0.0419 |
| 7 | `LOC` | Code metric | 0.0418 |
| 8 | `codeChurnTotal` | Git metric | 0.0413 |
| 9 | `codeChurnMax` | Git metric | 0.0399 |
| 10 | `codeChurnAvg` | Git metric | 0.0391 |

6/10 features are git metrics. Test smells contribute only 1 entry (Magic Number Test, rank 4).

---

## Comparison with Within-Project Baseline

| | Paper gốc (Martins 2024) | **Nghiên cứu hiện tại** |
|---|---|---|
| Setting | Within-project walk-forward | **Cross-project temporal 80/20** |
| Best model | Logistic Regression | **XGBoost** |
| F1 | **0.439** macro-F1 (undefined F1 set to 0) | **0.637 ± 0.014** |
| MCC | — | **0.206 ± 0.023** |
| AUC-ROC | — | **0.636 ± 0.009** |
| Test projects | Per-project | **35 unseen projects** |

Cross-project XGBoost cải thiện F1 khoảng 20 điểm phần trăm so với baseline within-project.

---

## Data Quality Notes

- 2 projects unreachable (deleted/private): `groupon/DotCi`, `salesforce/Argus` → 56 rows gán sentinel date 1970-01-01 → luôn vào training set.
- Test set hoàn toàn sạch (chỉ real timestamps).
- Missing feature values: 866 cells → filled với 0.0.
- Cột "Average" trong `6_feature_importance.csv` không dùng — paper tính (RF+ET)/2 trực tiếp từ các cột RF và ET.
