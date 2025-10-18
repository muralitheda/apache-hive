

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


## Q20. How to Manage Hive Partitions Automatically

### 1. **Automate Partition Discovery**

* Hive does **not automatically detect partitions** added directly to HDFS.
* Use:

  ```sql
  MSCK REPAIR TABLE table_name;
  ```

  * Scans the HDFS location of the table.
  * Adds any **new partitions** to Hive metadata.
* Can be **scheduled periodically** (e.g., via Oozie, Airflow, or cron jobs) to keep metadata in sync.

### 2. **Manage Partition Retention**

* Delete **old partitions** automatically to save storage and improve query performance.
* Example: Retain only the last 30 days of data:

  ```sql
  ALTER TABLE table_name DROP IF EXISTS PARTITION (dt<'2025-09-07');
  ```
* Can be automated with scripts or workflow schedulers.

### 3. **Automatic Partitioning During ETL**

* When loading data with Hive, you can **use dynamic partitioning**:

  ```sql
  SET hive.exec.dynamic.partition = true;
  SET hive.exec.dynamic.partition.mode = nonstrict;

  INSERT INTO TABLE table_name PARTITION (dt)
  SELECT col1, col2, col3, dt
  FROM staging_table;
  ```
* Hive automatically **creates new partitions** based on the `dt` column.

### ✅ Best Practices

1. Enable **dynamic partitioning** for frequent ETL loads.
2. Schedule **MSCK REPAIR TABLE** periodically to sync HDFS partitions with Hive metadata.
3. Automate **partition retention** to prevent small files and save storage.

---

## Q21. How to Recover Dropped Hive Tables with Partitions

### **1. External Table Dropped by Mistake**

* **Scenario:** Only metadata is lost; data in HDFS is intact.
* **Recovery Steps:**

Step 1. Create External Table
    
```sql
USE default;

-- Drop if exists
DROP TABLE IF EXISTS default.ext_sales;

-- Create external table
CREATE EXTERNAL TABLE default.ext_sales (
    order_id INT,
    customer STRING,
    amount DOUBLE
)
PARTITIONED BY (dt STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION '/user/hduser/ext_sales_data';

vi /home/hduser/sales1.csv
101,John,250.50
102,Alice,300.00

vi /home/hduser/sales2.csv
103,Bob,150.75
104,Eve,400.00

hadoop fs -mkdir -p /user/hduser/ext_sales_data/dt=2025-10-01
hadoop fs -mkdir -p /user/hduser/ext_sales_data/dt=2025-10-02

hadoop fs -put /home/hduser/sales1.csv /user/hduser/ext_sales_data/dt=2025-10-01/
hadoop fs -put /home/hduser/sales2.csv /user/hduser/ext_sales_data/dt=2025-10-02/

ALTER TABLE default.ext_sales ADD PARTITION (dt='2025-10-01') LOCATION '/user/hduser/ext_sales_data/dt=2025-10-01/';
ALTER TABLE default.ext_sales ADD PARTITION (dt='2025-10-02') LOCATION '/user/hduser/ext_sales_data/dt=2025-10-02/';

SELECT * FROM default.ext_sales;

```

Step 2: Drop Table by Mistake

```sql
DROP TABLE default.ext_sales;

--Data in /user/hduser/ext_sales_data/ still exists in HDFS.
```

Step 3: Recover External Table
```sql
-- Recreate external table
CREATE EXTERNAL TABLE default.ext_sales (
    order_id INT,
    customer STRING,
    amount DOUBLE
)
PARTITIONED BY (dt STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION '/user/hduser/ext_sales_data';

-- Repair partitions
MSCK REPAIR TABLE default.ext_sales;

-- Query to verify recovery
SELECT * FROM default.ext_sales;
```


### **2. Managed Table Dropped by Mistake**

* **Scenario:** Data is removed along with metadata, but can sometimes be recovered from **HDFS trash**.
* **Recovery Steps:**

Step 1: Create Managed Table

```sql
USE default;

DROP TABLE IF EXISTS default.mng_sales;

CREATE TABLE default.mng_sales (
    order_id INT,
    customer STRING,
    amount DOUBLE
)
PARTITIONED BY (dt STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ',';

LOAD DATA LOCAL INPATH '/home/hduser/sales1.csv' INTO TABLE default.mng_sales PARTITION (dt='2025-10-01');
LOAD DATA LOCAL INPATH '/home/hduser/sales2.csv' INTO TABLE default.mng_sales PARTITION (dt='2025-10-02');

SELECT * FROM default.mng_sales;

```

Step 2: Drop Managed Table by Mistake

```sql
-- Check the current value
hdfs getconf -confKey fs.trash.interval

-- if it is 0 then you have to update it in the core-site.xml
<property>
  <name>fs.trash.interval</name>
  <value>60</value> <!-- trash retention in minutes -->
  <description>HDFS trash retention interval for testing</description>
</property>


DROP TABLE default.mng_sales;
--Data goes to HDFS trash (if trash enabled).
```

Step 3: Recover Managed Table

* Check trash location, e.g., hadoop fs -ls -R /user/hive/.Trash/Current/default/mng_sales/
* Recreate table:

```sql
CREATE TABLE default.mng_sales (
    order_id INT,
    customer STRING,
    amount DOUBLE
)
PARTITIONED BY (dt STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ',';

LOAD DATA INPATH '/user/hive/.Trash/Current/default/mng_sales/dt=2025-10-01/sales1.csv' 
INTO TABLE default.mng_sales PARTITION (dt='2025-10-01');

LOAD DATA INPATH '/user/hive/.Trash/Current/default/mng_sales/dt=2025-10-02/sales2.csv' 
INTO TABLE default.mng_sales PARTITION (dt='2025-10-02');

SELECT * FROM default.mng_sales;

```

### ✅ Key Points

* **External table:** Only metadata lost → recover with `CREATE EXTERNAL TABLE + MSCK REPAIR`.
* **Managed table:** Data + metadata lost → recover from HDFS trash, then recreate table and reload.
* **Best Practice:** Schedule **backups of metadata and important managed tables** to avoid permanent loss.

---

## Q22. Discovering Partitions Created by External Systems

**Scenario:**
External tools like Spark or Sqoop may write data directly into Hive’s HDFS partition folders. Hive **does not automatically update its metastore**, so queries will not return results for these new partitions unless you explicitly update the metastore.

**Use Case Example:**

* Sqoop or Spark writes data to HDFS:

  ```
  /user/hive/warehouse/retail.db/custinfo/date=2018-02-20/
  ```
* Hive table `retail.custinfo` **does not see this partition** unless the metastore is updated.

---

### **Why we need to run MSCK REPAIR TABLE after each ingestion**

* Hive stores a list of partitions for each table in its **metastore**.
* If new partitions are **directly created in HDFS**, the metastore is unaware.
* Running either of the methods below **adds the new partitions to Hive metastore**, making them visible to queries.

---

### **1. Add Each Partition Manually**

```sql
ALTER TABLE retail.custinfo
ADD PARTITION (`date`='2018-02-20') 
LOCATION '/user/hive/warehouse/retail.db/custinfo/date=2018-02-20/';
```

* Useful for **small number of partitions** or one-off updates.

---

### **2. Use MSCK REPAIR TABLE**

```sql
MSCK REPAIR TABLE retail.custinfo;
```

* Scans the HDFS location for **any partitions not present in the metastore**.
* Automatically adds metadata for **all missing partitions**.
* **Recommended** for multiple new partitions added externally.

---

### **3. Loading via Hive Statements (No Repair Needed)**

* If you load data using Hive directly, e.g.:

