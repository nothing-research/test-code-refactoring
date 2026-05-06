# Tổng Quan Dữ Liệu và Pipeline Thực Nghiệm

Tài liệu này giải thích cách dữ liệu được tổ chức, gộp lại, các đặc trưng có trong dataset, nhãn phân loại, và sự khác biệt giữa cách tiếp cận của nghiên cứu gốc (Martins 2024) so với phương pháp hiện tại.

---

## 1. Cấu Trúc Dữ Liệu Thô

```
sources-rf/dataset/
├── per_project/          ← 59 file CSV, mỗi file = 1 dự án Java
│   ├── acs-aem-commons.csv
│   ├── admiral.csv
│   └── ... (57 file khác)
└── per_project_merged.csv  ← đã gộp, dùng cho thực nghiệm
```

Mỗi file trong `per_project/` đại diện cho một dự án Java mã nguồn mở trên GitHub. Mỗi **hàng** trong file là một **test file** (lớp Java chứa unit test) được đo tại một thời điểm commit cụ thể.

---

## 2. Quá Trình Gộp Dữ Liệu (`per_project/` → `per_project_merged.csv`)

Tất cả 59 CSV có cùng cấu trúc cột (schema). Việc gộp đơn giản là **nối dọc (vertical concatenation)** — xếp chồng tất cả 59 bảng thành một bảng duy nhất:

```python
import pandas as pd, os, glob

files = glob.glob("sources-rf/dataset/per_project/*.csv")
df = pd.concat([pd.read_csv(f) for f in files], ignore_index=True)
df.to_csv("sources-rf/dataset/per_project_merged.csv", index=False)
```

**Kết quả sau gộp:**
| Chỉ số | Giá trị |
|---|---|
| Tổng hàng (test files) | 5.213 |
| Tổng dự án | 59 |
| Tổng cột | 46 (4 metadata + 41 features + 1 nhãn) |

Cột `App` trong mỗi CSV ghi tên dự án (`owner/repo`, ví dụ `"Adobe-Consulting-Services/acs-aem-commons"`), cho phép phân biệt các hàng đến từ dự án nào sau khi gộp.

---

## 3. Đặc Trưng (Features)

Sau khi loại bỏ các cột metadata (`App`, `SHA`, `Tag`, `TestFilePath`) và nhãn (`isRefactored`), còn lại **41 đặc trưng** chia thành 3 nhóm:

### 3.1 Mùi Kiểm Thử — Test Smells (21 đặc trưng)

Được phát hiện tự động bằng công cụ **tsDetect**. Mỗi đặc trưng là số nguyên đếm số lần xuất hiện của một mẫu lỗi trong test file.

| Tên đặc trưng | Ý nghĩa |
|---|---|
| Assertion Roulette | Nhiều lệnh `assert` không có thông điệp mô tả → khó biết cái nào fail |
| Magic Number Test | Dùng hằng số số học trực tiếp (ví dụ `assertEquals(42, result)`) |
| Verbose Test | Phương thức test quá dài, vi phạm nguyên tắc "one concern per test" |
| Eager Test | Một test gọi nhiều phương thức sản xuất khác nhau cùng lúc |
| Lazy Test | Nhiều test method gọi đúng một phương thức sản xuất duy nhất |
| Sleepy Test | Dùng `Thread.sleep()` — tạo ra test chậm và không ổn định |
| Unknown Test | Test method không có bất kỳ lệnh `assert` nào (không kiểm tra gì) |
| Empty Test | Test method hoàn toàn rỗng (không có câu lệnh) |
| Ignored Test | Được chú thích `@Ignore` hoặc `@Disabled` |
| Exception Catching Throwing | Dùng `try/catch` thay vì `assertThrows` |
| Dependent Test | Phụ thuộc vào thứ tự chạy của các test khác |
| General Fixture | `setUp()` khởi tạo các trường không được dùng trong mọi test |
| Mystery Guest | Test đọc/ghi file ngoài, DB — tạo ra dependency ẩn |
| Print Statement | Có lệnh `System.out.println()` trong test |
| Redundant Assertion | Assert luôn luôn đúng (`assertTrue(true)`) |
| Sensitive Equality | So sánh bằng qua `toString()` thay vì `equals()` |
| Conditional Test Logic | Dùng `if/for/switch` trong thân test |
| Constructor Initialization | Thiết lập fixture trong constructor thay vì `setUp()` |
| Default Test | Tên lớp test được IDE tự sinh (ví dụ `TestCase1`) |
| Resource Optimism | Dùng tài nguyên ngoài mà không kiểm tra sự tồn tại trước |
| Duplicate Assert | Lặp lại cùng một lệnh `assert` trong một test method |

### 3.2 Chỉ Số Mã Nguồn — Code Metrics (7 đặc trưng)

