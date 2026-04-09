# Olist E-Commerce Data Pipeline
### Medallion Architecture with PySpark and Delta Lake

----

## Overview

An end-to-end data engineering pipeline built on the Brazilian Olist e-commerce dataset. The pipeline ingests raw CSV data, applies layered transformations following the Medallion architecture, and produces business-ready aggregates for analytics consumption.

**Tools:** PySpark | Delta Lake | Databricks | Unity Catalog

----

## Architecture

```
/Volumes/olist_project/raw/olist_files/    ← Raw CSV storage (Unity Catalog Volume)
            ↓
olist_project.bronze.*                     ← Raw ingestion (6 Delta tables)
            ↓
olist_project.silver.*                     ← Cleaned & typed (6 Delta tables)
            ↓
olist_project.gold.*                       ← Business aggregates (3 Delta tables)
```

### Notebooks

| Notebook | Responsibility |
|---|---|
| 1. Data Inspection | Profiling raw CSVs, null analysis, schema review |
| 2. Raw → Bronze | Ingest CSVs to Delta with explicit schema enforcement |
| 3. Bronze Data Quality Check | Null percentages, row counts, anomaly detection |
| 4. Bronze → Silver | Type casting, null handling, standardization, flagging |
| 5. Silver → Gold | Business aggregations, OPTIMIZE, VACUUM |

---

## Dataset

[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

99,441 orders placed between 2016 and 2018 across Brazil. Contains order lifecycle data, customer locations, product catalog, payments, and reviews.

**Tables used:** orders, customers, order_items, payments, reviews, products

---

## Pipeline Design Decisions

**Why explicit schema in Bronze instead of inferSchema?**
inferSchema performs a two-pass scan (performance cost) and infers types at runtime — meaning a source schema change silently corrupts downstream tables. Explicit StructType enforces a contract with the source. If the source breaks that contract, the pipeline fails loudly at Bronze rather than propagating bad data to Silver and Gold.

**Why keep delivery date nulls in Silver instead of dropping or filling?**
`order_delivered_customer_date` being null means the order has not been delivered yet — it is business state, not missing data. Dropping these rows would remove in-progress orders from analysis. Filling with any value would be fabricating data.

**Why `payment_value` for revenue instead of `order_items.price`?**
`order_items.price` is the product list price before discounts, vouchers, and payment methods are applied. `payment_value` is what the customer actually paid — realized revenue. For finance reporting, only realized revenue is meaningful.

**Why two-level aggregation for revenue?**
Payments is a one-to-many table with orders — one order can have multiple payment rows due to installments or mixed payment methods. Joining payments directly to orders without pre-aggregating inflates row counts and overstates revenue. Payments are aggregated to order level before joining to customers.

**Why left joins in Silver instead of inner joins?**
Inner joins silently drop rows that don't match. A product with a null category still has real revenue attached to it. Left joins preserve all order_items rows and let null categories flow through to Gold where they are labeled explicitly.

**Why monthly grain for Gold revenue table?**
Business reporting cycle is monthly. BI tools can roll monthly grain up to quarterly or yearly with a simple SUM. Building at daily grain would produce a larger table with no analytical benefit for this use case.

---

## Data Quality Issues Found

| Finding | Table | Column | Details |
|---|---|---|---|
| 14 financial anomalies | orders | order_approved_at | Orders marked delivered with no payment approval recorded — potential logging failures or financial risk |
| 160 missing approvals | orders | order_approved_at | Total orders with null approval across all statuses — expected for canceled orders |
| 88.3% null comments | reviews | review_comment_title | Expected business behavior — most customers submit score only without written comment |
| 58.7% null messages | reviews | review_comment_message | Same as above — rows retained, scores valid |
| 1.85% null categories | products | product_category_name | Replaced with "unknown" — products still carry valid revenue |
| City name inconsistency | customers | customer_city | Free-text entry produced mixed case and whitespace — standardized to lowercase + trim |

---

## Key Insights From The Data

**Late delivery rate by state:**
- National average: 10.44%
- Worst state (AL): 23.93% — 2.3x the national average
- Best state: 2.88%

AL state showing late delivery rates more than double the national average suggests a systemic logistics issue with the carrier or route serving that region — not random variance.

**Review behavior:**
88% of customers submit a review score without any written comment. Any sentiment analysis on this dataset must account for extreme selection bias — only dissatisfied or highly satisfied customers tend to write comments.

---

## Known Limitations

- **Full reload on every run** — pipeline uses overwrite mode. No incremental loading. Re-running ingests and overwrites all data rather than processing only new records.
- **No orchestration** — notebooks are executed manually. No scheduling, dependency management, or retry logic.
- **No alerting** — DQ failures are logged but do not trigger notifications.
- **Static source** — pipeline designed for CSV batch ingestion. Does not handle streaming or CDC from a live transactional database.

---

## What I Would Do Differently in Production

**Incremental loading with Delta MERGE**
Replace overwrite mode with a MERGE operation keyed on `order_id`. Use `updated_at` watermark from source to pull only changed records. Handles late-arriving data correctly without full table scans.

**Orchestration with Apache Airflow**
Define pipeline as a DAG with explicit dependencies: Bronze must complete before Silver, Silver before Gold. Add retry logic, SLA alerts, and backfill capability.

**Formal DQ framework**
Replace custom null check functions with Great Expectations. Define expectations as code, version them, and fail the pipeline on critical expectation breaches rather than logging and continuing.

**CDC from transactional source**
Replace static CSV ingestion with Debezium capturing change events from a PostgreSQL source. Events stream into Bronze as they occur rather than batch ingestion once per day.

**Monitoring and alerting**
Alert on: row count drops >5% between runs, null rate increases beyond threshold, Gold revenue deviating >10% from previous period, pipeline runtime exceeding SLA.

---

## How to Run

1. Upload CSVs to `/Volumes/olist_project/raw/olist_files/`
2. Run notebooks in order: 1 → 2 → 3 → 4 → 5
3. Verify row counts match between layers before proceeding to next notebook
4. Gold tables available at `olist_project.gold.*` after notebook 5 completes

---