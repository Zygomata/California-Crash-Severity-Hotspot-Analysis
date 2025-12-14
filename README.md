# California Crash Analytics & Injury Risk Study

## Overview

This project analyzes vehicle crashes across California to understand how **time of day**, **weekend vs. weekday**, and **highway vs. non-highway** conditions affect the likelihood of injuries in reported collisions. Using multi-year crash data from the **California Crash Reporting System (CCRS)**, the project builds a reproducible Python-based data pipeline and applies statistical methods to quantify relationships between crash context and injury outcomes.

---

## Research Question

> **To what extent do time of day, weekend or not, and the presence of a highway affect injury rates in vehicle crashes?**

Motivation:

- California has high traffic volume, dense urban areas, and busy highways that contribute to frequent crashes and injuries.  
- Understanding how injury rates vary by time, weekend status, and highway presence can inform targeted safety interventions and public health policy.

**Hypothesis:**  
Nighttime, weekends, and highway crashes are associated with **higher injury rates** due to reduced visibility, higher speeds, and riskier driving patterns.

---

## Data

### Source

- **California Crash Reporting System (CCRS)** – official crash and injury data for the state of California (2016–2025).  
- Data accessed via CA Open Data portal:
  - Crashes files (per year)
  - Injured (parties) files (per year)

### Key Tables

- **Crashes dataset**  
  - Example fields: `CollisionId`, `Crash Date Time`, `Crash Time Description`, `Day Of Week`, `IsHighwayRelated`, city/county codes, collision type.
- **Injured dataset**  
  - Example fields: `CollisionId`, `ExtentOfInjuryCode`, `InjuredPersonType`, `SeatPosition`, `SafetyEquipmentCode`.

### Final Analysis Dataset

After extraction, join, and preprocessing, the final modeling dataset includes:

- `ID` – new unique identifier (replaces `CollisionId` after resolving many-to-one injury records)  
- `is_injured` – binary injury flag (0 = low/no injury, 1 = higher severity)  
- `is_weekend` – 0 = Monday–Friday, 1 = Saturday/Sunday  
- `crash_time` – crash time bucketed to nearest hour (0–2300 in 24h format)  
- `is_highway` – 0 = non-highway, 1 = highway  

The dataset contains **~5 million+ rows** after preprocessing and deduplication.

---


---

## Data Pipeline

### 1. Extraction

1. Download yearly **crashes** and **injured** CSVs from the CCRS open data portal.  
2. Store raw files in a local “data lake” folder structure (e.g., `data/raw/2016_crashes.csv`).

Tools:

- **Python**, **Pandas**, Jupyter notebooks for reproducible workflows.  
- Files loaded with `pandas.read_csv` across years (2016–2025), then concatenated into large crash and injured dataframes (4M+ and 5M+ rows respectively).

### 2. Transformation & Merging

Key steps:

- Strip whitespace from column names and standardize schemas across years.  
- Concatenate yearly crash files into `df_crashes` and yearly injured files into `df_injured` using `pd.concat` (resulting in 4M–5M+ rows per table).  
- Rename `Collision Id` → `CollisionId` to ensure a consistent join key.  
- Outer join crashes and injured on `CollisionId` to capture both crash context and injury outcomes, creating a combined dataset with ~6.7M rows.

Then:

- Select only relevant fields: injury severity, day of week, crash time, and highway indicator.  
- Export intermediate combined dataframe to CSV for use in later notebooks, avoiding recomputation in large in-memory operations.

### 3. Cleaning & Preprocessing

Major preprocessing operations:

1. **De-duplication**  
   - Remove exact duplicate rows with `drop_duplicates`.  
   - Investigate remaining duplicate `CollisionId` values and determine they represent **multiple injured parties in the same crash** (different `ExtentOfInjuryCode` values).  
   - Introduce a new unique `ID` and drop `CollisionId` while preserving injury information.

2. **Column renaming and type optimization**  
   - Shorten variable names (e.g., `ExtentOfInjuryCode` → `is_injured`, `Crash Time Description` → `crash_time`).  
   - Downcast numeric types (`int64` → `int16`, `float64` → `float32`) to reduce memory usage from ~230MB to ~182MB for the final dataframe.

3. **Missing value handling**  
   - Use **mode imputation** for categorical fields: `is_injured`, `is_weekend`, `is_highway`.  
   - Use **median imputation** for `crash_time` after inspecting its skewed distribution with Seaborn histograms.