```sql
INSERT INTO TABLE retail.custinfo PARTITION (`date`)
SELECT * FROM staging_custinfo;
```

* Hive automatically updates the **metadata of the final table**.
* No need to run `ALTER TABLE` or `MSCK REPAIR TABLE`.

---

### **4. Automated Partition Discovery**

* Hive can **periodically discover partitions** from HDFS.
* Example to enable automated discovery every 10 minutes:

```sql
-- Set discovery frequency in seconds
SET metastore.partition.management.task.frequency=600;

-- Enable partition discovery on table
ALTER TABLE retail.exttbl SET TBLPROPERTIES ('discover.partitions' = 'true');
```

* Ideal for **frequent external loads** without manual intervention.

---

### **5. Option C: External Tools Registering Partitions Programmatically**

* Some tools (Spark, Sqoop, NiFi, or custom scripts) can **directly register partitions in Hive metastore** without relying on MSCK REPAIR TABLE.
* Example in **Spark**:

```python
df.write.mode("overwrite") \
    .partitionBy("date") \
    .format("hive") \
    .saveAsTable("retail.custinfo")
```

* Spark automatically:

  1. Creates HDFS directories for each `date` value.
  2. Updates Hive metastore with **new partitions**.
* No need to run `ALTER TABLE` or `MSCK REPAIR TABLE`.

---

### **Summary / Best Practices**

| Method                                   | When to Use                 | Notes                                         |
| ---------------------------------------- | --------------------------- | --------------------------------------------- |
| ALTER TABLE ADD PARTITION                | Few partitions              | Manual, time-consuming for large datasets     |
| MSCK REPAIR TABLE                        | Multiple new partitions     | Scans HDFS, can be slow for very large tables |
| Hive INSERT (dynamic/static)             | Loading from staging tables | Metadata updated automatically                |
| Automated Discovery                      | Frequent external loads     | Requires metastore task frequency setup       |
| External Tools Programmatic Registration | Spark, Sqoop, NiFi loads    | Metadata updated automatically at write time  |

---

✅ **Key Takeaways:**

1. Hive metastore must be aware of HDFS partitions for queries to return correct results.
2. Direct HDFS writes **do not update metastore**.
3. Choose **manual, repair, dynamic insert, automated discovery, or programmatic registration** depending on load frequency and table size.

---

## Q23. Configure Hive Table Partition Retention (Automatic Purging / Archiving)

**Scenario:**
You want to **archive and purge old data** from Hive tables, ensuring:

* Only recent partitions (e.g., last 1 year) are retained.
* Hive metastore metadata and/or actual data in HDFS is cleaned up.
* Works for **managed and external tables**, **partitioned and non-partitioned**.

---

### **Step 1: Create a Partitioned Table**

```sql
USE default;

DROP TABLE IF EXISTS default.customer;

CREATE TABLE default.customer (
    id INT,
    name STRING,
    amount DOUBLE
)
PARTITIONED BY (dt STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS ORC;
```

---

### **Step 2: Insert Sample Data**

```sql
INSERT INTO default.customer PARTITION (dt='2024-10-05') VALUES (1,'Alice',100.0);
INSERT INTO default.customer PARTITION (dt='2023-09-01') VALUES (2,'Bob',200.0);
INSERT INTO default.customer PARTITION (dt='2022-08-15') VALUES (3,'Charlie',300.0);
```

---

### **Step 3: Using `partition.retention.period` (Metadata Only)**

```sql
ALTER TABLE default.customer 
SET TBLPROPERTIES ('partition.retention.period'='365d');
```

> **Note:** This does **not physically delete partitions or data** in standard Hive. It may only mark old partitions in metadata, depending on the Hive distribution.

---

### **Step 4: Purge Old Partitions Programmatically**

#### **Option A: Hive Script / SQL**

```sql
-- Drop partitions older than 1 year
ALTER TABLE default.customer 
DROP IF EXISTS PARTITION (dt<'2023-10-07');  -- Use dynamic date in scripts
```

#### **Option B: Archive Recent Data Only**

* For **partitioned or non-partitioned tables**, create a new table with recent data:

```sql
-- Keep only last 1 year data
INSERT OVERWRITE TABLE default.customer
SELECT * 
FROM default.customer
WHERE dt >= '2023-10-07';
```

* This also acts as an **archive + purge mechanism**.

#### **Option C: MapReduce / Hive / Spark with Controlled Output Files**

* Set **reducers / repartition** to control number of output files and optimize performance:

**Hive Example:**

```sql
SET mapreduce.job.reduces=4;

INSERT OVERWRITE TABLE default.customer
SELECT * 
FROM default.customer
WHERE dt >= '2023-10-07';
```

**Spark Example:**

```python
from datetime import datetime, timedelta
from pyspark.sql import SparkSession

spark = SparkSession.builder.enableHiveSupport().getOrCreate()

cutoff = (datetime.now() - timedelta(days=365)).strftime('%Y-%m-%d')

df = spark.table("default.customer").filter(f"dt >= '{cutoff}'")
df.repartition(4).write.mode("overwrite").saveAsTable("default.customer")
```

* Ensures **old data is purged** and **recent data is archived efficiently** with controlled file output.

---

### **Step 5: Automation**

* Schedule **daily or weekly jobs** (Hive script, Spark, or MR) to automatically archive/purge old partitions.
* For external tables, only **metadata deletion** occurs; data must be handled separately or backed up.

---

### **Key Points / Best Practices**

1. `partition.retention.period` alone **does not physically purge** data.
2. Use **ALTER TABLE DROP PARTITION** or **INSERT OVERWRITE** to remove old data.
3. For large tables, consider **MapReduce/Spark jobs with controlled reducers/repartition** to optimize HDFS output files.
4. Works for **managed/external tables** and **partitioned/non-partitioned tables**.
5. Always **backup old data** if needed for compliance or audit purposes.

---

## Q24. Small Files Issue in Hive / HDFS? (Small Files Merge | Handling | Compaction | Archiving | Purging)


### 1. Problem Context

Small files are one of the biggest scalability problems in Hadoop & Hive.
Each file, directory, and block in HDFS is represented as an **object in the NameNode’s memory**, consuming roughly **150 bytes per entry**.

**Example:**
10 million files → ~3 GB of NameNode heap memory.
Result: slower NameNode performance, high GC overhead, and degraded Hive query execution.


### 2. Step 1 – Clean Zero-Byte Files

**Why:**
MapReduce/Spark jobs often create `_SUCCESS`, `_FAILURE`, or other **0-byte files**.
Though they use **no HDFS blocks**, they **consume NameNode metadata memory**.

**Command:**

```bash
hdfs dfs -ls -R /path/to/hdfs/dir | awk '$1 !~ /^d/ && $5 == "0" { print $8 }' | xargs -n100 hdfs dfs -rm
```

**Notes:**

* Moves files to `.Trash`, then auto-cleared by `fs.trash.interval`.
* Ideal for **daily cleanup** or **cron jobs** in staging environments.


### 3. Step 2 – Use Metadata Governance Tools

**Why:**
To identify which pipelines (NiFi/Spark streaming) cause excessive small file creation.

**Tools:**

* Cloudera Navigator
* Apache Atlas
* AWS Glue Data Catalog

**Purpose:**
Audit, lineage, and metadata tracking. Preventive step for optimizing ingestion design.


### 4. Step 3 – Reduce Partition Explosion

**Problem:**
Too many partitions = too many directories + small files.

**Solution:**
Partition **wisely** (daily instead of hourly). Use **bucketing** for finer granularity.

**Hive Config:**

```sql
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
SET hive.exec.max.dynamic.partitions=1000;
SET hive.exec.max.dynamic.partitions.pernode=100;
```

