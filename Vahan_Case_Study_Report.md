# Vahan — Product Analytics Internship Case Study
**Candidate:** George Jose  
**Dataset:** 18,198 leads, one row per lead  
**Target metric:** First Trip (FT) conversion  
**Submission Date:** August 2026

---

## Executive Summary

FT conversion in this dataset is extremely rare, at a base rate of **0.297%** — only about 54 positive cases across 18,198 leads. The single clearest theme across all three parts of the analysis is that **cohort quality and speed to first attempt** drive conversion far more than call volume or expressed interest. The top three cohorts by FT rate all exceed 1,400 leads and convert at roughly 3x the dataset average, while the largest cohort by volume (OLX, 5,182 leads) converts at a fraction of that rate. For the machine learning model, accuracy is a misleading headline number at this class ratio — the meaningful signals are recall, precision, and what each error type costs the business. The practical takeaway: prioritise high-FT cohorts, follow up fast after upload, and treat "Interested" as a leading indicator at best — not a conversion signal.

---

## 1. Problem Context

Vahan operates a lead-to-driver funnel where leads are uploaded in batches from various sources (cohorts), assigned to telecallers, and worked through a series of funnel stages: Attempted → Connected → Interested → Onboarded → First Trip (FT). The business objective is to maximise the number of leads that complete a First Trip, since FT is the revenue-generating event.

The dataset captures 18,198 individual lead records across 16 distinct cohorts, uploaded over a period in July 2026. Each row contains funnel stage indicators and timing metadata. The target variable is `FT_after_upload` — a binary flag indicating whether the lead completed a First Trip after being uploaded.

### Dataset at a Glance

| Property | Value |
|---|---|
| Total leads | 18,198 |
| Unit of analysis | One row per lead |
| Number of cohorts | 16 |
| FT-positive leads (`FT_after_upload`) | ~54 |
| FT-positive leads (`FT_after_first_attempt`) | ~17 |
| Overall FT conversion rate | 0.297% |
| Target variable | `FT_after_upload` (binary) |

> **Note on FT column divergence:** The dataset contains two FT columns — `FT_after_upload` (54 positives) and `FT_after_first_attempt` (17 positives). These differ by 37 leads. The divergence is expected: `FT_after_upload` counts all leads that eventually completed a First Trip after being uploaded, regardless of when the first call was made. `FT_after_first_attempt` counts only those who converted specifically after the first call attempt. Leads that converted after a second or later attempt are counted in the former but not the latter. All analysis in this report uses `FT_after_upload` as the target, since it is the broader and more operationally complete measure of conversion yield per cohort.

The extreme rarity of the positive class (FT) is the single most important data characteristic — it shapes every analytical decision, from cohort ranking methodology to model evaluation.

---

## 2. Part 1 — Top 3 Best-Performing Cohorts

### 2.1 Metric Choice

Cohorts were ranked by **FT rate**, defined as:

```
FT rate = FT_after_upload / Uploaded Leads × 100
```

Raw FT count was explicitly **not used** as the ranking metric. Raw counts automatically favour the largest cohorts regardless of conversion quality — OLX, with 5,182 leads, would rank first on count despite a very low FT rate. Rate normalises for cohort size and measures what actually matters: how efficiently each source converts its own leads into trips.

### 2.2 Volume Threshold

A minimum threshold of **30 uploaded leads** was applied before ranking. A cohort of 5 leads with 1 FT shows a 20% FT rate, but that rate is noise rather than signal — a single lead difference shifts the percentage dramatically. The 30-lead floor ensures that every ranked cohort has a statistically stable and operationally meaningful rate estimate.

### 2.3 Results

| Rank | Cohort | Leads | FT Count | FT Rate | Connect Rate | Interest Rate | Median Hrs to Attempt |
|---:|---|---:|---:|---:|---:|---:|---:|
| 1 | Single Referral > 7 days- 24th Jul | 1,500 | 14 | 0.93% | 47.46% | 1.01% | 48.5 hrs |
| 2 | Khanna- 2W 26th Jul | 1,546 | 14 | 0.91% | 41.72% | 3.48% | 15.0 hrs |
| 3 | PreOb-Ob Fees Paid 29th Jul (set 1) | 1,483 | 7 | 0.47% | 45.36% | 15.85% | 15.0 hrs |

