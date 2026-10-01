# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Shweta Gunjal
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/shweta1805/flyrank-ml-internship
- **Date:** October 2026

## 0. Abstract

This study investigates whether historical search-performance signals can help identify content pages that are likely to experience a meaningful decline in future Google Search impressions. Using the anonymized FlyRank ML Internship warehouse, the analysis constructs content-month observations from Google Search Console and GA4 search and engagement signals, with a decline defined as more than a 20% decrease in next-month impressions. A time-aware logistic regression model is compared with a Week-4 rule-based baseline, using March–December 2025 for training, January–March 2026 for validation, and April–May 2026 as a held-out test period. At a validation-selected threshold of 0.40, the logistic regression model achieved 0.565 precision, 0.958 recall, and 0.711 F1 on the held-out test period, compared with 0.579 precision, 0.381 recall, and 0.460 F1 for the baseline. The resulting workflow is intended to prioritize content pages for human review or refresh consideration and does not establish that content refreshes cause improvements in search performance.

## 1. Problem framing

This project supports the decision of which content pages should be prioritized for human review or refresh consideration.

The unit of analysis is one anonymized content item observed during one calendar month. For each content-month observation, the system uses currently observable search and engagement signals to estimate whether the page is likely to experience a meaningful decline in Google Search impressions during the following month.

The output is a prediction that can be used to prioritize pages for review. A human can then investigate the page and decide whether a refresh or other action is appropriate.

The cost of a wrong prediction is asymmetric: prioritizing a page that does not decline can consume review or editorial resources, while missing a page that later declines can delay investigation. For this reason, precision is an important evaluation metric, while recall and F1 are also reported to show the broader precision–recall trade-off.

The analysis is decision support rather than a causal system. A predicted decline does not establish why performance may change, and the results do not prove that refreshing a page will improve Google Search performance.

## 2. Data safety

The analysis uses the public-safe FlyRank ML Internship warehouse hosted on Hugging Face.

The primary table is `fact_content_daily_performance`, which contains daily search and engagement performance for anonymized content. Supporting reference tables include `dim_clients` and `dim_content`.

The modeling window covers March 2025 through May 2026, with June 2026 retained as the future outcome month for May 2026. Daily records are aggregated to the content-month level before modeling.

The predictive features are:

- GSC impressions
- GSC clicks
- GSC CTR, derived from clicks divided by impressions
- GSC average position
- GA4 sessions
- GA4 engaged sessions

The following information is deliberately excluded from predictive features:

- client names, domains, URLs, and private queries
- credentials and other private information
- client and content hash IDs
- report dates
- data-availability flags
- future-period performance
- the decline label and other label-derived fields
- product-generated fields that could encode the target

Future performance is used only to construct the target. A content-month is labeled as a decline when next-month impressions are less than 80% of current-month impressions. Future-period impressions are never supplied to the model as features.

GA4 missing values are treated as unavailable observations rather than automatically interpreted as zero. Missing-value handling is performed inside the modeling pipeline using median imputation with missingness indicators.

The analysis uses anonymized, public-safe data and does not include client-identifying information, private search queries, credentials, or raw private exports. The results are observational and predictive; they do not establish a causal effect of content refreshes on Google Search performance.

## 3. Baseline

The Week-4 baseline is a transparent rule-based scoring system that combines search-position opportunity and impression-volume opportunity.

Position points are assigned as follows:

- Positions 1–3: 0 points
- Positions 4–10: 2 points
- Positions 11–20: 3 points
- Positions 21+: 1 point

Impression volume is divided into low, medium, and high groups using thresholds derived from the training data:

- Low: 0 points
- Medium: 1 point
- High: 2 points

The total baseline score is the sum of position and volume points:

- Score 4–5 → Refresh
- Score 2–3 → Monitor
- Score 0–1 → Low priority

For evaluation, `Refresh` is treated as the positive prediction. The baseline is evaluated on exactly the same held-out April–May 2026 test period as the ML model.

On the held-out test period, the baseline achieved:

- Precision: 0.579
- Recall: 0.381
- F1: 0.460

The baseline provides a transparent comparison because its rules are simple, interpretable, and based only on current-period observable signals. Its thresholds are fixed independently of the held-out test outcomes.

## 4. Model / analysis

The analysis uses a time-aware logistic regression model to estimate whether a content-month observation will experience a more than 20% decline in Google Search impressions during the following month.

The target is defined as:

`decline_label = 1` when next-month impressions are less than 80% of current-month impressions.

The model uses six current-period features:

- GSC impressions
- GSC clicks
- GSC CTR
- GSC average position
- GA4 sessions
- GA4 engaged sessions

These features represent observable search visibility, click behavior, search position, and engagement signals available during the prediction month.

The preprocessing pipeline uses median imputation with missingness indicators for numeric features, followed by standardization. The classifier is logistic regression with balanced class weights.

