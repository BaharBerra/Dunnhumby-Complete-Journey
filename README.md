# Dunnhumby Complete Journey: Customer Analytics and Coupon Targeting

An end-to-end analytics project built on roughly two years (102 weeks) of grocery retail data: **raw data → dbt modelling → statistics and machine learning in Python → a Power BI data story**.

The core question: **Where is sales growth coming from, which customers are we losing, and who should receive a coupon?**

## Key findings

| Topic | Finding |
|---|---|
| Sales | Sales grew 6.7% over the last 13 weeks. Growth came from basket value (+8.4%) while visits fell 1.5% |
| Segments | The loyal segment is 29% of households but 57% of sales. The only segment with falling visits is "Frequent small-basket" (-11.4%) |
| Future spend | The highest past-spend group spends about 8.9x more in the future than the lowest |
| Coupons | Households that previously bought the couponed product redeem about 5.5x more often |
| Basket | Pasta and pasta sauce are bought together about 10x more often than expected (lift 9.8) |
| Coupon model | On a time-based test (last 7 campaigns), sending coupons to the half with the highest model score reaches 86% of redeemers (50% under random selection) |

> All results describe **associations**, not causal effects. The model finds who is likely to redeem, not who spends more because of the coupon. Measuring the effect requires a test campaign with a holdout group, which the final report page proposes.

## Architecture

```
Raw CSVs (BigQuery: raw)
        │
        ▼
  dbt: staging → intermediate → marts → reports      (BigQuery: dbt_* dataset)
        │
        ▼
  Python (Colab): hypothesis tests, segmentation, classification, regression, Apriori
        │
        ▼
  BigQuery: ml_output (result tables)
        │
        ▼
  Power BI: displays tables only, no calculations
```

**Design choice:** All calculations live in dbt and Python; Power BI only displays results. This keeps the dashboard and the notebook consistent.

## Data

[Dunnhumby "The Complete Journey"](https://www.dunnhumby.com/source-files/): 2,500 households, 102 weeks, with transaction, product, campaign, coupon and coupon-redemption tables.

Raw files are not included in this repository. Download the dataset and load it into a BigQuery `raw` dataset.

## dbt layers

| Layer | Content | Models |
|---|---|---|
| `staging/raw` | Type and naming cleanup of raw tables | 8 |
| `intermediate` | Enriched transactions, baskets, weekly household activity, category co-occurrence, future spend | 10 |
| `marts` | Dimension and fact tables (household, product, campaign, date, transaction, basket, coupon) and ML datasets | 13 |
| `reports` | `rpt_` tables feeding the dashboard pages | 6 |

Key models:
- `mart_household_segmentation_features`: clustering inputs
- `mart_household_spend_dataset`: future-spend (CLV) target and predictors
- `mart_coupon_response_dataset`, `rpt_coupon_response`: coupon response (household × campaign)
- `mart_category_pairs`: basket co-occurrence

**Data quality:** uniqueness, not-null and relationship tests in `schema.yml` and the model yml files, plus row-count checks in `tests/`.

## Python analysis (Colab notebook)

| Section | Content |
|---|---|
| Cross-check | `rpt_coupon_response` values are recomputed independently from raw data and compared |
| A. Coupon redemption (H1–H7) | Logistic regression with household- and campaign-clustered standard errors, odds ratios |
| B. Future spend (H9–H11) | PPML (Poisson pseudo-maximum likelihood), 5-fold CV against a naive baseline |
| C. Segments (H8, H12, H13) | K-Means (k=4), stability analysis (bootstrap ARI/Jaccard), category profiles |
| D. Basket (H14, H15) | Apriori association rules, segment × coupon category test |
| E. Summary | One test family with Benjamini-Hochberg correction |
| F. Coupon model | Time-based test, ROC AUC, recall with confidence intervals, recall by segment |
| G. Output | Results written to the BigQuery `ml_output` dataset |

**Methodological choices**
- Time-based train/test split: train on earlier campaigns, test on the latest ones.
- Methods that handle repeated households (clustered standard errors, household-grouped CV, household-level permutation test).
- Baselines: constant prediction, single-feature model, gradient boosting comparison.
- Descriptive and inferential results are reported separately, and limitations are stated explicitly (small segments, limited segment stability).

## Power BI report

The report follows a storytelling flow:

Sales → Segments → Future spend → Order interval → Household journey → Coupon redemption → Who gets the coupon → ML → Basket affinity → Campaign recommendation

Each page has a finding as its headline, a "What to do" box and the next question.

## Setup

```bash
# 1. Dependencies
pip install dbt-bigquery

# 2. Define your BigQuery connection in profiles.yml (not included in the repo)

# 3. Run and test the models
dbt build

# 4. Run the notebook in Colab to write the ml_output tables
# 5. Rebuild the rpt_ models that depend on ml_output
dbt run --select tag:needs_ml_output
```

Notes:
- Replace the `database` value in `models/schema.yml` with your own GCP project ID.
- The notebook only writes to BigQuery when `WRITE_TO_BQ = True` (default `False`, to prevent accidental overwrites).
- The `ml_output` dataset must be in the same location as the dbt dataset, since BigQuery does not allow cross-location joins.

## Limitations

- **No causality:** The data has no control group, so the effect of coupons on spending cannot be measured.
- **Limited period:** About two years from a single retailer; seasonality and price effects are not modelled.
- **Segments are tendencies:** The bootstrap stability of k=4 is limited; segment boundaries are not sharp.
- **Small cells:** Some segments have very little coupon data, so rates for them are not interpreted.

## Tech stack

BigQuery · dbt · Python (pandas, scikit-learn, statsmodels, mlxtend) · Google Colab · Power BI · Fivetran

## Repository structure

```
├── models/
│   ├── staging/raw/        # raw data cleanup
│   ├── intermediate/       # intermediate transformations
│   ├── marts/              # dimensions, facts and ML datasets
│   └── reports/            # dashboard tables, ml_output source definition
├── tests/                  # row-count checks
├── dbt_project.yml
└── README.md
```

## Created By
- `Bahar Berra Uyar`
- `Tansu Uludağ`
- `Canseli Aydoğmuş`
- `Ümit Kargacı`