### 2.4 Key Insight — Interest ≠ Conversion

Cohort #3 has a far higher interest rate (15.85%) than cohorts #1 and #2 (~1–3%), yet it converts at a lower FT rate. This decoupling is important: expressed interest reflects early-funnel intent, while FT reflects actual driver behaviour. An analyst who sorted by interest rate would select a lower-performing cohort over a higher-performing one. The data makes clear that **interest is a weak proxy for FT conversion** — a finding that is reinforced again in the machine learning results.

Cohort #1 (Single Referral) stands out particularly: its median time to first attempt is 48.5 hours — much longer than most cohorts — yet it still leads on FT rate. This suggests that the quality of the referral itself is the dominant signal, and that these leads convert even when follow-up is slower. Cohort #2 (Khanna 2W) follows up in 15 hours and achieves a near-identical FT rate, suggesting faster follow-up compensates for lower lead quality.

---

## 3. Part 2 — Aggregation Query

### 3.1 Grain Decision

The aggregation is performed at the **`lead_source`** level — one row per cohort. This is the correct business grain because:

- Leads are acquired and managed by source
- Budget allocation, team assignments, and follow-up strategy decisions are made at the cohort level
- A date grain would fragment the same cohort across multiple days and make cross-cohort comparison impossible
- A lead-level grain would simply replicate the raw data without adding analytical value

### 3.2 SQL Query

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

> **Data hygiene note:** The column `upload_to_first_attempt_P50 (hrs)` contains spaces and parentheses and requires backtick or quote escaping depending on the SQL dialect. Standardising column names to `snake_case` is recommended for production use.

### 3.3 Aggregated Output (12 of 16 cohorts shown — ≥ 30 leads)

> **Scope note:** The dataset contains 16 cohorts in total. Four cohorts — JobHai (two variants), "50K 2W4", and "50K 2W5" — collectively account for only ~7 leads combined and are excluded from this table because they fall below the 30-lead analytical threshold. They contribute no FT conversions and their per-cohort metrics are not statistically stable. The SQL query above returns all 16 rows; this table filters to the 12 cohorts with ≥ 30 leads for presentation clarity.

| Cohort | Leads | Attempted | Connected | Interested | Onboarded | FT Count | FT Rate | Connect Rate | Interest Rate |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Single Referral > 7 days- 24th Jul | 1,500 | 1,454 | 690 | 7 | 30 | 14 | 0.93% | 47.46% | 1.01% |
| Khanna- 2W 26th Jul | 1,546 | 1,376 | 574 | 20 | 39 | 14 | 0.91% | 41.72% | 3.48% |
| PreOb-Ob Fees Paid 29th Jul (set 1) | 1,483 | 1,433 | 650 | 103 | 15 | 7 | 0.47% | 45.36% | 15.85% |
| PreOb-Ob Fees Paid 29th Jul (set 2) | 1,558 | 1,519 | 735 | 122 | 14 | 7 | 0.45% | 48.39% | 16.60% |
| AI Connected but not Connected by TC- Set 1 | 1,480 | 1,331 | 694 | 29 | 9 | 5 | 0.34% | 52.14% | 4.18% |
| AI Connected but not Connected by TC- Set 2 | 1,193 | 1,066 | 466 | 26 | 6 | 2 | 0.17% | 43.71% | 5.58% |
| Khanna AI | 886 | 245 | 137 | 2 | 3 | 1 | 0.11% | 55.92% | 1.46% |
| OLX - Ashwin - 2W - 17 Jul | 5,182 | 962 | 372 | 9 | 3 | 4 | 0.08% | 38.67% | 2.42% |
| 2W3W - 3WAHD - Khanna - 3W - 17 Jul | 495 | 495 | 212 | 7 | 0 | 0 | 0.00% | 42.83% | 3.30% |
| 2W3W - 3WCNG - Khanna - 3W - 17 Jul | 530 | 530 | 291 | 1 | 0 | 0 | 0.00% | 54.91% | 0.34% |
| 2W3W - 3WEV - Khanna - 3W - 17 Jul | 201 | 201 | 91 | 1 | 0 | 0 | 0.00% | 45.27% | 1.10% |
| AI Connected band Not Interested | 2,137 | 1,359 | 637 | 21 | 0 | 0 | 0.00% | 46.87% | 3.30% |