4. **Feature engineering & recoding**  
   - `is_injured`: Map multiple categorical injury descriptions to a binary indicator (grouping minor/no visible injuries vs. more serious/fatal injuries).  
   - `is_weekend`: Map Sunday/Saturday to 1, all other days to 0.  
   - `is_highway`: Map boolean to 0/1.  
   - `crash_time`:  
     - Round to the nearest hundred (e.g., 1630 → 1600) for hourly buckets.  
     - Normalize values at or above 2400 back into 0–2300 (e.g., 2500 → 100), while intentionally keeping them on the original calendar day for interpretability of “weekend nights.”

---

## Analysis Methods

All analysis is performed in Python using Pandas, Seaborn, and standard stats/ML libraries in Jupyter notebooks.

### 1. Descriptive Statistics

- Compute descriptive summaries for key variables (injury rate, weekend share, highway share, distribution of crash times).  
- Use Seaborn distribution plots to understand the shape of `crash_time` and guide imputation strategy.

**Advantage:** Fast, high-level overview of central tendency and spread.  
**Limitation:** Limited insight for categorical predictors and interactions.

### 2. Logistic Regression

- Model injury likelihood (`is_injured` = 0/1) as a function of continuous `crash_time` using a logistic regression model.  
- Evaluate performance and confusion matrix to understand true positive vs. true negative rates.

Findings:

- The model is very good at predicting **non-injury** cases (majority class) but struggles with injured cases due to severe **class imbalance** (many more non-injured than injured records).  
- Overall accuracy is misleadingly high because the model can guess “no injury” most of the time and still perform well on paper.

**Advantage:** Simple, interpretable, and quick to run on large datasets.  
**Limitation:** Single predictor and imbalance reduce its predictive usefulness for high-risk cases.

### 3. Chi-Square Tests & Contingency Tables

- Build contingency tables for `is_injured` vs. `is_weekend` and `is_injured` vs. `is_highway`.  
- Run **Chi-squared tests of independence** to assess whether injury occurrence is statistically associated with weekend status and highway presence.

Results:

- Very small p-values (near zero) indicate **strong evidence** that:
  - Injury occurrence is dependent on **weekend vs. weekday** status.  
  - Injury occurrence is dependent on **highway vs. non-highway** location.

**Advantage:** Well-suited for large-sample categorical association testing, no normality assumption.  
**Limitation:** Does not convey *magnitude* or *direction* of association, only presence of dependency.

---

## Key Findings

- **Time of day, weekend status, and highway presence all show relationships with injury outcomes** in California crash data.  
- Logistic regression confirms issues with class imbalance: the model is biased toward non-injury predictions.  
- Chi-square tests provide strong statistical evidence that injury rates differ between:
  - Weekend vs. weekday crashes.  
  - Highway vs. non-highway crashes.

**Limitations:**

- Class imbalance between injured and non-injured cases reduces predictive power and can mislead accuracy-based metrics.  
- Heavy imputation for `is_injured` (around one-third of missing values) may introduce bias into estimates and significance tests.

---

## Future Work

Potential extensions of the project:

1. **Richer feature set**  
   - Incorporate additional factors: weather, road surface, lighting, speed limits, and vehicle type to refine injury risk modeling.

2. **Improved modeling and imbalance handling**  
   - Use techniques like resampling (over/under-sampling), class weights, or more advanced models (tree ensembles, gradient boosting) to better capture injury cases.  
   - Explore interaction terms (e.g., weekend x time of day, highway x time of day) and nonlinear effects.

3. **Production Data Pipeline**  
   - Move from notebook-only workflows to a parameterized Python/ETL pipeline with orchestration (e.g., Airflow) to automatically ingest new CCRS data releases.

---

## Getting Started

### Requirements

- Python 3.11+  
- Recommended packages:
  - `pandas`
  - `numpy`
  - `seaborn`
  - `matplotlib`
  - `scikit-learn` or `statsmodels` (for logistic regression)
  - `jupyter`

### Quickstart

1. Clone the repository and navigate into it.  
2. Download raw CCRS crash and injured CSVs from the CA Open Data portal into `data/raw/`.  
3. Run the notebooks in order:
   - `01_import.ipynb` – combines yearly crashes/injured data and exports merged CSV.  
   - `02_preprocess.ipynb` – cleans, imputes, and engineers final modeling dataset.  
   - `03_analysis.ipynb` – runs descriptive stats, chi-square tests, and logistic regression.

---

## References

- California Crash Reporting System (CCRS) – CA Open Data Portal.  
- Pandas documentation for large-scale dataframe processing and type optimization.  
- Seaborn documentation for statistical visualization used in distribution plots.  
- McKinney, W. *Python for Data Analysis: Data Wrangling with Pandas, NumPy, and Jupyter.*




  [Download the PDF]([https://github.com/Zygomata/California-Crash-Severity-Hotspot-Analysis/blob/main/D610_Task_2_writeup.pdf])
  
</html>

