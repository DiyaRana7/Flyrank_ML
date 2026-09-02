# Capstone Report — Content Refresh Opportunity Scoring

- **Author:** Diya Rana
- **Lane:** Refresh / Content Opportunity Scoring
- **Date:** September 2026

> **Decision-support note:** This project helps prioritize pages for human review. It does not claim to predict Google's ranking algorithm or prove that refreshing a page causes better search performance.

## Abstract

This project asks which content pages should be prioritized for review when editorial review capacity is limited. Using the 30,000-row anonymized FlyRank starter dataset, the analysis builds a Random Forest classifier from observable search-performance, freshness, age, and engagement signals and compares it with a transparent Week-4 baseline rule. Validation uses a client-grouped holdout so that clients in the test set are not present in the training set. On the held-out test set, the Random Forest achieved 0.878 accuracy, 0.839 precision, 0.957 recall, 0.894 F1, 0.968 ROC-AUC, 0.973 average precision, and 1.000 Precision@50, outperforming the baseline on every shared evaluation metric. The result supports using a model-assisted ranking workflow for review prioritization, while keeping final refresh decisions with human editors because the target is an observed current-window proxy rather than a clean future outcome.

## 1. Introduction / Problem Statement

Content teams often have more pages than they can manually review at once. This project addresses the practical question: **Which content pages should an editor review first?**

The unit of analysis is one anonymized content page. The intended output is a ranked review queue that helps an editor decide whether a page should be refreshed, improved, monitored, or left unchanged.

A false positive can waste editorial time, while a false negative can cause a potentially important page to be missed. A transparent rule provides an interpretable baseline, while machine learning can combine several observable signals.

## 2. Data

The project uses the **30,000-row anonymized starter dataset**.

- **Rows:** 30,000
- **Unique pages:** 30,000
- **Unique clients:** 32
- **Columns:** 44

The dataset contains observable content and search-performance fields including 90-day impressions, clicks, sessions and users; recent and previous 30-day performance; content age; freshness; CTR; average position; engagement; scroll measures; and AI traffic percentage.

### Target

The target is:

```text
trend_direction == "down"
```

This is an **observed current-window proxy**, not a clean future-window outcome. Therefore, results describe performance on this defined proxy and should not be presented as proof of future decline.

### Exclusions

The following fields are excluded from model features:

- `content_id`
- `client_id`
- `trend_direction`
- `trend_pct`

Identifiers are used only for grouping and validation checks. Outcome-derived trend fields are not used as predictive features.

The dataset is anonymized and public-safe; client names, domains, private queries, credentials, and raw URLs are not included in the public report.

## 3. Methodology

### 3.1 Features

The final feature set contains 21 observable signals:

```text
impressions_90d, clicks_90d, sessions_90d, users_90d,
engaged_sessions_90d, days_with_impressions, days_with_sessions,
impressions_last_30d, clicks_last_30d, sessions_last_30d,
impressions_prev_30d, clicks_prev_30d, sessions_prev_30d,
content_age_days, days_since_last_update, ctr, avg_position,
engagement_rate, scroll_rate, ai_traffic_pct
```

Missing numeric values are filled using feature medians.

### 3.2 Transparent Week-4 Baseline

The baseline score adds:

- **+2** if `days_since_last_update >= 180`
- **+1** if `days_since_last_update >= 365`
- **+2** if `impressions_90d >= 500`
- **+1** if `impressions_90d >= 5000`
- **+1** if `avg_position > 10`

A score of **4 or more** produces `REVIEW_REFRESH`; otherwise the page is assigned `MONITOR`.

### 3.3 Random Forest

A Random Forest classifier with 200 trees was used:

```text
n_estimators = 200
random_state = 42
class_weight = "balanced"
n_jobs = -1
```

The model was selected because it can combine multiple numeric signals and capture non-linear interactions without requiring a highly complex model.

### 3.4 Validation

A **client-grouped holdout** was used with `test_size=0.33` and `random_state=42`.

- Training rows: **18,690**
- Test rows: **11,310**
- Training clients: **21**
- Test clients: **11**
- Client overlap: **0**
- Training declining rate: **0.546**
- Test declining rate: **0.536**

The baseline and Random Forest are evaluated on exactly the same test rows.

### 3.5 Evaluation Metrics

