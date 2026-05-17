# Data Quality Report — Salary Survey 2021

> **Dataset:** Ask a Manager Salary Survey (2021)  
> **File gốc:** `salary_survey_raw.csv`  

---

## 1. Tổng quan Dataset Gốc

### 1.1 Thống kê cơ bản

| Chỉ số | Giá trị |
|--------|---------|
| Số hàng (raw) | 2,800 |
| Số cột | 17 |
| Duplicate toàn hàng | 38 (1.4%) |
| Tổng null (tất cả cột) | 9,501 |
| Cột có null | 9 / 17 |

### 1.2 Bảng so sánh Trước / Sau xử lý

| Chỉ số | Trước | Sau | Ghi chú |
|--------|-------|-----|---------|
| Số hàng | 2,800 | **2,762** | Xóa 38 duplicate |
| Số cột | 17 | **28** | Thêm 11 cột mới (encoding + features) |
| Duplicate | 38 | **0** | |
| Null `annual_salary` | 391 (14%) | **391** (có flag) | MNAR — giữ, thêm `salary_was_missing` |
| Null `additional_monetary_comp` | 1,538 (55%) | **0** | Điền 0 (null = không có khoản này) |
| Null `us_state` | 1,813 (65%) | **0** | Điền 'N/A' |
| Null `city` | 1,562 (56%) | **0** | Điền 'Unknown' |
| Null `income_context` | 2,269 (81%) | **0** | Điền 'none' (free-text hợp lệ) |
| Null `race` | 766 (27%) | **766** | MNAR — giữ nguyên |
| dtype `annual_salary` | object | **float64** | |
| dtype `timestamp` | object | **datetime64** | |
| dtype `currency` | object | **category** | Tiết kiệm ~60% bộ nhớ |

---

## 2. Vấn đề Phát hiện

| # | Tên vấn đề | Cột bị ảnh hưởng | Mức độ | Ví dụ cụ thể |
|---|------------|------------------|--------|--------------|
| 1 | **Duplicate toàn hàng** | Tất cả | Critical | 38 hàng giống nhau 100% |
| 2 | **dtype sai — salary** | `annual_salary` | Critical | `"56,894"`, `"$81577"`, `" 144,543 "` |
| 3 | **dtype sai — timestamp** | `timestamp` | Minor | Ít nhất 4 format: `04/12/2021 16:11:47`, `2021-04-26 03:04`, `04/15/2021` |
| 4 | **Missing MNAR — salary** | `annual_salary` | Critical | 391 null (14%) — người không muốn khai |
| 5 | **Missing MNAR — race** | `race` | Minor | 766 null (27%) — nhóm thiểu số ngại khai |
| 6 | **Missing hợp lệ — free text** | `income_context`, `additional_context_on_job_title` | OK | 81% và 35% null — trường tùy chọn |
| 7 | **Chuỗi không nhất quán — gender** | `gender` | Critical | `"Woman"`, `"woman"`, `"Female"`, `"female"` = 4 cách viết cho 1 giá trị |
| 8 | **Chuỗi không nhất quán — country** | `country` | Critical | `"United States"`, `"US"`, `"USA"`, `"United states"` = 4 cách cho 1 quốc gia |
| 9 | **Outlier nghi ngờ** | `annual_salary` | Minor | 3 hàng `salary = 1,200,000` — round number, lặp lại |
| 10 | **Currency không nhất quán** | `currency` | Minor | `"USD"` vs `"usd"` vs `"USD "` (khoảng trắng) |

---

## 3. Quyết định Xử lý

### 3.1 Duplicate
- **Quyết định:** `drop_duplicates(keep='first')`
- **Phương án khác:** `keep='last'` — không phù hợp vì không có lý do để tin lần gửi cuối chính xác hơn.
- **Rủi ro nếu không xử lý:** Một số nhóm demographics bị overcount, lương trung bình bị lệch.

### 3.2 dtype `annual_salary`
- **Quyết định:** Strip → remove `,` và `$` → `pd.to_numeric(errors='coerce')`
- **Phương án khác:** Regex phức tạp hơn — không cần thiết, dữ liệu chỉ có 2 ký tự lạ.
- **Rủi ro nếu không xử lý:** `df['annual_salary'].mean()` lỗi hoặc trả về NaN toàn bộ.

### 3.3 Missing `additional_monetary_comp`
- **Quyết định:** `fillna(0)`
- **Lý do:** Đây là khoản tiền bổ sung (bonus, commission). Người không điền = không có khoản này, không phải bỏ qua câu hỏi. Đây là trường hợp **hiếm** mà `fillna(0)` là **đúng** về mặt ngữ nghĩa.
- **Rủi ro:** Nếu một số người thực sự bỏ qua câu hỏi dù có bonus → underestimate total comp.

