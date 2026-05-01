# Datathon 2026 — The Gridbreaker
## Team DataFusion4

> **Phân tích dữ liệu và Dự báo Doanh thu Thương mại Điện tử Thời trang Việt Nam**

| Thành viên | Vai trò |
|---|---|
| Phạm Vũ Phương Anh | Team Lead, EDA & Báo cáo |
| Trần Minh Hoàng | Data Engineering & Pipeline |
| Phạm Ngọc Thế Kiều | Machine Learning & Forecasting |
| Phạm Huỳnh Gia Huy | Machine Learning & Visualization |

---

## Kết quả chính

| Chỉ số | Giá trị |
|---|---|
| Revenue R² (fold 2022) | **0.797** |
| COGS R² (fold 2022) | **0.801** |
| Revenue MAE | 561,863 VNĐ/ngày |
| COGS MAE | 482,240 VNĐ/ngày |
| Lost revenue xác định | ~0.6 tỷ VNĐ (2012–2022) |

---

## Cấu trúc thư mục

```
Datathon2026_DataFusion4/
│
├── data/
│   ├── raw/                        # Dữ liệu gốc (14 file CSV)
│   └── processed/                  # Dữ liệu đã xử lý (xuất từ Phần 2.1)
│
├── figures/                        # Toàn bộ hình vẽ từ EDA (xuất từ Phần 2.2)
│
├── notebooks/                      # Toàn bộ mã nguồn
│   ├── Data_Exploration.ipynb      # Khám phá dữ liệu ban đầu
│   ├── Phan1_TracNghiem.ipynb      # Phần 1: 10 câu trắc nghiệm
│   ├── Phan2.1_Clean&merge.ipynb   # Phần 2.1: Xử lý & làm sạch dữ liệu
│   ├── Phan2.2_ExploratoryDataAnalysis.ipynb  # Phần 2.2: EDA 4 tầng
│   └── Phan3_MachineLearning.ipynb # Phần 3: Mô hình dự báo
│
├── results/
│   └── submission.csv              # File nộp Kaggle (Date, Revenue, COGS)
│
├── .gitignore
└── README.md                       # Mô tả cấu trúc thư mục và hướng dẫn chạy lại kết quả.
```

---

## Hướng dẫn chạy lại kết quả

### Yêu cầu môi trường

```bash
pip install -r requirements.txt
```

Hoặc cài thủ công các thư viện chính:

```bash
pip install pandas numpy scikit-learn lightgbm xgboost optuna shap \
            matplotlib seaborn squarify lunardate pyarrow
```

> **Python**: 3.9+ được khuyến nghị

---

### Bước 1 — Xử lý dữ liệu (Phần 2.1)

**Notebook:** `notebooks/Phan2.1_Clean&merge.ipynb`

Chạy toàn bộ các cell theo thứ tự. Notebook sẽ:
- Load 14 file CSV gốc từ `data/raw/`
- Làm sạch, chuẩn hóa kiểu dữ liệu và join thành 4 bộ Master Dataset
- Xuất 4 file `.parquet` vào `data/processed/`

**Output:** `sales_master.parquet`, `ops_master.parquet`, `ops_sales.parquet`, `customer_master.parquet`

---

### Bước 2 — Phân tích EDA (Phần 2.2)

**Notebook:** `notebooks/Phan2.2_ExploratoryDataAnalysis.ipynb`

Chạy toàn bộ các cell theo thứ tự. Notebook sẽ:
- Load 4 file master từ `data/processed/` (yêu cầu hoàn thành Bước 1)
- Thực hiện phân tích 4 tầng: Descriptive → Diagnostic → Predictive → Prescriptive
- Xuất toàn bộ hình vào thư mục `figures/`

**Output:** 13 file hình PNG trong `figures/`

---

### Bước 3 — Mô hình dự báo (Phần 3)

**Notebook:** `notebooks/Phan3_MachineLearning.ipynb`