Đo cấu trúc tĩnh của file test, không cần chạy code.

| Tên | Ý nghĩa |
|---|---|
| `NumberOfMethods` | Tổng số phương thức (bao gồm cả non-test) |
| `LOC` | Số dòng mã (Lines of Code) |
| `NOM` | Number of Methods (chỉ số CK — Chidamber & Kemerer) |
| `WMC` | Weighted Methods per Class — tổng độ phức tạp cyclomatic của tất cả phương thức |
| `RFC` | Response for a Class — số phương thức có thể được gọi khi nhận một message |
| `NAs` | Number of Attributes — số trường/thuộc tính trong class |
| `AsD` | Average Statement Depth — độ sâu câu lệnh trung bình (lồng ghép if/for) |

### 3.3 Chỉ Số Lịch Sử Git — Git Metrics (13 đặc trưng)

Được thu thập từ GitHub REST API bằng cách truy vấn lịch sử commit của từng test file. Đây là nhóm đặc trưng **bổ sung so với nghiên cứu gốc** — không có sẵn trong dataset của Martins.

| Tên | Ý nghĩa |
|---|---|
| `commits` | Tổng số commit đã chạm vào file này |
| `Contributors` | Số developer duy nhất đã commit vào file |
| `minorContributors` | Số contributor chiếm ít hơn 5% tổng commit |
| `contributorsExperience` | Kinh nghiệm tổng hợp của các contributor (điểm tổng hợp) |
| `codeChurnTotal` | Tổng số dòng thay đổi (thêm + xóa) qua tất cả commit |
| `codeChurnMax` | Số dòng thay đổi lớn nhất trong một commit duy nhất |
| `codeChurnAvg` | Trung bình số dòng thay đổi mỗi commit |
| `addLinesCount` | Tổng số dòng được thêm vào qua tất cả commit |
| `addLinesMax` | Số dòng thêm nhiều nhất trong một commit duy nhất |
| `addLinesAvg` | Trung bình số dòng thêm mỗi commit |
| `removedLinesCount` | Tổng số dòng bị xóa qua tất cả commit |
| `removedLinesMax` | Số dòng xóa nhiều nhất trong một commit duy nhất |
| `removedLinesAvg` | Trung bình số dòng xóa mỗi commit |

---

## 4. Nhãn (Label)

Chỉ có **một nhãn nhị phân**: `isRefactored`

| Giá trị | Ý nghĩa |
|---|---|
| `1` (True) | Test file này đã trải qua ít nhất một thao tác tái cấu trúc đặc thù cho test trong lịch sử commit của dự án |
| `0` (False) | Test file này **không** được tái cấu trúc trong giai đoạn nghiên cứu |

**Phân bố lớp:** mất cân bằng — lớp dương (isRefactored = 1) chiếm khoảng **30%** tổng số mẫu, lớp âm chiếm 70%. Đây là điều thường thấy trong bài toán dự đoán chất lượng phần mềm.

**Cách nhãn được xác định:** Martins sử dụng công cụ phát hiện tái cấu trúc (RefactoringMiner hoặc tương tự) trên lịch sử commit của từng dự án. Nếu file test xuất hiện trong ít nhất một commit được xác định là "refactoring commit" → nhãn = 1.

---

## 5. Xử Lý Dấu Thời Gian Commit

Dataset gốc của Martins **không có** thông tin ngày commit. Để thực hiện phân chia theo thời gian, chúng tôi thu thập thêm:

1. Mỗi hàng có cột `SHA` (mã hash của commit gần nhất).
2. Truy vấn GitHub API: `GET /repos/{owner}/{repo}/commits/{sha}` → lấy `committer.date`.
3. Tổng cộng 1.105 SHA duy nhất; 1.077 có timestamp thực (2 dự án không thể truy cập).
4. Hai dự án không truy cập được (`groupon/DotCi` — HTTP 451 và `salesforce/Argus` — HTTP 404) được gán **ngày sentinel `1970-01-01`**, đảm bảo luôn nằm trong tập huấn luyện.

---

## 6. Sự Khác Biệt: Nghiên Cứu Gốc vs Phương Pháp Hiện Tại

### Tóm tắt nhanh

