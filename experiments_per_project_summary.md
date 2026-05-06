# Tóm tắt kết quả `experiments_per_project.ipynb`

Notebook `experiments_per_project.ipynb` chạy bài toán phân loại nhị phân để dự đoán một test file có được refactor hay không (`isRefactored`) trên bộ dữ liệu `sources-rf/dataset/per_project_merged.csv`.

## Thiết lập thí nghiệm

| Hạng mục | Giá trị |
|---|---|
| Bài toán | Cross-project classification cho test refactoring prediction |
| Nhãn dự đoán | `isRefactored` |
| Số project sau khi xử lý timestamp | 59 |
| Số dòng dữ liệu | 5,213 |
| Số feature dùng cho học máy | 41 |
| Khoảng thời gian commit thật | 2010-10-27 đến 2023-08-16 |
| Split | Temporal 80/20 theo `commit_date` |
| Cutoff | 2022-02-14 |
| Train | 4,172 dòng, 56 project |
| Test | 1,041 dòng, 35 project |
| Tiền xử lý | Fill missing value bằng 0.0, `StandardScaler`, SMOTE trên train set |
| Số lần chạy | 5 seed khác nhau |
| Sweep `n_estimators` | 10, 50, 100, 200, 500 |

## Chất lượng dữ liệu timestamp

Notebook truy vấn GitHub API để lấy timestamp cho từng commit và cache vào `commit_timestamps_cache.csv`.

| Chỉ số | Giá trị |
|---|---:|
| Dòng có timestamp thật | 5,157 / 5,213 |
| Dòng dùng sentinel date | 56 / 5,213 |
| Project có đủ timestamp | 59 |
| Project có một phần timestamp | 0 |
| Project không có timestamp | 2 |
| Missing feature values | 866 |

Hai project không lấy được timestamp là `groupon/DotCi` và `salesforce/Argus`. Các dòng không có timestamp được gán sentinel date `1970-01-01`, nên luôn nằm trong train set. Test set chỉ gồm các dòng có timestamp thật.

## Experiment A: so sánh model

Kết quả dưới đây là trung bình và độ lệch chuẩn qua 5 lần chạy. Split train/test cố định theo thời gian; seed thay đổi ảnh hưởng đến SMOTE và các model có thành phần ngẫu nhiên.

| Model | Precision | Recall | F1 | MCC | AUC-ROC |
|---|---:|---:|---:|---:|---:|
| Dummy (Baseline) | 0.000 ± 0.000 | 0.000 ± 0.000 | 0.000 ± 0.000 | 0.000 ± 0.000 | 0.500 ± 0.000 |
| Logistic Regression | 0.629 ± 0.003 | 0.548 ± 0.004 | 0.586 ± 0.004 | 0.154 ± 0.007 | 0.586 ± 0.002 |
| Naive Bayes | 0.634 ± 0.005 | 0.217 ± 0.003 | 0.323 ± 0.004 | 0.082 ± 0.005 | 0.561 ± 0.004 |
| Decision Tree | 0.593 ± 0.014 | 0.555 ± 0.040 | 0.573 ± 0.023 | 0.089 ± 0.029 | 0.545 ± 0.015 |
| KNN | 0.591 ± 0.005 | 0.523 ± 0.009 | 0.555 ± 0.006 | 0.081 ± 0.009 | 0.551 ± 0.002 |
| SVM (Linear) | 0.629 ± 0.005 | 0.551 ± 0.009 | 0.588 ± 0.006 | 0.155 ± 0.009 | 0.586 ± 0.002 |
| AdaBoost | 0.614 ± 0.008 | 0.480 ± 0.028 | 0.539 ± 0.020 | 0.113 ± 0.018 | 0.591 ± 0.003 |
| Gradient Boosting | 0.641 ± 0.003 | 0.591 ± 0.016 | 0.615 ± 0.009 | 0.187 ± 0.008 | 0.630 ± 0.006 |
| Random Forest | 0.641 ± 0.009 | 0.607 ± 0.018 | 0.624 ± 0.013 | 0.192 ± 0.022 | 0.632 ± 0.008 |
| Extra Trees | 0.613 ± 0.005 | 0.630 ± 0.016 | 0.621 ± 0.010 | 0.146 ± 0.013 | 0.609 ± 0.002 |
| **XGBoost** | **0.645 ± 0.010** | **0.630 ± 0.019** | **0.637 ± 0.014** | **0.206 ± 0.023** | **0.636 ± 0.009** |

