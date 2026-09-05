# Methodology and Analysis Workflow

## 1. Study Objective
This project investigates delivery lead time in Brazilian e-commerce (Olist data) with three linked goals:

1. Quantify how much lead-time variability is explained by seller and geography.
2. Estimate the effects of routing and shipment characteristics on delivery time.
3. Check whether findings are stable under robust methods and resampling.

The analysis is framed as an experiment-based workflow: data audit -> cleaning -> feature engineering -> exploratory analysis -> multilevel modeling -> robustness and bootstrap checks.

## 2. Data Sources
We use the following tables from the Olist dataset:

1. `olist_orders_dataset.csv`
2. `olist_order_items_dataset.csv`
3. `olist_products_dataset.csv`
4. `olist_customers_dataset.csv`
5. `olist_sellers_dataset.csv`
6. `olist_geolocation_dataset.csv`
7. `product_category_matrix.csv` (project-level mapping table for theme features)

Main analysis script/notebook:

1. `code/delivery_multilevel_robust_bootstrap_v2.Rmd`

Derived analysis artifacts are saved in:

1. `code/output/` and `output/` (RDS/CSV, charts, tables)

## 3. Data Audit and Integrity Checks
We first perform structural checks before modeling:

1. Table dimensions and variable summaries.
2. Missingness report by table/column.
3. Duplicate key checks:
   - `seller_id` in sellers
   - `customer_id` in customers
   - `order_id` in orders
   - `(order_id, order_item_id)` in order_items
4. Referential integrity checks using anti-joins:
   - orders missing customers
   - order_items missing orders/products/sellers
5. Multi-seller order structure:
   - count of distinct sellers per order
   - share of orders with more than one seller

These checks establish whether the core keys and joins used later are reliable.

## 4. Outcome Construction and Initial Diagnostics
The raw outcome is delivery lead time:

1. `purchase_ts` from `order_purchase_timestamp`
2. `delivered_ts` from `order_delivered_customer_date`
3. `lead_time_days_raw = delivered_ts - purchase_ts` (in days)

We inspect:

1. Distribution shape (histogram, quantiles)
2. Missing/invalid cases
3. Tail behavior and plausible trimming thresholds

The outcome is right-skewed, so the model outcome is:

1. `log_lead_time = log1p(lead_time_days_raw)`

An auxiliary speed metric is also created:

1. `speed_z = -scale(log_lead_time)` (higher means faster delivery)

## 5. Cleaning Rules and Analysis Sample
The main cleaned sample follows strict, explicit rules:

1. Keep delivered orders with valid purchase and delivered timestamps.
2. Keep nonnegative lead time values.
3. Restrict to orders linked to exactly one seller.
4. Attach minimal geography (customer and seller state).
5. Create `is_same_state` indicator.
6. Add time controls from purchase timestamp:
   - purchase year
   - weekday
   - season (DJF, MAM, JJA, SON)

This ensures each modeled order has an unambiguous seller-level grouping and a valid outcome.

## 6. Feature Engineering
### 6.1 Shipment and monetary predictors
Order-level aggregates are built from order items and products:

1. `n_items`
2. `total_value`
3. `total_freight`
4. `total_weight_g`
5. `total_volume_cm3`

Transformations:

1. `log_items`, `log_value`, `log_freight`, `log_wgt`, `log_vol`
2. z-scored versions: `z_log_*`

### 6.2 Category and theme features
Product categories are mapped to order level by value-weighted dominant category:

1. Main category per order (`main_cat_pt`)
2. Rare categories collapsed to reduce sparse-level instability
3. Theme mapping via `product_category_matrix.csv`:
   - `is_hedonic`
   - `is_experience`
   - `theme_name`

### 6.3 Region and population enrichment
State-level population (2017) is merged, then mapped to macro-regions:

1. Customer and seller regions (`North`, `Northeast`, `Central-West`, `Southeast`, `South`)
2. `is_same_region`
3. Population terms and transformations (`log_customer_pop`, `log_seller_pop`, z-scores)

## 7. Locked Analysis Datasets
To keep modeling reproducible and stable across reruns, intermediate datasets are saved:

1. `orders_eda.rds/csv`
2. `orders_eda_strict.rds/csv`

`orders_eda_strict` is used for final model fitting and robustness sections.

