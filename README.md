# Introduction to Data Engineering - Via Snowflake Fundamentals

Welcome! This guide covers the **four essential pillars** of Snowflake mastery - ideal for interview prep, onboarding, or building solid mental models. At the same time, it's s soft-landing introduction to **modern data engineering** concepts.

---

## Table of Contents
[What to Expect](#0-what-to-expect)  

**Block 1 - A bit of DevOps & Architecture**

1. [Snowflake Architecture & Performance Fundamentals](#1-snowflake-architecture--performance-fundamentals)  
   - [1.1 Why Snowflake Exists](#11-why-snowflake-exists)  
   - [1.2 The Three-Layer Architecture](#12-the-three-layer-architecture)  
   - [1.3 Storage Layer: Data & Micro-Partitions & Pruning](#13-storage-layer-data--micro-partitions--pruning)  
   - [1.4 Compute Layer: Virtual Warehouses & Parallelism](#14-compute-layer-virtual-warehouses--parallelism)  
   - [1.5 Cloud Services Layer: The “Brain”](#15-cloud-services-layer-the-brain)  
   - [1.6 Caching: Remembering Previous Work](#16-caching-remembering-previous-work)  
   - [1.7 Security Overview - Understanding RBAC](#17-security-overview---understanding-rbac)  
   - [1.8 Performance Optimization Techniques](#18-performance-optimization-techniques)  
   - [1.9 Visual Summary](#19-visual-summary)  
   - [1.10 Quick Recap Checklist](#110-quick-recap-checklist)  

**Block 2 - Data Transformation & Automation**

2. [Procedural SQL, Streams & Tasks](#2-procedural-sql-streams--tasks)  
   - [2.1 Stored Procedures & Snowflake Scripting](#21-stored-procedures--snowflake-scripting)  
   - [2.2 Incremental Strategies- Full Incremental Loads vs. Partitioned Incremental Loads](#22-incremental-strategies-full-incremental-loads-vs-partitioned-incremental-loads)  
   - [2.3 Understanding MERGE Statements - Performing Efficient Upserts](#23-understanding-merge-statements---performing-efficient-upserts)  
   - [2.4 Streams - Tracking Table Changes (Change Data Capture)](#24-streams---tracking-table-changes-change-data-capture)  
   - [2.5 Tasks - Scheduling & Automation](#25-tasks---scheduling--automation)  

**Block 3 - Data Ingestion & Orchestration & (some) Data Governance**

3. [S3 Integration & Data Loading](#3-s3-integration--data-loading)  

4. [Orchestration & ELT Design](#4-orchestration--elt-design)  

---

## What to Expect

This guide follows the natural flow of a modern data platform - from **how Snowflake works under the hood**, to **how we build and automate transformations**, and finally **how we ingest and orchestrate data end to end**.

# Block 1 - A bit of DevOps & Architecture  
Understanding how Snowflake works under the hood - its architecture, performance layers, and optimization principles.

### 1. Snowflake Architecture & Performance Fundamentals  
We begin with the *foundations*.  
Before building any pipeline, we must understand how Snowflake actually runs queries - its three-layer architecture (storage, compute, and cloud services), how data is partitioned, cached, secured, and optimized. This section builds the mental model every engineer needs to reason about performance and cost.

### 2. Procedural SQL, Streams & Tasks  
Once we know how Snowflake executes queries, we move into *building logic*.  
Here we learn how to:
- Write data transformation logic inside Snowflake (procedures and scripting)  
- Apply incremental patterns with `MERGE`  
- Detect changes automatically with **Streams (CDC)**  
- Automate everything with **Tasks**  
In other words, this section turns Snowflake from a query engine into a full **data-transformation and automation platform**.

### 3. S3 Integration & Data Loading  
Until now, we’ve assumed data already existed inside Snowflake tables.  
In reality, most data originates elsewhere - in raw CSVs, JSON API dumps, Parquet datasets, or partner S3 feeds.  
This section shows **how to connect Snowflake to external storage (S3)** and **load data efficiently and securely** into internal tables - the ingestion bridge between the outside world and your analytical layer.

### 4. Orchestration & ELT Design  
Finally, once ingestion and transformation logic are ready, we bring everything together under a governed orchestration layer.  
Here we see how Matillion can visually model ELT flows, manage dependencies, and deploy jobs that tie ingestion, transformation, and scheduling into a cohesive, maintainable pipeline.

> Think of these four sections as consecutive layers of maturity:  
> **Architecture → Transformation → Ingestion → Orchestration.**  
> By the end, you’ll understand not only *how Snowflake works*, but *how to operate it as a complete data platform*.
---

Let's begin!

## 1. Snowflake Architecture & Performance Fundamentals

### 1.1 Why Snowflake Exists

Before Snowflake, most companies used **on-premises data warehouses** such as Oracle, Teradata, or SQL Server.  
These systems were powerful, but they suffered from three major limitations:

| Problem | Description |
|----------|--------------|
| **Rigid scalability** | To handle more data or users, you had to buy and install new hardware. |
| **Compute & storage tightly coupled** | You paid for both, even when idle. A heavy query could block everyone else. |
| **Complex maintenance** | Patching, backups, tuning indexes - all manual. |

Snowflake was born as a **cloud-native data warehouse** designed to solve these issues through:
- **Elastic scalability** → pay only for what you use  
- **Separation of compute and storage** → multiple teams can query the same data independently  
- **Automatic optimization** → no manual partitioning or indexing required  

---

### 1.2 The Three-Layer Architecture

Snowflake is built around a **3-layer architecture**, each layer with its own purpose.

| Layer | Description | Analogy |
|-------|--------------|----------|
| **Storage Layer** | Stores all data in *micro-partitions* (compressed, columnar chunks of ~16 MB). | The library shelves holding books (data). |
| **Compute Layer** | Executes queries through *virtual warehouses* (clusters of compute nodes). | Librarians reading and summarizing books. |
| **Cloud Services Layer** | Coordinates everything: SQL parsing, optimization, metadata, RBAC, caching. | The library manager who knows where every book is and assigns tasks. |

---

### 1.3 Storage Layer: Data & Micro-Partitions & Pruning

#### Fundamentals
- Data is stored in immutable, compressed **micro-partitions** (~16 MB uncompressed).
- Each partition keeps **metadata**: min/max values, row counts, null counts.
- Snowflake automatically decides how to partition - no manual setup.
- This metadata allows **partition pruning** (skipping irrelevant data).

First, let's make clear the difference between **S3 partitioning** and **Snowflake micro-partitioning**.

#### S3 Partitionning vs. Snowflake Partitioning

| Concept | Where it happens | Who defines it | Example |
|----------|------------------|----------------|----------|
| **Folder partitioning** | In S3 (external data) | You | `s3://sales/year=2025/month=10/day=01/` |
| **Micro-partitioning** | Inside Snowflake | Automatic | Snowflake splits the table into small internal chunks |

**S3 partitioning** helps *organize* data before loading and optimizes query performance.  
**Micro-partitioning** helps *query* data efficiently after loading, so it's also a layer of query performance.

#### Example

Suppose you have 1 billion orders loaded into Snowflake:

| Partition | Date range | Row count |
|------------|-------------|------------|
| P1 | 2023-01-01 → 2023-03-31 | 250 M |
| P2 | 2023-04-01 → 2023-06-30 | 250 M |
| P3 | 2023-07-01 → 2023-09-30 | 250 M |
| P4 | 2023-10-01 → 2023-12-31 | 250 M |

Query:
```sql
SELECT SUM(amount)
FROM orders
WHERE order_date >= '2023-07-01';
```

Snowflake automatically scans only partitions **P3** and **P4**.  
This is **partition pruning**, and it happens inside the **storage layer**.

---

### 1.4 Compute Layer: Virtual Warehouses, Parallelism & Optimizer

#### Fundamentals
- A **virtual warehouse** = compute cluster that runs queries.
- Each warehouse can be sized (`X-SMALL` to `4X-LARGE`) and scaled *up* or *out*.
- Multiple warehouses can access the same data **independently** (no locking).
- Compute cost = active warehouse time × size.

#### Parallel Processing

Snowflake uses **Massively Parallel Processing (MPP)**:
- Each warehouse has many compute nodes.
- Each node processes a subset of micro-partitions **at the same time**.
- Results are combined and returned.

**Analogy**  
> Instead of one person reading 1000 books sequentially,  
> 100 people each read 10 books and share their summaries.  
> That’s parallelism. This saves a lot of time (100x faster in this case)!

#### Example
```sql
SELECT region, SUM(amount)
FROM orders
GROUP BY region;
```
- The Snowflake **"Optimizer"** divides a table into micro-partitions.
- 8 compute nodes (in a MEDIUM warehouse) each read a portion in **parallel**.
- Nodes send partial results → Snowflake merges them → result returned.

---

### 1.5 Cloud Services Layer: The “Brain”

The **Cloud Services Layer** is Snowflake’s *central nervous system*.  
It coordinates all query operations and ensures that storage, compute, and user permissions work together smoothly.  
No data is stored here - it’s all **metadata, orchestration, and optimization**.

---

#### The Optimizer - The Query "Planner"

When you submit a SQL query, the **Optimizer** (inside this layer) becomes the planner and strategist.

**Its goal:**  
-> Transform your SQL statement into the *most efficient execution plan possible.*

**Key Responsibilities:**
1. **Parse and rewrite SQL** - validate syntax, resolve references, and simplify logic.  
2. **Build an execution plan** - decide how to access tables, join them, and aggregate results.  
3. **Leverage metadata cache** - determine which micro-partitions to read and which to skip (*partition pruning*).  
4. **Select execution strategies** - e.g., hash join vs. broadcast join, aggregation order, etc.  
5. **Distribute work** - assign tasks to compute nodes for parallel execution.  

**Analogy:**  
> Think of the Optimizer as a head chef planning the entire recipe before anyone starts cooking.  
> The compute layer are the sous-chefs who just execute the plan efficiently.

---

#### Example: The Optimizer in Action

```sql
SELECT c.country, SUM(o.amount)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.country;
```
Here’s what happens before any data is read:

1. **Parsing:** The optimizer validates the SQL syntax.  
2. **Planning:** It inspects metadata - finds that only 20% of micro-partitions have `order_date >= '2024-01-01'`.  
3. **Optimization:** It selects a **hash join** (both tables large) and determines partition pruning rules.  
4. **Distribution:** It assigns subsets of data to each compute node to process in parallel.  
5. **Execution:** The compute layer receives this plan and executes it exactly as instructed.

---

#### Other Components of the Cloud Services Layer

Beyond the optimizer, this layer also manages:

| Function | Description |
|-----------|--------------|
| **Metadata management** | Keeps catalog of all databases, schemas, tables, and micro-partitions. |
| **Security & RBAC** | Verifies that the active role has privileges for each object accessed. |
| **Caching coordination** | Controls result cache and metadata cache usage. |
| **Session management** | Tracks active users, queries, and warehouses. |
| **Query compilation** | Translates SQL plans into execution instructions for compute clusters. |

---

#### Example - Full Cloud Services Flow

```sql
SELECT * FROM orders WHERE region = 'Europe';
```

1. **Authentication & RBAC:** Check if the user’s role can read `orders`.  
2. **Optimization:** Read partition metadata → find which contain `'Europe'`.  
3. **Planning:** Build the execution plan and send it to the compute layer.  
4. **Execution:** Compute layer reads only the relevant partitions in parallel.  
5. **Aggregation & Return:** Results are combined and returned to the user.

> The Cloud Services Layer doesn’t store data  it stores *knowledge about data*.  
> It’s what makes Snowflake **smart, elastic, and self-optimizing.**

---

### 1.6 Caching: Remembering Previous Work

Caching = reusing previous data or results to save time and cost.

| Cache Type | Where | Stores | Reused When |
|-------------|--------|---------|--------------|
| **Result cache** | Cloud Services Layer | Final results of identical queries | Same SQL *and* same data snapshot |
| **Metadata cache** | Cloud Services Layer | Micro-partition statistics | Always used for pruning |
| **Local cache** | Compute Layer | Recently read micro-partitions | Same warehouse runs again before suspension |

---

#### Deep Dive - “Same SQL” and “Same Data”

Snowflake has multiple caches that operate at different levels. As seen above, they are the _Result Cache_, _Local Cache_, and _Metadata Cache_.

Each one “remembers” something different. Let’s break them down:

#### Result Cache (Same SQL, Same Data)

**What it stores**
- The final output (rows/aggregates) of a query.  
- Stored for 24 hours in the *Cloud Services layer*.

**“Same SQL”** means the text of the query is identical - even a space or comment breaks the match.  
**“Same data”** means the underlying tables haven’t changed since the cache was created.

```sql
-- First run
SELECT COUNT(*) FROM orders WHERE region='Europe';
-- Second run
SELECT COUNT(*) FROM orders WHERE region='Europe';
```
The second query returns instantly from **result cache** because:
- Query text is identical.
- The `orders` table has not changed.

If new rows are inserted, the cache is invalidated because the data snapshot changed.

---

#### Local Cache (Same Data Accessed Again by the Same Warehouse)

**What it stores**
- The *raw micro-partitions* (data chunks) recently read from the storage layer.  
- Stored in the *Compute layer (warehouse memory)*.

**“Same data”** means the *same micro-partitions are being requested again*, and the warehouse hasn’t suspended.

```sql
SELECT COUNT(*) FROM orders WHERE region='Europe';
SELECT SUM(amount) FROM orders WHERE region='Europe';
```
These two queries are *not identical* SQL, so result cache can’t be used.  
But the same data (`Europe` partitions) is already in memory → reused from **local cache**.

> Think of it like fetching pages of a book to your desk: 
> -> you can re-read or analyze them differently without going back to the library.

---

#### Metadata Cache (Used by the Optimizer)

The **Metadata Cache** is always active and operates behind the scenes in the **Cloud Services Layer**.

**What It Stores**
- Summary information (metadata) about each **micro-partition**:
  - Minimum and maximum values per column, Row counts and null counts, File and partition locations, Data versioning information.

This metadata is lightweight but extremely valuable. 

-> It allows the **optimizer** to know where data is located **without reading it first**.

**Why It Matters**
Before any query runs, the optimizer:
1. Reads the metadata cache.  
2. Determines which micro-partitions are **relevant** for your filters.  
3. Skips the rest entirely - a process called **partition pruning**.

This saves both time and compute cost because irrelevant data is never scanned.

#### Example
```sql
SELECT SUM(amount)
FROM orders
WHERE order_date >= '2024-01-01';
```

Snowflake checks metadata:  
- Partition 1 → max(order_date) = 2023-12-31 → ❌ skipped
- Partition 2 → min(order_date) = 2024-01-01 → ✅ scanned

No actual data is read until the optimizer knows exactly which partitions matter!

---

### 1.7 Security Overview - Understanding RBAC (Role-Based Access Control)

Snowflake’s security model is built on **Role-Based Access Control (RBAC)** -  
every user acts *through a role*, and that role defines *what they can see or do*.

#### Core Principles

| Concept | Description | Analogy |
|----------|--------------|----------|
| **Users** | Represent individual people or service accounts. | People entering a building. |
| **Roles** | Define *sets of privileges* (what actions are allowed). | ID badges giving access to rooms. |
| **Privileges** | Specific permissions (e.g., SELECT, INSERT, USAGE). | Keys that open certain doors. |
| **Objects** | The things being protected (databases, schemas, tables, etc.). | The rooms themselves. |
| **Grants** | Connect roles ↔ privileges ↔ objects ↔ users. | Giving a badge access to a door. |

**Users never get direct privileges** - only roles do. This makes permission management cleaner and auditable.

---

#### Basic Hierarchy

```text
Role → Privilege → Object
User → Role
```

Or in English:
> A user has a role.
> A role has privileges.
> Privileges apply to objects (databases, schemas, tables...).

#### Example - Creating and Assigning Roles

```sql
-- 1. Create a role for analysts
CREATE ROLE analyst;

-- 2. Grant privileges to that role
GRANT USAGE ON DATABASE sales TO ROLE analyst;
GRANT USAGE ON SCHEMA sales.public TO ROLE analyst;
GRANT SELECT ON TABLE sales.public.orders TO ROLE analyst;

-- 3. Assign role to user
GRANT ROLE analyst TO USER caio;
```

Now user **Caio** can query the `orders` table via their `analyst` role. This avoids potential problems such as deleting data, accessing other databases or seeing sensitive information.

---

#### Role Hierarchies (Parent/Child Roles)

Roles can inherit from other roles.  
For example:

```sql
GRANT ROLE analyst TO ROLE senior_analyst;
GRANT ROLE senior_analyst TO ROLE analytics_admin;
```

This creates a **role tree**:

```
analytics_admin
   └── senior_analyst
         └── analyst
```

The higher roles automatically inherit privileges from the lower ones.

---

#### System Roles

Snowflake includes built-in roles:

| Role | Description |
|-------|--------------|
| **ACCOUNTADMIN** | Full control over the account (everything). |
| **SECURITYADMIN** | Manages roles, users, and grants. |
| **SYSADMIN** | Manages databases, schemas, and objects. |
| **PUBLIC** | Default role with minimal privileges. |

**Best practice:** Use _least privilege_ - give users the minimum necessary role to do their job.

---

#### Best Practices

| Practice | Why it matters |
|-----------|----------------|
| Use role hierarchies | Simplifies permission management. |
| Never assign privileges directly to users | Keeps audits and role rotations clean. |
| Create separate roles for read, write, admin | Clear separation of duties. |
| Regularly review grants | Prevent privilege creep. |

---

#### Summary

| Level | Example | Description |
|--------|----------|-------------|
| **Database** | `GRANT USAGE ON DATABASE sales TO ROLE analyst;` | Grants visibility to a database. |
| **Schema** | `GRANT USAGE ON SCHEMA sales.public TO ROLE analyst;` | Allows seeing objects in a schema. |
| **Table** | `GRANT SELECT ON TABLE sales.public.orders TO ROLE analyst;` | Allows querying a table. |
| **Role assignment** | `GRANT ROLE analyst TO USER caio;` | Gives user access through a role. |

**Roles → Privileges → Objects → Users**: This chain defines *everything* about access in Snowflake.

---

### 1.8 Performance Optimization Techniques

Performance tuning in Snowflake means helping the **Optimizer** and the **Compute Layer** do *less work*.  
That means:  
- Reading less data  
- Caching more effectively  
- Using compute power efficiently  

---

#### Clustering (Organizing Data for Pruning)

**Definition:**  
Clustering organizes rows inside micro-partitions so that values of one or more columns are physically close together.  

**Why it matters:**  
It improves **partition pruning** - Snowflake can skip entire chunks of data because related values are stored together.

**Example:**  
```sql
ALTER TABLE orders CLUSTER BY (country, order_date);
```

Now all rows with the same `country` and `order_date` ranges are near each other - pruning becomes sharper.

You can inspect clustering quality with:  
```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('orders');
```

**Analogy:**  
> Imagine sorting a library so that all “Europe” books are on one shelf - the librarian finds them faster!

---

#### Views vs. Materialized Views

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

The second one **runs instantly** since results are precomputed.

---

#### Profiling (Measuring Query Performance)

**Definition:**  
Profiling means examining how a query actually ran - which steps were slow, how many partitions were read, and how compute was distributed.

**How to Access:**  
- Snowflake UI → **Query History → Profile**  
- SQL via `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`

| Component | Description | Example insight |
|------------|--------------|-----------------|
| **Query Plan Graph** | Visual DAG showing each step (scan, join, aggregate). | Detect unbalanced joins or bottlenecks. |
| **Bytes scanned** | Total data read. | Too high = poor pruning or missing clustering. |
| **Partitions scanned** | How many micro-partitions were accessed. | Should be much smaller than total. |
| **Elapsed time per step** | Execution time per operation. | Helps find slow joins or aggregations. |

**Example Insight:**  
If profiling shows most time spent on “join,” try clustering or filtering earlier.

---

#### Core Optimization Areas

| Area | Technique | Why it helps | Example or Tip |
|-------|------------|--------------|----------------|
| **1. Query pruning** | Use filters on columns Snowflake can prune. | Reduces number of micro-partitions scanned. | Filter on `order_date` instead of `DATE(order_date)`. |
| **2. Clustering keys** | Group large tables by frequently filtered columns. | Keeps data ordered for faster pruning. | `ALTER TABLE orders CLUSTER BY (country, order_date);` |
| **3. Warehouse sizing** | Choose the right size and enable auto-suspend. | Balances cost and speed. | Short bursts → medium; long queries → large. |
| **4. Materialized views** | Store pre-aggregated results. | Avoids recomputation for dashboards. | `CREATE MATERIALIZED VIEW mv_sales AS SELECT region, SUM(amount)...` |
| **5. Caching** | Reuse result/local/metadata caches. | Saves compute and I/O. | Run related queries in the same warehouse. |
| **6. Query profiling** | Inspect execution plans. | Identify bottlenecks. | Use the Web UI or `QUERY_HISTORY()`. |

---

#### Example - Optimizing a Query

```sql
SELECT customer_id, SUM(amount)
FROM orders
WHERE country = 'DE'
GROUP BY customer_id;
```

**Possible improvements:**

1. **Clustering**  
   - Cluster by `(country)` or `(country, order_date)` for better pruning.  

2. **Avoid Functions on Filter Columns**  
   - Don’t wrap filters in functions like `LOWER(country) = 'de'`; this prevents pruning.  

3. **Scale Appropriately**  
   - Scale **up** for faster single queries or **out** for concurrency.  

4. **Materialized View**  
   - Precompute if it’s a common aggregation:  
     ```sql
     CREATE MATERIALIZED VIEW mv_orders_de AS
     SELECT customer_id, SUM(amount)
     FROM orders
     WHERE country = 'DE'
     GROUP BY customer_id;
     ```

5. **Inspect the Query Profile**  
   - Check *Query History → Profile* for large scans or skewed joins.

---

#### Visualization - How Query Pruning Works

```
orders table
├── Partition 1 → country=US  (skipped)
├── Partition 2 → country=ES  (skipped)
├── Partition 3 → country=DE  ✅ scanned
├── Partition 4 → country=DE  ✅ scanned
└── Partition 5 → country=FR  (skipped)
```

Out of 5 partitions, only 2 are read - 60% of the work is saved.

---

#### Fundamental Rule of Thumb

> **Snowflake doesn’t get slow - it just scans more data than necessary.**  
> Optimization means guiding the optimizer to touch as few micro-partitions as possible.

---

#### Bonus: Useful Views & Functions

| View / Function | Description | Example |
|------------------|--------------|----------|
| `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` | Query metadata for performance review. | Find queries with long execution time. |
| `TABLE_STORAGE_METRICS` | See table size and micro-partition stats. | Identify bloated tables for reclustering. |
| `SYSTEM$CLUSTERING_INFORMATION('table_name')` | Shows clustering depth and effectiveness. | Monitor large tables for re-clustering needs. |

---

#### Optimization Checklist

- [ ] My filters enable **partition pruning** (no unnecessary functions).  
- [ ] I use **clustering keys** for very large, frequently filtered tables.  
- [ ] My **warehouses auto-suspend** when idle to save cost.  
- [ ] I **reuse warehouses** to take advantage of caching.  
- [ ] I **profile queries** regularly to identify slow joins or large scans.  
- [ ] I use **materialized views** for repeated aggregations.  

---

#### Summary - What Optimization Really Means

> Snowflake optimization isn’t about “tuning knobs.”  
> It’s about *helping the optimizer do less work*.  
>  
> - Store data intelligently (partitioning, clustering).  
> - Write filter-friendly SQL.  
> - Use warehouses efficiently.  
> - Let caching and materialization handle repetition.  
>  
> In short: **read less, compute smarter, and pay only for what you need.**

---

### 1.9 Visual Summary

```
                        ┌───────────────────────────────┐
                        │   Cloud Services Layer        │
                        │ (Parsing • Optimization • RBAC│
                        │  Metadata • Caching)          │
                        └──────────────▲────────────────┘
                                       │
                                       │ sends plan
                                       ▼
          ┌────────────────────────────┴────────────────────────────┐
          │             Compute Layer (Virtual Warehouses)          │
          │  • Executes queries in parallel (MPP)                   │
          │  • Maintains local cache                                │
          └──────────────▲──────────────────────────────────────────┘
                         │
                         │ reads micro-partitions
                         ▼
          ┌────────────────────────────────────────────────────────┐
          │                Storage Layer                           │
          │ • Data in compressed micro-partitions (~16 MB)          │
          │ • Metadata for pruning                                 │
          │ • Physically stored on S3 / Blob storage               │
          └────────────────────────────────────────────────────────┘
```

---

### 1.10 Quick Recap Checklist

- [ ] I can explain **why Snowflake was created** and its three-layer design.  
- [ ] I understand **the difference between S3 partitions and micro-partitions.**  
- [ ] I can describe **the Optimizer’s role** in planning queries.  
- [ ] I know what **parallel processing** and **clustering** mean.  
- [ ] I can explain **caching types** and how each one works.  
- [ ] I can apply at least **three optimization techniques** (pruning, caching, views).  
- [ ] I know how to **profile a query** to find performance bottlenecks.  

---
# Block 2 - Data Transformation & Automation  
Building logic, handling incremental data, and automating updates natively inside Snowflake.

---

## 2. Procedural SQL, Streams & Tasks

### 2.1 Stored Procedures & Snowflake Scripting

#### Why Procedural SQL Exists

Standard SQL is **declarative** - you describe *what* you want (e.g., “select all orders from Germany”), and the engine decides *how* to get it.  

However, in real-world pipelines, you often need **multi-step logic**:
- Conditional flows (`IF / ELSE`)
- Loops
- Variables and intermediate results
- Error handling
- Controlled transactions (BEGIN / COMMIT / ROLLBACK)

That’s where **Procedural SQL** and **Stored Procedures** come in - they let you write **scripts** that orchestrate complex transformations *inside Snowflake itself*.

---

#### Two Ways to Write Logic in Snowflake

| Language | Description | When to Use |
|-----------|--------------|-------------|
| **Snowflake Scripting (SQL dialect)** | Native procedural SQL syntax (similar to PL/SQL or T-SQL). | When you want to stay entirely in SQL. |
| **JavaScript Stored Procedures** | Use JavaScript with SQL commands embedded (`snowflake.execute()`). | When logic or branching is complex (loops, string ops, API calls). |

Both are **executed in Snowflake’s compute layer**, close to the data - no need to move data to Python or an external system.

---

#### Example 1 - Simple Snowflake Scripting Block

```sql
BEGIN
    LET country := 'DE';
    LET total := (SELECT COUNT(*) FROM orders WHERE country = :country);
    RETURN 'Total orders for ' || :country || ' = ' || :total;
END;
```

**How it works - step by step**

1. **`BEGIN ... END` - Defines the scripting block**  
   This marks the start and end of a self-contained Snowflake Scripting block.  
   Everything inside executes as a single unit - like a small anonymous procedure.

2. **`LET country := 'DE';` - Declares and initializes a variable**  
   - `LET` is used to define variables.  
   - Here, we create a variable called `country` and assign it the value `'DE'`.  
   - Variables make your SQL more dynamic - you can reuse them in other queries.

3. **`LET total := (SELECT COUNT(*) FROM orders WHERE country = :country);` - Executes a query into a variable**  
   - This line runs a subquery and stores its result (the count of rows) into `total`.  
   - The colon (`:`) before `country` means “use the variable’s value.”  
   - After this line, `total` holds, for example, `13412`.

4. **`RETURN 'Total orders for ' || :country || ' = ' || :total;` - Returns the result**  
   - The `||` operator concatenates strings and variable values.  
   - This creates a text output like:  
     ```
     Total orders for DE = 13412
     ```
   - The block ends immediately after returning this string.

5. **Execution context**  
   - You can run this block directly in the Snowflake worksheet - no need to save it as a stored procedure.  
   - It executes atomically: if any part fails, no partial result is left behind.

---

**Output Example:** 
```
Total orders for DE = 13412
```

**In short:**  
This block demonstrates the *core idea* of Snowflake Scripting - using variables, SQL queries, and logic inside a single executable block, all running **inside Snowflake’s compute engine**, close to your data.

---

#### Example 2 - Stored Procedure Using JavaScript

Sometimes, more control or looping is needed. Snowflake also supports **JavaScript procedures** with embedded SQL.

```sql
CREATE OR REPLACE PROCEDURE merge_incremental()
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
    var cmd = `
        MERGE INTO fact_orders t
        USING staging_orders s
        ON t.order_id = s.order_id
        WHEN MATCHED THEN UPDATE SET amount = s.amount
        WHEN NOT MATCHED THEN INSERT (order_id, amount) VALUES (s.order_id, s.amount);
    `;
    snowflake.execute({sqlText: cmd});
    return "Incremental merge completed.";
$$;
```

**Explanation (step-by-step):**

1. **`CREATE OR REPLACE PROCEDURE merge_incremental()`**  
   Defines a stored procedure named `merge_incremental`. The parentheses indicate no input parameters.

2. **`RETURNS STRING`**  
   The procedure will return a text message once it completes (in this case, a success message).

3. **`LANGUAGE JAVASCRIPT`**  
   Tells Snowflake this procedure is written in JavaScript (not SQL Scripting).

4. **`AS $$ ... $$`**  
   Everything inside the double dollar signs **is the actual code body**.

5. **`var cmd = \` ... \`;`**  
   Declares a JavaScript variable `cmd` containing a SQL statement as a string (here, the MERGE logic).  
   The backticks (`` ` ``) let you use multi-line SQL text easily.

6. **`MERGE INTO fact_orders ...` - Upsert logic**  
   - This SQL statement performs an **upsert** (update + insert) between two tables:  
     the target table `fact_orders` and the source `staging_orders`.  
   - It matches rows by `order_id`:  
     - If a match exists → the row is **updated** (for example, the order already exists, but the `amount` has changed since the last load - we update the existing record accordingly).  
     - If no match exists → the row is **inserted** as a new record (for example, a new order has been placed that didn’t exist in the fact table before).  
   - This pattern keeps the fact table synchronized with the latest transactional data  
     in a **single atomic operation**, avoiding separate `UPDATE` and `INSERT` steps.  
   - ⚠️ **Note:** This operation **does not maintain historical versions** of changed rows.  
     When a value (like `amount`) is updated, the old value is overwritten - it’s a *current-state sync*, not a *historical log*.  

7. **`snowflake.execute({sqlText: cmd});`**  
   Executes the SQL command stored in `cmd` within Snowflake’s compute environment.

8. **`return "Incremental merge completed.";`**  
   Returns a simple string message back to the caller.

---

**In summary:**  
This stored procedure dynamically builds and executes a MERGE statement in JavaScript.  
It’s especially useful when:
- you need to loop through multiple tables or schemas,  
- build SQL strings programmatically, or  
- handle logic that SQL alone can’t express cleanly (conditions, exceptions, logs).


**Key idea:**  
- You can dynamically build SQL text and execute it.  
- Perfect for looping over tables, handling errors, or running conditionally.

---

#### Examples of Control Flow Syntax (Quick Reference)

| Structure | Example | Purpose |
|------------|----------|----------|
| **IF / ELSE** | `IF v_total > 0 THEN RETURN 'ok'; ELSE RETURN 'empty'; END IF;` | Conditional logic. |
| **FOR loop** | `FOR rec IN (SELECT * FROM table) DO ... END FOR;` | Iterate over query results. |
| **WHILE loop** | `WHILE v_count < 10 DO ... END WHILE;` | Repeat until condition met. |
| **TRY / CATCH** | `EXCEPTION WHEN OTHERS THEN RETURN 'error';` | Error handling. |
| **Transactions** | `BEGIN; ... COMMIT;` | Atomic group of statements. |

---

#### Snowflake Stored Procedure Analogy

> Think of a stored procedure as a **recipe card** you keep in the kitchen (the database).  
> Each time you “CALL” it, Snowflake follows the exact steps, using the freshest ingredients (current data).

---

#### Best Practices

| Tip | Why it matters |
|------|----------------|
| Keep procedures small and modular. | Easier debugging and reuse. |
| Use variables for parameters (dates, regions, etc.). | Avoid hardcoding. |
| Include exception handling. | Prevent partial data loads. |
| Log operations into an audit table. | Improves observability. |
| Combine with **Streams + Tasks** (next sections). | Enables automated incremental pipelines. |

---

#### Quick Recap

- [ ] I understand the difference between **declarative SQL** and **procedural SQL**.  
- [ ] I know the two types of stored procedures: **SQL scripting** and **JavaScript**.  
- [ ] I can create and call a basic stored procedure.  
- [ ] I know how to use **variables, loops, and conditionals**.  
- [ ] I understand why stored procedures are used in ETL/ELT workflows.

---

### 2.2 Incremental Strategies: Full Incremental Loads vs. Partitioned Incremental Loads

In data pipelines, not all updates are equal.  
Some processes need to refresh **all data since the last load**, while others only process **the most recent slice of data** (for example, *today’s sales*).

These two strategies are known as **full incremental loads** and **partitioned incremental loads**.

---

#### Clarifying the Word “Partition”

In this context, the word *partition* does **not** refer to:
- **S3 partitions** - the folder-based organization in external storage (e.g. `year=2025/month=10/`), or  
- **Snowflake micro-partitions** - the automatic internal storage chunks Snowflake uses for pruning.

Here, *partition* means a **logical subset of data** based on a business key - usually a **date column** such as `sales_date`, `event_date`, or `updated_at`.  
It’s the *portion of data* we choose to process at a given time (e.g., “today’s data”).

---

#### Full Incremental Loads

**Definition:**  
Reload and synchronize *all changed or new records* since the last load, regardless of their date or logical partition.

**Example use cases:**
- Refreshing a large fact table (`fact_sales`) using all rows from `staging_sales`.
- Jobs where late-arriving data can modify multiple past days.

**Advantages:**
- Ensures complete consistency across all dates.  
- Captures late updates and corrections in historical data.

**Drawbacks:**
- Scans and compares much more data → **higher compute cost**.  
- **Slower** for very large staging tables.

---

#### Partitioned Incremental Loads

**Definition:**  
Process only a **specific logical partition** (e.g., one day’s worth of data).  
The load logic filters by a column like `sales_date = CURRENT_DATE()`.

**Example use cases:**
- Daily jobs that only ingest *today’s* transactions.  
- Streaming or near-real-time ingestion where only the most recent data matters.

**Advantages:**
- **Faster** and **cheaper** - only a small subset of rows is processed.  
- Ideal for append-only event or transaction tables.

**Drawbacks:**
- Historical corrections (e.g., backdated orders) are ignored unless reprocessed manually.  
- Requires **extra handling** for *late-arriving* or *out-of-order* data.

---

#### Why It Matters in Snowflake

Snowflake’s architecture (with automatic **micro-partition pruning** and **metadata caching**) makes both patterns efficient.  
However, choosing the right **incremental strategy** has major performance and cost implications:

| Scenario | Recommended Pattern |
|-----------|--------------------|
| Frequent updates across multiple dates | Full Incremental |
| Append-only or daily event ingestion | Partitioned Incremental |
| Low-latency dashboards or streaming micro-batches | Partitioned Incremental |
| Historical corrections or late-arriving data | Full Incremental |

---

**Summary:**  
- Both methods rely on **MERGE statements** to synchronize source and target tables.  
- Example 1 (next section) demonstrates a **Full Incremental Load** (all rows).  
- Example 2 refines that with a **Partitioned Incremental Load**, filtering by date *inside* the MERGE - processing only the relevant logical partition of data.

### 2.3 Understanding MERGE Statements - Performing Efficient Upserts

#### Why MERGE Exists

In data pipelines, you often need to **synchronize a target table** (e.g., `fact_sales`)  
with new or changed records from a source table (e.g., `staging_sales`).

Historically, this was done using **two separate SQL statements**:
1. `UPDATE` existing rows (to refresh records that already exist in the target).
2. `INSERT` new rows (for records not yet present).

That approach worked, **but it required multiple steps and manual transaction handling**  
to ensure that both actions succeeded together - otherwise, data could become inconsistent.

The **MERGE** statement was introduced to solve this problem by combining both actions into  
a **single atomic operation**: if any part fails, nothing is applied. 

This makes MERGE not only faster, but also safer and easier to maintain in production pipelines.

---

#### MERGE Syntax

```sql
MERGE INTO <target_table> AS t
USING <source_table> AS s
ON <join_condition>
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT (...columns...) VALUES (...values...);
```

**Breakdown:**
- `MERGE INTO`: defines the target table to be updated.  
- `USING`: defines the source table (the incoming data).  
- `ON`: defines how to match rows between the two tables.  
- `WHEN MATCHED`: what to do when the key already exists (usually an `UPDATE`).  
- `WHEN NOT MATCHED`: what to do when the key doesn’t exist (usually an `INSERT`).  

---

#### Example 1 - Basic MERGE for Daily Sales Loads ("(Daily) Full Incremental Load")

**Business Goal:**  
This MERGE keeps the `fact_sales` table synchronized with the latest daily data from `staging_sales`.  
Each day, new transactions arrive in the staging area. The MERGE ensures:
- existing sales are **updated** if their amounts change (e.g., refunds, adjustments), and  
- new orders are **inserted** seamlessly.  

This pattern is a classic **Daily Incremental Load**, common in finance, retail, and e-commerce pipelines.

```sql
MERGE INTO fact_sales AS t
USING staging_sales AS s
ON t.order_id = s.order_id
WHEN MATCHED THEN
    UPDATE SET
        t.amount     = s.amount,
        t.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
    INSERT (order_id, amount, sales_date, created_at, updated_at)
    VALUES (s.order_id, s.amount, s.sales_date, CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP());
```

---

### How it works - step by step

1. **`MERGE INTO fact_sales AS t`**  
   - Defines the **target table** (`t`) that will be updated or inserted into.  
   - Think of this as your main *fact table* that stores finalized, clean data.

2. **`USING staging_sales AS s`**  
   - Defines the **source table** (`s`) - where new or changed records arrive daily.  
   - Typically populated by ingestion jobs or external data loads.

3. **`ON t.order_id = s.order_id`**  
   - This is the **matching condition** (the join key).  
   - Snowflake compares each `order_id` in the source against the target.  
   - If a match is found, the existing row in the target is a **candidate for update**.

4. **`WHEN MATCHED THEN UPDATE SET ...`**  
   - If the same `order_id` exists in both tables, update specific columns - here, we refresh `amount` and `updated_at`.  
   - This ensures that any price corrections or adjustments are reflected in the fact table.

5. **`WHEN NOT MATCHED THEN INSERT (...) VALUES (...)`**  
   - If no row exists with that `order_id`, Snowflake inserts a **new record** into `fact_sales`.  
   - ⚠️ **Note:** You must explicitly list all target columns and their corresponding values - for example, `(order_id, amount, sales_date, created_at, updated_at)` followed by `VALUES (s.order_id, s.amount, s.sales_date, CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP())`.  
   - This explicit mapping prevents data misalignment if column orders change or if the staging table contains extra fields.  
   - ✅ **In practice**, data teams handle large tables (sometimes with hundreds of columns) by **automating this step** using:
     - **dbt macros** to dynamically generate the column list (`{{ get_insert_columns('fact_sales') }}`),  
     - **Python or SQL templating** for dynamic SQL in stored procedures, or  
     - **views** that expose only the needed columns, simplifying the MERGE statement.  
   - This approach keeps the MERGE operation both **safe** and **maintainable**, even at scale.

6. **Atomic execution**  
   - Both `UPDATE` and `INSERT` actions are part of one **atomic operation**.  
   - If anything fails (e.g., constraint violation), the entire transaction rolls back -  
     leaving the target table consistent.

7. **Merging ALL rows (Full Incremental Sync) vs. Filtering Source Table in Merging (Partitioned Incremental Sync)**  
   - The next example introduces a **key variation**: instead of merging *all rows* from `staging_sales`, it filters the **source inside the MERGE** (e.g., `WHERE sales_date = CURRENT_DATE()`).  
   - This makes the operation more efficient for **daily or real-time pipelines**, as only the relevant subset of data (today’s transactions) is processed atomically.  
   - In other words, Example 1 handles a **full incremental sync**, while Example 2 focuses on a **partitioned, filtered sync**.

---

### Visualizing the MERGE

**Before the MERGE**

*fact_sales* (target)
| order_id | amount | updated_at |
|-----------|---------|------------|
| 1 | 100 | 2025-10-18 |
| 2 | 200 | 2025-10-18 |

*staging_sales* (source)
| order_id | amount | sales_date |
|-----------|---------|------------|
| 2 | 220 | 2025-10-19 |
| 3 | 150 | 2025-10-19 |

---

**Step-by-step:**

- Row with `order_id = 2` exists in both tables → **UPDATE** → amount changes from **200 → 220** and timestamp refreshes.  
- Row with `order_id = 3` exists only in source → **INSERT** → new sale added to the fact table.  
- Row with `order_id = 1` exists only in target → **no change**.

---

**After the MERGE**

*fact_sales* (target)
| order_id | amount | updated_at |
|-----------|---------|------------|
| 1 | 100 | 2025-10-18 |
| 2 | 220 | 2025-10-19 |
| 3 | 150 | 2025-10-19 |

The fact table now reflects both the updated and newly inserted records - all done in a single, efficient statement.

---

**Summary:**
- MERGE automates incremental synchronization.  
- It eliminates the need for separate `UPDATE` and `INSERT` statements.  
- It keeps your fact table up-to-date with minimal logic and guaranteed consistency.  
- ⚠️ **Note:** This operation overwrites existing data (no history tracking).  
  If you need to preserve historical versions, you’d need a **Type 2 SCD (Slowly Changing Dimension)** design instead.

---

#### Example 2 - MERGE with a Filtered Source ("(Daily or Real-Time) Partitioned Incremental Load")

**Main Difference from Example 1:**  
The previous MERGE processed *all available records* in `staging_sales`.  
This one filters the **Source inside the MERGE itself**, so only *today’s data* is processed - a much more efficient pattern for daily or real-time pipelines.

---

**Business Goal:**  
This MERGE is used in a **scheduled job** that only processes the current day’s data, avoiding unnecessary work.  
By filtering the source inside the MERGE, it ensures only *today’s* rows are evaluated atomically (Snowflake evaluates the filter and applies the merge as one transaction) - perfect for near-real-time ingestion pipelines where you only want to update the active partition.  

Common use cases include **daily sales snapshots**, **rolling inventory updates**,  
or **transaction feeds** coming from external systems like Stripe or Shopify.

---

```sql
MERGE INTO fact_sales AS t
USING (
  SELECT * FROM staging_sales WHERE sales_date = CURRENT_DATE()
) AS s
ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET amount = s.amount
WHEN NOT MATCHED THEN INSERT (order_id, amount, sales_date)
VALUES (s.order_id, s.amount, s.sales_date);
```

---

### How it works - step by step

1. **Filtered Source Subquery**  
   - The `USING (...)` clause contains a subquery:  
     ```sql
     SELECT * FROM staging_sales WHERE sales_date = CURRENT_DATE()
     ```  
   - This limits the source to only *today’s* rows before any MERGE logic runs.  
   - Unlike pre-filtering the table in a temporary step, this approach is **atomic** -  
     Snowflake evaluates the filter and applies the merge as one transaction.

2. **Join Condition**  
   - `ON t.order_id = s.order_id` compares the filtered source with the target.  
   - Only today's orders are checked - historical rows remain untouched.

3. **Matched Rows → UPDATE**  
   - If an order from today already exists in the fact table, update its amount (for example, a price correction).  
   - The target remains current without duplicating records.

4. **Unmatched Rows → INSERT**  
   - If an order doesn’t exist yet, insert it with the same columns: `(order_id, amount, sales_date)`.  
   - This captures new daily transactions efficiently.

5. **Performance Advantage**  
   - Filtering inside the MERGE avoids scanning the entire staging table.  
   - This is especially useful for **large tables** with historical data, as only a small, relevant subset (today’s "partition") is processed.

6. **Atomic Operation**  
   - The MERGE (with its internal filter) is evaluated as a single, consistent transaction - ensuring that no records are missed or double-processed if another job runs concurrently.

---

### Mini Example - Visualizing the Filtered Merge

**Before the MERGE**

*fact_sales* (target)
| order_id | amount | sales_date |
|-----------|---------|-------------|
| 100 | 90 | 2025-10-19 |
| 101 | 120 | 2025-10-19 |

*staging_sales* (source)
| order_id | amount | sales_date |
|-----------|---------|-------------|
| 101 | 130 | 2025-10-20 |
| 102 | 50 | 2025-10-20 |
| 103 | 75 | 2025-10-18 |  

---

**Step-by-step:**  

- The filter `WHERE sales_date = CURRENT_DATE()` selects only rows for **2025-10-20**.  
  So only `order_id = 101` and `102` are considered.  
- `order_id = 101` → **MATCHED** → UPDATE `amount` from `120 → 130`.  
- `order_id = 102` → **NOT MATCHED** → INSERT as a new record.  
- `order_id = 103` (from two days ago) → ignored (outside the filter).

---

**After the MERGE**

**fact_sales (target)**
| order_id | amount | sales_date |
|-----------|---------|-------------|
| 100 | 90 | 2025-10-19 |
| 101 | 130 | 2025-10-20 |
| 102 | 50 | 2025-10-20 |

Only the active day’s partition was processed - faster, safer, and fully atomic.

---

**Summary:**
- Filtering **inside** the MERGE keeps the operation atomic and efficient.  
- Perfect for **daily partitions** or **streaming ingestion** use cases.  
- Avoids unnecessary reads and reduces cost for large staging tables.  
- ⚠️ **Note:** Like before, this MERGE overwrites the current state - it doesn’t maintain history.


---

#### Analogy

> Think of `MERGE` as a **smart librarian** updating a catalog:  
> - If a book already exists → they **update** its details (e.g., a new edition).  
> - If it’s a new book → they **add** it to the shelf.  
> - Everything happens in **one organized step**, not two separate passes.  
>
> In a **full incremental load**, the librarian checks **the entire catalog** every time.  
> In a **partitioned incremental load**, they only look at **today’s new arrivals** -  
> faster, cheaper, and still perfectly accurate for the day’s work.


---

#### Common MERGE Patterns

| Scenario | Typical Columns | Purpose |
|-----------|------------------|----------|
| **Fact table upsert** | `order_id`, `sales_date` | Load new transactions and update existing ones (daily loads). |
| **Dimension table refresh** | `customer_id` | Sync changing attributes like name, tier, or status (Type 1 SCD). |
| **CDC (Change Data Capture)** | `record_id`, `operation` | Apply inserts/updates/deletes from a change log stream. |

---

#### Best Practices

| Practice | Why it matters |
|-----------|----------------|
| **Cluster or partition the target table** | Improves performance through micro-partition pruning on join keys. |
| **Filter the source** | Avoids scanning irrelevant rows (especially for daily or partitioned merges). |
| **Update only necessary columns** | Reduces micro-partition rewrites and storage overhead. |
| **Track affected rows** | Use `RESULT_SCAN(LAST_QUERY_ID())` to monitor how many rows were updated or inserted. |
| **Combine with Streams or Tasks** | Enables automated, continuous incremental merges (covered next). |
| **Use staging schemas for safety** | Prevents accidental overwrites of production data. |

---

#### Inspecting Merge Results

After a MERGE finishes, Snowflake stores summary statistics for that query execution.  
You can access them immediately using `RESULT_SCAN(LAST_QUERY_ID())`:

```sql
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

Typical output:

```
rows_inserted | rows_updated | rows_deleted
--------------+--------------+--------------
1024 | 354 | 0
```

This is useful for **logging**, **monitoring**, or **data quality checks** - especially inside stored procedures or automated tasks.

---

#### Quick Recap Checklist

- [ ] I understand how `MERGE` combines **INSERT** and **UPDATE** into one atomic statement.  
- [ ] I can explain the difference between **full** and **partitioned** incremental loads.  
- [ ] I know how to interpret the `ON`, `WHEN MATCHED`, and `WHEN NOT MATCHED` clauses.  
- [ ] I know when to apply `MERGE` to facts, dimensions, or CDC streams.  
- [ ] I can check merge results using `RESULT_SCAN(LAST_QUERY_ID())`.  

---

**Summary:**  
> The `MERGE` command is the backbone of incremental loading in Snowflake -  
> enabling atomic, auditable synchronization between staging and target tables.  
> In the next section, you’ll see how **Streams** build on top of this concept  
> to track table changes automatically.

---

## 2.4 Streams - Tracking Table Changes (Change Data Capture)

### DML - Data Manipulation Language

Before we can talk about **Change Data Capture (CDC)**, we need to revisit a fundamental idea every data engineer should know from day one:

> **Data changes over time.**  
> Tables aren’t static - new rows appear, existing ones evolve, and others are removed.

This concept is formalized in SQL under the umbrella of **DML (Data Manipulation Language)** - the commands responsible for modifying data in a table.

| Action | Description |
|---------|--------------|
| **INSERT** | Add new records. |
| **UPDATE** | Modify existing records. |
| **DELETE** | Remove records. |

As data engineers, our job is to **capture and process these changes efficiently**.  
That means designing systems that can detect when and how data has changed - not just new inserts, but also updates and deletions - without reloading entire tables.

> In everyday terms: data is **added**, **modified**, or **removed**.  
> In database terms: a **DML statement** performs an **INSERT**, **UPDATE**, or **DELETE**.

### Understanding Change Data Capture (CDC)

In modern data pipelines, it’s inefficient to reload entire tables every time data changes.  
Instead, we use **Change Data Capture (CDC)** - a design pattern that focuses on detecting and processing **only the rows that changed** (INSERT, UPDATE, DELETE) since the last run.

**Goal:**  
> Capture **only** the **delta**, not the full dataset.

Different systems implement CDC differently:

| CDC Method | How It Works | Example Tools |
|-------------|--------------|----------------|
| **Log-based CDC** | Reads database transaction logs to detect row-level changes. | `Debezium`, `Fivetran`, `Kafka Connect` |
| **Trigger-based CDC** | Uses triggers to log changes into audit tables. | `PostgreSQL`, `MySQL` triggers |
| **Timestamp-based CDC** | Filters rows using last-updated timestamps. | `dbt` incremental models, `SQL` scripts |
| **Metadata-based CDC (Snowflake Streams)** | Uses Snowflake’s internal metadata to track inserts, updates, and deletes. | ✅ Native to Snowflake |

---

### Why Streams Exist

In previous sections, we saw how **MERGE** can synchronize data between tables:  
- *Full Incremental MERGE* → merges **all records** from staging each day.  
- *Partitioned Incremental MERGE* → merges **only a specific time window** (e.g., `WHERE sales_date = CURRENT_DATE()`).

However, both approaches rely on **manual logic** to determine *what changed*.  
You, the engineer, decide which data to merge - using timestamps, partitions, or custom filters.

That works well for **append-only data** (e.g., new daily transactions) but becomes fragile when the source data also includes **updates** and **deletes**.

In those cases:
- You might miss changes if timestamps are inconsistent.  
- You might reload unnecessary data, wasting compute.  
- And most importantly - you can’t easily detect **deleted rows**,  
  since a deleted record simply disappears from the source table.

We need something smarter - a mechanism that can automatically track all **DML operations** (`INSERT`, `UPDATE`, and `DELETE`) as they happen, without scanning the whole table.

That’s exactly what **Streams** provide.

---

### From MERGE to Streams - The Missing Link

So far, our MERGE logic could **INSERT** and **UPDATE** rows efficiently.  
But what happens if the source table also contains **DELETIONS** - rows that no longer exist?  
A normal MERGE can’t detect those automatically, because it only sees the current snapshot of the source table.

That’s where **Streams** come in.

Streams continuously record **all DML operations** - `INSERT`, `UPDATE`, and `DELETE` - that occur on a table, storing this information as metadata. When you query the stream, you see not just new and changed rows, but also rows that were deleted.

| Concept | Role |
|----------|------|
| **MERGE** | Applies the changes (`INSERT`, `UPDATE`, or `DELETE`) to the target. |
| **STREAM** | Detects and exposes all the changes (the *memory* of what happened). |

Together, they form a **native Change Data Capture (CDC) pipeline** -  
where the stream tracks what changed, and the MERGE applies those changes atomically.

---

### How Streams Work

1. You create a **stream** on a table (typically a staging or raw table).  
2. Snowflake automatically logs every DML change (`INSERT`, `UPDATE`, `DELETE`) in internal metadata.  
3. When you query the stream, it returns **only the changed rows** since the last read.  
4. Once those changes are processed in a DML operation (like MERGE), the stream **advances its offset**, so the same changes aren’t returned again.

---

### Example 1 - Creating and Querying a Stream

**Business Goal:**  
This pattern demonstrates the *first step* in building a **Change Data Capture (CDC)** pipeline: creating a **Stream** on a source or staging table to automatically track every change.  

By querying the stream, analysts or data engineers can inspect **which rows changed** (inserts, updates, or deletes) before those changes are applied downstream.  

Common use cases:
- Auditing or debugging changes in staging tables  
- Monitoring incremental loads without full table scans  
- Feeding deltas into downstream transformations or dashboards  

```sql
-- 1. Create a stream to track changes on staging_sales
CREATE OR REPLACE STREAM stream_staging_sales
ON TABLE staging_sales;

-- 2. Insert new data into the source table as an example of new data coming in
INSERT INTO staging_sales VALUES (101, '2025-10-20', 120.50, 'DE');

-- 3. Query the stream to view only this delta
SELECT * FROM stream_staging_sales;
```

**Output Example:**

```
order_id | sales_date | amount | country | METADATA$ACTION | METADATA$ISUPDATE
---------+-------------+--------+----------+-----------------+-----------------
101 | 2025-10-20 | 120.50 | DE | INSERT | FALSE
```

- `METADATA$ACTION` → shows the type of change (`INSERT`, `UPDATE`, or `DELETE`)  
- `METADATA$ISUPDATE` → TRUE if the row was modified from a previous version  

---

### Step-by-Step Visualization

**Before any change**

*staging_sales*
| order_id | amount | country |
|-----------|---------|----------|
| 100 | 90 | ES |

**After an UPDATE**

*staging_sales*
| order_id | amount | country |
|-----------|---------|----------|
| 100 | **100** | ES |

**What the Stream shows**

*stream_staging_sales*
| order_id | amount | country | METADATA$ACTION | METADATA$ISUPDATE |
|-----------|---------|----------|-----------------|------------------|
| 100 | 100 | ES | UPDATE | TRUE |

✅ **Key difference:**  
Instead of scanning or filtering the table, the stream *already knows* what changed - Snowflake records it automatically as metadata.

---

### Example 2 - Using a Stream in a MERGE (Native CDC Pattern)

Now let’s connect this back to the MERGE concept we already know.

**Business Goal:**  
This pattern combines **Streams + MERGE** to form a fully automated **Change Data Capture (CDC)** workflow.  
The Stream acts as the memory - it records *what* changed - while MERGE applies those changes atomically to the target table.  

Each run processes **only new or modified rows** since the last run, eliminating manual filtering logic and enabling near–real-time synchronization between systems.  

Common use cases:
- Incremental updates to fact or dimension tables  
- Real-time synchronization across environments (e.g., staging → production)  
- Event-driven data ingestion pipelines from transactional sources like Stripe or Shopify

```sql
MERGE INTO fact_sales AS t
USING (
  SELECT order_id, sales_date, amount, country, METADATA$ACTION
  FROM stream_staging_sales
) AS s
ON t.order_id = s.order_id
WHEN MATCHED AND s.METADATA$ACTION = 'DELETE' THEN DELETE
WHEN MATCHED AND s.METADATA$ACTION = 'UPDATE' THEN UPDATE SET t.amount = s.amount
WHEN NOT MATCHED AND s.METADATA$ACTION = 'INSERT' THEN
    INSERT (order_id, sales_date, amount, country)
    VALUES (s.order_id, s.sales_date, s.amount, s.country);
```

---

### How This Differs from Previous MERGE Examples

| Step | Incremental MERGE | Stream-Based MERGE |
|------|--------------------|--------------------|
| **Data Source** | `staging_sales` table | `stream_staging_sales` object |
| **Change Detection** | Manual - filters by date or timestamp | Automatic - metadata tracks deltas |
| **Data Volume Scanned** | Potentially large (full or partitioned data) | Minimal (only changed rows) |
| **Scheduling** | Batch (daily/hourly) | Continuous (micro-batch or near real-time) |
| **Maintenance** | Must manage “last load” timestamp | Snowflake advances offset automatically |

---

### Side-by-Side Visualization

#### Previous Incremental MERGE

*Staging Table (input):*

| order_id | amount | country | sales_date |
|-----------|---------|----------|-------------|
| 100 | 90 | ES | 2025-10-19 |
| 101 | 120.50 | DE | 2025-10-20 |

You manually filtered:  
```sql
SELECT * FROM staging_sales WHERE sales_date = CURRENT_DATE();
```

Then merged - scanning and deciding what changed.

---

#### Stream-Based MERGE

**Stream (input):**

| order_id | amount | country | METADATA$ACTION |
|-----------|---------|----------|-----------------|
| 101 | 120.50 | DE | INSERT |
| 100 | 100 | ES | UPDATE |

Snowflake *already* filtered and tagged these as deltas - so MERGE can just apply them atomically without scanning or filtering.

---

### In Summary

> This MERGE pattern no longer depends on timestamps, partitions, or full scans.  
> The **stream acts as the memory** of what changed,  
> and **MERGE** is the action that applies those changes to keep your target table always up-to-date.

---

### Analogy

> Think of a **security camera** watching your table:  
> - It doesn’t record every second of footage.  
> - It only keeps clips where something changed.  
> - When you check the camera (query the stream), you see exactly what happened since the last time you looked.

---

### Stream Types

| Type | Description | Typical Use |
|-------|--------------|-------------|
| **Standard Stream** | Tracks inserts, updates, and deletes since last consumption. | Most incremental ETL or CDC pipelines. |
| **Append-Only Stream** | Tracks only inserts (no updates/deletes). | Immutable event data, append-only logs. |

---

### Important Notes

| Concept | Description |
|----------|--------------|
| **Consumption** | Once a stream is read inside a DML (like MERGE), its offset advances - those changes won’t appear again. |
| **Retention Period** | Default is 14 days; unconsumed changes expire afterward. |
| **Dependencies** | Dropping or replacing the source table invalidates the stream. |
| **Performance** | Streams store only metadata deltas - very lightweight. |

---

### Inspecting Stream Status

```sql
SHOW STREAMS;
```

Typical output:

```
name | table_name | type | stale | mode
---------------------+------------------+-----------+-------+--------------
STREAM_STAGING_SALES | STAGING_SALES | STANDARD | false | append_only
```


---

### Best Practices

| Practice | Why It Helps |
|-----------|--------------|
| Create streams on **staging or raw tables** | Keeps transformation logic separate from CDC tracking. |
| **Consume regularly** (e.g., via Tasks) | Prevents data loss when retention expires. |
| Combine with **MERGE** | Enables efficient, incremental upserts. |
| Use **append-only** mode | Optimized for immutable data (no updates/deletes). |
| Add **audit columns** (`created_at`, `updated_at`) | Simplifies validation and troubleshooting. |

---

### Quick Recap Checklist

- [ ] Streams implement **native Change Data Capture (CDC)** in Snowflake.  
- [ ] Each query returns only the **delta** since the last consumption.  
- [ ] Streams work perfectly with **MERGE** to apply changes incrementally.  
- [ ] Must be consumed regularly (default retention = 14 days).  
- [ ] Streams + Tasks = fully automated CDC pipelines.

---

**Summary:**  
> Snowflake Streams are a *built-in CDC mechanism*.  
> They automatically record all inserts, updates, and deletes in a table,  
> so your pipelines can process **only what changed** - efficiently, atomically, and without manual filtering.

## 2.5 Tasks - Scheduling & Automation

### Why Tasks Exist

In the previous sections, we built the foundations of a **modern Snowflake pipeline**:
- **Stored Procedures** → reusable logic  
- **MERGE** → synchronization between source and target tables  
- **Streams** → automatic detection of inserts, updates, and deletes  

Now we need a way to make this pipeline **run automatically** - every hour, every night, or whenever new data arrives. That’s the job of **Tasks** in Snowflake.

A **Task** is a built-in scheduler that can:
- Run SQL statements or stored procedures on a defined schedule  
- Trigger other tasks to form multi-step workflows  
- Handle dependency chains so steps run in the right order  
- Operate entirely inside Snowflake (no Airflow, no cron)

---

### Orchestration and DAGs - The Broader Context

#### Orchestration
In data engineering, **orchestration** means coordinating multiple tasks that must run in sequence to produce a result.

Example:
1. Load raw data  
2. Transform it into analytics-ready tables  
3. Refresh dashboards  

Each step depends on the one before it.

#### DAG - Directed Acyclic Graph
A **DAG** is the structure that represents those dependencies. It’s a **graph** of nodes (tasks) and arrows (dependencies).

- **Directed** → Flow goes one way (Task A → Task B)  
- **Acyclic** → No loops - a task never triggers itself again  

Example DAG:

    Load → Transform → Aggregate → Publish

This same principle underlies `Airflow`, `Prefect`, and `Snowflake Tasks`.

---

### How Snowflake Tasks Work

A **Task** is a Snowflake object that runs a SQL command or stored procedure:

- Runs on a defined **schedule** (`CRON` or simple interval)  
- Uses a chosen **warehouse** for compute  
- Can depend on other tasks to form a DAG  
- Can be suspended, resumed, or monitored  

| Concept | Description |
|----------|-------------|
| **SQL Statement** | The query or procedure to execute |
| **Schedule** | When it runs (`USING CRON` or `SCHEDULE='1 hour'`) |
| **Warehouse** | Compute resource to use |
| **Dependencies** | Other tasks that must finish first |

---

### Example 1 - Basic Task for MERGE

**Business Goal:**  
Automatically update the `fact_sales` table every hour with new and changed records from `staging_sales`. This ensures continuous data freshness without manual execution.

```sql
    CREATE OR REPLACE TASK task_merge_sales
      WAREHOUSE = my_wh
      SCHEDULE = '1 HOUR'
    AS
      MERGE INTO fact_sales AS t
      USING staging_sales AS s
      ON t.order_id = s.order_id
      WHEN MATCHED THEN UPDATE SET t.amount = s.amount
      WHEN NOT MATCHED THEN
        INSERT (order_id, amount, sales_date)
        VALUES (s.order_id, s.amount, s.sales_date);
```

**Key points**
- The warehouse `my_wh` resumes automatically for each run.  
- The schedule `'1 HOUR'` executes hourly.  
- Tasks can be paused/resumed:

```sql
   ALTER TASK task_merge_sales SUSPEND;
   ALTER TASK task_merge_sales RESUME;
```
---

### Example 2 - Task Triggering a Stored Procedure (Best Practice)

In production, it’s better to call a **stored procedure** inside a task. This keeps orchestration logic clean and reusable.

**Business Goal:**  
Execute the `merge_stream_sales()` procedure (which performs a Stream + MERGE CDC operation) every hour. This pattern creates a **self-updating pipeline** where changes are detected, merged, and maintained automatically.

Common use cases:
- Hourly CDC updates  
- Periodic fact table synchronization  
- End-to-end ELT pipelines (extract → transform → load)

```sql
    CREATE OR REPLACE TASK task_stream_merge_sales
      WAREHOUSE = my_wh
      SCHEDULE = 'USING CRON 0 * * * * UTC'
    AS
      CALL merge_stream_sales();
```
---

### Example 3 - Chained Tasks (Task DAG)

Snowflake supports dependent tasks - define multi-step workflows directly in SQL.

**Business Goal:**  
Ensure that raw data is loaded from S3 before merging it into `fact_sales`.

Pipeline flow:

    task_load_raw → task_merge_fact

```sql
    -- 1. Load raw data
    CREATE OR REPLACE TASK task_load_raw
      WAREHOUSE = my_wh
      SCHEDULE = 'USING CRON 0 * * * * UTC'
    AS
      COPY INTO staging_sales FROM @s3_stage;

    -- 2. Merge into fact table
    CREATE OR REPLACE TASK task_merge_fact
      WAREHOUSE = my_wh
      AFTER task_load_raw
    AS
      CALL merge_stream_sales();
```

---

### Simple ASCII DAG Visualization

```
        ┌──────────────────┐
        │  task_load_raw   │
        │ (COPY from S3)   │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ task_merge_fact  │
        │ (Stream + MERGE) │
        └──────────────────┘
```

Each run executes sequentially and atomically - no external scheduler required.

---

### Monitoring and Management

| Command | Purpose |
|----------|----------|
| `SHOW TASKS;` | List all tasks and their status |
| `ALTER TASK ... RESUME;` | Enable a suspended task |
| `ALTER TASK ... SUSPEND;` | Disable a task |
| `SYSTEM$TASK_HISTORY('task_name')` | View logs and errors |
| `DROP TASK task_name;` | Remove a task |

Example query:
```sql
    SELECT *
    FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TASK_HISTORY())
    WHERE TASK_NAME = 'TASK_MERGE_SALES'
    ORDER BY SCHEDULED_TIME DESC;
```
---

### Best Practices

| Practice | Why It Helps |
|-----------|--------------|
| Use stored procedures inside tasks | Keeps code modular and reusable |
| Chain tasks using `AFTER` | Builds maintainable DAGs |
| Use CRON for fine-grained control | Precise scheduling |
| Always specify a warehouse | Tasks need compute |
| Monitor `TASK_HISTORY` | Detect failures early |
| Suspend unused tasks | Avoid extra costs |

---

### Quick Recap Checklist

- [ ] I understand orchestration and DAGs.  
- [ ] I can create a basic Task.  
- [ ] I can schedule SQL and stored procedures.  
- [ ] I can chain tasks using `AFTER`.  
- [ ] I know how to monitor and manage tasks.  

---

**Summary:**  
> **Tasks** are Snowflake’s built-in orchestration layer.  
> They automate SQL or procedural logic and enable **end-to-end pipelines**  
> that run on schedule - entirely inside Snowflake, no external tools required.

## 2.6 Why Streams Instead of Timestamps?

### The Context

Before Snowflake introduced **Streams**, most incremental data pipelines relied on **timestamp-based logic** to detect new or modified rows. The pattern looked something like this:

```sql
    SELECT *
    FROM staging_sales
    WHERE updated_at > (SELECT MAX(updated_at) FROM fact_sales);
```

This approach works - but it also carries several risks and long-term operational drawbacks. As data systems scale, those weaknesses become significant both technically and from a business perspective.

---

### Comparing the Two Approaches

| Aspect | Timestamp-Based Incremental Load | Stream-Based CDC |
|--------|----------------------------------|------------------|
| **Change Detection** | Relies on application or ETL logic to update `updated_at` | Automatically tracked by Snowflake metadata |
| **Accuracy** | Can miss updates if timestamps are delayed or inconsistent | Guaranteed to capture every DML event (INSERT, UPDATE, DELETE) |
| **Deletes** | Not detected (rows simply disappear) | Tracked explicitly via `METADATA$ACTION = 'DELETE'` |
| **Data Latency** | Often requires wide safety windows (reloading overlaps) | Processes only true deltas - real-time or micro-batch ready |
| **Maintenance** | Requires managing last-run checkpoints and time zones | Fully managed - stream offset advances automatically |
| **Compute Cost** | Re-reads entire time windows repeatedly | Scans only changed rows - cost scales with data volatility |
| **Auditing & Replay** | Difficult to reconstruct exact history | Stream metadata provides deterministic replay window |
| **Complexity** | Simple to start, brittle at scale | Slight setup overhead, but robust and self-maintaining |

---

### Technical Limitations of Timestamps

1. **Human Dependency** - Assumes every system updates `updated_at` correctly, which is unreliable across multiple sources.  
2. **Time Drift** - Late or out-of-order data can break “last timestamp” logic.  
3. **Deletes Are Invisible** - A deleted row has no timestamp, so the warehouse never knows it’s gone.  
4. **Inefficient Reprocessing** - Pipelines often reload overlapping windows (“last 24 hours”) to catch late data, wasting compute.

---

### Why a Tech Lead Chooses Streams

From a **business and architectural** perspective, Streams provide several long-term advantages:

| Benefit | Business Impact |
|----------|----------------|
| **Data integrity & trust** | Every change (insert, update, delete) is guaranteed to be captured - critical for audit and compliance. |
| **Operational efficiency** | Less custom logic and maintenance; lower failure rates. |
| **Compute savings** | Streams only process changed rows, drastically reducing warehouse time. |
| **Real-time readiness** | Enables near–real-time CDC without external systems. |
| **Simpler architecture** | Removes the need for checkpoint logic and timestamp comparisons. |
| **Auditability** | Provides deterministic replay and historical traceability for debugging and governance. |

**In short**:  
> **Timestamps detect changes if everything goes right.  
> Streams detect changes even when things go wrong.**

---

### Analogy

> A timestamp-based pipeline is like asking employees what time they *think* they arrived.  
> A Stream-based pipeline is like reading the building’s entry logs - every entry, exit, and edit is recorded precisely and automatically.

---

### When Timestamps Are Still Fine

Not every project needs the full complexity or robustness of Streams. For smaller or append-only systems, **timestamp-based pipelines** can still be perfectly valid.

| Scenario | Recommended Approach | Why |
|-----------|----------------------|-----|
| **Simple append-only data** (no updates/deletes) | Timestamps | Easy to implement; no change tracking needed. |
| **Prototyping or proof-of-concept** | Timestamps | Quick setup; focus on testing business logic. |
| **Third-party sources with reliable `updated_at`** | Timestamps | Consistent update logic makes it safe. |
| **Large-scale, frequently updated datasets** | Streams | Avoids missing changes and reduces compute costs. |
| **Compliance, auditing, or CDC integration** | Streams | Ensures full change tracking and replayability. |

---

### Key Takeaways

- **Timestamps** are simple and quick - best for lightweight, append-only pipelines.  
- **Streams** are robust and enterprise-ready - designed for systems with updates, deletes, or audit requirements.  
- **Tech leads choose Streams** for long-term maintainability, cost savings, and data trustworthiness.  

---

**Summary:**  
> Choosing Streams over timestamps isn’t about complexity - it’s about **reliability and scalability**.  
> Streams turn incremental logic from a manual guessing game into a **deterministic, metadata-driven CDC system**,  
> ensuring that every insert, update, and delete is captured - accurately, efficiently, and at scale.

---

# Block 3 - Data Ingestion & Orchestration & (some) Data Governance 
Connecting external data sources (like S3) and coordinating full EL->T flows with orchestration tools.

---

## 3.1 Stages in Snowflake - The Bridge Between Storage and Compute

### Setting the Context: ETL vs ELT

Before learning how Snowflake loads data, let’s recall a foundational concept in data engineering:  
- the difference between **ETL** and **ELT**.  

| Term | Stands For | Sequence | Where Transformation Happens | Example Tooling |
|------|-------------|-----------|------------------------------|-----------------|
| **E-T-L** | Extract → Transform → Load | Data is transformed *before* loading into the warehouse. | In external systems (e.g., `Spark`, `Python`, `Informatica`). | Traditional data warehouses |
| **EL-T** | Extract & Load → Transform | Raw data is first *loaded as-is* into the warehouse, then transformed there. | Inside Snowflake using SQL or `dbt`. | Modern cloud data platforms |

---

### Why Modern Architectures Use EL->T

In the past (ETL era), compute was expensive and warehouses were rigid. You transformed data **before** loading it to reduce storage costs.  

Today, with Snowflake and other cloud-native systems:
- **Compute is elastic** - you can scale up/down instantly.  
- **Storage is cheap** - it’s efficient to store raw data.  
- **SQL is powerful** - you can transform data directly inside the warehouse.  

That’s why most Snowflake pipelines follow the **ELT pattern**:
1. **Extract** → pull data from APIs, databases, or files into cloud storage (like S3).  
2. **Load** → use **Snowflake Stages** and `COPY INTO` to bring it into Snowflake.  
3. **Transform** → apply business logic with SQL, dbt, or stored procedures.

> In short: ELT embraces Snowflake’s strengths - scalability, simplicity, and cost-efficiency.

---

### What Are Stages?

A **Stage** is a *location where files live before being loaded into Snowflake*.  
It acts as the **bridge** between external file storage (like S3) and Snowflake’s **internal tables**.

When you run a `COPY INTO` command, Snowflake looks for files in a stage.

| Stage Type | Where Data Physically Lives | Example Use |
|-------------|-----------------------------|--------------|
| **Internal Stage** | Inside Snowflake-managed cloud storage | Quick ad hoc file uploads |
| **External Stage** | Outside Snowflake, e.g. AWS S3, Azure Blob, GCS | Production pipelines connected to data lakes |
| **Named Stage** | A user-created object referencing internal or external storage | Reusable for automated data loads |

---

### Analogy

> Think of a **Stage** as a **loading dock** for your data warehouse.  
> Raw files (from S3, local uploads, or APIs) are temporarily parked here before being “moved” into structured tables.

---

### Internal vs External Stages

| Type | Description | Example Definition | Typical Use |
|------|--------------|--------------------|--------------|
| **Internal Stage** | Files are stored directly within Snowflake’s environment. | `@%my_table` (table stage) or `@my_internal_stage` | Small uploads, manual data testing, secure internal pipelines. |
| **External Stage** | Points to cloud storage like S3, Azure Blob, or GCS via credentials or IAM roles. | `CREATE STAGE s3_stage URL='s3://my-bucket/path/' CREDENTIALS=(AWS_KEY_ID=… AWS_SECRET_KEY=…);` | Large-scale production pipelines, automated ingestion. |

---

### Example - Creating an External Stage for S3

```sql
    CREATE OR REPLACE STAGE s3_stage
    URL = 's3://company-data/raw/sales/'
    CREDENTIALS = (AWS_KEY_ID = 'AKIA...' AWS_SECRET_KEY = 'XXXXXX')
    FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1);
```

This definition:
- Connects Snowflake to an **S3 bucket path**.  
- Authenticates using **AWS credentials** (or IAM role).  
- Defines a **file format** (so Snowflake knows how to read the files).

---

### Example - Loading Data from the Stage into a Table

```sql
    COPY INTO staging_sales
    FROM @s3_stage
    FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1)
    ON_ERROR = 'CONTINUE';
```

- `COPY INTO` loads files from the stage into your table (`staging_sales`).  
- The process runs **in parallel** across multiple files and Snowflake compute nodes.  
- `ON_ERROR='CONTINUE'` ensures bad rows don’t block the load.

---

### What Happens Under the Hood

When you run a `COPY INTO` from a stage:
1. Snowflake’s **Cloud Services Layer** reads file metadata from S3.  
2. The **Compute Layer** (warehouse) loads and parses data in parallel.  
3. The **Storage Layer** saves it as compressed micro-partitions.  
4. Successfully loaded files are marked in metadata to prevent reloading duplicates.

> **Result:** high-speed, fault-tolerant, idempotent ingestion from S3 directly into Snowflake.

---

### Business Goal

**Goal:**  
Establish a secure and reliable connection between Snowflake and cloud storage (S3) so that data can be ingested automatically into the warehouse for downstream analysis.

**Why It Matters:**  
- Eliminates manual uploads or third-party connectors.  
- Ensures data pipelines are **repeatable, auditable, and scalable**.  
- Enables combining ingestion with **Tasks and Streams** for continuous ELT.

Common Use Cases:
- Daily ingestion of raw event logs from S3.  
- Loading CSV exports from SaaS systems (Stripe, Shopify, Salesforce).  
- Integrating third-party data vendors who deliver S3 files.

---

### Key Takeaways

- **Stages** are the entry point for data loading into Snowflake.  
- **External Stages** connect Snowflake directly to S3 or other cloud storage.  
- **COPY INTO** moves data from the Stage into structured tables.  
- Together, they form the “Extract → Load” part of modern **ELT pipelines**.  
- You can then transform the data using SQL, dbt, or stored procedures inside Snowflake.

---

**Summary:**  
> Stages are Snowflake’s built-in bridge between external storage and the warehouse.  
> They enable seamless, secure, and scalable data loading - forming the foundation of every modern ELT pipeline.

---

## 3.2 File Formats, Schema Inference & Schema Drift

### Why File Formats Matter

When Snowflake loads files from S3 (CSV, JSON, Parquet), it must know **how to read them** -  
where columns start and end, what separates fields, whether headers exist, and how to interpret data types.  

That logic is defined by a **File Format**.

| File Type | Structure | Schema Inference | Typical Use |
|------------|------------|------------------|--------------|
| **CSV** | Flat, delimited text | ❌ None (must define target columns manually) | SaaS exports, logs |
| **JSON** | Nested, semi-structured | ✅ Partial (new keys appear dynamically in `VARIANT`) | APIs, event data |
| **Parquet** | Binary, columnar | ✅ Full (schema stored in metadata) | Large analytical datasets |

**Example - Define a CSV File Format**
```sql
    CREATE OR REPLACE FILE FORMAT csv_sales_format
      TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1;
```
Then load files with:

    COPY INTO staging_sales
    FROM @s3_stage
    FILE_FORMAT = (FORMAT_NAME = 'csv_sales_format');

---

### Schema Inference - Letting Snowflake “Read” the File

Some file formats can describe their structure automatically.

| Format | Schema Inference Support | Notes |
|---------|---------------------------|--------|
| **CSV** | ❌ No | Table schema must match file exactly. |
| **JSON** | ✅ Flexible | New keys automatically appear when stored as `VARIANT`. |
| **Parquet** | ✅ Complete | Snowflake reads schema from Parquet metadata. |

**Example - Preview Parquet File Structure**
```sql
    SELECT * FROM @s3_stage (FILE_FORMAT => 'PARQUET');
```
Snowflake infers column names and types directly from Parquet metadata - you can even create a table directly from that inferred schema.

---

### Schema Drift - A Real-World Problem

**Schema drift** means the **structure of incoming files changes over time** - new columns appear, old ones disappear, or data types evolve.  

This doesn’t mean Snowflake automatically adjusts - it means your *data changed*, and your ingestion pipeline must decide **how to respond**.

| Type of Drift | Example | Impact |
|----------------|----------|--------|
| **New column** | New field `discount` added to CSV | `COPY INTO` fails - column mismatch |
| **Removed column** | `region` column missing in next file | Column loads as NULL |
| **Type change** | `amount` becomes `"100.00"` (string) | Causes cast errors or inconsistent data |

---

### Example - Schema Drift in a CSV Pipeline

**Yesterday’s file (3 columns)**  
| order_id | amount | region |
|-----------|---------|--------|
| 100 | 90 | ES |

**Today’s file (4 columns)**  
| order_id | amount | region | discount |
|-----------|---------|--------|----------|
| 101 | 120 | DE | 10 |

**Result:**  
A `COPY INTO` will fail because Snowflake expects 3 columns but receives 4.  
This is **schema drift** - the data changed, but your ingestion logic didn’t.

---

### Why It Matters - A Data Governance Perspective

In the real world, schema drift is not a “bug” - it’s a **signal** that your data contracts are changing. It’s both a **data quality** and a **data governance** issue.

| Governance Lens | Why It Matters |
|------------------|----------------|
| **Data Quality** | Undetected drift can cause silent data loss or broken joins. |
| **Traceability** | Without logging schema versions, it’s hard to audit what changed and when. |
| **Accountability** | Teams need to know which upstream system introduced the change. |
| **Risk Management** | In regulated environments, silent schema drift can lead to compliance violations (e.g., missing mandatory fields). |

A robust pipeline doesn’t try to *prevent* schema drift - it **monitors and reacts** to it.

---

### How to Handle Schema Drift Responsibly

| Approach | Description | When to Use |
|-----------|--------------|-------------|
| **Manual evolution** | Use `ALTER TABLE ADD COLUMN` when drift occurs | Stable CSV pipelines |
| **Flexible ingestion** | Load data as `VARIANT` (for JSON/Parquet) | For frequent or unpredictable drift |
| **Schema validation layer** | Compare new file structure to expected schema before loading | For critical or regulated pipelines |
| **Metadata tracking** | Store schema versions and ingestion timestamps in a control table | For governance and debugging |
| **Alerting** | Use Tasks or dbt tests to notify when new fields appear | For automated observability |

**Example - Detecting Drift Before Load**
```sql
    SELECT METADATA$FILENAME, COUNT(*)
    FROM @s3_stage (FILE_FORMAT => 'CSV')
    GROUP BY 1;
```
You can inspect staged files or compare column counts before executing the `COPY INTO` command, allowing your pipeline to log or alert if a mismatch is detected.

---

### Key Takeaways

- **File Formats** define how Snowflake interprets files.  
- **Schema Inference** helps Snowflake discover structure (especially in Parquet).  
- **Schema Drift** is not an error - it’s a *change in the data contract*.  
- **Governance-aware pipelines** monitor drift, log schema versions, and alert when changes occur.  
- The goal isn’t to stop drift - it’s to **detect and manage it before it breaks production**.

> In modern data systems, schema drift is inevitable.  
> Good data engineering doesn’t fear it - it plans for it.

