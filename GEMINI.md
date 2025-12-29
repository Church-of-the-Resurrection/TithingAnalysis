# Tithing Analysis & Forecasting
**Parish:** [Church of the Resurrection](https://resurrectionwichita.com)  
**Project Owner:** Parker Roth (Chairperson, Finance Council)  

This repository contains the GCP Dataform project used to ingest tithing data, calculate household/fund statistics, and generate multi-layer revenue forecasts for the Finance Council.

---

## ☁️ Infrastructure
- **GCP Project ID:** `finance-council`
- **Org ID:** `107477859580`
- **Platform:** Google Cloud Dataform (Core 3.0.35)
- **Primary Region:** `us`

---

## 📂 Project Structure

| Directory | Purpose |
| :--- | :--- |
| `definitions/sources/` | External table definitions for raw CSV imports (Contributions & Pledges). |
| `definitions/views/` | Core aggregations (Weekly/Monthly stats) and household-level metrics. |
| `definitions/models/` | ML training (ARIMA) and statistical lookup tables (Seasonality/Growth). |
| `definitions/forecasts/` | Individual forecast "layers" and the `combined_contributions` master view. |
| `definitions/queries/` | Diagnostic scripts and "Report Cards" for auditing forecast accuracy. |
| `docs/` | Deep-dive methodology and business logic documentation. |

---

## 📈 Forecasting Strategy (The 4-Layer Cake)
We use a **Component-Based Strategy** to ensure "Conservative Biasing" (preferring safe under-estimation over risky over-estimation).

1.  **Layer 1: Pledged Revenue** (Deterministic + Seasonality).
2.  **Layer 2: Unpledged Core Giving** (ARIMA_PLUS Machine Learning for gifts < $2k).
3.  **Layer 3: Unpledged Major Gifts** (Statistical Median for gifts > $2k).
4.  **Layer 4: Extraordinary/Restricted** (Statistical Median for non-operating funds).

*Full details can be found in [docs/forecasting_methodology.md](docs/forecasting_methodology.md).*

---

## 🚀 Execution & Tags
We use tags to manage the execution lifecycle of the project:

- `daily`: Refreshes external sources and re-calculates the heavy "caching" tables (Weekly/Monthly stats). Run this every morning.
- `forecasting`: Re-runs the ML models and updates the combined forecast views.
- `bulletin`: Specifically refreshes the stats required for the weekly parish bulletin.
- `debug`: Utility scripts for ad-hoc analysis.

**Example Command:**
```bash
dataform run --tags daily,bulletin
```

---

## 🛠️ Maintenance & Audits
- **ARIMA Retraining:** The Machine Learning model in `definitions/models/core_contribution_forecast.sqlx` must be manually retrained (re-run) on the 1st of every month to incorporate the previous month's final data.
- **Threshold Audit:** Use `definitions/queries/core_threshold_analysis.sqlx` annually to verify the $2,000 "Major Gift" threshold is still mathematically optimal.
- **Fund Codes:** Operating revenue is identified by a `1` prefix in the `FundName`. If the Chart of Accounts changes, the `IsOrdinary` logic in `monthly_fund_stats.sqlx` must be updated.
