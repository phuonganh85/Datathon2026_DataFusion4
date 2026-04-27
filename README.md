# 📈 DATATHON 2026 — Revenue Forecast

Dự báo doanh thu sử dụng LightGBM với Optuna tuning, được xây dựng cho cuộc thi **Datathon 2026 — DataFusion**.

---

## 🗂️ Cấu trúc Pipeline

| Bước | Mô tả |
|------|-------|
| 0. Setup | Import thư viện, cấu hình seed, khai báo đường dẫn & khoảng thời gian |
| 1. Load Data | Đọc `sales.csv`và các nguồn dữ liệu bổ sung |
| 2. External Aggregates | Tổng hợp daily từ web traffic, orders, order items, promotions, inventory |
| 3. Merge Master Frame | Gộp tất cả nguồn theo cột `Date` thành một DataFrame duy nhất |
| 4. EDA & Correlation | Phân tích tương quan với `log_revenue`, vẽ heatmap |
| 5. Feature Engineering | Hàm `build_features()` v3 — tạo lag, rolling, calendar, Tết, interaction features |
| 6. Build Training Dataset | Loại bỏ short-lag leakage, tính `COGS_RATIO` median |
| 7. Baseline Ridge | Ridge Regression chuẩn hóa làm benchmark |
| 8. Cross-Validation LightGBM | TimeSeriesSplit 5-fold với early stopping |
| 9. Optuna Tuning | Tối ưu siêu tham số LightGBM (50 trials) |
| 10. Best Params | Hiển thị kết quả Optuna |
| 11. Final Model | Retrain toàn bộ dữ liệu + Bias Correction từ 3 fold cuối |
| 12. Test Inference | Tạo features cho tập test (2023-01-01 → 2024-07-01), dự báo Revenue & COGS |
| 13. SHAP + Validation Plots | SHAP summary, bar chart top-20 features, biểu đồ thực tế vs dự báo |
| 14. Submission | Xuất file `submission.csv` |

---

## 📂 Dữ liệu đầu vào

Tất cả file được đặt tại `D:\Datathon2026_DataFusion4\data\processed\`

| File | Nội dung |
|------|---------|
| `sales.csv` | Doanh thu & COGS hàng ngày (train: 2012-07-04 → 2022-12-31) |
| `sample_submission.csv` | Template nộp bài (test: 2023-01-01 → 2024-07-01) |
| `web_traffic.csv` | Sessions, unique visitors, page views, bounce rate |
| `orders.csv` | Đơn hàng theo ngày, trạng thái, thiết bị |
| `order_items.csv` | Chi tiết sản phẩm, số lượng, giá, discount, promo |
| `promotions.csv` | Chương trình khuyến mãi (ngày bắt đầu/kết thúc, discount) |
| `inventory.csv` | Days of supply, fill rate theo ngày |

---

## 🔧 Feature Engineering

Hàm `build_features()` tạo **67 features** gồm:

- **Calendar:** `day_of_week`, `month`, `quarter`, `day_of_year`, `is_weekend`, `is_month_end/start`
- **Vietnamese Lunar Calendar:** `lunar_month`, `lunar_day`, `is_tet_week`, `is_pre_tet`, `is_post_tet`
- **Lag features:** `rev_lag_7`, `rev_lag_14`, `rev_lag_30`, `rev_lag_364`, `rev_lag_365`
- **Rolling statistics:** `rev_roll_mean_7/30/90`, `rev_roll_std_30`, `rev_roll_max_30`
- **Momentum:** `rev_momentum_7_30`
- **Fourier features:** sin/cos của chu kỳ 7 và 365.25 ngày
- **Interaction:** `weekend_x_promo`, `pretet_x_promo`
- **External signals:** web traffic, orders, promotions, inventory (lag 1, 3, roll 7)

---

## 🤖 Mô hình
### LightGBM (Final)
- **Objective:** Huber loss (α=0.5)
- **Tuning:** Optuna 50 trials, TimeSeriesSplit 5-fold
- **CV kết quả:**

| Fold | MAE | RMSE | R² |
|------|-----|------|----|
| 1 | 647,124 | 1,091,682 | 0.867 |
| 2 | 397,259 | 615,276 | 0.950 |
| 3 | 379,085 | 607,507 | 0.953 |
| 4 | 336,176 | 432,742 | 0.937 |
| 5 | 292,911 | 405,083 | 0.932 |
| **Avg** | **410,511 ± 123,697** | **630,458 ± 246,321** | — |

- **Validation (final):** MAE = 174,430 | RMSE = 235,723 | **R² = 0.9771**
- **Bias Correction:** +165,165 (trung bình 3 fold cuối)

### COGS Prediction
- `COGS = Revenue × COGS_RATIO` (median từ train set: **0.8264**)

---

## 📦 Thư viện yêu cầu

```
numpy
pandas
lightgbm
shap
optuna
seaborn
matplotlib
lunardate
scikit-learn
```

Cài đặt nhanh:
```bash
pip install numpy pandas lightgbm shap optuna seaborn matplotlib lunardate scikit-learn
```

---

## ▶️ Cách chạy

1. Đặt các file CSV vào `D:\Datathon2026_DataFusion4\data\processed\`
2. Mở `mle.ipynb` bằng Jupyter Notebook hoặc VS Code
3. Chạy tuần tự từ đầu đến cuối (`Run All`)
4. File `submission.csv` sẽ được lưu tại thư mục làm việc hiện tại

---

## 📤 Output

File `submission.csv` có định dạng:

```
Date,Revenue,COGS
2023-01-01,2672084.0,2208133.0
2023-01-02,2604463.0,2152252.0
...
```

---

## 📝 Ghi chú

- Random seed cố định: `SEED = 42`
- Target được transform bằng `log1p` trước khi huấn luyện, sau đó `expm1` để trả về giá trị thực
- Đảm bảo **không có data leakage**: `sales['Date'].max() < sample_sub['Date'].min()` được kiểm tra tự động
- Short-lag features (`rev_lag_1`, `rev_lag_7`, v.v.) bị loại khỏi feature set để tránh leakage trong inference