Kết quả chính:

- XGBoost là model tốt nhất theo F1, MCC và AUC-ROC.
- Random Forest và Extra Trees đứng sau XGBoost rất sát về F1.
- Nhóm tree-based ensemble tốt hơn rõ rệt so với Logistic Regression, SVM, KNN, Decision Tree và Naive Bayes.
- Dummy baseline có F1 bằng 0 vì luôn dự đoán lớp đa số.

## Experiment B: sweep `n_estimators`

Notebook chỉ sweep các model tree-based và boosting: Random Forest, Extra Trees, XGBoost, Gradient Boosting, AdaBoost.

| Model | Best `n_estimators` theo F1 | F1 mean | AUC mean |
|---|---:|---:|---:|
| AdaBoost | 500 | 0.548 | 0.598 |
| Extra Trees | 200 | 0.626 | 0.613 |
| Gradient Boosting | 10 | 0.639 | 0.619 |
| Random Forest | 200 | 0.632 | 0.637 |
| **XGBoost** | **500** | **0.645** | **0.640** |

Nhận xét:

- XGBoost cải thiện khi tăng số estimator và đạt F1 cao nhất ở `n_estimators=500`.
- Random Forest tăng mạnh từ 10 lên 200 estimator, sau đó gần như bão hòa.
- Extra Trees ổn định quanh 200 đến 500 estimator.
- Gradient Boosting đạt F1 tốt nhất ngay tại `n_estimators=10`, sau đó không cải thiện thêm.
- AdaBoost thấp hơn các ensemble còn lại, dù tăng estimator có giúp cải thiện nhẹ.

## Final evaluation

Với seed cố định `42`, notebook train lại toàn bộ model để tạo ROC curve và feature importance.

| Hạng mục | Giá trị |
|---|---:|
| Train project | 56 |
| Train sample sau SMOTE | 4,322 |
| Test project | 35 |
| Test sample | 1,041 |

Notebook lưu các hình:

- `fig1_class_distribution`
- `fig2_temporal_split`
- `fig3_experiment_a_metrics`
- `fig4_estimators_sweep`
- `fig5_roc_curves`
- `fig6_feature_importance`
- `fig7_feature_category`

Notebook cũng lưu các file CSV:

- `1_experiment_a_per_run.csv`
- `2_experiment_a_summary.csv`
- `3_experiment_b_sweep.csv`
- `4_experiment_b_best_nestimators.csv`
- `5_final_evaluation_seed42.csv`
- `6_feature_importance.csv`

## Feature importance

Feature importance được tính bằng trung bình importance của Random Forest và Extra Trees.

| Rank | Feature | Nhóm feature | Importance |
|---:|---|---|---:|
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

Ba feature quan trọng nhất đều là metric liên quan đến số dòng bị xóa: `removedLinesCount`, `removedLinesMax`, `removedLinesAvg`. Điều này cho thấy lịch sử thay đổi mã nguồn, đặc biệt là mức độ xóa/chỉnh sửa dòng, có tín hiệu mạnh trong việc dự đoán test refactoring.

## Kết luận

Trong thiết lập cross-project temporal, XGBoost là lựa chọn tốt nhất với F1 `0.637 ± 0.014`, MCC `0.206 ± 0.023` và AUC-ROC `0.636 ± 0.009`. Random Forest và Extra Trees có F1 gần tương đương, nhưng XGBoost ổn định hơn ở nhóm metric tổng thể.

Các metric Git/churn chiếm ưu thế trong top feature importance, trong khi test smell nổi bật nhất là `Magic Number Test`. Kết quả này gợi ý rằng thông tin lịch sử thay đổi file và churn có vai trò lớn hơn test smell riêng lẻ khi dự đoán refactoring trong bối cảnh cross-project.
