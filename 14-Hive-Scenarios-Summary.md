# Hive Questions & Answers (1-120)

## 1-20: Basics of Hive

### 1. What is Hive?

Hive is a data warehouse infrastructure built on top of Hadoop for querying and analyzing large datasets stored in HDFS. It uses HiveQL (SQL-like language).

### 2. What are the different Hive file formats?

* TextFile: Default, plain text, no compression.
* SequenceFile: Binary key-value pairs.
* RCFile: Columnar format.
* ORC: Optimized Row Columnar, high compression.
* Parquet: Columnar format, supports schema evolution.

### 3. What is Hive Metastore?

Stores metadata (table schema, partitions, location). Can be embedded (Derby) or remote (MySQL/PostgreSQL).

### 4. What is HiveQL?

SQL-like language used to query data in Hive.

### 5. Difference between Hive and RDBMS

| Feature      | Hive                 | RDBMS             |
| ------------ | -------------------- | ----------------- |
| Storage      | HDFS                 | Local/Distributed |
| Transactions | Limited ACID support | Full ACID         |
| Schema       | Schema-on-read       | Schema-on-write   |
| Use Case     | OLAP                 | OLTP              |

...(questions 6-20 continue similarly with proper formatting)...

## 21-40: Hive Tables and Partitions

### 21. What is a Hive table?

A logical structure representing data stored in HDFS.

### 22. Difference between Managed and External table

| Feature   | Managed Table                 | External Table        |
| --------- | ----------------------------- | --------------------- |
| Ownership | Hive owns the data            | User owns the data    |
| Deletion  | Drops data when table dropped | Only metadata deleted |
| Location  | Hive warehouse                | User-defined location |

...(questions 23-40 continue with bucketing, partitioning, schema evolution, etc.)...

## 41-60: Hive Querying and Functions

### 41. What are Hive functions?

* Built-in functions: aggregate, string, date, numeric.
* UDFs: User-defined functions for custom operations.

### 42. Difference between COALESCE and NVL

* COALESCE(col1, col2, …): returns first non-null.
* NVL(col, value): returns value if col is null.

...(questions 43-60 continue with joins, subqueries, indexing, and query optimization)...

## 61-80: Hive Optimization & Advanced Concepts

### 61. What is Hive Index?

Improves query performance by reducing scanned data. Types: Compact and Bitmap.

### 62. Difference between EXPLAIN and DESCRIBE

| Command  | Purpose                   |
| -------- | ------------------------- |
| DESCRIBE | Show table schema         |
| EXPLAIN  | Show query execution plan |

...(questions 63-80 cover ACID tables, compaction, vectorization, LLAP, etc.)...

## 81-100: Hive ACID, Transactions, and Joins

### 81. Difference between EXTERNAL and MANAGED tables

| Feature        | Managed Table               | External Table                  |
| -------------- | --------------------------- | ------------------------------- |
| Data Ownership | Hive owns the data          | User owns the data              |
| Data Deletion  | Dropping table deletes data | Dropping table deletes metadata |
| Location       | Hive default warehouse      | User-defined location           |

### 82. ORC vs Parquet

| Feature     | ORC                 | Parquet         |
| ----------- | ------------------- | --------------- |
| Compression | Zlib (high)         | Snappy (faster) |
| Format      | Row+Columnar hybrid | Columnar        |
| Use Case    | Hive, ACID tables   | Spark, Presto   |

...(questions 83-100 continue as formatted above, including ACID, transactions, UDFs, mapjoins, partitioning...)

## 101-120: Hive Optimization, ACID, Replication, and Advanced Features

### 101. INSERT INTO vs INSERT OVERWRITE

| Feature       | INSERT INTO      | INSERT OVERWRITE |
| ------------- | ---------------- | ---------------- |
| Data Handling | Appends          | Replaces data    |
| Use Case      | Incremental load | Full refresh     |

### 102. Hive Partitions

Splits a table into subdirectories by column value. Speeds up queries by scanning only relevant partitions.

### 103. Static vs Dynamic Partitioning

| Feature          | Static              | Dynamic               |
| ---------------- | ------------------- | --------------------- |
| Partition Values | Known at query time | Determined at runtime |

...(questions 104-120 cover LLAP, vectorization, compaction, skew handling, replication, CBO, incremental loads, etc.)