The evaluation uses Accuracy, Precision, Recall, F1, ROC-AUC, Average Precision, and Precision@50.

**Precision@50** is especially relevant because the practical use case is a limited review queue.

## 4. Model Evaluation and Results

### 4.1 Random Forest Performance

| Metric | Random Forest |
|---|---:|
| Accuracy | **0.878** |
| Precision | **0.839** |
| Recall | **0.957** |
| F1 | **0.894** |
| ROC-AUC | **0.968** |
| Average Precision | **0.973** |
| Precision@50 | **1.000** |

The test-set declining base rate is **0.536**.

### 4.2 Baseline vs Random Forest

| Metric | Week-4 Baseline | Random Forest |
|---|---:|---:|
| Accuracy | 0.479 | **0.878** |
| Precision | 0.549 | **0.839** |
| Recall | 0.157 | **0.957** |
| F1 | 0.245 | **0.894** |
| Precision@50 | 0.680 | **1.000** |

The Random Forest substantially outperformed the transparent baseline on every shared metric.

Recall increased from **0.157 to 0.957**, while Precision@50 increased from **0.680 to 1.000**. This makes the model particularly useful for prioritizing a small review queue on the defined proxy.

These results demonstrate stronger performance on the evaluated task. They do **not** establish that the model predicts Google's ranking system or that refreshing a selected page will improve traffic.

### 4.3 Error Analysis and Feature Importance

The notebook also generates a confusion matrix, false-positive and false-negative counts, and a top-10 feature-importance table.

Feature importance is descriptive: it shows which signals the model relied on, not which signals cause search decline.

## 5. Limitations and Honest Framing

1. The target is an observed current-window proxy rather than a clean future-window outcome.
2. The model should not be presented as proof that it predicts future search performance.
3. Client-grouped validation reduces client memorization risk but does not establish performance across future time periods.
4. The analysis is observational and does not establish causality.
5. Feature importance shows model reliance, not causal impact.
6. A page may decline because of seasonality, consolidation, low-volume noise, or other factors.
7. The project does not demonstrate that a content refresh causes recovery or increased traffic.
8. Final actions require human editorial review.

A stronger future version would define a genuinely future-looking target, such as decline or recovery in the next 30 days using only information available before that prediction window, and would use time-aware validation.

## 6. Ranked Recommendations

1. **Start with the highest-ranked opportunities.** Use the ranking to focus limited editorial capacity.
2. **Read the reason code before acting.** The reason code explains why a page received priority.
3. **Verify the page actually needs a refresh.** Check whether it is outdated, incomplete, inaccurate, or no longer competitive.
4. **Rule out alternative explanations.** Check seasonality, consolidation, low-volume noise, and other changes.
5. **Choose the action manually.** Refresh, improve, monitor, or take no action.
6. **Do not automate publishing, deletion, merging, or rewriting from the score alone.**
7. **Improve the next model with a future-looking target and time-based validation.**

## 7. Reproducibility

Key notebook:

```text
work/notebooks/capstone.ipynb
```

Generated public-safe artifacts:

```text
work/outputs/capstone_metrics.json
work/outputs/capstone_action_summary.csv
work/figures/capstone_feature_importance.png
work/figures/capstone_model_vs_baseline.png
```

The notebook uses random seed **42**, client-grouped validation, the same test rows for both methods, and records the main metrics in a JSON receipt.

To reproduce the analysis:

1. Clone the repository.
2. Open `work/notebooks/capstone.ipynb`.
3. Run the notebook from top to bottom.
4. Confirm zero client overlap between train and test.
5. Confirm the baseline and model use the same test rows.
6. Confirm the reported metrics and generated artifacts.

Only public-safe aggregated results should be published.

## 8. Acknowledgments & Data Credit

**Built on the FlyRank ML Internship dataset**

Data and internship context: FlyRank — https://flyrank.ai

## Conclusion

The capstone shows that the Random Forest provides a substantially stronger ranking signal than the simple Week-4 rule for identifying pages associated with the observed declining-trend proxy. Its strongest practical result is **Precision@50 = 1.000**, alongside high recall and F1. The model is therefore useful as a **decision-support layer for prioritizing human review**, not as an automated explanation of Google's ranking system. A future-looking target and time-aware validation would be the most important next improvements.
