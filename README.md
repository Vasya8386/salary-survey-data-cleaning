# Salary Survey — Data Cleaning Project

> **Môn:** Data Handling & Visualization — Bài 02  
> **Dataset:** Ask a Manager Salary Survey (2021)

---

## Mô tả

Repo này chứa toàn bộ quá trình làm sạch dataset lương thực tế (2,800 phản hồi) từ khảo sát Ask a Manager 2021. Bao gồm xử lý missing values, duplicate, outlier, chuẩn hóa chuỗi, encoding categorical và feature engineering.

## Cấu trúc repo

```
├── salary_cleaning.ipynb       # Notebook xử lý chính — tất cả cell đã chạy
├── data_quality_report.md      # Báo cáo 5 phần theo cấu trúc handout
├── salary_survey_raw.csv       # Dataset gốc (input)
├── salary_cleaned.csv          # Dataset sau khi xử lý (output)
└── README.md                   # File này
```

## Cách chạy

```bash
# 1. Clone repository:
git clone https://github.com/YOUR_USERNAME/salary-survey-data-cleaning.git

# 2. Cài thư viện cần thiết
pip install pandas numpy matplotlib scikit-learn rapidfuzz

# 3. Mở notebook
jupyter notebook salary_cleaning.ipynb

# 4. Run All Cells (Kernel > Restart & Run All)
```

> **Lưu ý:** Đặt `salary_survey_raw.csv` cùng thư mục với notebook trước khi chạy.

## Tóm tắt các vấn đề chính đã xử lý

| Vấn đề | Giải pháp |
|--------|-----------|
| 38 duplicate toàn hàng | `drop_duplicates(keep='first')` |
| `annual_salary` lưu dạng string có `,` và `$` | Strip → remove ký tự → `pd.to_numeric()` |
| `timestamp` có 4+ format khác nhau | `pd.to_datetime(format='mixed', errors='coerce')` |
| `gender`: 4 cách viết cho 3 giá trị thực | Lowercase → mapping dict |
| `country`: US/USA/United States/United states | Mapping dict chuẩn hóa |
| 391 null `annual_salary` (MNAR) | Giữ null + flag `salary_was_missing` |
| 1,538 null `additional_monetary_comp` | `fillna(0)` — null = không có bonus |
| Outlier salary = 1,200,000 (×3) | Winsorize tại 1st–99th percentile |
| Industry: 40+ unique values | Target Encoding thay vì OHE |
| Education, experience: ordinal | Ordinal Encoding (1–6 và 1–7) |

## Features mới được tạo

| Feature | Mô tả | Câu hỏi trả lời |
|---------|-------|----------------|
| `total_comp` | `annual_salary` + `additional_monetary_comp` | Tổng thu nhập thực tế là bao nhiêu? |
| `salary_per_exp_year` | Lương / số năm kinh nghiệm | Ai đang được trả lương "xứng đáng" hơn theo kinh nghiệm? |
| `age_group` | Gen Z / Millennial / Gen X / Boomer | Pattern lương theo thế hệ? |
| `education_level` | Ordinal 1–6 từ High School → PhD | Học vấn cao hơn có tương quan với lương cao hơn? |
| `exp_in_field_ord` | Ordinal 1–7 từ <1 năm → 41+ năm | Kinh nghiệm ảnh hưởng lương thế nào? |
| `industry_target_enc` | Mean salary theo industry | Ngành nào trả lương cao nhất? |
| `salary_was_missing` | Flag 0/1 | Nhóm không khai lương có đặc điểm gì khác? |
| `survey_month` | Tháng gửi khảo sát | Phân bố thời gian của survey? |

## Kết quả
- **Input:** 2,800 hàng × 17 cột
- **Output:** 2,751 hàng × 38 cột
- **Pipeline:** Hàm `clean_data(df)` — tái sử dụng được, có docstring và unit tests
