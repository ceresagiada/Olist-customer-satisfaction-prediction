# Olist Customer Satisfaction Prediction

Academic group project realised at the end of a Data Analytics intensive bootcamp at @Neoland, analysing customer satisfaction for a real Brazilian e-commerce dataset provided by Olist for @Kaggle.

---

## Project Overview

The primary objective of this project is to consolidate and clean a relational dataset of nearly 100,000 orders to build a predictive model that identifies factors driving customer satisfaction.

- **Objective 1**: Build and compare machine learning models to predict if a customer will be "completely satisfied" (Review Score = 5) based on order characteristics.
- **Objective 2**: Identify key factors for satisfaction to lead business strategies.

---

## Workflow Architecture

### 1. The Dataset

Olist Brazilian E-Commerce dataset ([Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)), consisting of 9 relational tables, of which 8 were used for this analysis, for a total of 46 columns and ~100k data entries.

| Table | Description |
|---|---|
| `olist_orders_dataset.csv` | Orders and dates |
| `olist_products_dataset.csv` | Product characteristics |
| `olist_customers_dataset.csv` | Customer information |
| `olist_sellers_dataset.csv` | Seller data |
| `olist_order_items_dataset.csv` | Items per order |
| `olist_order_payments_dataset.csv` | Payment information |
| `olist_order_reviews_dataset.csv` | Customer reviews |
| `product_category_name_translation.csv` | Category name translations (PT → EN) |

---

### 2. Data Cleaning & Preprocessing

The initial phase focused on transforming 8 raw relational tables into a single, modeling-ready dataset. The full pipeline is documented in the preprocessing notebook.

[View the Data Preprocessing Notebook](Olist%20-%201_Preprocessing.ipynb)

#### Data Consolidation
All 8 tables were merged using `order_id` as the primary key, with reviews as the base table (left join), producing a single flat dataset of ~99K rows and 24 columns, exported as `Olist_clean_data.csv`.

#### Cleaning Decisions by Table

**Orders**
- Timestamps normalized to daily granularity (time component removed).
- `delivered_status` derived as binary flag (1 = delivered, 0 = otherwise).
- `waiting_time` computed as days elapsed from purchase to delivery; undelivered orders imputed with `max_waiting_observed + 30` days to penalize without distorting the model.
- Redundant date columns dropped; only `order_purchase_timestamp` retained.

**Products & Categories**
- 70+ granular product categories translated from Portuguese to English and consolidated into **6 broad groups**: Home and Decoration, Electronics and Technology, Fashion and Personal Care, Leisure/Toys/Arts, Tools and Construction, and Miscellaneous.
- Dimensional nulls (weight, length, height, width) imputed with **median**; metadata nulls (description length, photos count) imputed with **0**; missing categories labeled as **"Unknown"**.
- `product_name_lenght` column dropped (no added value).

**Customers & Sellers**
- Zip code and city columns dropped to reduce fragmentation.
- State codes mapped to **5 Brazilian regions**: Southeast, South, Northeast, North, Central-West.

**Payments**
- Only 2.25% of orders used more than one payment method → simplified approach adopted.
- Dominant payment method (highest payment value) selected per order.
- Aggregated into 3 variables: `payment_type` (dominant method), `payment_value_sum` (total paid), `number_payments` (total installments).

**Reviews**
- ~1.3% of orders had multiple reviews → **most recent review kept** (reflects latest customer sentiment, e.g. after issue resolution).
- Text fields (`review_comment_title`, `review_comment_message`) excluded (text analysis out of scope).

**Items**
- Merged with Products table before aggregation.
- For multi-item orders: product characteristics (category, seller, etc.) taken from the **most expensive item** (most likely satisfaction driver); dimensional measures aggregated as **max**; financial measures aggregated as **sum**.
- Engineered variables: `number_items`, `total_price`, `total_freight_value`.



---

### 3. Predictive Analysis & Modeling

The full modeling pipeline is documented in the predictive analysis notebook.

[View the Predictive Analysis Notebook](Olist_PredictiveAnalysis.ipynb)