**Tip:**

* Avoid over-partitioning by time or key fields.
* Roll up small partitions into weekly/monthly aggregates.


### 5. Step 4 – Archive Older Data (HAR Files)

**Purpose:**
Archive infrequently accessed partitions into fewer, larger files using **Hadoop Archives (HAR)**.

**Enable Archive:**

```sql
SET hive.archive.enabled=true;
SET hive.archive.har.parentdir.settable=true;
SET har.partfile.size=1099511627776; -- 1 TB
```

**Archive Example:**

```sql
ALTER TABLE customer ARCHIVE PARTITION (year=2023, month=12);
```

**Unarchive Example:**

```sql
ALTER TABLE customer UNARCHIVE PARTITION (year=2023, month=12);
```

**Retention Example:**

```sql
ALTER TABLE customer SET TBLPROPERTIES ('partition.retention.period'='365d');
```

**Notes:**

* Archived partitions are read-only and slower to access.
* Useful for **cold data** or compliance retention.
* Combine with **purging** to delete data older than 1 year.


### 6. Step 5 – Merge Small Files (Hive or Spark)

**Goal:**
Reduce small file count for **active Hive tables**.

#### Option A: Hive Merge Settings

```sql
SET hive.merge.mapfiles=true;
SET hive.merge.mapredfiles=true;
SET hive.merge.smallfiles.avgsize=104857600;     -- 100 MB
SET hive.merge.size.per.task=209715200;          -- 200 MB
SET mapred.max.split.size=68157440;              -- ~65 MB
SET mapred.min.split.size=68157440;
SET hive.exec.mappers.bytes.per.mapper=268435456;  -- 256 MB
SET hive.exec.reducers.bytes.per.reducer=268435456;
```

**Merge Query (Off-Peak Hours):**

```sql
INSERT OVERWRITE TABLE table1
SELECT * FROM table1;
```

This will rewrite the data into fewer, large files.


#### Option B: Spark-Based Merge

```python
df = spark.read.orc("hdfs:///user/hive/warehouse/table1/")
df.repartition(4).write.mode("overwrite").orc("hdfs:///user/hive/warehouse/table1/")
```

Or:

```python
df.coalesce(4).write.mode("overwrite").parquet("hdfs:///user/hive/warehouse/table1/")
```

**Tip:**

* Use `repartition()` for shuffle and even file distribution.
* `coalesce()` for fewer files without shuffle.
* Run during off-peak or maintenance windows.



### 7. Step 6 – Apply Hive Compaction (ACID Tables)

**Applies To:**
Hive transactional tables (e.g., ORC with `transactional=true`).

**Enable Compactor:**

```sql
SET hive.compactor.initiator.on=true;
SET hive.compactor.worker.threads=2;
```

**Manual Compaction:**

```sql
ALTER TABLE sales_data COMPACT 'minor';
ALTER TABLE sales_data COMPACT 'major';
```

**Notes:**

* **Minor compaction:** merges delta files only.
* **Major compaction:** merges base + delta = full consolidation.
* Schedule with cron or Metastore triggers.



### 8. Step 7 – Purge or Archive Mechanism

For **retention management and cleanup**:

**Use Case:**
You want to keep **only the last 1 year** of partition data.

```sql
ALTER TABLE customer SET TBLPROPERTIES ('partition.retention.period'='365d');
```

Then purge older partitions manually or via scheduled job:

```sql
INSERT OVERWRITE TABLE customer
SELECT * FROM customer
WHERE order_date >= date_sub(current_date, 365);
```

**Optionally:**
Archive old data to HAR before deleting.



### 9. Step 8 – Reduce Files During Ingestion

When streaming data using **NiFi or Spark Structured Streaming**,
you can control file count before data lands in Hive:

**Spark Example:**

```python
df.write.mode("append").option("maxRecordsPerFile", 500000).saveAsTable("table1")
```

**NiFi Example:**
Batch flowfiles before pushing to HDFS.


### Summary Table

| Step | Strategy             | Purpose                       | Tools / Commands               |
| ---- | -------------------- | ----------------------------- | ------------------------------ |
| 1    | Clean 0-byte files   | Reduce metadata overhead      | HDFS script                    |
| 2    | Use governance tools | Identify small-file producers | Navigator / Atlas              |
| 3    | Reduce partitions    | Avoid partition explosion     | Hive configs                   |
| 4    | Archive old data     | Minimize cold storage files   | HAR / Hive archive             |
| 5    | Merge small files    | Optimize active tables        | Hive merge / Spark repartition |
| 6    | Apply compaction     | Consolidate delta files       | Hive ACID compactor            |
| 7    | Purge old data       | Retention enforcement         | Retention & purge SQL          |
| 8    | Control ingestion    | Prevent small files creation  | NiFi / Spark settings          |


### Key Takeaways

* Prevent small files **at ingestion time**.
* Periodically **merge and compact** active data.
* **Archive or purge** inactive data to free space and memory.
* Monitor NameNode heap utilization and file count regularly.
* Combine Hive + Spark + governance tools for full control.

---

## Q25. If we change the partition location of a Hive table using the `ALTER TABLE` command with the **NEW LOCATION** option, then the data for that partition in the table:

**A.** `also moves automatically to the new location`  
**B.** has to be dropped and recreated  
**C.** has to be backed up into a second table and restored   
**D.** has to be moved manually into the new location  

---

## Q26. How can you add a new partition for the month **April** in the above partitioned table?

````markdown

To add a new partition in the table `partitioned_transaction`, use the following command:

```sql
ALTER TABLE partitioned_transaction 
ADD PARTITION (month='April') 
LOCATION '/partitioned_transaction/month=April';
````

Alternatively, we can use **`MSCK REPAIR TABLE`** or **partition discovery**, but these methods are **not advisable for adding a single partition**.

---

## Q27. What is the default maximum dynamic partition that can be created by a mapper/reducer per node or totally? How can you change it?

By default, the **maximum number of dynamic partitions** that can be created by a **mapper or reducer per node** is **100**, and the **total number** of dynamic partitions that can be created is **1000**.

We can change these limits using the following commands:

```sql
SET hive.exec.max.dynamic.partitions.pernode = <value>;
```

To set the total number of dynamic partitions that can be created by one statement:

```sql
SET hive.exec.max.dynamic.partitions = <value>;
```

---


## Q28. Daily Incremental Load with Overlapping Weekly Data in Hive

You receive **daily data** from a **source system**, but **each file contains one week of data**.

For example:  
- On **2 Jan 2022**, the source file contains data from **25 Dec 2021 → 01 Jan 2022**.  
- On **3 Jan 2022**, the next file contains data from **26 Dec 2021 → 02 Jan 2022**, which overlaps with the previous load.

Your **Hive target table** already contains **3 years of data**, including these dates.  
The source data may include **updates**, **deletes**, or **new inserts** within this one-week window.

### ❓ **How do you handle this situation?**
- What type of Hive table should be created?  
- How should the data be loaded each day?

### ✅ **Answer**

#### **1. Table Design**
Create a **partitioned external Hive table** based on the transaction date (or load date).  
Use **ORC/Parquet** format for efficient updates and enable **ACID (transactional)** features if supported.

```sql
CREATE EXTERNAL TABLE customer_data (
  customer_id     STRING,
  name            STRING,
  amount          DOUBLE,
  ...
)
PARTITIONED BY (txn_date DATE)
STORED AS ORC
TBLPROPERTIES (
  'transactional'='true',
  'partition.retention.period'='1095d'  -- 3 years retention
);
````

#### **2. Loading Logic**

Since each day’s file overlaps with the previous week’s data, you should **overwrite only the affected partitions** (recent 7 days) instead of reloading the entire dataset.

