

## Q1. How to see the default value set for a config in Hive?

Run the `SET` command with the property name:

```sql
set hive.fetch.task.conversion;
```

**Answer:**

* It shows the current value (default or overridden).
* If not explicitly set in `hive-site.xml` or via `SET`, Hive returns the default value.

---

## Q2. Why does a `SELECT *` query in Hive not run MapReduce/Tez/Spark?

**Answer:**

* Controlled by `hive.fetch.task.conversion`.
* For simple queries (`SELECT`, `FILTER`, `LIMIT`), Hive skips MapReduce/Tez/Spark and directly fetches data.

**Values:**

* **none** – Always use MapReduce/Tez/Spark.
* **minimal** – Optimization only for very simple queries.
* **more** – Extended optimization for filters/projections (default).

---

## Q3. How to change Hive configs permanently within the current HQL session / cluster / job level?

**Answer:**

### 1. Session / Query Level (Temporary)

* Set properties for the current session only:

```sql
set hive.exec.engine=mr;
set hive.map.size=1024m;
set hive.vectorized.exec.enabled=true;
```

* Hive CLI / Script Level: `SET` command or `-hiveconf`
```bash
hive -hiveconf hive.exec.engine=mr -f my_query.hql
```

* These settings last **only for the current HQL session**.

### 2. Job / Oozie Level (Temporary)

* If you are using a job scheduler (like Oozie) that packages and executes your job, you can include a *custom `hive-site.xml`* containing only the necessary overrides.
* Settings are applied **for jobs using that config**.

### 3. Cluster / Admin Level (Permanent)

* Admin can change configs cluster-wide using:

  * `hive-site.xml` on all nodes
  * Cluster management tools like **Ambari, Cloudera Manager, EMR Manager, Dataproc Manager**

---

## Q4. Can I use multiple columns in a subquery `WHERE ... IN` clause in Hive?

**Answer:**

* **Hive does not support multi-column `IN` subqueries** like:

```sql
SELECT * FROM tableA
WHERE (A.id, A.name, A.Roll_no) IN (
    SELECT Id, name, roll_no FROM tableB
);
```

* **Workarounds:**

**Option 1: Use `CONCAT`**

```sql
SELECT * FROM tableA
WHERE CONCAT(A.id, A.name, A.Roll_no) IN (
    SELECT CONCAT(Id, name, roll_no) FROM tableB
);
```

**Option 2: Use `JOIN`**

```sql
SELECT a.*
FROM tableA a
INNER JOIN tableB b
ON a.id = b.id
AND a.name = b.name
AND a.rollno = b.rollno;
```

**Key:** Multi-column matching is supported via `JOIN` or `CONCAT`, not direct multi-column `IN`.

---

## Q5. How to skip header/footer rows from a table in Hive?

**Answer:**

* Use **table properties** when creating or altering the table:

```sql
-- Skip first 2 header rows
TBLPROPERTIES ("skip.header.line.count"="2");

-- Skip last 1 footer row
TBLPROPERTIES ("skip.footer.line.count"="1");
```

* Applies when reading data from **text/CSV files**.
* Useful for **ignoring file headers or summary/footer lines**.

**Key:** Only works for **external or managed tables with text-based formats**.

**Example:**

```sql
vi /home/hduser/students.csv
Name,Age,Class
John,15,10
Alice,14,9
Bob,16,11
Summary: 3 rows

hadoop fs -mkdir /user/hive/warehouse/students/
hadoop fs -put /home/hduser/students.csv /user/hive/warehouse/students/

--hive

DROP TABLE IF EXISTS default.students;

CREATE EXTERNAL TABLE default.students(
    Name STRING,
    Age INT,
    Class STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/students/'
TBLPROPERTIES (
    "skip.header.line.count"="1",
    "skip.footer.line.count"="1"
);

SELECT * FROM default.students;
John    15    10
Alice   14    9
Bob     16    11

```

✅ The header (`Name,Age,Class`) and footer (`Summary: 3 rows`) are **skipped automatically**.

---

## Q6. What is the maximum size of the `STRING` data type in Hive?

**Answer:**

* Maximum size of `STRING` in Hive is **2 GB** per value.
* **Practical limit** depends on:

  * **Available memory** – large strings may cause OOM errors.
  * **File format** – ORC/Parquet handle large strings more efficiently than text files.

**Key:** `STRING` is variable-length; actual usable size may be smaller than 2 GB.

---

Here’s a **short, GitHub-outline-friendly version** of your question:

---

## Q7. How to access data in multiple HDFS subdirectories using a single Hive table?

**Answer:**

