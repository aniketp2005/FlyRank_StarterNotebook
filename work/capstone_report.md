# Capstone Report ? Refresh / Content Opportunity Scoring

- **Author:** Vaishnavi
- **Lane:** Lane 2 ? Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Vaish03166/FlyRank_Internship_StarterNotebook
- **Date:** 2026-08-07

## 0. Abstract

This capstone asks which existing content pages a reviewer should inspect first when review capacity is limited. It uses the approved pseudonymized FlyRank warehouse release, aggregated from the March 2026 daily-performance partition to one row per client/content page. A transparent visibility-and-position-volatility baseline is compared with logistic-regression and random-forest ranking models using client-held-out validation. The notebooks write Precision@20, Precision@50, average precision, base rate, and validation receipts at run time; this report intentionally does not invent values that have not been freshly generated. The output is an explainable human-review queue, not an automated refresh decision or a causal claim about outcomes.

## 1. Problem framing

The decision is how to allocate limited reviewer attention across an existing content inventory. One model row represents one pseudonymized content page within one pseudonymized client. The output is a ranked review queue with a score, a review/monitor recommendation, and a reason code.

A false positive consumes an early review slot, while a false negative leaves a page with the defined short-window decline signal lower in the queue. Precision@K is therefore more useful than overall accuracy: an editor acts on the top 20 or top 50 pages, not every row at once.

## 2. Data safety

The work uses `fact_content_daily_performance` from the approved FlyRank internship warehouse release, specifically the `month=2026-03` development partition. Its native grain is daily ? client ? content; the notebooks aggregate it to client ? content and keep pages with at least 100 impressions in the feature half of March.

The feature window is March 1?15. The March 16?31 impression total is used only to build the decline proxy. Pseudonymous `client_hash_id` and `content_hash_id` are used for grouping, validation, and approved queue identification only?never as model features. The final features exclude the outcome-window total, the label, product scores/flags, and all client-identifying content. No names, domains, URLs, titles, or raw queries appear in `work/`.

## 3. Baseline

The baseline ranks pages by prior-half impressions and doubles the score when prior position volatility is at least five. It gives priority to pages with meaningful historical visibility, especially where search position has been unstable. It is evaluated on the same held-out pages, label, and Precision@K metric as the learned models, so a model must earn its added complexity.

## 4. Model / analysis

The target is a short-window decline proxy: `impressions_outcome < 0.8 ? impressions_prev`, where the first quantity is measured in March 16?31 and the second in March 1?15. It is a measured future-within-month proxy, not a claim that a page needs a refresh or will recover after one.

Final features are `log_impressions_prev`, `log_clicks_prev`, `avg_position_prev`, `position_volatility_prev`, and `active_days_prev`. Logistic regression is the readable reference; random forest tests non-linear combinations of the same observable signals. Neither model uses `impressions_outcome`, `is_declining`, identifiers, product scores, or existing decision flags.

## 5. Evaluation

The reported evaluation uses `GroupShuffleSplit` with `client_hash_id`, a 25% test share, and random seed 42. Holding out entire clients prevents pages from the same client appearing in both train and test. The notebooks show a random-row split only as a diagnostic; the client-group result is the substantive one.

Precision@20 and Precision@50 match the review-capacity decision, while average precision provides whole-ranking context. The held-out positive rate is printed beside the metrics. `work/outputs/w05_model_metrics.json` records the baseline/model comparison and `work/outputs/w06_validation_audit.json` records grouped-versus-random and deliberate leakage-test results after a fresh run.

The error review shows three held-out wrong cases using pseudonymous IDs only. Errors may reflect temporary search movement, measurement noise, or unobserved signals. Feature importance describes how the fitted model used data; it is not causal evidence.

## 6. Interpretation

The analysis tests whether prior visibility, clicks, search position, position volatility, and active days prioritize this decline proxy better than the rule. A stronger learned score supports only the narrow claim that it ranks the proxy more effectively on held-out clients; it does not explain why a page declined.

The validation notebook intentionally adds an outcome-window total as a leaky feature. Any score increase from that version is an audit check, not a reportable model result. This confirms why the final feature set stays strictly before the outcome window.

## 7. Recommendation

A reviewer starts with the out-of-fold queue in `work/outputs/w07_action_playbook_queue.csv`. `visible_position_unstable` means meaningful prior visibility plus high position volatility; `visible_search_history` means meaningful prior visibility; `measurable_decline_risk` covers other measurable candidates. The recommended action is `REVIEW_FOR_REFRESH` only when the playbook score clears its threshold; otherwise it is `MONITOR`.

A human must check current search context, user intent, factual accuracy, technical accessibility, and whether the movement is temporary. This queue must never automatically delete, rewrite, publish, or change content. Investigate shifts in positive rate, Precision@K, missing-position share, or prior-impression distribution before reuse.

## 8. Reproducibility

From a fresh clone, run:

```bash
pip install -r requirements.txt
```

In Colab, add the approved read-only `HF_TOKEN` through Secrets or the environment?never in a code cell. Run `w05_model.ipynb`, `w06_validation_audit.ipynb`, `w07_action_playbook.ipynb`, and `capstone.ipynb` in that order. All model/split seeds are 42. Their metric receipts are saved under `work/outputs/` and make the results traceable without committing raw data.

## 9. Acknowledgments & data credit

Built on the approved pseudonymized FlyRank ML Internship dataset and starter materials. Data credit: [FlyRank](https://flyrank.ai).

---

**Claims checklist:** conclusions are observed, measured, directional, or decision-support. This work does not claim to predict Google?s algorithm, prove that a refresh causes recovery, or expose client-identifying information.