##### **Step 1 – Enable dynamic partitioning**

```sql
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
```

##### **Step 2 – Overwrite the last 7 days**

When today = **03 Jan 2022**, load data for **26 Dec 2021 → 02 Jan 2022**:

```sql
INSERT OVERWRITE TABLE customer_data PARTITION (txn_date)
SELECT * 
FROM staging_customer
WHERE txn_date BETWEEN '2021-12-26' AND '2022-01-02';
```

👉 This overwrites only the last week’s partitions (updates/deletes handled by reloading them).

#### **3. Retention Management**

Automatically purge partitions older than 3 years using table property:

```sql
ALTER TABLE customer_data 
SET TBLPROPERTIES ('partition.retention.period'='1095d');
```

#### **4. Alternative (if ACID Merge is supported)**

If your Hive version supports ACID and MERGE statements, you can perform **record-level upserts** instead of partition-level overwrites:

```sql
MERGE INTO customer_data AS tgt
USING staging_customer AS src
ON tgt.customer_id = src.customer_id AND tgt.txn_date = src.txn_date
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```
✅ This ensures only changed records are updated.

### 🧠 **What Happens During Each Load**

| Date of Load | Data Range                | Action                                                    |
| ------------ | ------------------------- | --------------------------------------------------------- |
| 02 Jan 2022  | 25 Dec 2021 → 01 Jan 2022 | Initial load                                              |
| 03 Jan 2022  | 26 Dec 2021 → 02 Jan 2022 | Overwrites partitions for 26 Dec → 01 Jan; inserts 02 Jan |

Older data (before 26 Dec 2021) remains **untouched**.
Partitions older than 3 years are automatically **purged**.

### 🧭 **Data Flow Example**

```
Source System (daily file)
   ↓
GCS (staging zone)
   ↓
Hive External Table (partitioned by txn_date)
   ↓
BigQuery (ingestion → curated layers)
```

### 🏁 **Summary**

| Aspect            | Recommendation                                                       |
| ----------------- | -------------------------------------------------------------------- |
| Table Type        | External, partitioned by date                                        |
| Storage Format    | ORC / Parquet                                                        |
| Write Mode        | INSERT OVERWRITE (for last 7 days) or MERGE (if ACID)                |
| Dynamic Partition | Enabled                                                              |
| Retention         | 3 years (`partition.retention.period`)                               |
| Benefit           | Handles overlapping weekly data efficiently with incremental updates |

---

## Q29. When a partition is archived in Hive, it?

**Options:**  
A. Reduces space through compression  
B. Reduces the block size  
C. Reduces the length of records  
D. Reduces the number of files stored   

**Answer:** **D — Reduces the number of files stored**

**Notes:**

* Archived partitions are stored as **Hadoop Archives (HAR)**.
* You can **query them normally**.
* To **modify data**, you must **unarchive** first:

```markdown
ALTER TABLE table_name ARCHIVE PARTITION (partition_column='value');
ALTER TABLE table_name UNARCHIVE PARTITION (partition_column='value');
SELECT * FROM table_name WHERE partition_column='value';
```

---


## Q30. How do you purge data in a Hive partition table after N days?

You can configure **automatic partition retention** using the table property:

```markdown
ALTER TABLE customer 
SET TBLPROPERTIES ('partition.retention.period'='7d');
```

> This will automatically purge partitions older than **7 days**.

---

## Q31. If the directory for a partition does not exist (no data), but the metastore entry exists, what happens?

Example:

```sql
SELECT * FROM tablename WHERE datadt='2020-08-02';
```

HDFS location:

```
/hdfslocation/tablename/datadt=2020-08-01/file.txt (exists)
```

**Options:**
A. Error is thrown
B. MapReduce job is not triggered ✅
C. Result from a random partition is returned
D. No result is returned

**Answer:** **B — MapReduce job is not triggered**

> Explanation: Hive checks the metastore for partitions. If the partition exists but contains no data, the query **returns empty results** without triggering a MapReduce job.

---

## Q32. If data exists in HDFS but partition metadata is not added in the metastore, what happens?

Example:

```sql
SELECT * FROM tablename WHERE datadt='2020-08-01';
```

HDFS location:

```
/hdfslocation/tablename/datadt=2020-08-01/file.txt (exists)
```

**Options:**
A. Result will be returned or error thrown
B. MapReduce job is not triggered
C. Result from a random partition is returned
D. No result is returned ✅

**Answer:** **D — No result is returned**

> Explanation: Hive relies on the **metastore** to know which partitions exist.
> Even if the files are present in HDFS, **querying without partition metadata returns no results**.
> Use `MSCK REPAIR TABLE tablename;` or `ALTER TABLE ... ADD PARTITION` to register existing HDFS data with Hive.

---

## Q33. How to permanently drop data in a Hive table? Why use PURGE?

To permanently delete a Hive table and its data:

```markdown
DROP TABLE [IF EXISTS] table_name [PURGE];
```

### **Explanation**

* **Without PURGE**:

  * Table data moves to the **Trash directory**.
  * Data **can be recovered** if dropped accidentally.

* **With PURGE**:

  * Table data **bypasses Trash** and is **permanently deleted**.
  * Use **PURGE** when you are **sure you no longer need the data**, or when the table size is too large to retain in Trash.

> ⚠️ **Caution:** Once PURGE is used, data **cannot be recovered**.

---

## Q34. Hive Table Rename and Location

### **1. Renaming a Managed Hive Table**

* **Does the location change automatically?** ✅ **Yes**

  * Only if **no explicit location** was set (default location used).

```markdown
ALTER TABLE old_table_name RENAME TO new_table_name;
```

### **2. Renaming an External Hive Table**

* **Does the location change automatically?** ❌ **No**

  * External tables **retain their original location**, because Hive does not own the data.

```markdown
ALTER TABLE old_external_table_name RENAME TO new_external_table_name;
```

### **3. Changing a Hive Table Location**

```markdown
ALTER TABLE tblname SET LOCATION "hdfs:/dir1/dir2/dir3";
```

* This works for **both managed and external tables**.
* Use it to **move data** to a new HDFS directory or to point an external table to a different location.

---

## Q35. How to convert an array column into rows in Hive? What is the use of `explode`?

### **Scenario**

* Assume a table has an **array column** `x`:

```
x -> Array[a, b, c, d, e]
```

### **Use of `explode`**

* Converts **array or complex data types** into **separate rows**.
* Useful for **flattening nested data** for easier querying and analysis.

```markdown
SELECT explode(x) AS y 
FROM arrays;
```

**Result:**

```
a
b
c
d
e
```

> `explode` is commonly used with **arrays, maps, or structs** to transform them into **tabular row format**.

---

## Q36. Consider the query:

```sql
SELECT explode(city_locality) 
FROM ALL_LOCALITIES;
```

where `city_locality` is an **ARRAY** datatype. What does it return?

**Options:**
A. All the array elements as one row for each input array
B. Zero or more rows, for each element for each input array
C. Each of the array elements as one column for each input array
D. Zero or more columns for each element for each input array

✅ **Answer:** B — *Zero or more rows, for each element for each input array*

---


## Q37. How to generate surrogate keys / sequence numbers / primary keys / unique values in Hive or Spark Hive?

**1. Using ROW_NUMBER()**

```sql
SELECT ROW_NUMBER() OVER (ORDER BY name) AS surrogate_key, name, amount 
FROM customer;
```

Generates sequential numbers for each row.
✅ Works in Hive (v2.1+) and Spark SQL.

**2. Using UUID()**

```sql
SELECT UUID() AS unique_id, name, amount 
FROM customer;
```

