# Apex Retail Intelligence — Data Dictionary

## Source Datasets

### Customer (`customer_historical.csv`, `customer_incremental.csv`)
| Field | Type (Silver) | Notes |
|---|---|---|
| customer_id | int (PK) | Dropped if null/blank |
| age | double | Null → 0.0 |
| gender | string | Null → "Unknown" |
| income_bracket | string | Low / Medium / High |
| loyalty_program | string | Yes / No |
| membership_years | double | |
| churned | string | Yes / No — used in KPI 3 |
| marital_status | string | |
| number_of_children | double | |
| education_level | string | |
| occupation | string | |
| customer_zip_code | string | |
| customer_city | string | |
| customer_state | string | Used in KPI 3 |

### Product (`product_historical.csv`, `product_incremental.csv`)
| Field | Type (Silver) | Notes |
|---|---|---|
| product_id | int (PK) | Dropped if null/blank |
| product_name | string | |
| product_brand | string | |
| product_category | string | Used in KPI 4 |
| product_rating | double | 0–5 |
| product_review_count | double | |
| product_stock | double | |
| product_return_rate | double | 0–1, used in KPI 4 |
| product_size | string | |
| product_weight | double | |
| product_color | string | |
| product_material | string | |
| product_manufacture_date | string (timestamp) | |
| product_expiry_date | string (timestamp) | |
| product_shelf_life | double | days |
| unit_price | double | |

### Sales / Transactions (`sales_historical.csv`, `sales_incremental.csv`)
| Field | Type (Silver) | Notes |
|---|---|---|
| transaction_id | int (PK) | Dropped if null/blank |
| transaction_date | string (timestamp) | Used to derive dim_date |
| customer_id | int (FK) | |
| product_id | int (FK) | |
| quantity | double | |
| unit_price | double | |
| discount_applied | double | 0–1 fraction |
| payment_method | string | |
| store_location | string | Used as "region" in KPI 1 |
| transaction_hour | string/int | Used in KPI 5 |
| day_of_week | string | Used in KPI 5 |
| week_of_year | string/int | |
| month_of_year | string/int | |
| total_sales | double | Gross line revenue |
| promotion_id | string | FK to dim_promotion |
| promotion_type | string | Used in KPI 2 |
| holiday_season | string | Yes/No |
| season | string | |
| weekend | string | Yes/No |

## Silver Layer — Derived Columns

| Table | Added Column | Purpose |
|---|---|---|
| dim_customer | customer_sk | Surrogate key, unique per (customer_id, version) |
| dim_customer | effective_start_date / effective_end_date / is_current | SCD Type 2 history tracking |
| dim_product | product_sk | Surrogate key, unique per product_id |
| fact_sales_ledger | sales_sk | Surrogate key, unique per transaction_id |
| all Bronze tables | ingested_at | Ingestion timestamp for audit trail |
| all Bronze tables | source_batch | "historical" or "incremental" |

## Gold Layer — Star Schema

| Table | Grain | Key Columns |
|---|---|---|
| dim_customer | 1 row per active/historical customer version | customer_sk (PK), customer_id |
| dim_product | 1 row per product | product_sk (PK), product_id |
| dim_promotion | 1 row per promotion_id | promotion_sk (PK), promotion_id |
| dim_date | 1 row per calendar date seen in sales | date_sk (PK, yyyyMMdd int) |
| fact_sales | 1 row per transaction | sales_sk (PK), customer_sk / product_sk / promotion_sk / date_sk (FKs) |

## KPI Definitions

| # | KPI | Formula |
|---|---|---|
| 1 | Net Margin by Region | `SUM(total_sales) − SUM(total_sales × discount_applied)`, grouped by `store_location` |
| 2 | AOV by Promotion | `AVG(total_sales)`, grouped by `promotion_type` |
| 3 | Demographic Churn Heatmap | `SUM(churned='Yes') / COUNT(*) × 100`, grouped by `customer_state, loyalty_program` |
| 4 | Product Quality Index | `AVG(product_return_rate)` and `AVG(product_rating)`, grouped by `product_category` |
| 5 | Store Traffic by Hour | `COUNT(*)`, grouped by `transaction_hour, day_of_week` |
