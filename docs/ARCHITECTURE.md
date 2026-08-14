# Apex Retail Intelligence — Architecture

## 1. Medallion Architecture Overview

```
                 ┌────────────────────────────────────────────────────────────────┐
                 │                     INBOUND VOLUME (Unity Catalog)                │
                 │  historical_data/*.csv   incremental_data/*.csv   audit_*/*.csv   │
                 └───────────────────────────────┬────────────────────────────────┘
                                                   │  01_raw_landing_ingestion.py
                                                   ▼
   ┌────────────┐   all-string CSV   ┌────────────┐  Parquet + audit PASS/FAIL  ┌─────────────┐
   │  RAW ZONE  │ ─────────────────► │  LANDING   │ ───────────────────────────►│ Audit Log   │
   │ raw/<e>/<t>│                    │ landing/.. │                              │ (Delta)     │
   └────────────┘                    └─────┬──────┘                              └─────────────┘
                                            │  02_bronze_layer.py
                                            ▼
                          ┌───────────────────────────────────┐
                          │  BRONZE (Delta, append-only)        │
                          │  bronze_customer / _product / _sales│
                          │  + ingested_at, + source_batch      │
                          └───────────────────┬───────────────┘
                                               │  03_silver_layer.py
                                               ▼
        ┌──────────────────────────────────────────────────────────────────────┐
        │  SILVER (Delta MERGE, DQ rules applied)                               │
        │  dim_customer  (SCD Type 2 — effective_start/end_date, is_current)    │
        │  dim_product   (SCD Type 1 — overwrite in place)                      │
        │  fact_sales_ledger (immutable, window-deduplicated)                   │
        │  + customer_sk / product_sk / sales_sk surrogate keys                 │
        └───────────────────────────────┬────────────────────────────────────┘
                                         │  04_gold_layer_kpi.py
                                         ▼
   ┌───────────────────────────────────────────────────────────────────────────────┐
   │  GOLD — Star Schema (Unity Catalog: apex_retail.GOLD_tables)                    │
   │                                                                                  │
   │        dim_date        dim_promotion                                          │
   │            \               /                                                  │
   │             \             /                                                   │
   │   dim_customer ── fact_sales ── dim_product                                   │
   │                                                                                  │
   │  KPI tables: kpi_net_margin_by_region, kpi_aov_by_promotion,                   │
   │  kpi_demographic_churn_heatmap, kpi_product_quality_index,                     │
   │  kpi_store_traffic_by_hour                                                     │
   └───────────────────────────────────────────────────────────────────────────────┘
```

## 2. Design Decisions & Rationale

| Decision | Rationale |
|---|---|
| **All-string Raw zone** | Keeps ingestion resilient to upstream schema drift / dirty data — type errors surface at Silver, not mid-ingest. |
| **Parquet at Landing, Delta from Bronze onward** | Parquet is a cheap, dependency-free columnar hop; Delta is introduced once we need ACID transactions, `MERGE`, and time travel. |
| **Bronze append-only, no dedup** | Preserves a complete, replayable raw history — the single source of truth if Silver logic ever needs to be re-derived. |
| **Silver `MERGE INTO`, not watermarking** | The brief explicitly prohibits watermarking. Delta `MERGE` gives key-based idempotency that's immune to late-arriving data — a batch-friendly guarantee watermarking can't offer. |
| **SCD2 for customers** | Regulatory/marketing analytics often need "what did we know about this customer at time T" — full history is a real requirement, not an academic exercise. |
| **SCD1 for products** | Product attributes (price, stock, rating) are operational, not historical — the business only cares about the current state. |
| **Surrogate keys generated at Silver** | Decouples Gold-layer join performance from natural-key data quality (nulls, type drift, renames) in the source systems. |
| **Star Schema (not snowflake) at Gold** | Simpler joins, better BI-tool compatibility, matches the assignment's explicit `dim_*` / `fact_sales` table list. |
| **Audit halts the pipeline on FAIL** | Fault-tolerance in this brief means *fail loud and stop*, not *silently ingest bad data*. |

## 3. Idempotency & Fault Tolerance Summary

- **Landing**: audit reconciliation halts the run (raises an exception) on any row-count mismatch before Bronze is ever touched.
- **Bronze**: historical load is `overwrite` (safe to re-run); incremental load is `append` — by design, Bronze is a raw log, not a deduplicated table.
- **Silver**: every table is reconciled via Delta `MERGE INTO` keyed on the business primary key, so re-running the notebook against the same batch is a no-op (idempotent) at the business-record level, even though Bronze itself is an append log.
- **Gold**: rebuilt with `overwrite` from Silver on every run — Gold is a derived, disposable projection, so overwrite is the correct idempotent strategy here (no state to preserve beyond what Silver already tracks).

## 4. Unity Catalog Layout

```
apex_retail1 (catalog)
├── control.landing_paths          -- control table produced by notebook 01
├── audit.landing_audit_log        -- append-only audit history
├── bronze_tables.bronze_customer
├── bronze_tables.bronze_product
├── bronze_tables.bronze_sales
├── silver_tables.dim_customer     -- SCD2
├── silver_tables.dim_product      -- SCD1
├── silver_tables.fact_sales_ledger
└── GOLD_tables.dim_customer
    GOLD_tables.dim_product
    GOLD_tables.dim_promotion
    GOLD_tables.dim_date
    GOLD_tables.fact_sales
    GOLD_tables.kpi_net_margin_by_region
    GOLD_tables.kpi_aov_by_promotion
    GOLD_tables.kpi_demographic_churn_heatmap
    GOLD_tables.kpi_product_quality_index
    GOLD_tables.kpi_store_traffic_by_hour
```