* Hive can read data **recursively from subdirectories** using these settings:

```sql
-- Enable recursive reading of HDFS directories
SET mapred.input.dir.recursive=true;

-- Allow Hive to support subdirectories
SET hive.mapred.supports.subdirectories=true;
```

* Example HDFS layout:

```sql
hadoop fs -mkdir /user/hive/warehouse/students/cust1
hadoop fs -mkdir /user/hive/warehouse/students/cust1/cust2
hadoop fs -mkdir /user/hive/warehouse/students/cust3

hadoop fs -put /home/hduser/students.csv /user/hive/warehouse/students/cust1/
hadoop fs -put /home/hduser/students.csv /user/hive/warehouse/students/cust1/cust2/
hadoop fs -put /home/hduser/students.csv /user/hive/warehouse/students/cust3/

--hive
---- 1. Enable recursive reading of HDFS directories
SET mapred.input.dir.recursive=true;

---- 2. Allow Hive to support subdirectories
SET hive.mapred.supports.subdirectories=true;

DROP TABLE IF EXISTS default.students_r;

CREATE TABLE default.students_r(
    Name STRING,
    Age INT,
    Class STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/students/'
TBLPROPERTIES (
    "skip.header.line.count"="1",
    "skip.footer.line.count"="1"
);
  
SELECT * FROM default.students_r;
students.name	students.age	students.class
John	15	10
Alice	14	9
Bob	16	11
John	15	10
Alice	14	9
Bob	16	11
John	15	10
Alice	14	9
Bob	16	11
John	15	10
Alice	14	9
Bob	16	11

```
---


## Q9. How to resolve `OutOfMemoryError: Java heap space` in Hive when using UDFs?

**Scenario:**

* A UDF (custom java code or python code) works fine on small datasets (e.g., transcript masking).
* Fails with **OutOfMemoryError** when dataset grows to GBs (e.g., audio-to-text data).

**Solution:** Increase memory for MapReduce tasks:

```sql
-- Mapper memory
SET mapreduce.map.memory.mb=4096;
SET mapreduce.map.java.opts=-Xmx3686m; --JVM heap.

-- Reducer memory
SET mapreduce.reduce.memory.mb=4096;
SET mapreduce.reduce.java.opts=-Xmx3686m;  --JVM heap.
```

**Tips:**

* **Increase memory** for mappers/reducers: `mapreduce.map.memory.mb`, `mapreduce.reduce.memory.mb`, and corresponding `-Xmx`.
* **Optimize UDFs** to process rows individually and avoid large in-memory buffers.
* **Split large files** and/or use **efficient formats** (ORC/Parquet) and execution engines (Tez/Spark).

