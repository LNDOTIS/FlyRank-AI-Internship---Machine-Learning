# Capstone Report — Predicting Near-Term Organic Search Decline

- **Author:** Long Nguyen
- **Lane:** Growth / Recovery / Momentum Prediction — Content Decline Detection
- **Repo:** https://github.com/LNDOTIS/FlyRank-AI-Internship---Machine-Learning
- **Date:** 2026-08-14

## 0. Abstract

>Can historical search-performance and content-lifecycle signals identify content pages that are likely to experience a meaningful decline in organic search impressions over the following 30 days?
 
I used the FlyRank pseudonymized warehouse release, working at the page-client prediction grain with a March 2026 feature window and an April 2026 target window. The first learned model was Logistic Regression using five pre-decision features: previous-30-day impressions, previous-30-day clicks, previous-30-day average position, days with impressions, and content age.  
On the initial Week-5 evaluation, Logistic Regression reached Precision@20 = 0.60 and Precision@50 = 0.46 versus the transparent baseline's 0.35 and 0.38, while the later grouped-by-client validation showed that performance was lower under the more honest split.  
The resulting score is therefore best treated as decision-support for prioritizing human review, not as an autonomous instruction to refresh, rewrite, delete, or change a page.

## 1. Problem framing

### Decision

The decision supported by this work is:

> **Which content pages should an SEO/content team inspect first because their historical signals indicate elevated measured risk of near-term organic-impression decline?**

The unit of analysis is **one content page for one client at one prediction point**.

The output is a **ranked risk score** plus a human-readable reason code and suggested review action.

A human reviewer can use the queue to:
- prioritize pages for inspection;
- review historical search exposure and ranking context;
- decide whether the page needs a freshness, relevance, technical, or search-intent investigation;
- choose between refresh, investigation, monitoring, leaving unchanged, or escalation.

The cost of a wrong call is asymmetric. A false positive consumes editorial/SEO review capacity. A false negative can delay investigation of a page that subsequently loses search exposure. Because the model does not establish causality, the intended value is prioritization rather than automatic intervention.

### Research question

> **Can historical search-performance and content-lifecycle signals identify pages likely to experience a >20% decline in organic search impressions over the next 30 days?**

The target is:

`is_declining = 1` when future 30-day impressions are more than 20% below the previous 30-day impressions.

The prediction setup used in Week 5 was:
- feature window: 2026-03-01 to 2026-03-31;
- prediction point: 2026-04-01;
- target window: 2026-04-01 to 2026-04-30.

## 2. Data safety

### Warehouse release

The work uses the **FlyRank internship warehouse release** rather than the small starter CSV.

The warehouse contains:
- `dim_clients`: client-level metadata;
- `dim_content`: content metadata;
- `fact_content_daily_performance`: daily page/client performance;
- `fact_content_query_90d`: query-mix context.

The daily fact table contains 78,835,655 rows and covers 2025-01-27 through 2026-06-30. The work uses DuckDB to aggregate the hosted Parquet data and brings only the modeling frame into pandas.

The modeling frame produced in Week 5 contained **100,893 page-client observations**.

### Features

The final Week-5 feature set was:

| Feature | Meaning |
|---|---|
| `imp_prev30` | previous-30-day impressions |
| `clk_prev30` | previous-30-day clicks |
| `avg_position_prev30` | previous-30-day average search position |
| `days_with_impressions` | number of days with observed impressions in the feature window |
| `content_age_days` | content age at the prediction point |

These are historical or lifecycle signals available before the prediction point.

### Deliberately excluded

The following were not used as model features:

- `imp_next30` and `clk_next30`: future target-window measurements;
- `is_declining`: the target itself;
- `impression_change_pct`: directly uses the future target window and was deliberately tested as a leaky feature;
- `trend_direction` and `trend_pct`: label-derived/sibling fields and therefore not valid predictors;
- product decision fields such as priority scores, health scores, action types, refresh tiers, or flags;
- `client_hash_id` and `content_hash_id` as numerical/predictive inputs. They are pseudonymous identifiers used only for joins, grouping, and validation.

