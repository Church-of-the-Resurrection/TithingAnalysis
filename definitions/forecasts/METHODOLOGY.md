# Collection Forecasting Methodology

## Executive Summary
This project utilizes a **"Layer Cake" Component-Based Forecasting Strategy**. Instead of forcing a single machine learning model to predict highly variable revenue streams, we split the church's revenue into four distinct "layers" based on donor behavior and predictability. Each layer is forecasted using the mathematical model best suited for its specific volatility profile.

## The 4-Layer Strategy

### Layer 1: Pledged Revenue (The Foundation)
* **Description:** Revenue from donors who have made a formal commitment for the year. This is our highest-certainty revenue stream.
* **Method:** Deterministic Calculation.
* **Logic:**
    * Takes the **Total Pledged Amount** for the target year (e.g., 2026).
    * Applies a **Rolling 3-Year Seasonality Curve**. This curve is dollar-weighted (capturing the "January Lump Sum" behavior of major donors) rather than donor-weighted.
    * Applies a standard **Fulfillment Rate** (default 98%) to account for unfulfilled pledges.
* **Why:** Machine learning models often lag in detecting pledge drops. This method allows the forecast to react immediately to stewardship campaign results.
* **Files:** `pledged_contributions.sqlx`, `pledge_seasonality.sqlx`

### Layer 2: Unpledged Core Giving (The Trend)
* **Description:** "Plate collections" and regular electronic giving from non-pledged donors. Defined as Ordinary gifts under $2,500.
* **Method:** ARIMA_PLUS (Machine Learning).
* **Logic:**
    * Uses Google BigQuery's `ARIMA_PLUS` algorithm to detect trends and cycles.
    * **Filters:**
        * `IsOrdinary = TRUE` (Excludes Building/Flower funds)
        * `IsPledged = FALSE` (Prevents double-counting Layer 1)
        * `Amount <= 2500` (Excludes volatility from Layer 3)
    * **Safeguard:** Explicitly excludes the *current incomplete month* from training data to prevent the model from interpreting a partial month as a revenue crash.
* **Why:** Regular giving follows strong seasonal patterns (Summer slump, Easter bump) that ARIMA predicts highly accurately.
* **Files:** `core_contribution_forecast.sqlx`, `core_contributions.sqlx`

### Layer 3: Unpledged Major Gifts (The Volatility)
* **Description:** Large, irregular checks (> $2,500) from donors who did *not* pledge.
* **Method:** Statistical Baseline (Weighted Average).
* **Logic:**
    * Calculates a monthly baseline using a weighted mix: **70% Median / 30% Average**.
    * **Filters:**
        * `Amount > 2500`
        * `IsPledged = FALSE` (Critical check to avoid double-counting pledge payments)
        * `STARTS_WITH(FundName, '1')` (Strictly Ordinary/Operating funds)
    * **Why:** Major gifts are event-driven, not trend-driven. ARIMA models often hallucinate peaks after a one-time windfall. This method provides a conservative "Safe Floor" for budgeting.
* **Files:** `large_contributions.sqlx`

### Layer 4: Extraordinary & Restricted (The Pass-Through)
* **Description:** All non-operating revenue (Building Fund, Flowers, Missions, etc.).
* **Method:** Statistical Baseline (Median).
* **Logic:**
    * Calculates the **Median** of historical monthly giving for all funds *not* starting with "1".
    * **Why:** These funds are highly sporadic. The median ensures we budget conservatively and treat any surplus as a bonus, rather than relying on it for operating expenses.
* **Files:** `extraordinary_contributions.sqlx`

---

## Technical Implementation (Dataform)

The forecast is generated through a pipeline of SQLX files:

1.  **Sources:** Raw data is ingested from `contributions` and `pledges`.
2.  **Models:**
    * `pledge_seasonality` calculates the monthly distribution % from the last 3 years of data.
    * `core_contribution_forecast` trains the ARIMA model.
3.  **Forecast Views:**
    * `pledged_contributions`: Generates the Layer 1 forecast.
    * `core_contributions`: Formats the ARIMA output (Layer 2).
    * `large_contributions`: Calculates the statistical baseline for Layer 3.
    * `extraordinary_contributions`: Calculates the statistical baseline for Layer 4.
4.  **Reporting:**
    * `combined_contributions`: The final table that `UNION ALL`s the four layers together. This is the source of truth for the Budget Dashboard.

## Anti-Double-Counting Audit
To ensure data integrity, the following exclusion logic is enforced:

| Layer | Filter Logic |
| :--- | :--- |
| **Pledged** | `JOIN pledges` (In) |
| **Core** | `LEFT JOIN pledges WHERE GivingNumber IS NULL` AND `Amount <= 2500` |
| **Major** | `LEFT JOIN pledges WHERE GivingNumber IS NULL` AND `Amount > 2500` |

*This ensures that every dollar of revenue falls into exactly one bucket.*