# Tóm tắt so sánh: Cross-project, Walk-forward và Table 6.2

File này tóm tắt kết quả trong thư mục `ten-projects_results` và so sánh với bảng cũ trong ảnh `ten-projects/metrics_10_projects.jpg`.

## Nguồn dữ liệu

- Kết quả cross-project temporal: `ten-projects_results/cp_results_summary.csv`
- Kết quả walk-forward within-project: `ten-projects_results/wf_results_all_models.csv`
- Bảng cũ Table 6.2: `ten-projects/metrics_10_projects.jpg`

## Kết quả cross-project temporal

Trong thiết lập cross-project temporal, hai model tốt nhất là Random Forest và XGBoost:

| Model | F1 mean | F1 std | AUC-ROC mean |
|---|---:|---:|---:|
| Random Forest | 0.586 | 0.023 | 0.633 |
| XGBoost | 0.583 | 0.024 | 0.620 |
| SVM (Linear) | 0.518 | 0.026 | 0.533 |
| Logistic Regression | 0.493 | 0.041 | 0.541 |
| KNN | 0.449 | 0.032 | 0.494 |

Random Forest là model tốt nhất theo F1 trung bình, nhưng XGBoost theo rất sát.

## So sánh với walk-forward within-project

F1 trung bình theo từng model trong thiết lập walk-forward:

| Model | Walk-forward mean F1 | Cross-project mean F1 | Chênh lệch |
|---|---:|---:|---:|
| Logistic Regression | 0.516 | 0.493 | -0.023 |
| KNN | 0.514 | 0.449 | -0.065 |
| SVM (Linear) | 0.505 | 0.518 | +0.013 |
| Random Forest | 0.570 | 0.586 | +0.016 |
| XGBoost | 0.579 | 0.583 | +0.004 |

Cross-project temporal tốt hơn nhẹ so với walk-forward khi so cùng từng model đối với Random Forest, XGBoost và SVM (Linear). Ngược lại, Logistic Regression và KNN trong cross-project cho kết quả thấp hơn walk-forward.

Nếu đánh giá walk-forward bằng cách chọn model tốt nhất riêng cho từng project, F1 trung bình đạt 0.604. Giá trị này cao hơn model cross-project tốt nhất, tức Random Forest với F1 trung bình 0.586.

## So sánh với Table 6.2 cũ

Bảng Table 6.2 trong ảnh cũ báo cáo các giá trị F1 sau:

| Project | F1 |
|---|---:|
| emissary | 0.73 |
| google-http-java-client | 0.36 |
| graphhopper | 0.62 |
| htmlunit-driver | NaN |
| iotdb | 0.47 |
| itext7 | 0.66 |
| kubernetes-client | 0.30 |
| questdb | 0.58 |
| tetrad | 1.00 |
| zanata-platform | 0.40 |

Nếu tổng hợp các giá trị này:

- Macro-F1 khi bỏ qua dòng `NaN`: xấp xỉ 0.569
- Macro-F1 khi tính `NaN` thành 0: xấp xỉ 0.512

So với các giá trị tổng hợp này:

| Phương pháp | F1 |
|---|---:|
| Table 6.2 cũ, bỏ qua NaN | 0.569 |
| Table 6.2 cũ, tính NaN thành 0 | 0.512 |
| Cross-project Random Forest | 0.586 |
| Cross-project XGBoost | 0.583 |

Kết quả cross-project temporal tốt nhất cao hơn nhẹ so với F1 tổng hợp của Table 6.2 cũ khi bỏ qua `NaN`: Random Forest cao hơn khoảng +0.017, XGBoost cao hơn khoảng +0.014.

## Lưu ý quan trọng

Đây không phải là so sánh project-by-project hoàn toàn tương ứng, vì 10 project trong Table 6.2 cũ và 10 project trong thí nghiệm mới không trùng nhau hoàn toàn.

Những project có trong Table 6.2 cũ nhưng không có trong kết quả 10 project mới:

- `google-http-java-client`
- `htmlunit-driver`
- `iotdb`
- `tetrad`

Những project có trong kết quả 10 project mới nhưng không có trong Table 6.2 cũ:

- `opencga`
- `nexus-public`
- `concord`
- `acs-aem-commons`

Vì vậy, so sánh với Table 6.2 chỉ nên được mô tả là so sánh tổng hợp theo aggregate, không phải là so sánh ghép cặp nghiêm ngặt trên cùng một tập project.

## Kết luận

Thiết lập cross-project temporal với 5 model cho kết quả cải thiện nhẹ so với F1 tổng hợp trong Table 6.2 cũ, đặc biệt với Random Forest và XGBoost. Nó cũng nhỉnh hơn walk-forward khi so cùng từng model đối với Random Forest và XGBoost.

Tuy nhiên, walk-forward vẫn mạnh hơn nếu mỗi project được chọn model tốt nhất riêng. Trong trường hợp đó, walk-forward đạt F1 trung bình 0.604, cao hơn cross-project temporal tốt nhất là 0.586.