### 3.4 What the Table Reveals

The table surfaces a striking **volume-conversion divergence**. OLX is the largest cohort by far (5,182 leads — more than 3× the next largest), yet it converts at just 0.08% FT rate. Several cohorts with fewer than 600 leads produce zero FT conversions despite high connection rates. In contrast, the top two cohorts (each around 1,500 leads) generate 14 FTs each. The operational implication is clear: telecaller effort should be reallocated toward the high-FT-rate cohorts rather than concentrated on the high-volume ones.

---

## 4. Part 3 — Machine Learning Model: Drivers of FT Conversion

### 4.1 Model Setup

| Parameter | Value |
|---|---|
| Algorithm | RandomForestClassifier |
| Number of trees | 200 |
| Max depth | 6 |
| Class weight | `balanced` |
| Target variable | `FT_after_upload` (binary) |
| Train/test split | 75% / 25% (random) |

### 4.2 Feature Selection and Leakage Avoidance

**Features used:**
- `Attempted`
- `Connected`
- `Attempt per Lead`
- `tag_filled`
- `Interested`
- `upload_to_first_attempt_P50 (hrs)`
- `lead_source` (label-encoded as `leadsource_enc`)

**Features deliberately excluded:**

`OB_after_upload` and `FT_after_first_attempt` were both excluded to prevent data leakage.

- `FT_after_first_attempt` is essentially a direct indicator of the target — including it would give the model the answer it is supposed to predict.
- `OB_after_upload` is on the causal path to FT: leads onboard before they take a trip, so this variable is downstream of the feature space but upstream of the target.

Including either would dramatically inflate model performance in training but produce a model that cannot be used in production — because at the point of scoring a new lead, neither value is yet known. The model must predict from information available **at the time of lead upload**, not from information that only becomes available later in the funnel.

### 4.3 Data Quality Issue: `upload_to_first_attempt_P50 (hrs)`

> **Important disclosure:** `upload_to_first_attempt_P50 (hrs)` is the top-ranked feature in this model at importance 0.319 — yet it has significant data quality problems that must be acknowledged before interpreting results.
>
> - **29% missing values:** 5,266 of 18,198 rows have no value for this column. These were imputed with the column median before training. Median imputation is a reasonable default but erases real variation — a lead with a missing value may have genuinely not been attempted, or the timestamp may simply be absent due to a logging gap. The model cannot distinguish between these two interpretations.
>
> - **8.2% negative values:** 1,496 rows contain negative values, with extremes as low as **-611 hours**. A negative "hours from upload to first attempt" is operationally impossible — it means the system recorded a call attempt *before* the lead was uploaded. This is a data pipeline bug (likely a timezone mismatch or logging order error), not a real signal. These rows were retained for training without correction, which means the model has partially learned from nonsensical timestamps.
>
> **What this means for the headline finding:** The conclusion that "speed to first attempt drives FT conversion" is directionally credible — it is consistent with the Part 1 cohort analysis and with general sales operations intuition. However, the specific importance score (0.319) should be treated as an upper bound rather than a precise estimate, because a portion of that signal is drawn from imputed and corrupted values. Cleaning this column — removing negatives, investigating missingness patterns, and distinguishing "not attempted" from "timestamp absent" — is the single highest-priority data quality fix before any production use of this model.

### 4.4 Class Imbalance

The dataset is approximately **99.70% no-FT and 0.30% FT**. This is a textbook severe imbalance. The `class_weight='balanced'` parameter adjusts the loss function to penalise misclassification of the minority class more heavily, giving the model a better chance of learning the FT signal despite its rarity.

