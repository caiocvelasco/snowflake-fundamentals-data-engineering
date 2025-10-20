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

### 1.4 Compute Layer: Virtual Warehouses & Parallelism

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

#### Responsibilities
- **Parse & optimize** SQL queries.  
- **Decide** which micro-partitions to read (_pruning_).  
- **Assign** work to compute nodes.  
- **Check** permissions via RBAC.  
- **Manage** caches and query metadata.

#### Example
```sql
SELECT * FROM orders WHERE region = 'Europe';
```
1. Cloud Services parses query & checks role privileges.  
2. Reads metadata → only partitions containing `'Europe'`.  
3. Tells compute layer which partitions to read.  
4. Combines results and sends them back.

---

### 1.6 Caching: Remembering Previous Work

Caching = reusing previous data or results to save time and cost.

| Cache Type | Where | Stores | Reused When |
|-------------|--------|---------|--------------|
| **Result cache** | Cloud Services Layer | Final results of identical queries | Same SQL *and* same data snapshot |
| **Metadata cache** | Cloud Services Layer | Micro-partition statistics | Always used for pruning |
| **Local cache** | Compute Layer | Recently read micro-partitions | Same warehouse runs again before suspension |

---

## Deep Dive - “Same SQL” and “Same Data”

Snowflake has multiple caches that operate at different levels. As seen above, they are the _Result Cache_, _Local Cache_, and _Metadata Cache_.

Each one “remembers” something different. Let’s break them down:

### Result Cache (Same SQL, Same Data)

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

### Local Cache (Same Data Accessed Again by the Same Warehouse)

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

### Metadata Cache (Used by the Optimizer)

The **Metadata Cache** is always active and operates behind the scenes in the **Cloud Services Layer**.

#### What It Stores
- Summary information (metadata) about each **micro-partition**:
  - Minimum and maximum values per column, Row counts and null counts, File and partition locations, Data versioning information.

This metadata is lightweight but extremely valuable. 

-> It allows the **optimizer** to know where data is located **without reading it first**.

#### Why It Matters
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

Snowflake’s security model is built on **Role-Based Access Control (RBAC)** —  
every user acts *through a role*, and that role defines *what they can see or do*.

---

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

**Best practice:**  
Use _least privilege_ - give users the minimum necessary role to do their job.

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

🧭 **Roles → Privileges → Objects → Users**  
This chain defines *everything* about access in Snowflake.

---

### 1.8 Performance Optimization Techniques

| Area | Technique | Why it helps |
|-------|------------|--------------|
| **Query pruning** | Use filters on columns Snowflake can prune. | Reads fewer partitions. |
| **Clustering keys** | Group data manually by frequent filters. | Improves pruning for large tables. |
| **Warehouse sizing** | Scale wisely, use auto-suspend. | Reduces cost. |
| **Materialized views** | Pre-compute common aggregations. | Faster dashboards. |
| **Query profiling** | Use `QUERY_HISTORY()` or UI profiler. | Identify slow steps. |

#### Example
```sql
SELECT customer_id, SUM(amount)
FROM orders
WHERE country = 'DE'
GROUP BY customer_id;
```
Possible optimizations:
- Cluster by `(country)` for better pruning.  
- Materialize frequent aggregates.  
- Scale out if concurrency is high.

---

### 🧭 1.9 Visual Summary

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

### ✅ 1.10 Quick Recap Checklist

- [ ] I can explain why Snowflake was created.  
- [ ] I understand the difference between S3 partitions and Snowflake micro-partitions.  
- [ ] I know what “parallel processing” means.  
- [ ] I can describe the three layers and what happens in each.  
- [ ] I know what caching is and how each cache type works.  
- [ ] I can mention at least three performance optimization techniques.

---

Next step → **Procedural SQL: Stored Procedures, Streams, and Tasks** 🚀