Generates globally unique identifiers.
✅ Works in Hive 3.0+ and Spark SQL.


**3. Using monotonically_increasing_id() (Spark)**

```python
from pyspark.sql.session import SparkSession
from pyspark.sql.functions import monotonically_increasing_id

spark = SparkSession.builder.getOrCreate()
data = [("John", "Delhi"), ("Mary", "Mumbai"), ("Alex", "Chennai")]
df = spark.createDataFrame(data, ["name", "city"])
df.withColumn("surrogate_key", monotonically_increasing_id()).show()
```

Generates unique values per row (not sequential).
✅ Spark only.


**4. Using Column Concatenation**

```sql
SELECT CONCAT(id, '_', name) AS unique_key, name, amount 
FROM customer;
```

Combines columns to form unique composite keys.
✅ Works in Hive and Spark.


**5. Using UDF (Custom Function)**

```python
from pyspark.sql.session import SparkSession
from pyspark.sql.functions import udf
import uuid
spark = SparkSession.builder.getOrCreate()

data = [("John", "Delhi"), ("Mary", "Mumbai"), ("Alex", "Chennai")]
df = spark.createDataFrame(data, ["name", "city"])

uuid_udf = udf(lambda: str(uuid.uuid4()))
df.withColumn("unique_id", uuid_udf()).show()
```

Creates custom unique IDs (UUID or timestamp-based).
✅ Works in Hive (UDF) and Spark (UDF).


**Summary:**

* `ROW_NUMBER()` → Sequential surrogate key
* `UUID()` → Globally unique value
* `monotonically_increasing_id()` → Unique, non-sequential ID in Spark
* `CONCAT()` → Business key based on columns
* `UDF()` → Custom unique generation logic

---


## Q38. Hive Connectivity & Query Comparison Cheat Sheet

### 1️⃣ Hive Connectivity Methods

- **Hive CLI** (Thrift protocol)
- **Beeline CLI** (connects to HiveServer2, username/password optional if auth=NONE)
- **Hue** (Web UI)
- **Ambari Hive View** (Web UI)
- **SQL tools**: SQuirreL, Toad, DBeaver, DBVisualizer (JDBC)
- **Notebooks**: Jupyter, Zeppelin
- **Spark SQL**: spark.sql() queries



### 2️⃣ Connecting via Beeline

1. Start HiveServer2:
```bash
hiveserver2 &
````

2. Launch Beeline:

```bash
beeline
```

3. Connect:

```sql
!connect jdbc:hive2://localhost:10000/default
# Press Enter for username and password if auth=NONE

--hive
SET hive.server2.authentication;
# common values: KERBEROS, LDAP, NOSASL (or NONE / NONE-like), CUSTOM.
```

### 3️⃣ Spark Connection to Hive

* Using Hive Metastore:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("SparkHiveExample") \
    .enableHiveSupport() \
    .getOrCreate()

df = spark.sql("SELECT * FROM employees")
df.show()
```

* Using JDBC (HiveServer2 auth required if LDAP/Kerberos):

```python
df = spark.read.format("jdbc") \
    .option("url", "jdbc:hive2://localhost:10000/default") \
    .option("dbtable", "employees") \
    .load()
```

### 4️⃣ MINUS / EXCEPT Alternatives in Hive 2.x

* Hive **does not support MINUS or EXCEPT**.
* Use:

**LEFT ANTI JOIN**

```sql
SELECT a.*
FROM table_a a
LEFT ANTI JOIN table_b b
ON a.id = b.id;
```

**NOT IN**

```sql
SELECT *
FROM table_a
WHERE id NOT IN (SELECT id FROM table_b);
```

**NOT EXISTS**

```sql
SELECT *
FROM table_a a
WHERE NOT EXISTS (SELECT 1 FROM table_b b WHERE a.id = b.id);
```


### 5️⃣ Intersection / Common Rows

```sql
SELECT a.*
FROM table_a a
JOIN table_b b
ON a.id = b.id;
```

### 6️⃣ Union / Merge of Tables

```sql
SELECT id, name FROM table_a
UNION        -- removes duplicates
SELECT id, name FROM table_b;
```

* Use `UNION ALL` to include duplicates.

### 7️⃣ Full Comparison (Excess + Common)

```sql
SELECT COALESCE(a.id, b.id) AS id,
       a.name AS name_a,
       b.name AS name_b
FROM table_a a
FULL OUTER JOIN table_b b
ON a.id = b.id;
```

* `name_a IS NULL` → exists only in B
* `name_b IS NULL` → exists only in A
* Both not null → common row


### 8️⃣ Subqueries in Hive 2.x

* **SELECT clause subqueries are NOT supported**:

```sql
SELECT customer_num,
       (SELECT MAX(ship_charge) FROM orders) AS max_ship_chg
FROM customer;  -- ❌ Fails in Hive 2.x
```

* **Workaround using FROM / JOIN**:

```sql
SELECT c.customer_number,
       t.ship_chg
FROM customer c
CROSS JOIN (
    SELECT MAX(ship_charge) AS ship_chg
    FROM orders
) t;
```

* Supported in **WHERE** and **FROM** clauses only.
* Cross join creates a cartesian product; costly for large datasets.

### ✅ Summary

| Operation             | Hive 2.x Support | Hive Alternative                     |
| --------------------- | ---------------- | ------------------------------------ |
| MINUS / EXCEPT        | ❌                | LEFT ANTI JOIN / NOT EXISTS / NOT IN |
| INTERSECT             | ❌                | INNER JOIN                           |
| UNION                 | ✅                | UNION / UNION ALL                    |
| SELECT subquery       | ❌                | Use JOIN or CROSS JOIN               |
| WHERE / FROM subquery | ✅                | Supported                            |
| Full comparison       | ✅                | FULL OUTER JOIN + COALESCE           |

---


## Q39. Hive ACID / Transactional Tables

Hive supports **full ACID (Atomicity, Consistency, Isolation, Durability)** operations starting from Hive 0.14, enabling **INSERT, UPDATE, and DELETE** operations on transactional tables.

### Requirements:

1. **Transactional table**: Must be created with `TBLPROPERTIES ('transactional'='true')`.
2. **ORC file format**: ACID operations require ORC storage.
3. **Bucketed tables**: Table must be **bucketed** (`CLUSTERED BY`) for UPDATE/DELETE.
4. **Hive configurations**:

```sql
SET hive.support.concurrency=true;
SET hive.enforce.bucketing=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
SET hive.txn.manager=org.apache.hadoop.hive.ql.lockmgr.DbTxnManager;
SET hive.compactor.initiator.on=true;
SET hive.compactor.worker.threads=1;
```

### Example of a transactional table:

```sql
CREATE TABLE employees_orc (
    emp_id INT,
    name STRING,
    dept STRING,
    salary DOUBLE
)
CLUSTERED BY (emp_id) INTO 4 BUCKETS
STORED AS ORC
TBLPROPERTIES ('transactional'='true');
```

### DML Operations:

* **INSERT**

```sql
INSERT INTO employees_orc VALUES (1, 'John', 'IT', 50000);
```

* **UPDATE**

```sql
UPDATE employees_orc
SET salary = 60000
WHERE emp_id = 1;
```

* **DELETE**

```sql
DELETE FROM employees_orc
WHERE emp_id = 1;
```

### Compaction

* Hive writes updates/deletes as **delta files**, which can create small files.
* Compaction merges them:

  * **Minor compaction**: merges delta files within the same base.
  * **Major compaction**: merges base + delta files into a new base.

Schedule compaction via **cron**, Ambari, or Qubole.

### 1️⃣ Insert Overwrite on Partitioned Tables