Chạy toàn bộ các cell theo thứ tự. Notebook sẽ:
1. Load dữ liệu từ `data/processed/`
2. Xây dựng Baseline model (Multiplicative Decomposition)
3. Feature Engineering (66 đặc trưng)
4. Hyperparameter tuning với Optuna (50 trials/mô hình) — **có thể mất 15–30 phút**
5. Huấn luyện LightGBM + XGBoost Ensemble với Hill Climbing
6. Dự báo 2-Pass cho 2023 và 2024
7. Xuất `results/submission.csv`

> **Tái lập nhanh:** Nếu đã có `best_params.json`, notebook sẽ tự động load và bỏ qua bước Optuna tuning.

**Output:** `results/submission.csv`, `figures/Hình 3.*.png`, `best_params.json`

---

## 4 bộ Master Dataset

Dữ liệu thô (14 file CSV) được xử lý và hợp nhất thành **4 bộ Master Dataset** theo kiến trúc Star Schema, phục vụ cho cả EDA và Forecasting.

| Master Dataset | Grain | Nguồn bảng | Use case |
|---|---|---|---|
| `sales_master.parquet` | 1 dòng = 1 sản phẩm trong 1 đơn | order_items + products + orders + promotions | EDA đa chiều: category × size × segment × promo |
| `ops_master.parquet` | 1 dòng = 1 đơn hàng | orders + shipments + payments + returns(agg) + reviews(agg) + geography | Phân tích vận hành: giao hàng, hoàn tiền, đánh giá |
| `ops_sales.parquet` | 1 dòng = 1 đơn + sản phẩm đại diện | ops_master + sales_master(agg theo order_id) | Cross-domain: kết nối vận hành với sản phẩm |
| `customer_master.parquet` | 1 dòng = 1 khách hàng | customers + geography + orders(agg) + revenue(agg) | RFM segmentation, cohort retention, LTV |

**Quyết định thiết kế quan trọng:**
- `payments` **không** được join vào `sales_master` để tránh fan-out: một đơn có nhiều sản phẩm sẽ nhân đôi giá trị thanh toán theo số dòng sản phẩm.
- `reorder_flag` bị drop khỏi inventory (toàn bộ 60,247 dòng = 0, không có variance).
- 31,684 khách hàng chưa có đơn được giữ `NaN` thay vì `fillna(0)` để phân tích tỷ lệ kích hoạt.
- `signup_date` không đáng tin cậy: 73.8% đơn hàng có `order_date < signup_date` — ghi nhận bằng flag thay vì drop.

---

## Kiến trúc mô hình

```
Revenue(d) = Baseline(d)              +  Residual(d)
           = S(m,d) × g^Δy            +  LightGBM_Ensemble(features)
```

| Thành phần | Chi tiết |
|---|---|
| **Baseline** | Multiplicative decomposition: Seasonal profile × YoY geometric growth |
| **Mô hình ML** | LightGBM (objective=regression) + XGBoost (squarederror) |
| **Tuning** | Optuna TPE sampler, 50 trials, 2 fold cuối (2021, 2022) |
| **Cross-validation** | Expanding window theo năm: val = 2020, 2021, 2022 |
| **Ensemble** | Hill Climbing — composite metric 0.4×R² + 0.3×MAE + 0.3×RMSE |
| **Forecast** | 2-Pass: 2023 từ actuals 2022; 2024 từ predictions 2023 |
| **Features** | 66 features (train) / 62 features (test, bỏ rolling tránh error propagation) |

---

## CV Ablation — Quá trình cải thiện từng bước

Bảng so sánh kết quả CV expanding-window (val = 2021 và val = 2022) sau từng bước cải tiến, cho thấy đóng góp của từng thành phần trong pipeline:

| Approach | Rev MAE | Rev R² | COGS MAE | COGS R² |
|---|---|---|---|---|
| Baseline only (Seasonal × YoY) | 606,502 | 0.7310 | — | 0.7179 |
| + LightGBM (residual, chưa tuning) | ~640,000 | ~0.720 | — | ~0.610 |
| + Optuna tuning (50 trials, 2 fold) | 600,498 | 0.7477 | — | 0.6525 |
| + Lag/Rolling residual features | ~575,000 | ~0.780 | ~490,000 | ~0.790 |
| **+ Hill Climbing Ensemble (final)** | **561,863** | **0.7973** | **482,240** | **0.8011** |

