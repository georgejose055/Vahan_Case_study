# Vahan Case Study — Product Analytics Internship

**Candidate:** George Jose  
**Dataset:** 18,198 leads, one row per lead  
**Target:** First Trip (FT) conversion  
**Submission Date:** August 2026

---

## Overview

This repository contains the full submission for the **Product Analytics Internship Case Study at Vahan**. The case study explores the lead-to-FT (First Trip) conversion funnel across three structured tasks:

1. Identifying the top-performing cohorts by FT conversion rate
2. Writing a SQL aggregation query to summarize cohort-level performance
3. Building a machine learning model to identify the key drivers of FT conversion

---

## Repository Structure

```
Vahan_Case_study/
│
├── Vahan_Case_Study.xlsx                  ← Raw dataset (Raw Data tab)
├── Vahan_Case_Study_George.ipynb          ← Full analysis notebook (Parts 1, 2, 3)
├── Vahan_Case_Study_Report.md             ← Written report with reasoning and insights
│
└── output/
    ├── aggregated_cohort_table.csv        ← Full cohort aggregation (Part 2 output)
    ├── cohort_summary_full.csv            ← Cohort summary across all lead sources
    ├── top3_cohorts.csv                   ← Top 3 cohorts by FT rate (Part 1 output)
    ├── feature_importance.csv             ← Feature importance scores (Part 3 output)
    ├── feature_importance.png             ← Feature importance bar chart
    └── confusion_matrix.png               ← Model confusion matrix
```

---

## Dataset Overview

| Property | Value |
|---|---|
| Total leads | 18,198 |
| Unit of analysis | One row per lead |
| Target variable | `FT_after_upload` (binary) |
| FT conversion rate | ~0.297% (highly imbalanced) |

Key columns include: `lead_source`, `Attempted`, `Connected`, `Interested`, `OB_after_upload`, `FT_after_upload`, `FT_after_first_attempt`, `upload_to_first_attempt_P50 (hrs)`, `Attempt per Lead`, `tag_filled`

---

## Part 1 — Top 3 Cohorts

Cohorts were ranked by **FT rate = FT_after_upload / Uploaded Leads** rather than raw FT count. Raw counts favour large cohorts irrespective of conversion efficiency. A minimum threshold of **30 uploaded leads** was applied to exclude statistically unstable small cohorts.

### Result

| Cohort | Leads | FT Count | FT Rate | Connect Rate | Interest Rate |
|---|---:|---:|---:|---:|---:|
| Single Referral > 7 days- 24th Jul | 1,500 | 14 | 0.93% | 47.46% | 1.01% |
| Khanna- 2W 26th Jul | 1,546 | 14 | 0.91% | 41.72% | 3.48% |
| PreOb-Ob Fees Paid 29th Jul (set 1) | 1,483 | 7 | 0.47% | 45.36% | 15.85% |

> **Key insight:** The third cohort has a far higher interest rate (15.85%) than the top two, yet converts at a lower FT rate. This reveals that **expressed interest is not a reliable proxy for actual FT conversion**. The top two cohorts are the ones to prioritise operationally.

---

## Part 2 — Aggregation Query

The aggregation grain is **`lead_source`** — the business unit at which sourcing and follow-up decisions are made. Aggregating by date would fragment cohorts; aggregating at lead level would simply replay raw data.

```sql
SELECT
    lead_source,
    COUNT(*) AS total_leads,
    SUM(Attempted) AS attempted,
    SUM(Connected) AS connected,
    SUM(Interested) AS interested,
    SUM(OB_after_upload) AS onboarded,
    SUM(FT_after_upload) AS ft_count,
    ROUND(SUM(FT_after_upload) * 100.0 / COUNT(*), 2) AS ft_rate_pct,
    ROUND(SUM(Connected) * 100.0 / SUM(Attempted), 2) AS connect_rate_pct,
    ROUND(SUM(Interested) * 100.0 / SUM(Connected), 2) AS interest_rate_pct,
    ROUND(AVG(`upload_to_first_attempt_P50 (hrs)`), 1) AS median_time_to_attempt_hrs
FROM raw_leads
GROUP BY lead_source
ORDER BY ft_rate_pct DESC;
```