### 3.4 Missing `annual_salary` (MNAR)
- **Quyết định:** Giữ null + thêm cột flag `salary_was_missing`
- **Lý do:** MNAR — người không khai lương có thể là nhóm lương rất cao (muốn giữ bí mật) hoặc rất thấp (ngại so sánh). Impute bằng median sẽ kéo các hàng này về trung tâm, tạo bias hệ thống.
- **Rủi ro của quyết định này:** Mất 391 hàng trong bất kỳ phân tích nào cần salary.

### 3.5 Outlier `annual_salary`
- **Quyết định:** Winsorize tại 1st–99th percentile
- **Phương án khác:** Drop hàng outlier — mất ~2–3% dữ liệu, có thể xóa CEO hợp lệ.
- **Rủi ro:** Nén giá trị thực của nhóm thu nhập cao nhất.

### 3.6 Currency normalization
- **Quyết định:** `str.upper().strip()` → `astype('category')`
- **Lý do:** `"usd"` và `"USD"` là cùng currency; category dtype tiết kiệm ~60% bộ nhớ.

### 3.7 Industry encoding
- **Quyết định:** Target Encoding thay vì OHE
- **Lý do:** Industry có 40+ giá trị unique. OHE tạo 39 cột dummies, trong đó phần lớn sparse. Target Encoding (mean salary theo industry) giữ thông tin trong 1 cột duy nhất.
- **Rủi ro:** Data leakage nếu dùng trong ML pipeline — phải fit trên training set.

---

## 4. Quyết định Khó (Ambiguous)

### Vấn đề: `annual_salary` — Nên impute hay giữ null?

**Quan sát:** 391 hàng (14%) thiếu `annual_salary` — đây là biến mục tiêu chính của phân tích.

**Phân tích loại null:**
Khả năng cao là **MNAR** (Missing Not At Random):
- Người thu nhập rất cao có thể không muốn khai vì lo ngại so sánh.
- Người thu nhập thấp có thể ngại khai vì mặc cảm.
- Nếu MCAR (hoàn toàn ngẫu nhiên) → impute an toàn. Nhưng không có bằng chứng để khẳng định điều này.

**Phương án 1: Drop các hàng null**
- Ưu: Phân tích sạch, không cần giả định gì.
- Nhược: Mất 14% dữ liệu; nếu MNAR thì kết quả phân tích chỉ đại diện cho người sẵn sàng chia sẻ lương — **selection bias**.

**Phương án 2: Impute bằng median theo industry + education**
- Ưu: Giữ được 391 hàng, ước tính có điều kiện tốt hơn global median.
- Nhược: Nếu MNAR → impute kéo nhóm lương cao/thấp về trung bình → tạo bias hệ thống; kết quả phân tích lương trung bình sẽ bị underestimate ở đuôi phân phối.

**Quyết định chọn:** Giữ null + thêm flag `salary_was_missing = 1`

**Lý do:** Phân tích chính của dataset là *so sánh lương theo ngành và kinh nghiệm*. Nếu impute và không ghi rõ, người đọc report sẽ hiểu nhầm 100% dữ liệu là thực. Việc giữ null và có flag cho phép người dùng downstream tự quyết định: loại bỏ hàng thiếu hoặc xử lý riêng cho use case của họ. Đây là quyết định **bảo toàn tính trung thực** của dữ liệu.

---

## 5. Hạn chế Còn lại

Sau tất cả các bước xử lý, dataset vẫn có những vấn đề chưa giải quyết hoàn toàn:

| Hạn chế | Ảnh hưởng đến phân tích |
|---------|------------------------|
| **14% null `annual_salary`** (MNAR) | Phân tích lương trung bình có thể bị bias — nhóm lương cực cao/thấp underrepresented |
| **27% null `race`** (MNAR) | Không thể phân tích lương theo chủng tộc một cách tin cậy |
| **Currency không đồng nhất** | Dataset trộn lẫn USD/CAD/GBP/AUD — so sánh lương tuyệt đối giữa các quốc gia sẽ sai nếu không quy đổi tỷ giá |
| **Job title free-text** | Dù đã chuẩn hóa, vẫn còn hàng trăm job title unique. Phân tích theo nghề đòi hỏi gom nhóm kỹ hơn (e.g., NLP clustering) |
| **Self-reported data** | Không có cách xác minh salary; người dùng có thể khai sai (inflate hoặc deflate) |
| **Target encoding leakage** | `industry_target_enc` được tính trên toàn bộ dataset — nếu dùng trong ML model cần recalculate trên training set |
| **Timestamp không đủ granularity** | Một số hàng chỉ có ngày (không có giờ) → phân tích theo giờ trong ngày không đầy đủ |

---