### Leakage checks

The Week-6 audit found no future-window, label-derived, or product-decision contamination in the final five features.

A deliberately leaky experiment added `impression_change_pct`. The ROC-AUC increased from **0.65 to 1.00** under the random-split audit. This was treated as a successful leakage test, not as model performance. The feature was excluded from the final model.

No client names, URLs, raw queries, domains, titles, or other identifying information are used in this public report.

## 3. Baseline

The baseline was deliberately simple and transparent:

> A page receives a higher review score when it has meaningful historical search visibility and is stale enough to deserve attention; among qualifying pages, higher historical impressions receive a higher score.

The implementation used:
- a visibility threshold;
- a staleness threshold;
- historical impressions to rank qualifying pages;
- an explicit reason code for why the page entered the queue.

The baseline is a fair comparison because it answers the same operational question as the model: **which pages should be reviewed first?**

On the Week-5 evaluation frame, the baseline achieved:

| Ranking metric | Baseline |
|---|---:|
| Precision@20 | 0.35 |
| Precision@50 | 0.38 |
| Precision@100 | 0.36 |

The overall decline rate in the full Week-5 modeling frame was **51.48%** (51,944 declining vs 48,949 not declining). The test-set decline rate used in the Week-5 model comparison was **40.2%**.

This base rate matters: a ranking precision of 0.46 should not be interpreted without knowing that roughly 0.40 of the test observations were positive.

## 4. Model / analysis

### Method

I started with Logistic Regression because the task is binary classification with an observed future label, and because a readable model is useful for understanding the signals before adding complexity.

Random Forest was also evaluated as a stronger nonlinear alternative.

The Logistic Regression model is the main model discussed in this report because its behavior is easier to inspect and it already improved the transparent ranking baseline on the initial evaluation.

### Week-5 model comparison

On the same Week-5 evaluation frame:

| Model | Precision@20 | Precision@50 | Precision@100 | ROC-AUC | Precision | Recall | F1 | Base rate |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Logistic Regression | 0.60 | 0.46 | 0.54 | 0.62 | 0.472 | 0.656 | 0.549 | 0.402 |
| Random Forest | 0.45 | 0.44 | 0.43 | 0.58 | 0.436 | 0.604 | 0.506 | 0.402 |
| Transparent baseline | 0.35 | 0.38 | 0.36 | — | — | — | — | 0.402 |

The initial result therefore showed a measured ranking improvement over the hand-written baseline, particularly at the top of the queue.

The result should not be read as evidence that Logistic Regression is universally superior. The later grouped validation provides the more conservative view.

## 5. Evaluation

### Initial Week-5 validation

The initial Week-5 model evaluation was used as the development comparison.

It showed:
- Logistic Regression: Precision@20 = 0.60;
- Logistic Regression: Precision@50 = 0.46;
- Logistic Regression: ROC-AUC = 0.62;
- baseline: Precision@20 = 0.35;
- baseline: Precision@50 = 0.38.

### Honest grouped validation

Week 6 re-ran the Logistic Regression model with a **client-grouped split** so that no client appeared in both train and test.

The grouped split had:
- 84,384 training rows;
- 16,509 test rows;
- 32 training clients;
- 11 test clients;
- zero client overlap;
- test decline base rate = 0.402.

The before/after comparison was:

| Validation design | Accuracy | Precision | Recall | F1 | ROC-AUC | P@20 | P@50 | P@100 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Week-5 random split | 0.601 | 0.579 | 0.823 | 0.680 | 0.650 | 0.65 | 0.66 | 0.64 |
| Grouped by client | 0.512 | 0.440 | 0.786 | 0.564 | 0.621 | 0.60 | 0.46 | 0.54 |