> **Insight:** OLX contributes the highest volume (5,182 leads) but converts at a far lower FT rate than the top performers. Volume and conversion quality diverge — effort should be reallocated toward high-FT-rate sources.

---

## Part 3 — ML Model: Drivers of FT Conversion

### Model Setup

| Parameter | Value |
|---|---|
| Algorithm | Random Forest Classifier |
| Trees | 200 |
| Max depth | 6 |
| Class weight | `balanced` |
| Target | `FT_after_upload` (binary) |

### Features Used

`Attempted`, `Connected`, `Attempt per Lead`, `tag_filled`, `Interested`, `upload_to_first_attempt_P50 (hrs)`, `lead_source` (label encoded)

### Leakage Avoidance

`OB_after_upload` and `FT_after_first_attempt` were deliberately **excluded** as features. Both are downstream of or directly tied to the target and would give the model the answer key — inflating performance that would never hold in production.

---

### Model Performance

The dataset is 99.70% no-FT / 0.30% FT — a textbook severe class imbalance. **Accuracy alone is a misleading metric here.**

| Metric | No FT (0) | FT (1) |
|---|---|---|
| Precision | 1.00 | 0.01 |
| Recall | 0.76 | 0.43 |
| F1-score | 0.87 | 0.01 |
| Support | 4,536 | 14 |
| **Overall Accuracy** | **0.76** | |

### Confusion Matrix

| | Predicted No FT | Predicted FT |
|---|---:|---:|
| **Actual No FT** | 3,465 (TN) | 1,071 (FP) |
| **Actual FT** | 8 (FN) | 6 (TP) |

**Business interpretation:**
- **False negatives (8):** Missed FT leads — lost driver signups and revenue
- **False positives (1,071):** Wasted follow-up calls — a cost problem, not a revenue loss

The right operating threshold is a business decision based on the relative cost of a missed trip vs. a wasted call.

---

### Feature Importance

| Feature | Importance |
|---|---:|
| upload_to_first_attempt_P50 (hrs) | 0.3189 |
| leadsource_enc | 0.2918 |
| Attempt per Lead | 0.1676 |
| Attempted | 0.1418 |
| tag_filled | 0.0389 |
| Connected | 0.0330 |
| Interested | 0.0080 |

> **The two dominant signals are speed to first attempt and lead source — together they account for ~61% of predictive power.** `Interested` is nearly useless at 0.008, reinforcing the Part 1 finding that expressed interest does not reliably predict FT conversion.

---

## Key Business Takeaways

- **Prioritise cohort quality over volume.** High-volume cohorts like OLX do not convert well. The best sources by FT rate should receive disproportionate follow-up effort.
- **Call fast.** Speed to first attempt after upload is the single strongest predictor of FT conversion. Reducing this lag is one of the highest-leverage operational levers.
- **Do not rely on "Interested."** Stated interest is a weak signal with very low predictive importance. Operational decisions should be based on source quality and speed, not interest flags.
- **Evaluate model performance using business cost.** Accuracy is misleading at a 0.3% base rate. Recall, precision, and a cost-weighted threshold matter more.

---

## Limitations & Next Steps

- **Class imbalance** is the core constraint — only ~54 FT-positive leads in 18,198. SMOTE or cost-sensitive oversampling could improve recall.
- **Threshold tuning** — the default 0.5 threshold is not optimal. Tuning it to the FN/FP cost ratio would improve operating value.
- **Temporal validation** — a date-based train/test split would give a more honest estimate of production performance than random splitting.
- **More features** — time-of-day of call, agent-level performance, and follow-up count could add meaningful signal.

---

## Tools Used

- Python 3
- Pandas, NumPy
- Scikit-learn (RandomForestClassifier)
- Matplotlib, Seaborn
- SQL
- Jupyter Notebook

---

## Author

**George Jose**  
M.Sc. Data Science — CHRIST University, Bangalore  
GitHub: [@georgejose055](https://github.com/georgejose055)
