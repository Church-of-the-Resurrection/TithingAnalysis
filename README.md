# Church of the Resurrection: Tithing Analysis & Forecasting

This repository contains the **Dataform** source code for the Church of the Resurrection Finance Council's data pipeline. It automates the ingestion of tithing and pledge data to provide the parish leadership with accurate historical reporting and predictive financial forecasts.

## 🎯 Purpose
The goal of this project is to move from reactive budgeting to proactive financial stewardship. By analyzing donor behavior patterns, we can:
- Predict annual revenue with high precision.
- Identify trends in "Core" vs. "Major" giving.
- Provide real-time statistics for the weekly parish bulletin.
- Create a "Safe Floor" budget using conservative statistical forecasting.

## 🛠 Tech Stack
- **Google Dataform (Core 3.0):** Orchestrates the SQL pipeline and manages dependencies.
- **BigQuery:** The data warehouse where all calculations and ML (ARIMA) training occur.
- **SQLX:** The primary language used for defining data transformations.

## 📂 Key Documentation
- **[GEMINI.md](./GEMINI.md):** The technical entry point. Contains GCP Project IDs, ownership details, and execution tag documentation.
- **[Forecasting Methodology](./docs/forecasting_methodology.md):** A detailed breakdown of the "4-Layer Cake" strategy used to predict revenue.

## 🚀 Getting Started
To run this project, you must have access to the `finance-council` GCP project.

1.  **Install Dataform CLI:**
    ```bash
    npm i -g @dataform/cli
    ```
2.  **Initialize Project:**
    ```bash
    dataform install
    ```
3.  **Run the Daily Refresh:**
    ```bash
    dataform run --tags daily
    ```

## 📊 Maintenance
The pipeline is designed to be self-sustaining, but requires two manual monthly actions:
1.  **Upload New Data:** Ensure the `contributions` and `pledges` CSVs in the linked Cloud Storage buckets are updated.
2.  **Retrain Models:** Execute the `forecasting` tag at the start of each month to incorporate the latest donation trends.

---
*Maintained by the Finance Council Chairperson (Parker Roth).*
