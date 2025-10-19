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