The drop is itself an important result. It suggests that the initial random split gave a more optimistic estimate of generalization than the client-held-out evaluation.

The grouped result is the more appropriate evidence for the question:

> **Does the model work on pages from a client it did not see during training?**

### Error analysis

The Week-5 Logistic Regression evaluation contained:

- 9,355 correct predictions;
- 4,874 false positives;
- 2,280 false negatives.

Three high-scoring false positives illustrate the main difficulty:

| Example | Previous impressions | Previous clicks | Avg. position | Age | Model probability | Actual |
|---|---:|---:|---:|---:|---:|---|
| FP 1 | 42,185 | 4 | 9.10 | 70 days | 0.723 | not declining |
| FP 2 | 46,938 | 11 | 4.11 | 435 days | 0.671 | not declining |
| FP 3 | 41,655 | 13 | 5.06 | 257 days | 0.650 | not declining |

These cases show why a score is not a decision. Pages can have high exposure, limited clicks, or older lifecycle signals without actually declining in the next 30 days.

### Feature interpretation

Permutation importance under the grouped model was:

| Feature | Mean importance |
|---|---:|
| `days_with_impressions` | 0.0980 |
| `clk_prev30` | 0.0695 |
| `avg_position_prev30` | 0.0048 |
| `imp_prev30` | 0.0007 |
| `content_age_days` | -0.0029 |

The strongest measured contribution came from **days with impressions**, followed by **previous-30-day clicks**.

These results are directional rather than causal. They describe what the fitted model relied on when ranking this validation set; they do not show that changing a feature would cause future search performance to change.

## 6. Interpretation

Three observations are most useful.

### 1. Historical visibility is useful, but not sufficient

Pages with meaningful historical exposure provide a more useful population for review than pages with almost no observed search activity. This supports the basic idea behind the transparent baseline.

However, historical impressions alone do not identify future decline reliably enough to replace a learned ranking.

### 2. Engagement-related history matters

The grouped permutation audit placed previous-30-day clicks second among the five features. The Logistic Regression interpretation also showed an inverse relationship between historical clicks and the predicted decline probability, conditional on the other features.

This should be described as an observed model relationship, not as evidence that clicks protect pages from decline.

### 3. Freshness/decay is not a simple monotonic rule

The Week-7 descriptive freshness table was:

| Freshness bucket | n | Median previous impressions | Mean measured decline risk |
|---|---:|---:|---:|
| <=90 days | 33,070 | 745.0 | 46.63% |
| 91–180 days | 16,075 | 985.0 | 55.05% |
| 181–365 days | 37,434 | 881.5 | 53.83% |
| >365 days | 14,314 | 595.0 | 52.54% |

The pattern is **not monotonic**. The 91–180 day group had the highest measured risk, while the oldest group did not.

Therefore the defensible conclusion is:

> In this data, freshness is directionally associated with the measured review-priority signal, but the relationship is not a simple “older always means riskier” pattern.

Most importantly, the analysis does **not** show that refreshing a page causes traffic to improve.

## 7. Recommendation

The model output is a ranked review queue, not an automatic content-management system.

### Ranked actions

**1. High-risk + high-exposure — Prioritize for manual performance review**

Reason code: `high_risk_high_exposure`

Use when a page has a high model score and substantial historical search exposure.

Human question:
> If this page is genuinely at risk, is the potential value of reviewing it high enough to justify the review cost?

**2. High-risk + stale + visible — Review freshness and content relevance**

Reason code: `high_risk_stale_visible`

Human questions:
- Is the information still accurate?
- Has the topic changed?
- Does the page still match the search intent?
- Is the historical exposure meaningful enough to justify a refresh review?

**3. High-risk + weak position — Review search intent, relevance and ranking context**

Reason code: `high_risk_position`

Human questions:
- Has the search landscape changed?
- Does the page still match the intended query?
- Are there technical or indexing issues?
- Are competitors or SERP features changing the opportunity?