Even with this adjustment, the imbalance is severe enough that standard accuracy is a misleading headline metric. A model that predicts "no FT" for every lead achieves 99.70% accuracy while being completely useless for the business purpose.

### 4.5 Model Performance

#### Classification Report

| Class | Precision | Recall | F1-score | Support |
|---|---:|---:|---:|---:|
| 0 — No FT | 1.00 | 0.76 | 0.87 | 4,536 |
| 1 — FT | 0.01 | 0.43 | 0.01 | 14 |
| **Overall accuracy** | | | **0.76** | **4,550** |

**Reading the numbers correctly:**

- **Accuracy (0.76)** is high because the model correctly predicts "no FT" for the vast majority of leads. It is not a useful signal here.
- **Recall (0.43)** means the model identified 6 out of 14 actual FT leads in the test set. This is the metric that most directly reflects business value — how many real trips we avoided missing.
- **Precision (0.01)** means that of all the leads flagged as predicted-FT, only 1% actually converted. The model casts a wide net to find the few positives.

#### Confusion Matrix

| | Predicted No FT | Predicted FT |
|---|---:|---:|
| **Actual No FT** | 3,465 (TN) | 1,071 (FP) |
| **Actual FT** | 8 (FN) | 6 (TP) |

#### Business Cost Interpretation

The two error types carry fundamentally different costs:

**False Negatives (8 missed FT leads):** These are actual FT-converting leads that the model labelled as no-FT. The business consequence is a missed follow-up and a lost driver signup. At a 0.30% base rate, every individual FT is a rare and valuable event — each false negative represents a real trip and associated revenue not captured.

**False Positives (1,071):** These are leads flagged as predicted-FT that did not convert. The business consequence is wasted follow-up effort — telecaller time and call costs spent chasing leads that were unlikely to convert. At precision = 0.01, the model effectively tells you to call ~1,077 leads to recover ~6 trips.

**Threshold implication:** The optimal operating threshold is a business decision. If the cost of a wasted follow-up call is low and the value of a driver's first trip is high, the right move is to lower the decision threshold (accept more FPs to reduce FNs). If follow-up capacity is the binding constraint, the threshold should be raised to improve precision at the cost of some recall. The 0.5 default threshold used here is not necessarily the right one for this problem.

### 4.6 Feature Importance

| Feature | Importance | Interpretation |
|---|---:|---|
| upload_to_first_attempt_P50 (hrs) | 0.3189 | Speed of first follow-up after upload *(see data quality note in §4.3)* |
| leadsource_enc | 0.2918 | Which cohort the lead came from |
| Attempt per Lead | 0.1676 | How many calls were made per lead |
| Attempted | 0.1418 | Whether any call was attempted |
| tag_filled | 0.0389 | Whether the lead record was tagged |
| Connected | 0.0330 | Whether a connection was made |
| Interested | 0.0080 | Whether lead expressed interest |

**Together, speed-to-first-attempt and lead source account for ~61% of all predictive power.** The remaining features contribute meaningful but smaller marginal signal.

### 4.7 Plain-English Driver Narrative

**Speed to first attempt (0.319)** is the single most important predictor. Leads contacted quickly after upload are more likely to convert. This aligns with the behavioural intuition: a driver who has just registered their interest and receives a call within minutes is more engaged than one who receives a call days later. Every hour of delay after upload represents a decay in lead quality. *Note: this importance score is subject to the data quality caveats described in §4.3.*

**Lead source / cohort (0.292)** is the second most important feature. Where a lead comes from is nearly as predictive as how fast you call them. This is fully consistent with Part 1 — some cohorts convert at nearly 1% while others convert at close to zero, and the model learns this signal from the data.

**Attempt per Lead (0.168) and Attempted (0.142)** are mid-tier signals. They capture persistence — how hard the team chases a lead. More attempts correlate with more trips, but these are secondary to the quality of the source and the speed of initial contact.

**Interested (0.008)** is nearly irrelevant as a predictive feature. This is the sharpest finding: expressed interest contributes almost nothing to the model's ability to separate FT from no-FT. This directly mirrors the Part 1 insight, where the highest-interest cohort had a lower FT rate than the top two. Interest is a weak and potentially misleading signal — operational decisions should not be made on the basis of this variable alone.

