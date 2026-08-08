# Vahan — Product Analytics Internship Case Study
**Candidate:** George | **Dataset:** 18,198 leads, one row per lead | **Target metric:** First Trip (FT) conversion

---

## Executive Summary

Across all three tasks, one theme keeps surfacing: FT conversion is rare (0.297% base rate) and concentrated in a handful of cohorts. The top three cohorts by FT rate all exceed 1,400 leads and convert at roughly 3x the base rate, and the fastest-to-first-attempt cohorts dominate — speed and cohort choice matter far more than call volume or expressed interest. Because FT conversion is so rare, accuracy alone is a misleading yardstick; the model and the ranking both need to be read through a "how much does this actually cost/move the business" lens. The practical takeaway: prioritize high-FT cohorts, follow up fast after upload, and treat interest as a leading indicator, not a conversion signal.

---

## Part 1: Top 3 Best-Performing Cohorts

**Metric decision.** I ranked cohorts by FT rate (`FT_after_upload / Uploaded Leads`) rather than raw FT count. Raw counts reward volume, which would automatically favor the biggest cohorts even if their conversion is poor. Rate normalizes for size and measures what we actually care about: how well a lead source turns *its own* leads into trips.

**Volume threshold.** I restricted the ranking to cohorts with **at least 30 uploaded leads**. A cohort with, say, 5 leads and 1 FT shows a 20% FT rate, but that single conversion is noise — you can't act on a sample that small. The 30-lead floor ensures the rates we're ranking are statistically meaningful and stable enough to build operations around.

**Result — top 3 cohorts:**

| Cohort | Leads | FT count | FT rate | Connect rate | Interest rate |
|---|---|---|---|---|---|
| Single Referral > 7 days- 24th Jul | 1,500 | 14 | 0.93% | 47.46% | 1.01% |
| Khanna- 2W 26th Jul | 1,546 | 14 | 0.91% | 41.72% | 3.48% |
| PreOb-Ob Fees Paid 29th Jul (set 1) | 1,483 | 7 | 0.47% | 45.36% | 15.85% |

**Insight worth flagging:** cohort #3 has a *far* higher interest rate (15.85%) than #1 and #2 (~1–3%), yet a *lower* FT rate. In other words, people in that cohort say "yes, I'm interested" easily but don't convert to actual trips at the same pace. If I were picking top cohorts purely on interest, I'd pick the wrong one. This suggests interest and FT measure different things here: interest reflects early intent, FT reflects actual driver behavior. A strong analysis flags this nuance rather than just printing the top-3 by one number — cohort #1 and #2 are the ones to double down on; cohort #3 is a candidate for deeper investigation into *why* interest doesn't translate into trips.

---

## Part 2: Aggregation Query

**Grain decision.** I aggregated at **`lead_source`** (one row per cohort). This is the actionable business unit — leads are acquired by source, and operational decisions (budget allocation, follow-up strategy) are made at this level. A date grain would fragment cohorts across days and make them impossible to compare, while a finer grain (e.g., per phone number) is just the raw data again and adds no analytical value.

**Data-hygiene note.** The source column `upload_to_first_attempt_P50 (hrs)` contains spaces and parentheses, which requires backtick/quote escaping depending on SQL dialect — worth standardizing column names to snake_case as a cleanup step.

**SQL (assuming table `raw_leads`):**

