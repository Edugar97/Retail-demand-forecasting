# Retail Demand Forecasting 📈

### End-to-end demand forecasting for grocery retail — from raw transactional data to a business-ready decision tool.

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?logo=python&logoColor=white)
![LightGBM](https://img.shields.io/badge/LightGBM-Gradient_Boosting-9ACD32)
![Prophet](https://img.shields.io/badge/Prophet-Time_Series-1E90FF)
![Pandas](https://img.shields.io/badge/Pandas-Data_Wrangling-150458?logo=pandas&logoColor=white)
![Power BI](https://img.shields.io/badge/Power_BI-Dashboard-F2C811?logo=powerbi&logoColor=black)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebooks-F37626?logo=jupyter&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

> A complete machine-learning pipeline that forecasts next-day grocery sales for the five largest product families, reaching **92.8% forecast accuracy (MAPE ≈ 7.2%)** on a held-out test period and a **~40% reduction in RMSE versus a seasonal-naive baseline** — surfaced through an interactive Power BI dashboard designed for inventory and planning decisions.

![Retail Demand Forecasting Dashboard](reports/dashboard.png)

---

## 🎯 Business Problem

Forecasting error is expensive on both sides. **Overstock** ties up cash and drives waste in perishable categories; **stockouts** cost sales and erode customer trust. For a large grocery retailer, even a few points of forecast accuracy translate into meaningful savings in inventory and planning.

This project builds a daily demand forecast for **Corporación Favorita**, a major Ecuadorian grocery chain, at the **product-family level** — the granularity at which purchasing and replenishment decisions are actually made. The goal is not just a model with good error metrics, but a **decision tool a planner can open and act on**.

---

## 📊 Results at a Glance

All metrics are measured on the **test period (1 Jun – 15 Aug 2017, 380 family-days)**, which the model never saw during training or early stopping. Sales are **unit volumes** as provided by the dataset (it contains no prices).

| Metric | Value | What it means |
| :--- | :--- | :--- |
| **Forecast Accuracy (1 − MAPE)** | **92.82%** | On average, daily forecasts land within ~7% of actual sales |
| **MAPE** | **≈ 7.2%** | Mean absolute percentage error across families and days |
| **RMSE** | **15,176 units** | Typical error magnitude, penalizing large misses |
| **MAE** | **10,042 units** | Average absolute daily error per family |
| **R²** | **0.97** | The model explains 97% of the variance in daily sales |
| **Total Actual vs Forecast** | **52.73M vs 51.71M units** | Aggregate forecast within ~2% of realized sales |

### Model Benchmarking

Every model was evaluated on the same test window using a **chronological train / validation / test split** (no future leakage).

| Model | Role | MAE | RMSE | R² |
| :--- | :--- | ---: | ---: | ---: |
| Naive (previous day) | Baseline | 24,661 | 34,977 | 0.82 |
| Seasonal Naive (same weekday last week) | Baseline | 16,692 | 25,335 | 0.91 |
| Prophet (one model per family) | Statistical time-series benchmark | 22,696 | 31,414 | 0.86 |
| **LightGBM** | **Champion** | **10,042** | **15,176** | **0.97** |

LightGBM cuts RMSE by **~40%** versus the Seasonal Naive baseline (25,335 → 15,176).

> **How to read this comparison:** LightGBM and the naive baselines are **one-step-ahead** forecasts — they use the actual sales of the previous day and week as inputs. Prophet was fit once on data through December 2016 and forecasts months ahead, so part of its gap reflects that longer horizon, not only the algorithm.

### Accuracy by Product Family

| Family | Actual Sales (units) | Forecast Sales (units) | Accuracy |
| :--- | ---: | ---: | ---: |
| Grocery I | 19.20M | 18.35M | 92.52% |
| Beverages | 14.81M | 14.62M | 93.10% |
| Produce | 9.75M | 9.49M | 94.29% |
| Cleaning | 5.27M | 5.50M | 90.37% |
| Dairy | 3.69M | 3.75M | 93.81% |
| **Total** | **52.73M** | **51.71M** | **92.82%** |

---

## 🔑 What Drives Demand

The dashboard exposes **feature importance** (share of total LightGBM gain) so the model's logic is transparent to the business, not a black box. Short-term momentum dominates:

| Rank | Feature | Importance | Interpretation |
| :--- | :--- | ---: | :--- |
| 1 | `rolling_mean_7` | 39% | Average of the previous 7 days — recent demand level |
| 2 | `lag_7` | 33% | Same weekday last week — weekly rhythm |
| 3 | `lag_1` | 19% | Previous day — short-term momentum |
| 4 | `day_of_week` | 3% | Weekly seasonality |
| 5 | `day` | 1% | Calendar position |

**Takeaway:** recent demand and weekly seasonality explain over 90% of the model's signal, while calendar and promotion features add little on top. Because the strongest inputs are the last 7 days of actual sales, the model is best suited to **short-horizon (next-day) replenishment**; longer-range planning would need a recursive or multi-horizon setup.

---

## 🧠 Approach & Pipeline

The project follows a full, reproducible analytics workflow across three notebooks:

1. **Data understanding & quality checks** (`01_data_understanding.ipynb`) — 3,000,888 daily records across 54 stores and 33 product families (Jan 2013 – Aug 2017); verified no missing values or duplicates and found 31.3% zero-sales records concentrated in low-volume families.
2. **Exploratory Data Analysis** — trend, weekly and yearly seasonality (Sunday and December peak), product and store concentration (top 10 families = 93% of sales), and promotion effects (correlation 0.43 with sales).
3. **Feature engineering** (`02_modeling.ipynb`) — built per family on daily totals:
   - **Lag features** (`lag_1`, `lag_7`, `lag_30`)
   - **Rolling means** (`rolling_mean_7`, `rolling_mean_30`), shifted one day so they only use past data
   - **Calendar features** (year, month, day, day of week, quarter, week of year, weekend flag) and a **cyclical month encoding** (`month_sin`, `month_cos`)
   - **Promotion indicator** (`has_promotion`)
4. **Modeling** — chronological split (train: Jan 2013 – Dec 2016, validation: Jan – May 2017, test: Jun – Aug 2017); Naive and Seasonal Naive baselines; Prophet per family; and a **LightGBM** regressor trained with early stopping on the validation set.
5. **Evaluation** — RMSE, MAE, R² and MAPE on the test period, plus the **error distribution (APE %)** and accuracy by product family.
6. **Business layer** (`03_dashboard_preparation.ipynb`) — builds the dashboard dataset (accuracy per row, APE bins) and feeds an interactive **Power BI dashboard** with date and family filters, actual-vs-forecast tracking, and executive insights.

---

## 🛠️ Tech Stack

| Area | Tools |
| :--- | :--- |
| Language | Python 3.10+ |
| Data wrangling | pandas, numpy |
| Modeling | LightGBM, Prophet, scikit-learn |
| Visualization (analysis) | matplotlib, seaborn |
| Business intelligence | Power BI |
| Environment | Jupyter Notebook |

---

## 📁 Repository Structure

```
Retail-demand-forecasting/
├── data/
│   ├── train.csv                     # Kaggle source file (not tracked — see "How to Reproduce")
│   └── dashboard/
│       ├── dashboard_data.csv        # Dashboard-ready predictions (output of notebook 03)
│       ├── feature_importance.csv    # Feature importance table used by the dashboard
│       ├── model_metrics.csv         # MAE, RMSE, R², accuracy used by the dashboard
│       └── analisis.pbix             # Power BI report
├── images/
│   ├── feature_importance.png
│   ├── forecast_vs_actual.png
│   ├── model_comparison.png
│   └── retail_dashboard_template.svg # Dashboard layout template
├── models/
│   ├── lightgbm_model.pkl            # Trained LightGBM model
│   └── model_metadata.json           # Features, split dates and final metrics
├── notebooks/
│   ├── 01_data_understanding.ipynb   # Data quality + EDA + feature correlation
│   ├── 02_modeling.ipynb             # Baselines, LightGBM, Prophet, evaluation, exports
│   └── 03_dashboard_preparation.ipynb # Builds the Power BI dataset
├── reports/
│   ├── dashboard.png                 # Power BI dashboard (preview above)
│   ├── predictions.csv               # Raw test-set predictions
│   ├── test_predictions.csv          # Test-set predictions with error metrics
│   ├── feature_importance.csv
│   └── model_metrics.csv
├── requirements.txt
├── LICENSE
└── README.md
```

---

## ▶️ How to Reproduce

```bash
# 1. Clone the repository
git clone https://github.com/Edugar97/Retail-demand-forecasting.git
cd Retail-demand-forecasting

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # On Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Download the dataset
#    Kaggle → "Store Sales - Time Series Forecasting" (Corporación Favorita)
#    and place train.csv in data/

# 5. Run the notebooks in order (from the notebooks/ folder)
jupyter notebook notebooks/01_data_understanding.ipynb
jupyter notebook notebooks/02_modeling.ipynb
jupyter notebook notebooks/03_dashboard_preparation.ipynb

# 6. Open data/dashboard/analisis.pbix in Power BI Desktop
```

---

## 💡 What This Project Demonstrates

- **End-to-end ownership** — from raw data to a stakeholder-facing product, not just an isolated notebook.
- **Rigorous validation** — a chronological train / validation / test split and progressive baselines, all scored on the same test window.
- **Feature engineering that matters** — lag and rolling features that drive over 90% of the model's signal.
- **Gradient boosting in practice** — a LightGBM model with early stopping delivering a ~40% RMSE improvement over the seasonal-naive baseline.
- **Error analysis, not just headline metrics** — APE distribution and per-family accuracy to understand *where* the model works, with Cleaning as the weakest family.
- **Honest benchmarking** — documenting that the one-step-ahead setup favors lag-based models over a long-horizon Prophet forecast.
- **Translating models into decisions** — an interactive BI layer that turns predictions into inventory and planning insight.

---

## 👤 Author

**Raúl Eduardo García Cuéllar** — Data Analyst with hands-on enterprise experience in SAP HANA, SQL, and production data systems, combining ML/statistics with real-world business context.

- **GitHub** — [github.com/Edugar97](https://github.com/Edugar97)
- **Email** — [raulcuellar.rc97@gmail.com](mailto:raulcuellar.rc97@gmail.com)

---

## 📄 License

Distributed under the MIT License. See [`LICENSE`](LICENSE) for details.

*Data source: [Store Sales - Time Series Forecasting](https://www.kaggle.com/competitions/store-sales-time-series-forecasting) (Kaggle, Corporación Favorita).*