> **Ghi chú:** Với COGS, Baseline only (R²=0.7179) ban đầu tốt hơn LightGBM đơn lẻ (R²=0.6525) — residual COGS khó học hơn do phụ thuộc vào yếu tố cung ứng nội bộ chưa được phản ánh trong features. Hill Climbing Ensemble vượt cả hai bằng cách kết hợp LGBM=0.897 / XGB=0.103.

---

## No-leakage Guarantees

Toàn bộ pipeline được thiết kế để đảm bảo không có data leakage từ tập test vào quá trình huấn luyện:

1. **Baseline được fit lại từ scratch trong mỗi fold** (`upper_year = y - 1`): dự báo năm Y chỉ dùng dữ liệu đến năm Y-1, không nhìn thấy tương lai.
2. **Lag/Rolling features dùng `shift(1)` trước khi tính rolling**: không sử dụng giá trị cùng ngày khi tính momentum.
3. **Web traffic dùng lag 365 ngày**: không dùng sessions cùng ngày vì không có trong tập test — lag 365 hoàn toàn từ dữ liệu lịch sử.
4. **Rolling features bị loại khỏi test features**: tập test dùng 62 features (bỏ rolling) thay vì 66 để tránh error propagation khi predictions ngày trước làm input cho ngày sau.
5. **Macro flags (Tết, MegaSale, ngày lễ VN) tính từ lịch cố định**: không có lookahead, xác định được trước mà không cần dữ liệu tương lai.
6. **2-Pass forecast**: Pass 1 (2023) dùng actuals 2022; Pass 2 (2024) dùng predictions 2023 — Revenue/COGS thực tế của tập test **không bao giờ** được đưa vào features.
7. **Random seed = 42**, CV splits xác định bởi năm (deterministic).

```python
# Kiểm tra tự động ngay khi load dữ liệu
assert sales['Date'].max() < sample_sub['Date'].min(), 'LEAKAGE!'
assert 'Revenue' not in REV_FEATS_TEST
assert 'COGS'    not in COGS_FEATS_TEST
```

---

## Phát hiện chính từ EDA

**Vòng lặp mất mát kép:**
- **3.1%** doanh thu thuần bị mất qua hoàn tiền, chủ yếu do `wrong_size` (32.6% tổng đơn trả)
- **0.6 tỷ VNĐ** doanh thu bị bỏ lỡ khi hết hàng đúng lúc traffic đạt đỉnh (2012–2022)
- Retention Tháng 1 chỉ đạt **1.7%** (benchmark ngành 20–30%)

**3 hành động ROI cao nhất:**

| Hành động | Chi phí | Tiết kiệm/tháng | Hoàn vốn |
|---|---|---|---|
| Kill-Switch Ads (fill-rate <80%) | 5 M VNĐ | 80 M VNĐ | <0.1 tháng |
| AI Size Guide (Outdoor, GenZ) | 50 M VNĐ | 100 M VNĐ | 0.5 tháng |
| Safety Stock Z-95% | 200 M VNĐ | 150 M VNĐ | 1.3 tháng |

---

## Tính tái lập

| Yếu tố | Chi tiết |
|---|---|
| Random seed | `SEED = 42` xuyên suốt toàn bộ pipeline |
| Siêu tham số | Lưu trong `best_params.json` sau khi Optuna tuning |
| Phiên bản thư viện | Ghim trong `requirements.txt` |
| Kiểm tra leakage | `assert sales['Date'].max() < sample_sub['Date'].min()` tại thời điểm load |
| No test leakage | Revenue/COGS thực tế của tập test **không bao giờ** được đưa vào features |

---

## Ràng buộc tuân thủ

- ✅ Không sử dụng Revenue/COGS từ tập test làm đặc trưng
- ✅ Không sử dụng dữ liệu ngoài bộ dữ liệu được cung cấp (`lunardate` là thư viện tính toán, không phải dữ liệu ngoài)
- ✅ Mã nguồn đầy đủ, kết quả có thể tái lập với `SEED=42` và `best_params.json`

---

*Datathon 2026 — The Gridbreaker | VinTelligence & VinUniversity Data Science & AI Club*