```sql
SELECT
    lead_source AS leadsource,
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

**Aggregated output — sample rows (top and bottom performers):**

| lead_source | total_leads | attempted | connected | interested | onboarded | ft_count | ft_rate_pct | connect_rate_pct | interest_rate_pct |
|---|---|---|---|---|---|---|---|---|---|
| Single Referral > 7 days- 24th Jul | 1,500 | 1,454 | 690 | 7 | 30 | 14 | 0.93 | 47.46 | 1.01 |
| Khanna- 2W 26th Jul | 1,546 | 1,376 | 574 | 20 | 39 | 14 | 0.91 | 41.72 | 3.48 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |
| 2W3W - 3WAHD - Khanna - 3W - 17 Jul | 495 | 495 | 212 | 7 | 0 | 0 | 0.00 | — | — |

**What the table reveals.** There is a wide spread of performance across cohorts. The biggest cohorts are not the best: OLX, for example, contributes 5,182 leads — the largest volume in the dataset — yet converts at a far lower FT rate than the smaller high performers. Large-but-weak cohorts generate a lot of calling effort (attempts/connects) for almost no trips, while lean cohorts like the top two above turn ~1,500 leads into 14 FT each. In plain business terms: volume and conversion diverge, so effort should be re-allocated toward the high-FT-rate cohorts rather than the high-volume ones.

---

## Part 3: ML Model — What Drives FT Conversion

**Model setup.** RandomForestClassifier with 200 trees, `max_depth=6`, and `class_weight='balanced'`. Target: `FT_after_upload` (binary). Features: `Attempted`, `Connected`, `Attempt per Lead`, `tag_filled`, `Interested`, `upload_to_first_attempt_P50 (hrs)`, and `lead_source` (label-encoded).

**Leakage-avoidance decision.** I deliberately **excluded `OB_after_upload` and `FT_after_first_attempt` as features**. Both are downstream of, or directly tied to, the target: `FT_after_first_attempt` essentially reveals the outcome, and `OB_after_upload` is on the path to FT. Including either would give the model the "answer key" and inflate performance that would never hold in production, where we score a lead *before* we know these values. The model predicts from what we know early in the funnel only.

**Class imbalance — dealt with head-on.** The dataset is 99.70% no-FT / 0.30% FT (only ~54 FT leads in 18,198). This is a textbook severe imbalance, and it's why I will not lean on the 76% accuracy figure:

| | precision | recall | f1-score | support |
|---|---|---|---|---|
| **0 (No FT)** | 1.00 | 0.76 | 0.87 | 4,536 |
| **1 (FT)** | 0.01 | 0.43 | 0.01 | 14 |
| **accuracy** | | | 0.76 | 4,550 |

Accuracy is 76% simply because the model can be wrong on *every* FT lead and still be "76% correct" by getting the massive no-FT majority right. That number hides the real story. The meaningful signals are recall (0.43 — the model caught 6 of 14 actual FT leads) and precision (0.01 — most of what it flags as FT is wrong).

**Confusion matrix** (rows = actual, columns = predicted):

| | Predicted No FT | Predicted FT |
|---|---|---|
| **Actual No FT** | 3,465 (TN) | 1,071 (FP) |
| **Actual FT** | 8 (FN) | 6 (TP) |

**Reading it in business terms.** The two error types have very different costs:
- **False negatives (8 missed FT leads):** lost driver signups. These are real trips we predicted we wouldn't get, so we never follow up — and each is revenue walking out the door. With FT so rare, *every* missed FT is expensive.
- **False positives (1,071):** wasted follow-up effort. We chase leads we predict will convert but mostly won't. Because precision is 0.01, chasing *all* predicted-FT leads means calling ~1,077 leads to catch ~6 trips. This is a cost problem, not a lost-revenue problem.

Given this asymmetry, the "right" threshold is a business decision: if a follow-up call is cheap and a trip is valuable, we should trade precision for recall and cast a wider net (accept more FPs to catch more FNs).

**Feature importances — what actually drives conversion:**

| Feature | Importance |
|---|---|
| upload_to_first_attempt_P50 (hrs) | 0.319 |
| lead_source (encoded) | 0.292 |
| Attempt per Lead | 0.168 |
| Attempted | 0.142 |
| tag_filled | 0.039 |
| Connected | 0.033 |
| Interested | 0.008 |

**Plain-English narrative.** Two features dominate: **speed-to-first-attempt (0.319)** and **cohort choice (0.292)**. Together they account for ~61% of predictive power. The story: leads called *fast* after upload convert — the hour we take to make the first call is one of the single biggest levers on FT. Where the lead comes from (lead_source) matters almost as much, consistent with Part 1 showing cohort quality varies wildly. Notably, **`Interested` is nearly useless (0.008)** — expressed interest does almost nothing to predict actual conversion, which echoes the Part 1 insight where the highest-interest cohort had a mid-tier FT rate. Call volume metrics (`Attempted`, `Attempt per Lead`) are mid-tier signals — they matter, but as proxies for persistence/effort, not as the driver. The implication for the team: prioritize which cohorts you push and how fast you call them, before worrying about how many times or whether the lead says "interested."

---

## Limitations & Next Steps

- **Class imbalance is the core constraint.** With ~54 FT leads total, the model is starved for positive examples. I'd try **SMOTE/oversampling or cost-sensitive learning** (weight the FT class up beyond the standard `class_weight='balanced'`) to improve recall, and evaluate with precision/recall and a profit-style metric, not accuracy.
- **Threshold tuning.** The 0.5 default decision boundary isn't the right one here. I'd tune the threshold to reflect the FN/FP cost ratio (missed trip revenue vs. wasted follow-up call) and pick the operating point that maximizes expected value.
- **More features.** Time-of-day / day-of-week of the call, agent-level performance (who made the call), response outcomes, and follow-up count would all plausibly add signal. Also, cohort creation-date effects (e.g., is a cohort's performance stable or decaying over time?) would strengthen both Parts 1 and 3.
- **Temporal validation.** I'd split train/test by date rather than randomly, to confirm the model generalizes to cohorts seen after training — random splits over-lean on the exact cohorts in the dataset and can flatter performance.
