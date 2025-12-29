# Revenue Forecasting Methodology

## Executive Summary
This project utilizes a **"Layer Cake" Component-Based Forecasting Strategy**. Instead of forcing a single machine learning model to predict highly variable revenue streams, we split the church's revenue into four distinct "layers" based on donor behavior and predictability. Each layer is forecasted using the mathematical model best suited for its specific volatility profile.

## Core Philosophy: Conservative Biasing
A central tenet of this forecasting model is **"Conservative Biasing."** In non-profit finance, the cost of over-estimating revenue (budget shortfall) is far higher than under-estimating it (budget surplus).

* **Stable Layers (1 & 2):** targeted for **Precision**. We aim for <5% variance because these funds pay the salaries and bills.
* **Volatile Layers (3 & 4):** targeted for **Safety**. We intentionally use statistical medians ("Safe Floors") rather than averages. This means that in "windfall" months, the model will purposefully under-predict.
* **Success Metric:** A positive variance in Layers 3 & 4 is considered a success, not an error. It represents "Surplus Variance" that builds cash reserves.

## The 4-Layer Strategy

### Layer 1: Pledged Revenue (The Foundation)
* **Description:** Revenue from donors who have made a formal commitment for the year. This is our highest-certainty revenue stream.
* **Method:** Deterministic Calculation with Automated "Proxy" Projection.
* **Logic:**
    * **Actuals:** If pledge cards for the target year (e.g., 2026) have been entered, the model uses the real `TotalPledged` amount.
    * **Proxy Projection (The Safety Net):** If pledge cards are missing, the model calculates a "Proxy Total" using the formula: `Last Year's Total * (1 + Average Historical Growth Rate)`.
    * **Seasonality:** Applies a **Rolling 3-Year Dollar-Weighted Seasonality Curve** to capture "Lump Sum" behavior.
    * **Fulfillment:** Applies a standard **Fulfillment Rate** (default 98%) to account for unfulfilled pledges.
* **Files:** `pledged_contributions.sqlx`, `pledge_seasonality.sqlx`, `projected_pledge_totals.sqlx`

### Layer 2: Unpledged Core Giving (The Trend)
* **Description:** "Plate collections" and regular electronic giving from non-pledged donors. Defined as Ordinary gifts under $2,500.
* **Method:** ARIMA_PLUS (Machine Learning).
* **Logic:**
    * Uses Google BigQuery's `ARIMA_PLUS` algorithm to detect trends and cycles.
    * **Filters:** `IsOrdinary = TRUE`, `IsPledged = FALSE`, `Amount <= 2500`.
    * **Backcasting:** The view combines future predictions with historical "fitted values" (using `ML.DETECT_ANOMALIES`) to allow for seamless "Forecast vs. Actuals" analysis over past years.
* **Files:** `core_contribution_forecast.sqlx`, `core_contributions.sqlx`

### Layer 3: Unpledged Major Gifts (The Volatility)
* **Description:** Large, irregular checks (> $2,500) from donors who did *not* pledge.
* **Method:** Statistical Baseline (Median).
* **Logic:**
    * Calculates the **Median** of historical monthly giving for this specific segment.
    * **Why Median?** Major gifts are event-driven. Averages are easily skewed by one-time windfalls. The Median provides a "Safe Floor," ensuring we budget only for revenue that reliably appears, treating spikes as surplus.
    * **Filters:** `Amount > 2500`, `IsPledged = FALSE`, `STARTS_WITH(FundName, '1')`.
* **Files:** `large_contributions.sqlx`

### Layer 4: Extraordinary & Restricted (The Pass-Through)
* **Description:** All non-operating revenue (Building Fund, Flowers, Missions, etc.).
* **Method:** Statistical Baseline (Median).
* **Logic:**
    * Calculates the **Median** of historical monthly giving for all funds *not* starting with "1".
    * **Why:** These funds are highly sporadic. Blending in the "Average" would cause the model to hallucinate trends from past capital campaigns.
