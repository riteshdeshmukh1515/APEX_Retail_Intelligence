# Deployment Guide — Running This Pipeline on Real Databricks

## 1. Prerequisites
- A Databricks workspace with **Unity Catalog** enabled.
- A cluster (or Serverless SQL/Compute) running **Databricks Runtime 13.3 LTS or later** (ships
  with Delta Lake built in — no extra library install needed).
- CAN_MANAGE or CAN_CREATE permission on a Unity Catalog catalog (default catalog name used
  throughout: `apex_retail1` — change via notebook widgets if yours differs).

## 2. Upload the source data
1. In Catalog Explorer, create a Volume, e.g. `apex_retail1.landing_zone.inbound_data`.
2. Upload the **entire** `Datasets/` folder structure you were given into that Volume, preserving
   the subfolders exactly:
   ```
   inbound_data/
   ├── historical_data/{customer,product,sales}/*.csv
   ├── incremental_data/{customer_incremental,product_incremental,sales_incremental}/*.csv
   ├── audit_landing/*.csv
   └── audit_silver/*.csv
   ```
   (The brief explicitly calls out uploading *all* the audit files and load files — don't skip
   `audit_landing/` or `audit_silver/`.)

## 3. Import the notebooks
1. Workspace → your folder → **Import**.
2. Select all 5 files from `notebooks/` at once, format **Source File** (Databricks auto-detects
   the `# Databricks notebook source` header and multi-cell structure).
3. Confirm each imported file opens as a proper notebook with rendered `%md` cells, not a flat
   script.

## 4. Configure widgets (top of each notebook)
| Widget | Default | Set to |
|---|---|---|
| `catalog_root` | `/Volumes/apex_retail1/landing_zone/inbound_data` | your Volume path from step 2 |
| `pipeline_root` | `/Volumes/apex_retail1/pipeline/data` | any Volume/path with write access for raw/landing artefacts |
| `bronze_catalog` / `catalog` | `apex_retail1` | your Unity Catalog catalog name |
| `bronze_schema` | `bronze_tables` | leave as-is or rename |
| `silver_schema` | `silver_tables` | leave as-is or rename |
| `gold_schema` | `GOLD_tables` | **must match the brief's required schema name exactly** |

## 5. Run order
Either:
- **Option A (recommended for grading):** run `00_pipeline_orchestrator.py` — it chains all four
  phase notebooks via `dbutils.notebook.run` and stops immediately if any phase fails.
- **Option B (for step-by-step screenshots):** run `01` → `02` → `03` → `04` individually, in
  order, taking a screenshot of each notebook's final output cell as required by the "Silver
  Layer Script / Notebook" deliverable acceptance criteria.

## 6. Verify
```sql
SHOW TABLES IN apex_retail1.GOLD_tables;
SELECT * FROM apex_retail1.GOLD_tables.fact_sales LIMIT 10;
SELECT * FROM apex_retail1.GOLD_tables.kpi_net_margin_by_region;
```

## 7. Simulating an incremental (daily) run
1. Upload a new day's `*_incremental.csv` + matching `audit_landing/*.csv` /
   `audit_silver/*.csv` files into the same Volume paths (overwrite or versioned subfolder,
   depending on your ingestion convention).
2. Re-run `00_pipeline_orchestrator.py`. Because Silver uses `MERGE INTO`, previously-seen
   transactions/customers/products are reconciled (not duplicated) and only genuinely new/changed
   records flow through to Gold.

## 8. Troubleshooting
| Symptom | Likely cause | Fix |
|---|---|---|
| `LANDING AUDIT FAILED` exception | Row count in a CSV doesn't match its `audit_landing/*.csv` | Confirm you uploaded the correct/complete CSV; check for a trailing blank line or extra header. |
| `AnalysisException: Table already exists` on first run | A previous partial run created a table | Drop the specific Bronze/Silver/Gold table and re-run, or use the `overwrite` cells as-is (they self-heal). |
| Duplicate active `customer_id` assertion fails | Ran Silver notebook twice against changed logic mid-development | Drop `apex_retail.silver_tables.dim_customer` and re-seed from a clean historical + incremental run. |