---

## 5. Integrated Business Recommendations

The three parts of this analysis tell a consistent story. Taken together, the recommendations are:

1. **Reallocate telecaller effort toward high-FT-rate cohorts.** Single Referral and Khanna 2W convert at ~0.92% FT rate vs. a dataset average of 0.30%. Prioritising these sources over high-volume-but-low-converting ones like OLX would meaningfully increase FT yield from the same team capacity.

2. **Reduce time-to-first-attempt as a primary operational metric.** Speed to first attempt is the single strongest predictor of FT conversion in the model. Setting and monitoring SLAs on how quickly uploaded leads receive their first call is likely to have a measurable positive impact on FT rate.

3. **Do not use "Interested" as a triage or prioritisation signal.** The model assigns it near-zero importance (0.008), and the cohort analysis shows that high-interest cohorts do not reliably produce high FT rates. Interest reflects something real about early intent, but it does not translate to trips.

4. **Tune the decision threshold to business cost.** If the team has the capacity to work ~1,000 predicted-FT leads to recover ~6 real trips, the current threshold is acceptable. If capacity is limited, raising the threshold to improve precision would focus effort on the leads most likely to convert.

5. **Investigate cohort #3 (PreOb-Ob Fees Paid).** Its anomalously high interest rate (15.85%) with a mid-tier FT rate (0.47%) suggests that something specific to this cohort breaks the interest→FT pathway. Understanding why these leads say "yes" but do not complete a trip is a high-value diagnostic question.

---

## 6. Limitations and Next Steps

### Current Limitations

- **Severe class imbalance** — with only ~54 FT-positive leads in 18,198 records, the model is trained on very limited positive signal. Any performance metric should be interpreted with this in mind.
- **`upload_to_first_attempt_P50 (hrs)` data quality** — 29% missing (median-imputed) and 8.2% negative (retained as-is). The top feature's importance score is inflated by corrupted values. Cleaning this column is the highest-priority pre-production fix.
- **Random train/test split** — the current evaluation randomly assigns 25% of records to the test set. This may overestimate real-world performance if cohort characteristics change over time.
- **Feature scope** — the model uses only the variables available in the dataset. Many plausible drivers of FT conversion are absent.

### Recommended Next Steps

| Initiative | Expected Impact |
|---|---|
| Clean `upload_to_first_attempt_P50 (hrs)` — remove negatives, investigate missingness | Produce a trustworthy top feature; re-evaluate importance scores |
| SMOTE or cost-sensitive oversampling | Improve recall on FT-positive leads |
| Threshold tuning via precision-recall curve | Optimise operating point to business cost ratio |
| Temporal train/test split (by upload date) | More honest out-of-sample evaluation |
| Add time-of-day and day-of-week features | Capture call timing effects |
| Add agent-level performance features | Isolate telecaller quality from lead quality |
| Track cohort stability over time | Assess whether FT rate is consistent or decaying per cohort |

---

## 7. Technical Appendix

### Tools and Libraries

- **Python 3** — core language
- **Pandas** — data loading, cleaning, aggregation
- **Scikit-learn** — RandomForestClassifier, train/test split, classification metrics
- **Matplotlib / Seaborn** — visualisations (feature importance chart, confusion matrix)
- **SQL** — Part 2 aggregation query (PostgreSQL-compatible syntax)
- **Jupyter Notebook** — full analysis and code

### Output Files

| File | Description |
|---|---|
| `top3_cohorts.csv` | Top 3 cohorts by FT rate (Part 1) |
| `aggregated_cohort_table.csv` | Full cohort aggregation (Part 2) |
| `cohort_summary_full.csv` | Complete cohort summary across all sources |
| `feature_importance.csv` | Feature importance scores (Part 3) |
| `feature_importance.jpg` | Feature importance bar chart |
| `confusion_matrix.jpg` | Model confusion matrix |

---

*George Jose — M.Sc. Data Science, CHRIST University, Bangalore*  
*GitHub: [georgejose055](https://github.com/georgejose055)*
