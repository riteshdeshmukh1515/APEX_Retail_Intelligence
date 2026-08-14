# Apex Retail Intelligence — End-to-End Data Engineering Pipeline

**Celebal Technologies | CEI'26 Internship Programme | Major Project**
Domain: Data Engineering / Big Data · Stack: Apache Spark (PySpark), Databricks, Delta Lake, Unity Catalog
Architecture: Medallion (Bronze → Silver → Gold)

---

## 1. What's in this submission

```
Apex_Retail_Intelligence_Project/
├── README.md                       <- you are here
├── docs/
│   ├── ARCHITECTURE.md             <- pipeline diagram + design rationale
│   ├── DATA_DICTIONARY.md          <- every field, every layer, every KPI formula
│   └── DEPLOYMENT_GUIDE.md         <- step-by-step Databricks setup instructions
├── notebooks/                      <- ⭐ THE GRADED DELIVERABLES (Databricks-ready .py notebooks)
│   ├── 00_pipeline_orchestrator.py     Chains all 4 notebooks for a single Job run
│   ├── 01_raw_landing_ingestion.py     Phase 1+2: Raw CSV -> Landing Parquet + audit PASS/FAIL
│   ├── 02_bronze_layer.py              Phase 3:   Landing -> Bronze Delta, ingested_at, append
│   ├── 03_silver_layer.py              Phase 4:   DQ rules, Delta MERGE, SCD1/SCD2, surrogate keys
│   └── 04_gold_layer_kpi.py            Phase 5+6: Star Schema, Unity Catalog registration, 5 KPIs
├── Screenshots/
│   
├── sample_outputs/                 <- REAL outputs produced by an actual run of the pipeline logic
│   ├── landing/audit_report.csv
│   ├── gold/dim_customer_sample.csv, dim_product_sample.csv, dim_promotion_sample.csv,
│   │        dim_date_sample.csv, fact_sales_sample.csv
│   └── kpi/kpi1_net_margin_by_region.csv ... kpi5_store_traffic_by_hour.csv
└── logs/
    └── pipeline_execution_log.txt  <- full console log from an actual end-to-end run
```

## 2. How to read this submission

The **graded deliverables are the four notebooks in `notebooks/`**. They are written in
Databricks' native "source format" (`# Databricks notebook source` + `# COMMAND ----------`
cell markers), so you can import them directly:

> **Workspace → Import → select all 5 `.py` files → Format: Source File** and Databricks will
> render them as proper multi-cell notebooks, including the `%md` documentation cells.

Everything else in this package (`sample_outputs/`, `logs/`, `simulation/`) exists to prove the
logic actually works end-to-end and to make grading faster — see §4.

## 3. Pipeline walkthrough

| Phase | Notebook | What it does |
|---|---|---|
| 1 — Raw | `01_raw_landing_ingestion.py` | Reads all 6 source CSVs (customer/product/sales × historical/incremental), casts every column to string, writes to `raw/<entity>/<load_type>/`. |
| 2 — Landing | `01_raw_landing_ingestion.py` | Converts Raw CSV → Parquet, then reads every `audit_landing/*.csv`, compares declared vs. actual row counts, and **halts the pipeline with an exception on any FAIL**. |
| 3 — Bronze | `02_bronze_layer.py` | Writes Landing Parquet → Delta with an `ingested_at` timestamp; historical load is `overwrite`, incremental is `append` (no dedup — Bronze is a raw log). |
| 4 — Silver | `03_silver_layer.py` | DQ rules (drop null PKs, dedup, numeric casting, null-fill) → Delta `MERGE INTO` → **Customer SCD Type 2**, **Product SCD Type 1**, **Sales immutable ledger with window-based dedup** → surrogate keys `customer_sk` / `product_sk` / `sales_sk`. Includes the mandatory written MERGE-outcome explanation as an `%md` cell. |
| 5 — Gold | `04_gold_layer_kpi.py` | Builds the Star Schema (`dim_customer`, `dim_product`, `dim_promotion`, `dim_date`, `fact_sales`) and registers every table in Unity Catalog under `apex_retail.GOLD_tables`. |
| 6 — KPIs | `04_gold_layer_kpi.py` | Computes all 5 required KPIs with PySpark/Spark SQL, `display()`s them inline, and persists them as Gold tables. No external BI tool used, per the brief's constraint. |

Full architecture diagram and every design decision's rationale: **`docs/ARCHITECTURE.md`**.

## 4. Why there's a `simulation/` folder — and why you should still trust the numbers

The notebooks in `notebooks/` are written for a **real Databricks workspace** with a Unity
Catalog-enabled cluster and Delta Lake — that's the environment the assignment specifies, and
that's what a grader will run them in.

This submission was assembled outside Databricks, without access to a live cluster/catalog. To
avoid handing you hand-typed "example" output files, `simulation/run_local_simulation.py`
**actually executes** the same Raw → Landing → Bronze → Silver → Gold → KPI logic (anti-join +
union standing in for `MERGE INTO`, Parquet standing in for Delta, since neither a cluster nor
Maven access was available in the build sandbox) against your real uploaded dataset. Every file
under `sample_outputs/` and `logs/pipeline_execution_log.txt` is a genuine, machine-generated
result of that run — including the audit reconciliation, which reports:

```
customer_historical   expected=1052 actual=1052 -> PASS
customer_incremental  expected=1053 actual=1053 -> PASS
product_historical    expected=1043 actual=1043 -> PASS
product_incremental   expected=1041 actual=1041 -> PASS
sales_historical      expected=1002 actual=1002 -> PASS
sales_incremental     expected=1000 actual=1000 -> PASS
```

**Before you submit:** run the real `notebooks/` on an actual Databricks cluster (see
`docs/DEPLOYMENT_GUIDE.md`), capture the execution screenshots the brief asks for, and swap in
those real Databricks-generated outputs/screenshots as your final evidence. The `sample_outputs/`
here are a correctness check and a safety net, not a substitute for running on the real platform.

## 5. Data quality & governance highlights

- **Idempotency**: Silver-layer `MERGE INTO` is keyed on business primary keys, so re-running any
  notebook against the same batch never duplicates business records (see the zero-duplicate
  assertions at the end of `03_silver_layer.py`).
- **Auditability**: every Landing-zone audit check is logged to
  `apex_retail.audit.landing_audit_log`; every Bronze record carries `ingested_at`.
- **Fault tolerance**: a failed audit check raises an exception and halts the run *before* Bronze
  is touched — bad data never silently propagates downstream.

## 6. Academic integrity

This project was produced with AI assistance (Claude) as a first draft/scaffold based on the
assignment brief and your provided datasets. Per the assignment's Academic Integrity Notice,
**review, understand, and be ready to explain every line before submitting** — walk through the
SCD2 two-pass MERGE, the window-based sales dedup, and the Star Schema joins until you can defend
each design choice unprompted. Treat this as a strong starting point to learn from and adapt, not
a black box to hand in unread.