Hive supports **INSERT OVERWRITE** to update partitioned tables efficiently:

```sql
INSERT OVERWRITE TABLE sales PARTITION (year=2025, month=10)
SELECT * FROM sales_staging
WHERE year = 2025 AND month = 10;
```

* Replaces the partition data with new results.
* Works with **dynamic partitions**:

```sql
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT OVERWRITE TABLE sales PARTITION (year, month)
SELECT id, amount, year, month FROM sales_staging;
```


### 2️⃣ Using Storage Handlers (External Tables / NoSQL Integration)

Hive can interact with external storage systems using **storage handlers**. This allows DML operations on tables pointing to **NoSQL or search engines** like HBase, Phoenix, or Elasticsearch.

#### Example: Hive → HBase

```sql
CREATE TABLE hbase_employees (
  emp_id INT,
  name STRING,
  dept STRING
)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES (
  "hbase.columns.mapping" = ":key,cf:name,cf:dept"
);
```

* **Insert / Insert Select**

```sql
INSERT INTO hbase_employees VALUES (1, 'John', 'IT');

INSERT INTO hbase_employees
SELECT emp_id, name, dept FROM employees_orc;
```

* DML is supported if the storage handler implements it.
* Hive executes **insert-select queries** efficiently to push data into NoSQL tables.
* Supports **upserts** for some storage handlers (HBase, Phoenix).


### 3️⃣ Example: `sales` Table in Hive

#### a) Regular Partitioned Table

```sql
CREATE TABLE sales (
    sale_id BIGINT,
    product_id INT,
    customer_id INT,
    sale_date DATE,
    amount DOUBLE,
    quantity INT
)
PARTITIONED BY (year INT, month INT)
STORED AS ORC
TBLPROPERTIES (
    "orc.compress"="SNAPPY"
);
```

#### b) Transactional / ACID Table

```sql
CREATE TABLE sales_acid (
    sale_id BIGINT,
    product_id INT,
    customer_id INT,
    sale_date DATE,
    amount DOUBLE,
    quantity INT
)
PARTITIONED BY (year INT, month INT)
CLUSTERED BY (sale_id) INTO 8 BUCKETS
STORED AS ORC
TBLPROPERTIES (
    'transactional'='true',
    'transactional_properties'='default',
    'orc.compress'='SNAPPY'
);
```

### 4️⃣ Example DML on `sales` Table

**Insert into partitioned table**

```sql
INSERT INTO sales PARTITION (year=2025, month=10)
VALUES (1001, 101, 201, '2025-10-15', 500.0, 2);
```

**Insert overwrite a partition**

```sql
INSERT OVERWRITE TABLE sales PARTITION (year=2025, month=10)
SELECT sale_id, product_id, customer_id, sale_date, amount, quantity
FROM sales_staging
WHERE year = 2025 AND month = 10;
```

**Update ACID table**

```sql
UPDATE sales_acid
SET amount = 800.0
WHERE sale_id = 1002 AND year = 2025 AND month = 10;
```

**Delete from ACID table**

```sql
DELETE FROM sales_acid
WHERE sale_id = 1001 AND year = 2025 AND month = 10;
```

### ✅ Summary

| Feature                            | Hive Support / Notes                                          |
| ---------------------------------- | ------------------------------------------------------------- |
| Transactional / ACID tables        | ✅ Requires ORC, bucketing, transactional=true                 |
| DML (INSERT/UPDATE/DELETE)         | ✅ On ACID tables                                              |
| Insert overwrite partition         | ✅ Efficient for ETL updates                                   |
| External tables / Storage handlers | ✅ Insert-Select supported for HBase, Phoenix, ES              |
| Compaction                         | ✅ Minor / Major to merge delta files                          |
| Upserts via Hive                   | ✅ Storage-handler dependent                                   |
| Dynamic partitions                 | ✅ Supported with `hive.exec.dynamic.partition.mode=nonstrict` |

---


## Q40. Data Load Scenario:

* **Source Data:** Daily extract contains **1 week of data**.

  * Example: On 02-Jan-2022, the source file contains data **from 25-Dec-2021 to 01-Jan-2022**.
  * On 03-Jan-2022, the source file contains data **from 26-Dec-2021 to 02-Jan-2022**.
  * Source may have **inserts, updates, and deletes**.
* **Target Hive Table:** Contains **historical data** (3 years) and should be **updated** with the latest source data.

### 1️⃣ Hive Table Design

* Use a **partitioned table** based on the date column (`data_dt`) for efficient loading and management.
* Optionally, make it **ACID transactional** if updates/deletes need to be applied **without overwriting all partitions**.

#### Example Hive Table (Partitioned)

```sql
CREATE TABLE sales_history (
    sale_id BIGINT,
    product_id INT,
    customer_id INT,
    sale_date DATE,
    amount DOUBLE,
    quantity INT
)
PARTITIONED BY (data_dt DATE)
STORED AS ORC
TBLPROPERTIES (
    'transactional'='true',
    'orc.compress'='SNAPPY'
);
```

* **Partitioning** by `data_dt` allows efficient **INSERT OVERWRITE** per partition instead of rewriting the whole table.
* **ACID** support allows **UPDATE/DELETE** if needed.

### 2️⃣ Loading Strategy

Since the source data contains overlapping 1-week data:

1. **Identify partitions to refresh**:

   * For 02-Jan-2022 source (25-Dec-2021 → 01-Jan-2022), partitions `2021-12-25` → `2022-01-01` are affected.
   * For 03-Jan-2022 source (26-Dec-2021 → 02-Jan-2022), partitions `2021-12-26` → `2022-01-02` are affected.

2. **Use INSERT OVERWRITE on affected partitions**:

```sql
-- Example for 02-Jan-2022 load
INSERT OVERWRITE TABLE sales_history PARTITION (data_dt)
SELECT sale_id, product_id, customer_id, sale_date, amount, quantity, data_dt
FROM sales_source
WHERE data_dt BETWEEN '2021-12-25' AND '2022-01-01';
```

* **INSERT OVERWRITE** ensures **updates and deletes** from source are reflected in the target table for those partitions.
* Other partitions (not in the affected date range) remain unchanged.

3. **Next day load (03-Jan-2022)**:

```sql
INSERT OVERWRITE TABLE sales_history PARTITION (data_dt)
SELECT sale_id, product_id, customer_id, sale_date, amount, quantity, data_dt
FROM sales_source
WHERE data_dt BETWEEN '2021-12-26' AND '2022-01-02';
```

* Partitions `2021-12-26` → `2022-01-02` are **refreshed**.
* **No effect** on partitions before `2021-12-26`.

### 3️⃣ Key Considerations

* **Partitioning**:

  * Always partition on the **date column (`data_dt`)** to efficiently overwrite only affected partitions.

* **Table Type**:

  * **ACID transactional table** if updates and deletes are frequent.
  * Otherwise, **non-transactional ORC table** is sufficient if source always sends full data per partition.

* **Compaction**:

  * For ACID tables, schedule **minor/major compactions** to merge delta files.

* **Automation**:

  * Maintain a **list of affected partitions** for each daily load.
  * Script the **INSERT OVERWRITE** statements per partition dynamically.

### 4️⃣ Optional: Using Staging Table

1. Load **daily source data** into a **staging table**:

```sql
CREATE TABLE sales_stage LIKE sales_history STORED AS ORC;
```

2. Perform **partition-wise overwrite** from staging to history table:

```sql
INSERT OVERWRITE TABLE sales_history PARTITION (data_dt)
SELECT *
FROM sales_stage
WHERE data_dt BETWEEN <start_date> AND <end_date>;
```

* Keeps staging data separate and allows **validation** before updating the main table.