* See references:

  * [AWS EMR guide](https://aws.amazon.com/premiumsupport/knowledge-center/emr-hive-outofmemoryerror-heap-space/)
  * [StackOverflow discussion](https://stackoverflow.com/questions/29673053/java-lang-outofmemoryerror-java-heap-space-with-hive)

**Key:** Hive UDFs run inside JVM; **insufficient memory in mappers/reducers** causes heap space errors.

---

## Q10. Hive query failed with "vertex error" — how to fix it?

**Answer:**

* This error usually occurs when **Tez or Spark execution fails** on a specific vertex/task.
* **Quick fix:** switch the execution engine to MapReduce:

```sql
SET hive.execution.engine=mr;
```

* This forces Hive to run the query using MapReduce instead of Tez/Spark.
* Reference: [StackOverflow discussion](https://stackoverflow.com/questions/49420108/hive-vertex-failed-vertexname-map)

**Key:** Vertex errors often indicate **engine-specific issues**; using MR can be a workaround for stability.

---

## Q11. How to read fixed-width datasets in Hive?

**Answer:**
Hive does not natively support fixed-width files, but you can handle them using **`substr()`** or **`RegexSerDe`**.

### Option 1: Using `substr()`

* **Example file `students_fixed.txt`:**

```
vi /home/hduser/students_fixed.txt
Jackdosan Rag      37
Ecooler Eddiee     27

hadoop fs -mkdir /user/hive/warehouse/students_fixed
hadoop fs -put /home/hduser/students_fixed.txt /user/hive/warehouse/students_fixed/
```

* **Step 1: Create table with single string column**

```sql
DROP TABLE IF EXISTS default.raw_students;

CREATE TABLE default.raw_students(line STRING)
LOCATION '/user/hive/warehouse/students_fixed/';
```

* **Step 2: Parse fields using `substr()`**

```sql
SELECT
    TRIM(SUBSTR(line,1,19)) AS name,
    CAST(TRIM(SUBSTR(line,20,2)) AS INT) AS age
FROM default.raw_students;
```

* **Optional:** Insert into a CSV-style table

```sql
CREATE TABLE default.students_parsed(name STRING, age INT)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

INSERT INTO default.students_parsed
SELECT
    TRIM(SUBSTR(line,1,19)),
    CAST(TRIM(SUBSTR(line,20,2)) AS INT)
FROM default.raw_students;
```

### Option 2: Using RegexSerDe

* **Step 1: Create table with RegexSerDe**

```sql
DROP TABLE IF EXISTS default.students_regex;

CREATE TABLE default.students_regex (
    name STRING,
    age STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
    "input.regex" = "(.{19})(.{2})"
)
LOCATION '/user/hive/warehouse/students_fixed/';
```

* **Step 2: Query and cast fields**

```sql
SELECT TRIM(name) AS name, CAST(TRIM(age) AS INT) AS age
FROM default.students_regex;
```

**Key:**

* `substr()` + `trim()` works for **simple parsing**.
* **RegexSerDe** is cleaner for **structured fixed-width files**.

---

## Q12. How to Remove Duplicates in Hive?

### 1. Create sample data

```sql
USE default;

-- Drop if exists for rerun convenience
DROP TABLE IF EXISTS default.oldtable;

-- Create sample table
CREATE TABLE default.oldtable (
  id INT,
  name STRING,
  phone STRING,
  ts STRING
);

-- Insert sample records (with duplicates)
INSERT INTO default.oldtable VALUES
(1, 'Aarav',  '999110045', '2023-08-10 10:00:00'),
(1, 'Aarav',  '999110045', '2023-08-10 10:00:00'),
(2, 'Meera',  '988220078', '2023-08-12 11:30:00'),
(2, 'Meera',  '988220078', '2023-08-13 09:45:00'),
(3, 'Rohit',  '977330012', '2023-08-09 08:15:00'),
(3, 'Rohit',  '977330012', '2023-08-09 08:15:00'),
(4, 'Kavya',  '966440078', '2023-08-14 15:20:00'),
(4, 'Kavya',  '966440078', '2023-08-15 16:00:00');
```

---

### 2. Option 1 — Small dataset (In-place deduplication)

#### Case A: Remove exact duplicates

```sql
INSERT OVERWRITE TABLE default.oldtable
SELECT DISTINCT * FROM default.oldtable;
```

#### Case B: Keep only latest record per person

```sql
INSERT OVERWRITE TABLE default.oldtable
SELECT id, name, phone, ts
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY id, name, phone ORDER BY ts DESC) AS rnk
    FROM default.oldtable
) t
WHERE rnk = 1;
```

**Result after cleanup:**

| id | name  | phone      | ts                  |
| -- | ----- | ---------- | ------------------- |
| 1  | Aarav | 999110045 | 2023-08-10 10:00:00 |
| 2  | Meera | 988220078 | 2023-08-13 09:45:00 |
| 3  | Rohit | 977330012 | 2023-08-09 08:15:00 |
| 4  | Kavya | 966440078 | 2023-08-15 16:00:00 |

---

### 3. Option 2 — Large dataset (Use a new table, then swap)

```sql
DROP TABLE IF EXISTS default.newtable;

CREATE TABLE default.newtable AS
SELECT id, name, phone, ts
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY id, name, phone ORDER BY ts DESC) AS rnk
    FROM default.oldtable
) t
WHERE rnk = 1;

-- Option 1: Replace old table
DROP TABLE default.oldtable;
ALTER TABLE default.newtable RENAME TO oldtable;

-- Option 2: If old table cannot be dropped
INSERT OVERWRITE TABLE default.oldtable SELECT * FROM default.newtable;
DROP TABLE default.newtable;
```

---

### Key points

* Use `DISTINCT` for literal duplicate rows.
* Use `ROW_NUMBER()` for deduplication by key (keep latest).
* Use the new-table approach for large tables — safer, less memory pressure, and easier rollback.

| Aspect              | Explanation                                                                                                                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Memory Management   | `INSERT OVERWRITE` on a huge table requires Hive to read and rewrite the entire dataset in one go. This can consume a lot of memory and may trigger `OutOfMemoryError` or Tez/Spark job failures. |
| Parallel Processing | When creating a new table with `CREATE TABLE AS SELECT`, Hive can better distribute the load across mappers and reducers — often faster and safer.                                                |
| Failure Isolation   | If something fails during the new-table creation, your original data remains intact. With `INSERT OVERWRITE`, partial failures can corrupt or lose data.                                          |
| Easier Rollback     | You can verify the data in the new table before replacing the old one. If it’s wrong, just drop the new table — no harm done.                                                                     |
| Performance Boost   | `CTAS` (`CREATE TABLE AS SELECT`) is optimized in Hive — it skips SerDe overhead and writes data faster.                                                                                          |

---

## Q13. How to Prevent Duplicate Data Loads in Hive (Immutable Tables)

### Explanation

An **immutable table** in Hive is used to prevent accidental duplicate data loads — for example, in static or confirmed **`Dimension`** data such as **calendar, geography, or store** tables.

* The **first insert** into an immutable table succeeds.
* Any **subsequent insert** fails if the table already contains data.
* This helps avoid silent duplication when a data load script runs multiple times.
* However, `INSERT OVERWRITE` is still **allowed**, even when the table is immutable.

### Usage

```sql
USE default;

DROP TABLE IF EXISTS customers1;

CREATE EXTERNAL TABLE customers1 (
  userid STRING,
  name STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION '/user/hduser/custdata1'
TBLPROPERTIES ("immutable" = "true");
```

**Default:** `TBLPROPERTIES("immutable"="false")`

### Example Behavior

```sql
-- Works: first load
INSERT INTO customers1 VALUES ('1', 'Aarav'),('2', 'Meera');

-- Fails: second insert attempt
INSERT INTO customers1 VALUES ('3', 'Phebe');

-- Works: overwrite existing data
INSERT OVERWRITE TABLE customers1 VALUES ('4', 'Rohit');
```

### Key Takeaways

* **Use for static or reference data that should never be appended again.**
* Prevents duplicate data due to rerun scripts.
* Allows controlled updates via `INSERT OVERWRITE`.

---


## Q14. How to Change a Hive Table from External to Managed and Vice Versa?

```sql

USE default;

DROP TABLE IF EXISTS customers3;

CREATE EXTERNAL TABLE customers3 (
  userid STRING,
  name STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS ORC
LOCATION '/user/hduser/custdata2'
TBLPROPERTIES ("orc.compress"="SNAPPY");

-- Convert External → Managed
ALTER TABLE customers3 SET TBLPROPERTIES('EXTERNAL'='FALSE');

-- Verify
DESCRIBE FORMATTED customers3;

-- Convert Managed → External
ALTER TABLE customers3 SET TBLPROPERTIES('EXTERNAL'='TRUE');

-- Verify
DESCRIBE FORMATTED customers3;
```

**Note:**
Only metadata changes — **data location remains the same.**   
**Managed tables** delete data on `DROP`; **External tables** don’t.  

---

## Q15. Explain End-to-End Data Management & Data Lineage Explanation

In our project, we follow a multi-zone data lifecycle — **Raw**, **Curated**, and **Presentation/Discovery**.

**1. Raw Zone:**
All incoming data — from databases, APIs, Kafka, or files — first lands in a **data lake** such as HDFS, S3, or GCS.
This zone stores data in its original form, partitioned by date, reusable, versioned, and acts as the single source of truth.

**2. Curated Zone:**
Data from the raw zone is cleaned, validated, and transformed using **Spark or Hive**.
We apply deduplication, enrichment, and business rules here.
Processed outputs are stored in optimized formats (Parquet/ORC) in Hive or BigQuery — ideal for analytics, ETL, and data modeling.

**3. Presentation / Discovery Zone:**
This layer serves different consumers:

* For **batch analytics**, we use Hive, Presto, or BigQuery.
* For **real-time access**, we load data into **NoSQL systems** like HBase, Cassandra, or Elasticsearch for dashboards and fast lookups.

**Storage choices:**

* **File systems (HDFS/S3)** → raw, immutable data.
* **Hive/BigQuery** → curated analytical data.
* **HBase/Cassandra/ES** → frequently updated or low-latency data.

**Typical Flow:**
Kafka / Source DB → Raw (HDFS/S3) → Spark / Hive → Curated (Hive / BigQuery) → NoSQL / BI Layer (HBase, Cassandra, ES)

This approach ensures **traceability, consistency, and performance** across the full data lifecycle — from ingestion to analytics and real-time delivery.

---

## Q16. Who Consumes Data from External Tables and How Processed Data Reaches Reporting?

**1. Data Consumption:**

* External Hive tables are primarily consumed by **other applications** and **reporting/visualization tools**.
* Processed data is typically stored in **Hive** using **ORC format** with **Snappy compression** for optimized storage and fast access.

**2. Data Access for Reporting:**

* **JDBC/ODBC connections** allow applications or BI tools to query Hive tables directly.
* Visualization tools like **Tableau, QlikView, Power BI** can connect via ODBC/JDBC to generate dashboards and reports.

**3. Example Data Pipeline:**

```
Source → Extract → Load → Transform → Load → Reporting
```

* **Source:** RDBMS, flat files, or streaming (Kafka).
* **Extract:** Using Sqoop, Hive (managed tables), Spark, or NiFi.
* **Load:** Into Hive (staging/curated) or NoSQL (for low-latency queries).
* **Transform:** Using HiveQL, Spark SQL, or ETL scripts.
* **Load to Target:** Hive external tables, NoSQL, RDBMS, or file system.
* **Reporting:** Access via BI tools or applications using JDBC/ODBC connections.

**4. Example Scenarios:**

* **Batch Pipeline:** Extract sales data from Oracle → Load to Hive staging → Transform using Spark → Load into Hive external table → Reporting via Tableau.
* **Real-Time Pipeline:** Stream customer activity from Kafka → Spark Streaming → Load into HBase → Dashboard queries for real-time analytics.

This setup ensures **efficient data processing, optimized storage, and accessible reporting** for both batch and real-time requirements.

---

## Q17. How to Point `.hql` Scripts to Production Instead of Non-Prod

### Problem

* The existing `.hql` scripts are **hardcoded** to a non-production database/schema.
* Need a way to run the same script against **production** without manually changing the database name.

### Solution: Parameterize the Hive Script

#### 1. Using `hivevar` Parameters

**Inside `xyz.hql`:**

```sql
SELECT * 
FROM ${db_name}.oldtable
WHERE ts='${load_dt}';
```

**Command:**

```bash
hive -hivevar db_name='default' -hivevar load_dt='2023-08-10 10:00:00' -f /home/hduser/xyz.hql
```

* `${db_name}` and `${load_dt}` are replaced at runtime.
* Works for **any environment** by just changing the parameter values.

#### 2. Using a Parameter File

**Inside `abc.hql`:**

```sql
SELECT *
FROM ${hiveconf:db_name}.oldtable
WHERE ts='${hiveconf:load_dt}';
```

**Inside `paramfile.txt`:**

```
set db_name=default;
set load_dt=2023-08-10 10:00:00;
```

**Command:**

```bash
hive -f /home/hduser/abc.hql -i /home/hduser/paramfile.txt
```

**Advantages:**

* No need to edit `.hql` files for different environments.
* Single command can switch between **dev, test, or prod**.
* Supports multiple parameters (`dbname`, `load_dt`, `engine`, etc.).

✅ **Best Practice:**

* Always parameterize `.hql` scripts using `${variable}` and pass environment-specific values via `hivevar` or a parameter file.
* Avoid hardcoding environment names to prevent accidental data access or overwrite.

---


## Q18. All About Partitions in Hive

### **Why Partitions are Important**

* Partitions allow Hive to **organize large datasets** into smaller, manageable chunks based on column values (e.g., `date`, `region`, `hour`).
* They **improve query performance** by enabling **partition pruning**, so Hive reads only relevant data.

---

### **Drawbacks of Improper Partitioning**

1. **Small Partition Volumes (Small Files)**

   * Example: `date/hour` partitions with 5 MB of data each.
   * Result: Mappers/reducers process tiny amounts of data → poor resource utilization.

2. **High Namenode Overhead**

   * Each partition and file is stored as a separate HDFS object.
   * Small partitions + high cardinality columns → **large metadata**, increasing Namenode load.

3. **Metadata Mismatch**

   * If partitions are created in HDFS but **not added to Hive metadata**, queries **return no results**.
   * Always run `ALTER TABLE table_name ADD PARTITION ...` or use `MSCK REPAIR TABLE table_name`.

4. **Complexity in Multi-Level Partitions**

   * Example: `country/state/city/date`
   * Multi-level partitions require careful management to avoid errors and ensure all partitions are discoverable.

---

### **Best Practices**

* Keep partition size **reasonably large** (e.g., 256 MB – 1 GB).
* Avoid using **high cardinality columns** for partitioning.
* Always **sync HDFS partitions with Hive metadata**.
* Use **multi-level partitions** only when necessary and carefully manage them.

---

## Q19. What Happens to Partitions When Dropping Hive Tables?

| Table Type | Drop Table Behavior     | Partitions Dropped? | Data Dropped? |
| ---------- | ----------------------- | ------------------- | ------------- |
| Managed    | Metadata + data removed | Yes                 | Yes           |
| External   | Only metadata removed   | No                  | No            |

---

