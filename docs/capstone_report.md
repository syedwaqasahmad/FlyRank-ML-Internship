---
title: Content Visibility Churn Report
---

# Capstone Report — Content Visibility Churn (Freestyle Lane)

- **Author:** Syed Waqas Ahmad
- **Lane:** Freestyle — Content Visibility Churn (inspired by customer churn analysis, reframed onto FlyRank's search data)
- **Repo:** https://github.com/syedwaqasahmad/FlyRank-ML-Internship
- **Date:** August 2026

## 0. Abstract
This study asks whether observable, pre-decision search signals — visibility, freshness, ranking position, and content depth — can predict which content pages will decline in search visibility ("churn"), well enough to prioritize a limited weekly content-review queue. Using a single-snapshot anonymized dataset of 30,000 pages across 32 clients, we compare a carefully audited hand-written rule against a Random Forest model under a client-holdout validation split. The honest result is nuanced: the rule holds up well at the very top of the ranked list, while the model edges ahead further down. Along the way, we caught and corrected two leakage issues — one in our own feature set, one hiding inside our own baseline formula — underscoring that honest validation matters more than a flashy headline number. This output is intended strictly as decision-support for a human content reviewer, not an automated publishing decision.

## 1. Problem Framing
**Unit of analysis:** one content page (`content_id`).
**Decision this supports:** which pages a content/SEO reviewer should check first each week, given they cannot review every page.
**Output:** a ranked queue combining a model probability score with human-readable reason codes.
**Action a human takes:** review the flagged page and decide whether it needs a content refresh.
**Cost of a wrong call:** a false positive wastes limited reviewer time on a page that was never at risk; a false negative means a genuinely declining page goes unreviewed and keeps losing visibility.
**Why ML helps here:** the ML-06 signal audit showed that individual signals (staleness, position, word count) each behave *counterintuitively* on their own (all three tested OPPOSITE of the naive hypothesis), while a *combination* of signals was strongly predictive (94.1% churn rate vs. 54.2% base rate, though covering only 0.06% of pages). A model can weigh many interacting signals jointly across the full dataset — a simple hand rule cannot.

## 2. Data Safety
**Data used:** `data/raw/content_refresh_anonymized.csv` — a single anonymized snapshot of 30,000 pages, 32 clients, 44 columns.
**Deliberately excluded:** `trend_pct` — the exact percentage `trend_direction` (our label) is computed from; confirmed as direct leakage in ML-05 (clean feature-set AUC 0.673 vs. 1.000 with `trend_pct` included). Any FlyRank product decision flags (health scores, workflow triage labels) were never present in this dataset by design. `client_id` and `content_id` are used only as grouping/join keys, never as model features.
**Confirmed:** no client names, domains, URLs, or keywords appear anywhere in `work/`.

## 3. Baseline
Our transparent hand rule scores pages using two reason codes: "stale and still visible" (not updated in 180+ days, but still earning 500+ impressions in 90 days) and "page-one decay risk" (ranking in the top 10 but over 180 days old). An earlier third reason code — "declining with demand" — was dropped after ML-08 revealed it was built directly from the label field itself (`trend_direction == "down"`), making it circular rather than genuinely predictive; the corrected baseline below excludes it.

This is a fair comparison because it's evaluated on the exact same client-holdout test split and metric as the model.

## 4. Model / Analysis
**Method:** Random Forest classifier (200 trees, balanced class weights) — chosen for handling a mix of numeric and categorical features without heavy preprocessing, and for being less overfitting-prone than a single decision tree.
**Target:** `is_churned` = 1 if `trend_direction == "down"`, else 0 (a same-window proxy, not a true forward-looking outcome — see Limitations in Section 6).
**Features used:** 15 numeric signals (impressions, search volume, competition, CPC, word/char count, days with impressions/sessions, content age, days since update, CTR, average position, engagement rate, scroll rate, AI traffic %) plus 8 categorical tiers (competition level, content type, main intent, age/freshness/word-count/impression/position tiers) — 54 columns total after encoding.
**Deliberately left out:** `trend_pct` (leakage), product flags (not present), raw ID columns (join keys only).

## 5. Evaluation
**Split:** client-holdout (`GroupShuffleSplit` grouped by `client_id`, 75/25, random_state=42) — not a random row split. This choice mattered enormously: a naive random split inflated Precision@20 from an honest 0.700 to an artificially inflated 0.950, simply by letting the model partially memorize client-specific patterns rather than learn transferable signal.

**Base rate (majority class):** 54.2% of all pages are "churned" (declining) — this is the number any precision score should be read against.

| k | Base rate | Fair baseline precision | Model precision |
|---|---|---|---|
| 20 | 0.542 | 0.75 | 0.70 |
| 50 | 0.542 | 0.66 | 0.68 |

Both the baseline and the model clearly beat the base rate at both k values — the real question is which beats the other, and the answer is: neither, consistently. The baseline is sharper at k=20; the model edges ahead at k=50.

**Error analysis (from ML-08):** of 7,115 held-out test rows, 3,034 were misclassified by the model. The model's top false-positive errors tended to be pages with moderate-to-high impressions and `trend_direction` of "up" or "stable" — i.e., pages the model flagged as risky based on staleness/position signals that were, in this snapshot, actually holding steady or growing. This suggests the model leans hard on `impressions_90d` and `avg_position` (the two highest feature importances, 11.0% and 10.3% respectively) and can over-flag genuinely stable-but-aging pages.

## 6. Interpretation
The Random Forest's top features by importance were `impressions_90d` (11.0%), `avg_position` (10.3%), `days_with_impressions` (9.0%), `content_age_days` (7.6%), and `days_with_sessions` (6.7%) — visibility and position dominate, which lines up with intuition.

A genuine surprise came from the ML-06 signal audit: staleness, position, and word count *each individually* behaved opposite to conventional SEO wisdom (stale pages churned *less* than fresh ones; low-ranked pages churned *less* than top-ranked ones; thin pages churned *far less* than long pages). Only the *combination* of staleness + visibility was a strong standalone signal (94.1% churn rate), and even then it covered just 17–35 pages out of 30,000. This is a well-understood "no effect" for the individual signals, and it's a valid, useful result: it means naive single-variable heuristics would have been actively misleading here.

## 7. Recommendation
The final output (ML-10) is a ranked action queue combining the model's probability score with human-readable reason codes (e.g. "stale and still visible," "high model risk"), so a reviewer can see not just *which* pages to check, but *why*. A FlyRank editor would use this tomorrow by pulling the top 20-50 rows each week as their review starting point — not as an automated publishing decision.

**Confidence:** moderate. Both the model and baseline clearly beat the 54.2% base rate, but neither dominates the other, and this has only been validated on one 30,000-page snapshot from 32 clients — not the full warehouse, a different time period, or clients outside this sample.

**Limits:** this is observational and single-snapshot; no causal claims are made. The `is_churned` label itself is a same-window proxy, not a true forward-looking outcome — a stronger future version would define churn using signals from an earlier window predicting a later, held-out window.

## 8. Reproducibility
**Random seeds:** `random_state=42` used throughout (train/test split and Random Forest).
**Re-run from a fresh clone:**
```bash
git clone https://github.com/syedwaqasahmad/FlyRank-ML-Internship
cd FlyRank-ML-Internship
pip install -r requirements.txt
# Then open work/notebooks/capstone.ipynb in Colab or Jupyter and Run All
```
**Environment:** standard `pandas`, `numpy`, `scikit-learn`, `matplotlib` (see `requirements.txt`).
**Metrics committed:** results table and chart are generated fresh each run inside `work/notebooks/capstone.ipynb`; the chart is saved to `work/outputs/capstone_results_chart.png`.

## 9. Acknowledgments & Data Credit
Built on the FlyRank ML Internship dataset. Learn more at [https://flyrank.ai](https://flyrank.ai).

---

**Claims checklist:** ✅ observed / measured / directional / decision-support language used throughout · ✅ base rate (54.2%) reported alongside all precision numbers · ✅ no causal claims · ✅ no "predicted Google's algorithm" language · ✅ no client-identifying details anywhere · ✅ numbers match the capstone notebook's fresh run.
