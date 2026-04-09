# Project Takeaways — Olist Pipeline

## What This Project Taught Me

---

### 1. Data engineering is about decisions, not just code

Every transformation in this pipeline required a justification:
- Why keep null delivery dates instead of dropping them?
- Why use payment_value instead of price for revenue?
- Why left join instead of inner join?

The code is secondary. The reasoning is what separates a data engineer from someone who runs notebooks.

---

### 2. Explicit over implicit — always

| Implicit (what beginners do) | Explicit (what engineers do) |
|---|---|
| inferSchema | StructType with named fields |
| Overwrite without thinking | Document overwrite limitations |
| Default VACUUM retention | RETAIN 168 HOURS explicitly stated |
| Inner joins | Left joins with documented reason |

Every implicit assumption is a future bug waiting to surface in production at 2am.

---

### 3. Bronze is a contract, not a copy

Bronze is not "CSV files in Delta format." Bronze is a schema contract with your source system. If the source violates that contract — renamed column, new column, corrupted row — the pipeline fails loudly at Bronze. Nothing bad reaches Silver or Gold.

FAILFAST mode enforces this. It is not optional in production.

---

### 4. Nulls have meaning — don't destroy them

The worst thing you can do with a null is fill it with something wrong:
- `order_delivered_customer_date = null` means order not delivered yet
- Filling it with `order_purchase_timestamp` means "delivered instantly"
- That fabricated data flows to Gold and produces wrong delivery time metrics

Ask why a null exists before touching it. Sometimes null is the correct answer.

---

### 5. The fan-out problem is silent and dangerous

Payments is one-to-many with orders. A naive join inflates row counts without any error or warning. PySpark will happily multiply your revenue by 3x and show you a clean result.

Always verify: `COUNT(*) vs COUNT(DISTINCT key)` after every join. If they don't match, you have a fan-out.

---

### 6. Z-ORDER is about cardinality and query patterns

Z-ORDER only helps when:
- The column has high cardinality (many distinct values)
- Queries frequently filter on that column

Z-ORDER on `order_status` (8 values) = almost useless
Z-ORDER on `order_purchase_timestamp` (thousands of values) = significant file skipping

Know why you're running a command, not just that you should run it.

---

### 7. What I would build next

**Gap identified:** This pipeline uses overwrite mode — it is a batch full-reload pipeline, not a production incremental pipeline.

**Next project goal:** Build an incremental pipeline using:
- Delta MERGE (upsert) instead of overwrite
- Watermark-based filtering to process only new records
- Simulated streaming source to practice CDC patterns

---

### 8. Concepts I need to go deeper on

- [ ] Delta MERGE syntax and performance optimization
- [ ] Apache Airflow DAG design for pipeline orchestration
- [ ] Partitioning strategy — when to partition vs Z-ORDER
- [ ] Great Expectations for production data quality
- [ ] Spark execution plan reading — understand what `.explain()` shows
- [ ] Broadcast joins — when and why to use them for small dimension tables

---

### 9. Interview questions this project prepares me for

**"Walk me through a pipeline you built"**
I can describe every layer, every decision, and every tradeoff from memory.

**"How do you handle nulls in a pipeline?"**
Not "drop them" — understand their business meaning first, then decide.

**"What is the Medallion architecture?"**
Bronze = raw contract, Silver = clean and typed, Gold = business aggregates.

**"What is a fan-out problem?"**
One-to-many join inflating row counts — always verify count after joins.

**"What does OPTIMIZE do in Delta Lake?"**
Compacts small files into larger ones. Z-ORDER sorts data within files to enable file skipping on high-cardinality filter columns.

**"What is time travel in Delta Lake?"**
Delta transaction log records every write operation. You can query any previous snapshot using VERSION AS OF or TIMESTAMP AS OF. VACUUM with short retention destroys this capability.

---

### 10. The habit that matters most

**Verify before proceeding.**


A pipeline that silently drops 10% of rows is worse than a pipeline that fails loudly. Silent data loss is the most dangerous bug in data engineering.

---