| Khía cạnh | Nghiên cứu gốc (Martins 2024) | Phương pháp hiện tại |
|---|---|---|
| **Phạm vi** | Trong dự án (within-project) | Liên dự án (cross-project) |
| **Chiến lược chia dữ liệu** | Walk-forward theo commit/release | Phân vị thứ 80 của ngày commit (14/02/2022) |
| **Đơn vị fold** | Mỗi commit = một fold | Một fold duy nhất, cố định |
| **Bộ phân loại** | 8 (LR, RF, SVM, NB, DT, ET, KNN, MLP) | 11 (thêm AdaBoost, GB, XGBoost) |
| **Lựa chọn đặc trưng** | Mutual Information > 0.01 mỗi fold | Không lọc, dùng đủ 41 đặc trưng |
| **Cân bằng dữ liệu** | Nhiều phương pháp (SMOTE, ADASYN, ...) | SMOTE duy nhất, chỉ trên tập train |
| **Đánh giá phương sai** | Không (một lần chạy) | 5 lần chạy độc lập (seed khác nhau) |
| **Chỉ số Git** | Có sẵn trong dataset | Tự thu thập thêm qua GitHub API |
| **Kết quả tốt nhất** | Macro-F1 = 0,439 (Logistic Regression; quy F1 không xác định về 0) | F1 = 0,637 (XGBoost) |

### 6.1 Cách Tiếp Cận Gốc: Walk-Forward Trong Dự Án

```
Dự án A
─────────────────────────────────────────────────────
Commit 1 → Commit 2 → Commit 3 → ... → Commit N

Fold 1:  Train=[C1]              Test=[C2]
Fold 2:  Train=[C1, C2]          Test=[C3]
Fold 3:  Train=[C1, C2, C3]      Test=[C4]
...
```

Mỗi dự án được xử lý **độc lập**. Mô hình được huấn luyện và đánh giá trên cùng một dự án. Ở mỗi fold, chỉ dùng các commit trong quá khứ để dự đoán commit kế tiếp.

**Ưu điểm:** Tôn trọng thứ tự thời gian, mô hình học được đặc thù của từng dự án.

**Hạn chế:** Không thể áp dụng cho dự án mới chưa có dữ liệu lịch sử có nhãn. Khi dự án còn ít commit (fold đầu), tập train quá nhỏ để học hiệu quả. Mỗi dự án chạy riêng → không tận dụng được thông tin từ các dự án khác.

Pipeline gốc còn có thêm bước lọc đặc trưng bằng **Mutual Information** ở mỗi fold (loại bỏ feature có MI < 0,01), và hỗ trợ nhiều phương pháp cân bằng lớp có thể cấu hình qua command line.

### 6.2 Phương Pháp Hiện Tại: Cross-Project Temporal Split

```
56 dự án (train)         35 dự án (test)
──────────────────────── | ─────────────────────────
Commit dates < 14/02/2022 | Commit dates ≥ 14/02/2022
4.172 test files          | 1.041 test files
```

Toàn bộ 59 dự án được gộp thành một dataset duy nhất. Ngưỡng cắt thời gian được tính là phân vị thứ 80 của ngày commit (14/02/2022). Mô hình được huấn luyện trên tất cả dữ liệu trước ngưỡng, đánh giá trên tất cả dữ liệu sau ngưỡng — **bao gồm các dự án hoàn toàn mới chưa từng thấy khi train**.

Để đo phương sai (mức độ ổn định của kết quả), thực nghiệm được lặp lại **5 lần** với các hạt giống ngẫu nhiên (seed) khác nhau cho SMOTE và khởi tạo mô hình. Phân chia train/test giữ nguyên, chỉ phần stochastic thay đổi.

**Ưu điểm:** Đánh giá khả năng tổng quát hóa thực sự (mô hình không thấy dự án test lúc train). Phản ánh tình huống thực tiễn khi triển khai công cụ cho dự án mới. Khai thác được toàn bộ dữ liệu từ nhiều dự án.

**Hạn chế:** Không học được đặc thù riêng của từng dự án. Dữ liệu từ các dự án khác nhau có thể có phân bố mùi kiểm thử khác nhau, gây khó khăn cho việc chuyển giao.

### 6.3 Tại Sao Kết Quả Liên Dự Án Lại Cao Hơn?

Điều này có vẻ nghịch lý — cross-project (khó hơn về mặt lý thuyết) lại cho F1 cao hơn. Có hai lý do chính:

1. **Tập train lớn hơn nhiều:** 4.172 mẫu từ 56 dự án thay vì vài chục mẫu từ một dự án duy nhất ở các fold đầu của walk-forward.
2. **Chỉ số Git là tín hiệu mạnh và phổ quát:** `removedLinesCount`, `codeChurnTotal` — các file bị xóa nhiều dòng là ứng viên tái cấu trúc bất kể dự án nào. Tín hiệu này không phụ thuộc vào coding style của từng team, nên chuyển giao tốt.

---

## 7. Giá Trị Bị Thiếu

866 giá trị thiếu trong tập feature (chủ yếu là các chỉ số Git — file chỉ có một commit nên không tính được max/avg). Tất cả được điền bằng **0**, nhất quán với ngữ nghĩa "không có hoạt động" của các đặc trưng đếm số lần.