## 8. Exploratory Data Analysis (EDA)
EDA is used to motivate model structure and variable blocks:

1. Lead-time variation by customer/seller state and region.
2. Same-state vs cross-state and same-region contrasts.
3. Shipment predictors vs outcome (correlations and binned trends).
4. Season and theme differences.
5. Correlation heatmaps for collinearity diagnostics.

The EDA supports progressive model building:

1. M0: variance decomposition only
2. M1: routing effects
3. M2: shipment effects
4. M3: season + theme controls

## 9. Core Statistical Models
All main models are linear mixed-effects models using `lme4::lmer`.

### 9.1 Null crossed random-effects model (M0)
Outcome:

1. `log_lead_time ~ 1 + (1 | seller_id) + (1 | customer_state)`

Purpose:

1. Estimate variance components
2. Compute ICC / proportion of variance due to clustering

### 9.2 Progressive fixed-effects models

1. M1: add routing (`is_same_state`, `is_same_region`)
2. M2: add shipment predictors (`z_log_wgt`, `z_log_vol`, `z_log_value`, `z_log_freight`, `z_log_items`)
3. M3: add season and theme

Model comparison tools:

1. Likelihood-ratio tests (`anova`)
2. AIC/BIC
3. Marginal/conditional R2 (`performance::r2`)

## 10. Robustness and AMS Extensions
Robustness checks are designed to reflect AMS topics and stress-test conclusions.

### 10.1 Robust location/scatter and outliers

1. MCD estimator (`robustbase::covMcd`) on multivariate shipment predictors
2. Robust Mahalanobis distances and chi-square thresholding
3. Outlier diagnostics via distance plots and MCD diagnostic plots

### 10.2 Robust univariate summaries

1. Mean, 10% trimmed mean, winsorized mean, median
2. MAD, IQR, Huber M-location
3. State-level robust lead-time summaries

### 10.3 Robust regression checks (flat model)

1. OLS baseline
2. Huber M-regression (`MASS::rlm`)
3. LTS regression (`robustbase::ltsReg`)

### 10.4 Clustering on seller effects

1. Extract seller random intercepts from mixed model
2. Fit Gaussian mixture models (`mclust::Mclust`) with BIC model selection
3. Optional robust clustering (`tclust`) when package/setup allows

### 10.5 Bootstrap inference

1. Nonparametric OLS bootstrap (pairs resampling)
2. Nonparametric OLS residual bootstrap
3. Parametric mixed-model bootstrap (`lme4::bootMer`)

Target coefficients:

1. same-state
2. same-region
3. freight
4. item count
5. JJA season effect

### 10.6 Additional sensitivity checks

1. Drop correlated predictor (`z_log_vol`) and compare key coefficients
2. Tail trimming (winsorize at 95th percentile) and refit final model
3. Region-specific refits
4. Robust mixed-effects refit (`robustlmm::rlmer`) where feasible

## 11. Interpretation Scale
Main coefficients are estimated on the log scale and converted to percentage effects using:

1. `% change = 100 * (exp(beta) - 1)`

This supports practical interpretation of routing, freight, and seasonal effects in expected delivery-time terms.

## 12. Reproducibility and Execution Notes
Key reproducibility decisions:

1. Fixed random seed for stochastic procedures.
2. Locked intermediate datasets for stable reruns.
3. Explicit package checks for optional robustness modules (`mclust`, `tclust`, robust packages).
4. Consistent sample definition for core models and sensitivity checks.

Implementation note:

1. Some chunks were hardened to avoid package conflict issues (e.g., masked verbs from `plyr`), using explicit namespace calls and base-R alternatives for counting operations.

## 13. Limitations
Important design/data limitations considered in interpretation:

1. Outcome is full order-level lead time; first-mile vs last-mile delays are not separable.
2. Seller metadata is limited (no direct capacity/service-level variables).
3. Route distance/carrier-level operational data is not fully observed.
4. Some robustness modules depend on local package availability and numerical conditioning.

## 14. Deliverables Produced
Main outputs generated by this workflow:

1. Cleaned and strict modeling datasets (`RDS` and `CSV`)
2. EDA visualizations and summary tables
3. Progressive mixed-model results (M0-M3)
4. Robustness outputs (MCD, robust regressions, clustering, bootstrap summaries)
5. Final interpretation sections with effect-size translation