**4. High-risk + General manual review**

Reason code: `high_risk_general_review`

Use when the model assigns a high risk score but no single diagnostic signal is dominant enough to justify a more specific action.

**Suggested action**: broader manual review.

Human questions:
- What evidence makes this page look risky?
- Is the risk consistent with its historical search performance?
- Is there a content, relevance, technical, or search-intent explanation?
- Is there enough potential value to justify further investigation?

**Important**: this category should be treated as a fallback, not as evidence that a specific problem has been identified. The model is flagging the page for review; it is not diagnosing the cause of decline.

### Human-review checklist

Before taking action, a reviewer should check:

1. search intent;
2. content relevance;
3. freshness;
4. current search context;
5. historical evidence;
6. business context;
7. technical SEO issues;
8. measurement quality.

The final human decision should be recorded separately from the model recommendation.

### What should NOT be automated

The system should not automatically:

- publish content changes;
- rewrite a page;
- delete a page;
- change search intent;
- change canonical URLs;
- change internal linking strategy;
- change metadata without review;
- declare a page a ranking failure;
- claim that a refresh will improve traffic;
- claim a causal mechanism for Google's ranking;
- override business priorities.

### Cost/value thinking

The queue is most valuable when review capacity is scarce.

A sensible operating principle is:

> **Review the highest-risk, highest-exposure pages first, but only after checking that the page has enough observed history to make the review worthwhile.**

The cost of a false positive is reviewer time. The cost of a false negative is a missed opportunity to investigate a potentially declining page. The model does not estimate the monetary value of either outcome, so those costs remain a human/business decision.

### Monitoring and retraining

The queue should be re-evaluated when:

- Precision@K falls materially below the validated reference;
- ROC-AUC or ranking lift degrades across fresh periods;
- the decline base rate shifts substantially;
- the distribution of impressions/clicks/position changes;
- new clients or content populations differ materially from the development population;
- feature missingness or measurement coverage changes;
- the definition of the business action changes.

Retraining should be considered after a sustained distribution or performance change, not because one evaluation period is noisy.

## 8. Reproducibility

The project is organized as a sequence of notebooks:

1. `work/notebooks/w01_research_question.ipynb`
2. `work/notebooks/w02_ml_task_framing.ipynb`
3. `work/notebooks/w03_data_contract.ipynb`
4. `work/notebooks/w03_feature_leakage_check.ipynb`
5. `work/notebooks/w04_signal_audit.ipynb`
6. `work/notebooks/w04_baseline_score.ipynb`
7. `work/notebooks/w05_model.ipynb`
8. `work/notebooks/w06_validation_audit.ipynb`
9. `work/notebooks/w07_action_playbook.ipynb`
10. `work/notebooks/capstone.ipynb`

Repository:

https://github.com/LNDOTIS/FlyRank-AI-Internship---Machine-Learning

The main modeling environment uses Python, pandas, NumPy, DuckDB, Hugging Face access, scikit-learn, and matplotlib.

The warehouse is accessed through DuckDB over hosted Parquet data. The Hugging Face token is supplied at runtime rather than committed to the repository.

The principal random seed used in the modeling notebooks is **42**.

The warehouse grain was checked before aggregation, and the final feature set was audited for future-window, label-derived, and product-decision leakage.

## 9. Acknowledgments & data credit

Built on the **FlyRank ML Internship dataset**.

Data credit: [FlyRank](https://flyrank.ai)

This project was completed as part of the FlyRank ML Internship workflow and uses the pseudonymized internship warehouse release supplied for the program.

---

## Claims checklist

The report intentionally uses:

- **observed** for descriptive patterns;
- **measured** for evaluation results;
- **directional** for model interpretation;
- **decision-support** for recommendations.

It does not claim that refreshing content causes traffic improvement, does not claim to predict Google's ranking algorithm, and does not expose client-identifying information.

