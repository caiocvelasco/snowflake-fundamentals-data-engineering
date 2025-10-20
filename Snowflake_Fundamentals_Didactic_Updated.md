# Snowflake Fundamentals

Welcome! This guide covers the **four essential pillars** of Snowflake mastery - ideal for interview prep, onboarding, or building solid mental models.

---

## Table of Contents

1. [Snowflake Architecture & Performance Fundamentals](#1-snowflake-architecture--performance-fundamentals)
2. Procedural SQL, Streams & Tasks  
3. S3 Integration & Data Loading  
4. Matillion Orchestration & ELT Design  

---

## 1. Snowflake Architecture & Performance Fundamentals

### 1.4 Compute Layer: Virtual Warehouses, Parallelism & Optimizer

#### 🧠 The Optimizer (The "Planner")

The **Optimizer** is the *brain* of query execution. It lives in the **Cloud Services Layer** and decides how your query should be executed before any compute starts.

**Responsibilities:**
1. Parse and rewrite SQL into an optimized plan.
2. Use metadata cache to choose which partitions to read (pruning).
3. Choose join/aggregation strategies (hash, broadcast, etc.).
4. Assign work to each compute node for parallel processing.

**Analogy:**  
> The optimizer is like a chef planning the entire recipe before cooking.  
> The compute layer just follows the plan.

**Example:**  
```sql
SELECT c.country, SUM(o.amount)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.country;
```
- The optimizer checks partition metadata.
- Determines 20% of partitions match filter.
- Chooses a hash join plan and distributes work across nodes.

---

### 1.8 Performance Optimization Techniques (with Clustering, Views & Profiling)

#### 🧩 Clustering (Organizing Data for Pruning)

**Definition:**  
Clustering organizes rows inside micro-partitions so that values of one or more columns are physically close together.

**Why it matters:**  
It improves **partition pruning** by making partitions more homogeneous.

**Example:**  
```sql
ALTER TABLE orders CLUSTER BY (country, order_date);
```
Now all rows with the same `country` and `order_date` ranges are near each other — pruning becomes sharper.

You can inspect clustering quality with:  
```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('orders');
```

**Analogy:**  
> Imagine sorting a library so that all “Europe” books are on one shelf — the librarian finds them faster!

---

#### 🧩 Views vs Materialized Views

| Type | Stores Data? | Updated Automatically? | When to Use |
|------|----------------|----------------------|--------------|
| **View** | ❌ No | N/A | For lightweight, dynamic queries. |
| **Materialized View** | ✅ Yes | ✅ Yes | For heavy or repeated aggregations. |

**Example:**
```sql
CREATE VIEW v_orders_summary AS
SELECT region, SUM(amount) AS total FROM orders GROUP BY region;

CREATE MATERIALIZED VIEW mv_orders_summary AS
SELECT region, SUM(amount) AS total FROM orders GROUP BY region;
```

The second query runs instantly since results are precomputed.

---

#### 🧩 Profiling (Measuring Query Performance)

**Definition:**  
Profiling = examining how a query actually ran — which steps were slow, how many partitions were read, and how compute was distributed.

**How to Access:**  
- Snowflake UI → **Query History → Profile**  
- SQL via `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`

| Component | Description | Example insight |
|------------|--------------|-----------------|
| **Query Plan Graph** | Visual DAG showing each step (scan, join, aggregate). | Detect unbalanced joins. |
| **Bytes scanned** | Total data read. | Too high = poor pruning. |
| **Partitions scanned** | How many micro-partitions accessed. | Should be low. |
| **Elapsed time per step** | Time for each operator. | Helps find bottlenecks. |

**Example Insight:**  
If profiling shows most time spent on “join,” try clustering or filtering earlier.

---

#### Fundamental Rule of Thumb

> **Snowflake doesn’t get slow — it just scans more data than necessary.**  
> Optimization = guiding the optimizer to touch as few micro-partitions as possible.
