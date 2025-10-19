## Q1. Important considerations before performing Hive query performance tuning:

### **1. Table Design & Storage Considerations**

Before tuning, understand how your table is created:

* **Storage format:** Text, ORC, Parquet, Avro, RCFile, etc.
* **Compression:** Is compression enabled (SNAPPY, ZLIB, GZIP)?
* **Partitioning:** Helps prune unnecessary data.
* **Bucketing:** Useful for efficient joins and sampling.

### **2. Gather Table Statistics**

Manual observation and metadata collection help guide optimizations:

* **Data growth rate:** How fast the table size is increasing.
* **Table size & column types:**

  * Example: Large textual columns like chat/call transcripts or narrative feedback (~1.5 GB) might impact performance.
  * **Vectorization:** Avoid reading too many rows at once for heavy textual data (default 1024 rows).
  * **Sqoop fetch size:** Optimize while importing data.
* **Number of columns & rows:**

  * For wide tables, consider **columnar formats** like ORC or Parquet.
* **Cardinality of columns:** Helps decide partitions and buckets for efficient query execution.

### **3. Query Patterns & Clauses**

Understanding your query patterns helps pick optimizations:

* **Inline queries or CTEs**
* **WHERE clauses:** Prefer partition filters for pruning.
* **Joins:** Use bucketing to improve performance on large tables.
* **Aggregations / GROUP BY / HAVING / ORDER BY / LIMIT**
* **Set operations:** UNION / INTERSECT

### **4. Data Writing Considerations**

* **Frequency:** One-time load vs. regular batch loads.
* **Serialization format:** Avro, JSON, Text, ORC, Parquet, RCFile.
* **Compression:** Choose a format that balances read/write speed (e.g., ORC/Parquet with SNAPPY).
* **Reading efficiency:** Columnar formats + compression improve scan times.

### **5. Logging & Monitoring**

* Introduce **logger tables** to capture:

  * Execution time
  * Status of HQL statements
  * Helps identify slow steps in pipelines

### **6. Execution Plan Analysis**

* Use **EXPLAIN** to understand Hive query execution:

  * Join strategy (MapJoin, SortMergeJoin, etc.)
  * Whether **predicate pushdown** or other optimizations are applied

### **7. Environment & Configuration Settings**

Tune both default and query-specific settings:

* **Execution engine:** MR vs. Tez vs. Spark
* **Join types:** Broadcast vs. MapJoin
* **Reducers:** Count, split size, number of reducers
* **CBO (Cost-Based Optimizer)**
* **Vectorization** and parallel execution
* **Heap memory & other JVM settings**

### **Key Takeaways**

* Performance tuning starts **before writing queries**—it’s about understanding **table design, data size, query patterns, and environment settings**.
* Use **columnar formats, compression, partitions, and bucketing** strategically.
* Always validate with **EXPLAIN** and **execution logs** before finalizing optimizations.

---

## Q2. Hive common interview questions:

### **1. Hive MR vs Impala vs Tez**

| Feature          | Hive MR                     | Hive Tez          | Impala                               |
| ---------------- | --------------------------- | ----------------- | ------------------------------------ |
| Execution engine | MapReduce                   | Tez DAG           | Native MPP engine                    |
| Latency          | High                        | Medium            | Low (real-time queries)              |
| Throughput       | Good for batch              | Faster than MR    | High throughput, interactive queries |
| Use case         | Batch processing, large ETL | Faster batch jobs | Interactive analytics, BI dashboards |
| Integration      | Hadoop ecosystem            | Hadoop ecosystem  | Hadoop ecosystem (HDFS)              |

> **Key takeaway:** MR is slow, Tez faster batch, Impala low-latency interactive queries.

### **2. Manually added partition folder not fetched**

* Hive doesn’t automatically recognize new folders.
* **Solutions:**

  * `ALTER TABLE table_name ADD PARTITION (partition_col='value') LOCATION 'path';`
  * **Fool-proof:** `MSCK REPAIR TABLE table_name;`

> **MSCK repair** scans the table location and automatically adds all missing partitions.

### **3. When to choose partitioning & bucketing**

* **Partitioning:**

  * Use when a column has **high cardinality and is often filtered** (`WHERE partition_col=value`).
  * Reduces data scanned → faster queries.
* **Bucketing (sorted buckets):**

  * Use when performing **joins or aggregations** on columns.
  * Improves performance by **reducing shuffle**, enabling **bucketed joins**.
* **Example:**

  ```sql
  CREATE TABLE sales(
      id INT,
      product STRING,
      amount FLOAT
  )
  PARTITIONED BY (year INT)
  CLUSTERED BY (product) INTO 8 BUCKETS
  STORED AS ORC;
  ```

### **4. AVRO vs PARQUET vs ORC**

| Feature          | AVRO                            | Parquet        | ORC                                    |
| ---------------- | ------------------------------- | -------------- | -------------------------------------- |
| Format           | Row-oriented                    | Columnar       | Columnar                               |
| Compression      | Supports                        | Supports       | High compression                       |
| Schema evolution | Easy                            | Moderate       | Moderate                               |
| Performance      | Good for writes                 | Good read/scan | Best for Hive, vectorized read         |
| Use case         | ETL pipelines, schema evolution | Analytics & BI | Hive batch queries, heavy aggregations |

> **Key takeaway:** Use ORC/Parquet for analytics; AVRO when schema evolution is needed.

### **5. Performance improvement in Hive**

* Use **columnar storage** (ORC/Parquet)
* **Partitioning** & **bucketing**
* Enable **vectorization**
* Use **Tez execution engine** instead of MR
* **Predicate pushdown** (filter early)
* Avoid **small files**, use **merge**
* Use **map-side joins** (for small tables)
* Use **CBO (Cost-Based Optimizer)**
* Proper **heap & reducer tuning**

### **6. Table streaming / stream table concept in joins**

* **Idea:** When joining tables, small tables can be **streamed into memory** (hash table)
* **Best practice:**

  * Stream the **smaller table**
  * Larger table is scanned sequentially
* **Hash table:** Used to avoid skew and allow **map-side joins**

> Helps in **avoiding shuffle** and improving join performance.

### **7. Hive Index**

* **Concept:** An index table stores **pointers/addresses** to original data rows for faster lookup.
* **Performance:**

  * Reduces scan of entire table for **filtering operations**.
  * Mostly useful for selective queries.
* **Caveats:**

  * Hive Indexing deprecated from Hive 3.0 onwards.
  * **Alternative:** Materialized views or **ORC/Parquet**, which inherently support **column pruning and indexing**.

---