The model does not use client or content identifiers, future-period performance, the decline label, or other label-derived fields as predictive features. Future performance is used only to construct the target.

This approach fits the Refresh / Content Opportunity Scoring lane because the output is a prediction that can be used to prioritize content pages for human review. The model identifies pages that may warrant attention; it does not determine the cause of a decline or automatically decide that a page should be refreshed.

## 5. Evaluation

The evaluation uses a time-aware split to preserve the forecasting setting:

- **Training:** March–December 2025 — 406,006 labeled observations
- **Validation:** January–March 2026 — 403,654 labeled observations
- **Test:** April–May 2026 — 368,222 labeled observations

The validation period was used to select the classification threshold. A threshold of 0.40 was selected before evaluating the test set because it produced the highest validation F1 among the tested thresholds, with validation precision of 0.339, recall of 0.947, and F1 of 0.499.

The final model and the Week-4 baseline were both evaluated on the same held-out April–May 2026 test period.

| Approach | Precision | Recall | F1 |
|---|---:|---:|---:|
| Week-4 baseline | 0.579 | 0.381 | 0.460 |
| Logistic regression | 0.565 | 0.958 | 0.711 |

The logistic regression model achieved substantially higher recall and F1 on this test period, while the Week-4 baseline had slightly higher precision. This represents a different precision–recall trade-off rather than a universal ranking of the two approaches.

### Model confusion matrix

At the 0.40 threshold, the logistic regression model produced:

```text
[[ 10108 151913]
 [  8601 197600]]
```

The model correctly identified 197,600 of the 206,201 observed decline cases in the test period, while 8,601 decline cases were missed. It also generated 151,913 false positives among the 162,021 non-decline cases.

The test period contained 206,201 decline cases and 162,021 non-decline cases, corresponding to an observed decline rate of approximately 56.0%. This temporal distribution differs from the training and validation periods and should be considered when interpreting the test results.

## 6. Interpretation

The model identifies a relationship between current-period search and engagement signals and the probability of a future impression decline. The held-out results show that the model can identify a large share of the observed decline cases, but the predictions also include many false positives.

The model's high recall at the selected threshold is useful when the cost of overlooking potentially declining pages is important. However, its precision of 0.565 means that a substantial number of pages flagged by the model did not subsequently meet the decline definition. The model should therefore be interpreted as a screening and prioritization mechanism rather than a definitive classification of content health.

The test period also has a substantially different decline rate from the training and validation periods. This distribution shift means the observed test metrics should not be assumed to represent performance in every future period.

The model does not establish which feature caused a decline, why a particular page declined, or whether a content refresh would reverse the decline. The analysis is predictive and observational rather than causal.

A useful interpretation of the result is therefore: the workflow can help surface pages for human investigation using currently observable search and engagement signals, while editorial or SEO decisions still require additional context and human judgment.

## 7. Recommendation

The output supports a ranked human-review workflow for content pages that may experience a future search-impression decline.

### Recommended workflow

1. **Prioritize pages flagged by the model** for initial review, especially when the predicted probability is high.
2. **Review current search performance** including impressions, clicks, CTR, and average position before taking action.
3. **Use the Week-4 baseline signals** as an interpretable secondary check for position and impression-volume opportunity.
4. **Investigate the page manually** for possible content, search-intent, technical, or other relevant factors before deciding on a refresh.
5. **Record the editorial decision and monitor future performance** rather than treating the model prediction as proof that a refresh is required.

### Confidence and limits

Confidence in the workflow is moderate. The logistic regression model achieved 0.565 precision, 0.958 recall, and 0.711 F1 on the observed April–May 2026 test period, but the test period had a substantially higher decline rate than the training and validation periods.

The model is therefore best used as a screening and prioritization tool. It does not explain the cause of a decline, guarantee that a page will decline, or establish that refreshing content will improve Google Search performance.

## 8. Reproducibility

The analysis can be reproduced from the project repository:

https://github.com/shweta1805/flyrank-ml-internship

The main capstone notebook is:

`work/notebooks/capstone.ipynb`

The notebook contains the warehouse connection, monthly aggregation, target construction, feature preparation, time-aware train/validation/test split, model training, threshold selection, held-out evaluation, baseline comparison, and result tables.

The analysis uses `random_state=42` for the logistic regression model. The train/validation/test periods are fixed by calendar month:

- Training: March–December 2025
- Validation: January–March 2026
- Test: April–May 2026

The Hugging Face warehouse is accessed using the `HF_TOKEN` secret in the Colab environment. The token itself is not stored in the repository.

The repository contains the completed capstone notebook and the earlier internship assignment notebooks used to develop the workflow. No credentials or private client data are committed to the repository.

A fresh rerun should execute the notebook from the warehouse setup through the final evaluation cells and verify that the reported metrics and tables are regenerated.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