### ✅ Summary

| Requirement                             | Hive Solution                                             |
| --------------------------------------- | --------------------------------------------------------- |
| Historical data maintenance             | Partitioned table by `data_dt`                            |
| Source contains overlapping 1-week data | Refresh only affected partitions using `INSERT OVERWRITE` |
| Updates / deletes from source           | Use ACID transactional table                              |
| Large dataset / performance             | ORC storage with compression, partition pruning           |
| Multiple days / months                  | Automate partition detection and overwrite per partition  |


This approach ensures **reinstated weekly data** from the source is correctly reflected in Hive, **without touching unaffected historical partitions**, supporting incremental loads, updates, and deletes efficiently.

---

Here’s a **concise version** of the Hive views explanation for interviews:

---

## Q41. Views in Hive

### What is a view?

* A **logical table** storing a **query**, not data.
* Simplifies queries, enforces **security**, masks sensitive fields, and filters data.

### Example

```sql
CREATE VIEW emp_view AS
SELECT name, age
FROM students
WHERE age > 1;

SELECT * FROM emp_view;

DROP VIEW emp_view;
```

### Key Points

* **Normal view**: No data stored, reflects base table changes automatically.
* **Materialized view**: Stores data, needs **refresh** to update.
* Can filter or aggregate to show **different rows/columns** than base table.

### Example of materialized view

```sql

CREATE TABLE stock_prices (
  stock_id STRING,
  price DOUBLE,
  market STRING,
  ts STRING
) STORED AS ORC;

INSERT INTO stock_prices VALUES
('AAPL', 178.5, 'NASDAQ', '2025-10-15 09:00:00'),
('GOOG', 139.2, 'NASDAQ', '2025-10-15 09:00:00'),
('TCS', 3600.0, 'BSE', '2025-10-15 09:00:00');

CREATE MATERIALIZED VIEW stock_prices_mv AS
SELECT stock_id, price, ts
FROM stock_prices
WHERE market='NASDAQ';

SELECT * FROM stock_prices_mv;

INSERT INTO stock_prices VALUES
('MSFT', 410.8, 'NASDAQ', '2025-10-16 09:00:00'),
('INFY', 1520.0, 'BSE', '2025-10-16 09:00:00');

SELECT * FROM stock_prices_mv;

# By default, Hive does not auto-refresh materialized views. We must manually trigger it: Version 3.x
ALTER MATERIALIZED VIEW stock_prices_mv REBUILD;

# Hive version 2.x
DROP MATERIALIZED VIEW IF EXISTS stock_prices_mv;

CREATE MATERIALIZED VIEW stock_prices_mv AS
SELECT stock_id, price, ts
FROM stock_prices
WHERE market = 'NASDAQ';


SELECT * FROM stock_prices_mv;

```

* Useful for **time-sensitive data** (e.g., stock prices at 9am, 10am, 11am).

✅ **Summary**: Normal views = logical, always in sync; materialized views = stored data, faster queries, refresh required.


## Q42. Will a new column in the base table appear in the view?

* **If created with specific columns:** New columns **won’t appear**.
* **If created with `SELECT *`:** New columns **will appear automatically**.

**Example:**

```sql
CREATE VIEW emp_view AS SELECT empid, empname FROM employee;  -- No new cols shown
CREATE VIEW emp_view_all AS SELECT * FROM employee;           -- New cols visible
```

---

## Q43. Can we load data into Hive without using HDFS?

✅ **Yes**, by using the `LOCAL` keyword.

```sql
LOAD DATA LOCAL INPATH '/home/hduser/data/sales.csv'
INTO TABLE sales;
```

* Loads data **directly from the local file system**.
* Hive copies it into the table’s HDFS location automatically.

**Other options:**

* `CREATE TABLE new_tbl AS SELECT * FROM existing_tbl;` → uses HDFS internally.

**Summary:**
`LOAD DATA LOCAL INPATH` is the best way to load local files into Hive without manually uploading to HDFS.

---

## Q44. Does Hive depend only on HDFS as a storage layer?

❌ **No.** While **HDFS** is the **default storage layer**, Hive can work with **multiple storage systems** using **storage handlers**.

### Examples:

* **Cloud storages:** Amazon S3, Google Cloud Storage, Azure Data Lake
* **Databases / NoSQL:** HBase, Cassandra, Druid, Elasticsearch, BigQuery (BigLake)
* **Local file systems:** For testing or small-scale setups

### Example:

```sql
CREATE EXTERNAL TABLE hbase_emp(
  empid STRING,
  empname STRING
)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,details:name")
TBLPROPERTIES ("hbase.table.name" = "emp_details");
```

✅ **Summary:**
Hive can use **any storage backend** — HDFS is default, but not mandatory.

Excellent question — and very relevant when using **Hive with non-HDFS backends** like **MySQL**.

Here’s the clear answer 👇

---

## Q45. If Hive connects to MySQL, will YARN advantages still apply?

🧩 **Partially yes — but with limitations.**

Let’s break it down:


### **1️⃣ Hive’s execution still runs on YARN**

Even if your **data source** is MySQL (or any external DB):

* Hive queries are still **compiled, optimized, and executed** by Hive’s engine (MapReduce, Tez, or Spark).
* These jobs still **run on YARN**, so you get:

  * **Cluster resource management**
  * **Parallel execution**
  * **Fault tolerance**

✅ Example:

```sql
CREATE EXTERNAL TABLE mysql_sales
STORED BY 'org.apache.hive.storage.jdbc.JdbcStorageHandler'
TBLPROPERTIES (
  "hive.sql.database.type" = "MYSQL",
  "hive.sql.jdbc.driver" = "com.mysql.jdbc.Driver",
  "hive.sql.jdbc.url" = "jdbc:mysql://localhost:3306/salesdb",
  "hive.sql.dbcp.username" = "root",
  "hive.sql.dbcp.password" = "root",
  "hive.sql.table" = "sales"
);
```

When you run a query like:

```sql
SELECT * FROM mysql_sales WHERE amount > 1000;
```

Hive still runs a **distributed job on YARN**.


### **2️⃣ But data processing is limited by MySQL’s connector**

When you use **JDBC-based storage handlers**:

* Hive pulls data from MySQL into mappers or Spark executors.
* Data **may not be fully parallelized** — depends on how the connector splits queries.
* Heavy queries might overload MySQL, not YARN.

So while **Hive’s engine uses YARN**, **data I/O performance** is limited by the MySQL connector, not HDFS parallelism.

### **3️⃣ Summary**

| Feature         | Using HDFS    | Using MySQL (via JDBC)            |
| --------------- | ------------- | --------------------------------- |
| Runs on YARN    | ✅ Yes         | ✅ Yes                             |
| Parallel I/O    | ✅ Distributed | ⚠️ Limited                        |
| Fault tolerance | ✅ Full        | ⚠️ Partial (depends on connector) |
| Best for        | Big Data      | Reference / Lookup data           |


✅ **In short:**
You’ll still get YARN’s **resource management and fault tolerance**, but not the **full parallel read/write benefits** of HDFS-based tables.

Hive-on-MySQL is great for **joining external reference data**, not for **large-scale distributed processing**.


## Q46. Is multiline comment supported in Hive script?

❌ **No**, Hive does **not** support multiline comments.

You can only use **single-line comments** in Hive scripts:

```sql
-- This is a single-line comment
# This is also a single-line comment
```

### Examples of multiline comment (not supported in Hive)

These work in other languages but **not in Hive**:

* **Scala / Java:** `/* comment */`
* **Python:** `''' comment '''` or `""" comment """`

✅ **Summary:**
Hive supports only **single-line comments** using `--` or `#`; **multiline comments are not supported.**

