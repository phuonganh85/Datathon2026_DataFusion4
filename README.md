#  DATATHON 2026 — Revenue Forecast

Dự báo doanh thu và chi phí (Revenue & COGS) cho giai đoạn **2023-01-01 → 2024-07-01** sử dụng LightGBM kết hợp recursive forecasting, với dữ liệu lịch sử từ 2012.

---

## 🗂️ Cấu trúc dự án

```
project/
├── data/
│   └── processed/
│       ├── sales.csv              # Doanh thu lịch sử theo ngày
│       ├── sample_submission.csv  # Template nộp bài
│       ├── web_traffic.csv        # Dữ liệu lưu lượng web (tuỳ chọn)
│       ├── orders.csv             # Thông tin đơn hàng (tuỳ chọn)
│       ├── order_items.csv        # Chi tiết từng mặt hàng (tuỳ chọn)
│       ├── promotions.csv         # Dữ liệu khuyến mãi (tuỳ chọn)
│       └── inventory.csv          # Tồn kho (tuỳ chọn)
├── mle.ipynb                      # Notebook chính
└── submission.csv                 # Output: file nộp bài
```

---

##  Yêu cầu cài đặt

```bash
pip install lightgbm shap optuna seaborn lunardate scikit-learn pandas numpy matplotlib
```

---

##  Pipeline tổng quan

| Bước | Mô tả |
|------|-------|
| **0. Setup** | Import thư viện, cấu hình đường dẫn, khoảng thời gian train/test |
| **1. Load Data** | Đọc `sales.csv`, `sample_submission.csv` và các nguồn dữ liệu bổ sung |
| **2. Daily Aggregates** | Tổng hợp web traffic, orders, order items, promotions, inventory theo ngày |
| **3. Merge Master Frame** | Ghép tất cả nguồn dữ liệu vào một bảng duy nhất theo `Date` |
| **4. Correlation Heatmap** | Phân tích tương quan giữa các feature với `log_revenue` để định hướng feature engineering |
| **5. Feature Engineering** | Tạo đặc trưng thời gian, sự kiện đặc biệt, lag, rolling statistics, Fourier |
| **6. Build Training Dataset** | Chuẩn bị `X` và `y`, tính `COGS_RATIO` |
| **7. Baseline Ridge** | Mô hình Ridge cơ bản để đánh giá benchmark |
| **8. Cross-Validation** | TimeSeriesSplit 5-fold để đánh giá LightGBM |
| **9. Learning Curve** | Chẩn đoán bias-variance tradeoff |
| **10. Optuna Tuning** | Tối ưu siêu tham số với 50 trials |
| **11. Final Model** | Train lại toàn bộ dữ liệu, tính bias correction |
| **12. Recursive Forecasting** | Dự báo từng ngày cho kỳ test |
| **13. SHAP + Validation Plots** | Giải thích mô hình và trực quan hoá kết quả |
| **14. Submission** | Xuất `submission.csv` |

---

##  Chi tiết kỹ thuật

### Feature Engineering

**Calendar features:**
- `dayofweek`, `month`, `day`, `quarter`, `is_weekend`, `is_month_end`
- `is_vn_holiday` — các ngày lễ cố định của Việt Nam (1/1, 30/4, 1/5, 2/9)
- `day_in_month_norm` — vị trí trong tháng được chuẩn hoá

**Trend:**
- `year_idx`, `year_idx_sq` — xu hướng tuyến tính và bậc hai

**Sự kiện đặc biệt (Vietnamese context):**
- `days_to_tet`, `days_since_tet`, `is_pre_tet`, `is_post_tet` — proximity với Tết Âm lịch
- `days_to_next_mega`, `days_since_mega`, `is_mega_sale` — các ngày sale lớn (9/9, 10/10, 11/11, 12/12)

**Lag & Rolling:**
- Lag: 1, 7, 364, 365 ngày (lag 1 và 7 bị loại khỏi recursive forecast để tránh lookahead)
- Rolling mean: 7, 30 ngày; `rev_momentum_7_30`

**Fourier:**
- 3 cặp sin/cos để mô hình hoá chu kỳ năm

**External features (nếu có dữ liệu):**
- Web traffic, orders, order items — dùng `lag1` và `roll7` để tránh lookahead bias
- Promotions — `promo_count`, `promo_max_discount`, `promo_intensity`, `has_promo_today`
- Interaction: `weekend_x_promo`, `pretet_x_promo`

### Mô hình

- **Baseline:** Ridge Regression với StandardScaler
- **Chính:** LightGBM với `objective='huber'` (robust với outliers)
- **Tuning:** Optuna TPE, 50 trials, 3 fold cuối của TimeSeriesSplit
- **Bias correction:** Hiệu chỉnh offset từ fold validation gần nhất với kỳ test

### Dự báo (Recursive Forecasting)

Dự báo từng ngày một theo thứ tự thời gian. Giá trị đã dự báo được cập nhật vào `known_log` và dùng làm lag input cho các ngày tiếp theo. COGS được tính theo tỷ lệ trung vị `COGS/Revenue` từ tập train.

---

##  Kết quả

| Metric | Mô hình |
|--------|---------|
| Cross-validation | 5-fold TimeSeriesSplit trên tập train |
| Đánh giá | MAE, RMSE, R² |
| Bias correction | Áp dụng offset từ fold cuối |

Kết quả cụ thể được in ra console sau mỗi bước và hiển thị qua biểu đồ validation.

---

##  Output

File `submission.csv` gồm 3 cột:

```
Date,Revenue,COGS
2023-01-01,xxxxx,xxxxx
...
```

---