#### Target Variable
`review_score` (1–5) was binarized into `good_review`:
- **1** → Review score = 5 ("completely satisfied")
- **0** → Review score ≤ 4 ("not completely satisfied")

#### Feature Engineering
Beyond cleaning, the following features were engineered for modeling:

| Feature | Description |
|---|---|
| `seller_topN` | Binary flag: seller is in the top ~20% that drive 80% of sales (Pareto) |
| `product_topN` | Binary flag: product is in the top ~30% that drive 80% of sales (Pareto) |
| `repeat_customer` | Binary flag: customer had previously placed an order on the platform |
| `review_delay` | Days between purchase date and review submission date |
| `order_purchase_season` | Season of purchase (Summer, Fall, Winter, Spring — Southern Hemisphere) |

#### Feature Selection & Preprocessing
- `delivered_status` and `number_payments` dropped for excessive polarization (>95% concentration in a single value → negligible discriminative power for tree-based models).
- `customer_state` and `seller_state` minor regions regrouped into "Other" to reduce dimensionality.
- `payment_type` minor methods (voucher, debit card) grouped into "Other".
- `total_price` and `total_freight_value` dropped after multicollinearity analysis (VIF > 10); `payment_value_sum` retained as it captures both.
- Categorical features one-hot encoded; numerical features left in original scale (tree models are robust to skewness and outliers).

#### Models Trained & Compared

**Train/test split**: 80/20 with stratification and fixed random state (42) for reproducibility.

Four classification models were built and evaluated:

| Model | Key Hyperparameters | Class Imbalance Handling |
|---|---|---|
| **Decision Tree** | `max_depth=4` | `class_weight='balanced'` |
| **Random Forest** | `max_depth=6`, 100 trees | `class_weight='balanced'` |
| **XGBoost** | `max_depth=3`, 100 rounds | `scale_pos_weight` (ratio of negatives/positives) |
| **Logistic Regression** | `max_iter=100000`, RFE (6 features) | `class_weight='balanced'` |

#### Evaluation Metrics
All models were evaluated on: **Accuracy**, **Sensitivity (Recall)**, **Specificity**, **F1-Score**, and **ROC-AUC**.

---

### 4. The Dashboard

An interactive Power BI dashboard was built on the consolidated cleaned dataset to visualize business insights.

[View the interactive Dashboard through Power BI](Olist%20-%204_Dashboard.pbix)
[View the Dashboard as pdf](Olist%20-%204_Dashboard.pdf)

**Key KPIs displayed:**
- **98K** Total Orders | **R$ 16M** Total Revenue | **R$ 13M** Total Profit | **17.31** Average Delivery Days | **4.10** Average Review Score

**Visualizations included:**
- Average Review Score vs. Delivery Time (days) — clear negative correlation
- Orders by Delivery Time (days and period) split by satisfaction
- Orders by Review Score (distribution across 1–5 scale)
- Orders by Product Category and Satisfaction
- Average Satisfaction by Region (map view)

---

## Results & Business Impact

**Key insight from the dashboard**: Review score drops sharply and consistently as delivery time increases, making **delivery time the primary driver of customer satisfaction**. Orders delivered within 1 week have significantly higher satisfaction than those taking 1–2 weeks or more.

**Key business findings:**
- Delivery speed is the most impactful lever on customer satisfaction.
- ~57K orders (≈58% of total) received a 5-star rating; improving delivery performance is the clearest path to increasing this ratio.
- The Home & Decoration and Electronics categories generate the highest order volumes and are therefore the highest-priority segments to optimize.
- The Southeast region concentrates ~70% of orders, making it the primary target for logistics investment.

---

## Tech Stack

| Tool | Usage |
|---|---|
| **Python 3** | Core programming language |
| **pandas** | Data manipulation and merging |
| **NumPy** | Numerical operations |
| **scikit-learn** | ML models (Decision Tree, Random Forest, Logistic Regression), preprocessing, evaluation |
| **XGBoost** | Gradient boosting classifier |
| **statsmodels** | VIF multicollinearity analysis |
| **Matplotlib / Seaborn** | Exploratory data visualization |
| **Power BI** | Interactive business intelligence dashboard |
| **Jupyter Notebook** | Development and documentation environment |