---

## Q47. Does Hive support record-level Insert, Update, or Delete?

🧩 **By default:**
Hive supports **only record-level inserts**, not updates or deletes.

### ✅ **Insert Example**

```sql
INSERT INTO employee VALUES (101, 'John', 'HR');
```

### ❌ **Update/Delete — Not supported by default**

Hive was originally designed for **batch processing**, not row-level transactions.


### ✅ **To enable record-level Update/Delete**

You must use **ACID (transactional) tables**, which require:

* `transactional = true`
* **ORC file format**
* **Bucketing enabled**
* Hive **2.x or later**

```sql
CREATE TABLE employee_txn (
  empid INT,
  empname STRING,
  dept STRING
)
CLUSTERED BY (empid) INTO 4 BUCKETS
STORED AS ORC
TBLPROPERTIES ('transactional'='true');
```

Then you can do:

```sql
UPDATE employee_txn SET dept='Finance' WHERE empid=101;
DELETE FROM employee_txn WHERE empid=102;
```

✅ **Summary:**

| Operation | Default         | With ACID Table |
| --------- | --------------- | --------------- |
| INSERT    | ✅ Supported     | ✅ Supported     |
| UPDATE    | ❌ Not supported | ✅ Supported     |
| DELETE    | ❌ Not supported | ✅ Supported     |

Hive supports full DML only for **ACID-enabled ORC transactional tables**.

---

## Q48. How to run Hive queries from CLI without entering Hive prompt & pass arguments?**

* Run query directly:

  ```bash
  hive -e "SELECT * FROM txns WHERE state='California' limit 5 ;"
  ```

* Pass arguments: [NOT WORKING]

  ```bash
  hive -hivevar state='California' -e "SELECT * FROM txns WHERE state='${hivevar:state}' limit 5 ;"
  ```

* Run script file with variable:

  ```bash
  vi file.hql
  SELECT * FROM default.txns WHERE state='${hivevar:state}' limit 5;
  
  hive -hivevar state='California' -f file.hql
  ```

✅ Use `-e` for inline queries, `-f` for files, and `--hivevar` to pass parameters.

---

## Q49. What kind of database/data warehouse application is suitable for Hive?

Hive is **not a full-fledged database or data warehouse system** — rather, it serves as a **complementary or supplementary layer** on top of Hadoop for analytical and batch processing workloads.

It is best suited for:

* ✅ **Data warehousing and ETL** — performing large-scale aggregations, joins, and summarizations.
* ✅ **Batch analytics** — queries over huge, immutable datasets (TB–PB scale).
* ✅ **Historical trend analysis** — where low-latency isn’t critical.

Hive is **not ideal** for:

* ❌ OLTP (Online Transaction Processing)
* ❌ Real-time analytics or frequent updates/deletes

**Reason:**
The design of **Hadoop + HDFS** focuses on **high-throughput sequential reads/writes**, not low-latency random access. Hence, Hive is a **read-heavy, append-optimized** system used primarily for **big data analytics**, not as a transactional database.

---

## **Q50. Challenges faced when migrating data to Hive from RDBMS/DB/DWH**

| **Challenge**                              | **Issue**                                                                                                          | **Mitigation / Notes**                                                                              | **Priority / Impact** |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- | --------------------- |
| **Count Mismatch**                         | Differences in row counts due to constraints, column mismatches, or partial failures                               | Pilot/parallel runs, data quality frameworks, reconciliation (`--validate` in Sqoop, Spark scripts) | High                  |
| **SLA & Latency**                          | High latency on small ad-hoc queries; difficult to meet SLAs                                                       | Use optimized engines (Tez, Impala, Presto, Spark), query optimization                              | High                  |
| **Data Type & Size Mismatch**              | Hive and RDBMS data types differ                                                                                   | Testing and mapping strategy to handle conversions                                                  | Medium                |
| **Duplicates**                             | Source data may contain duplicates                                                                                 | Deduplicate using Hive queries or Spark transformations                                             | Medium                |
| **Performance**                            | Hive performance can be poor for certain workloads                                                                 | Partitioning, bucketing, ORC/Parquet formats, Tez/Spark execution, query tuning                     | High                  |
| **Cost Management**                        | Large data volumes incur high compute/storage costs                                                                | Optimize ETL pipelines, use compression, archive old data                                           | Medium                |
| **CDC (Change Data Capture)**              | Need to track inserts/updates/deletes from source systems                                                          | Sqoop (CDC), NiFi (CDC), Spark DeltaLake for incremental loads                                      | High                  |
| **ACID & DML**                             | Record-level insert/update/delete limited                                                                          | Hive 3+ ACID tables with ORC format and bucketing (still evolving)                                  | Medium                |
| **SCD (Slowly Changing Dimensions)**       | Applying SCD types is challenging                                                                                  | Use incremental loads, timestamp/versioning logic                                                   | Medium                |
| **Constraints (PK, FK, NOT NULL, UNIQUE)** | Hive does not enforce constraints physically                                                                       | Constraints are mostly for representation; batch data from DB/DWH is assumed clean                  | Low                   |
| **Small File Load Issues**                 | Many small files degrade HDFS and query performance                                                                | Compaction (minor/major), Hadoop archive, `INSERT OVERWRITE` with tuned map/reducer sizes           | High                  |
| **Unsupported Features**                   | Select-clause subqueries, multi-column subqueries, some Python/Java functions, `MINUS`/`EXCEPT`, format mismatches | Workarounds using joins, `LEFT JOIN/NOT EXISTS`, or pre-processing in Spark                         | Medium                |

---

## **Q51. Why and where Hive is best suited for data warehouse applications**

Hive is **designed for analytical workloads** on large-scale data stored in Hadoop. It is **not suitable for OLTP**, but it excels as a data warehouse tool.

### **Key Characteristics Making Hive Suitable for DW**

1. **Relatively static or incremental data**

   * Hive works best with **append-heavy or read-mostly datasets** rather than rapidly changing transactional data.
   * Example: Daily sales or log data that is added but rarely updated.

2. **Low requirement for fast response time**

   * Hive is **batch-oriented**, so **sub-second query latency is not expected**.
   * It’s suitable for **scheduled analytics, reporting, and ETL pipelines**.

3. **Data does not change rapidly**

   * Hive tables are typically **immutable or slowly changing**, which fits well with data warehouse workloads.

4. **Integration with other engines for optimization**

   * Challenges like latency and performance can be mitigated using:

     * **In-memory engines:** Spark, Tez
     * **Query acceleration engines:** Presto, Impala, LLAP
     * **NoSQL integration** for specialized storage and fast lookups


### **Why Hive is not for OLTP**

* Hive **does not support low-latency, row-level transactional updates** efficiently.
* It lacks **real-time insert/update/delete capabilities** required for online transaction processing.
* Hive is closer to **OLAP (Online Analytical Processing)** — optimized for **querying, aggregating, and analyzing large datasets**.


### **Use Cases in Data Warehousing**

| **Use Case**              | **Description**                                                              |
| ------------------------- | ---------------------------------------------------------------------------- |
| Analytics & Reporting     | Generating daily/weekly/monthly reports from terabytes of historical data    |
| Data Mining               | Mining large datasets for patterns, trends, and insights                     |
| ETL/ELT Pipelines         | Transforming and loading large-scale batch data from RDBMS/NoSQL into Hadoop |
| Historical Trend Analysis | Analyzing slowly changing dimensions (SCD1, SCD2) or archival data           |

✅ **Summary:**
Hive is **ideal for batch-oriented, large-scale analytical workloads**, making it a strong choice for **data warehouse applications**, but it **cannot replace OLTP systems**.

---