* **Files:** `extraordinary_contributions.sqlx`

## Technical Implementation (Dataform)

The forecast is generated through a pipeline of SQLX files:
1.  **Sources:** Raw data ingestion (`contributions`, `pledges`).
2.  **Models:** Seasonality calculations, Pledge Growth analysis, and ARIMA model training.
3.  **Forecast Views:** Individual outputs for Layers 1–4.
4.  **Reporting:** `combined_contributions` unites all layers for the final dashboard.

## Operational Maintenance
* **ARIMA Model Retraining:** The machine learning model (`Models/core_contribution_forecast`) is **stateful**. It does *not* automatically update when new data arrives. You must explicitly run the `CREATE OR REPLACE MODEL` operation at least **once per month** (ideally on the 1st) to incorporate the latest month's data into the trend lines.
* **Fulfillment Rate Calibration:** The 98% fulfillment rate in Layer 1 is hardcoded. This should be audited annually in January by comparing `TotalPledged` vs `TotalGiven` for the closed year.

## Critical Business Dependencies
* **Fund Code Schema:** The model identifies "Ordinary/Operating" funds strictly by checking if `FundName` starts with the character `'1'`. Any changes to the Chart of Accounts that alter this prefix must be updated in the Dataform definitions immediately.
* **Major Gift Threshold:** The distinction between "Core" (Layer 2) and "Major" (Layer 3) is a hard threshold of **$2,500**. This should be adjusted for inflation every 3-5 years if the volume of "Major" gifts becomes too large for the manual baseline approach.

## Debugging & Analysis Guide

To verify model performance, we use a two-step diagnosis process.

* **The "Current Month" Rule:** Never evaluate forecast accuracy for the current calendar month. Because donations are often back-weighted (heavier on Sundays or month-end), a partial month will always look like a forecast failure. Always filter reports to `forecast_month < DATE_TRUNC(CURRENT_DATE(), MONTH)`.

### Step 1: The Executive "Report Card"
**File:** `forecast_accuracy_report`
* **Purpose:** High-level check of "Did we hit the budget number?"
* **Metric:** Look at `error_pct`.
    * **Stable Months (Oct/Nov):** Target range is **+/- 5%**.
    * **Volatile Months (Aug/Sep):** Target range is **0% to +15%** (Positive Error = Surplus).
    * **Negative Spikes:** A large negative error (e.g., -20%) indicates a genuine revenue shortfall that requires immediate attention.

### Step 2: The Diagnostic Matrix
**File:** `forecast_diagnostic_matrix`
* **Purpose:** Root cause analysis when the "Report Card" shows a miss.
* **How to read:**
    * **Row:** Find the month with the variance.
    * **Columns:** Scan `Var_L1`, `Var_L2`, `Var_L3`, `Var_L4`.
* **Common Scenarios:**
    * **High Negative `Var_L1`:** Pledged donors are behind on payments. **Action:** Stewardship reminder letter.
    * **High Negative `Var_L2`:** Plate collection is soft. **Action:** Check attendance numbers or weather events.
    * **High Positive `Var_L3`:** A major donor gave a large unpledged gift. **Action:** Celebrate the surplus; do *not* adjust the forecast (this is the "Safe Floor" working).
    * **High Variance `Var_L4`:** A special collection occurred (e.g., Disaster Relief) that wasn't budgeted. **Action:** Ignore. This is pass-through money.

## Anti-Double-Counting Audit
To ensure data integrity, the following exclusion logic is enforced:

| Layer | Filter Logic |
| :--- | :--- |
| **Pledged** | `JOIN pledges` (In) |
| **Core** | `LEFT JOIN pledges WHERE GivingNumber IS NULL` AND `Amount <= 2500` |
| **Major** | `LEFT JOIN pledges WHERE GivingNumber IS NULL` AND `Amount > 2500` |

*This ensures that every dollar of revenue falls into exactly one bucket.*