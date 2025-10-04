## 🚀 Hive Real-time Scenario Questions & Scenarios

-----

### ⚙️ I. Core Hive Query Execution and Configuration

#### Q1. Why does a simple `SELECT *` query not run a MapReduce/Tez/Spark job in Hive?

This is due to the **Hive Fetch Task Optimization**. The property **`hive.fetch.task.conversion`** lowers the latency overhead of MapReduce/Tez/Spark by skipping the job execution for simple queries.

  * 💡 **Explanation:** Queries that only involve fetching, filtering, or limiting data (e.g., `SELECT`, `FILTER`, `LIMIT`) can bypass the distributed execution engine.
  * ⚙️ **Values:** `none`, `minimal`, and `more`.
  * **Example Query (Bypasses MapReduce):**
    ```sql
    SELECT * FROM table_name LIMIT 10;
    ```
  * **Check the Default Value:**
    ```sql
    SET hive.fetch.task.conversion;
    ```

#### Q2. How to change Hive configurations?

Configurations can be changed at multiple levels:

| Level | Method | Example |
| :--- | :--- | :--- |
| **Current Session/Query** | Use the **`SET`** command. | `SET hive.exec.engine=mr;` |
| **Cluster/Job Level (User)** | Change in **`hive-site.xml`** and submit via Oozie or similar job scheduler. | N/A |
| **Cluster Level (Admin)** | Change in cluster management tools (Ambari, Cloudera Manager, EMR, Dataproc). | N/A |

-----

### 🧠 II. Advanced HiveQL and Logic

#### Q3. Can I use a multi-column subquery in the `WHERE` clause in Hive?

Hive typically does not support multi-column subqueries using the `IN` clause (e.g., `WHERE (col1, col2) IN (SELECT col1, col2...)`).

**❌ Non-Working Example:**

```sql
SELECT * FROM tableA
WHERE (A.id, A.name, A.Roll_no) IN (
    SELECT Id, name, roll_no FROM tableB
);
```

**✅ Solution 1: Using `CONCAT` (Concatenation)**

```sql
SELECT * FROM tableA
WHERE CONCAT(A.id, A.name, A.Roll_no) IN (
    SELECT CONCAT(Id, name, roll_no) FROM tableB
);
```

**✅ Solution 2: Using an `INNER JOIN` (Preferred Method)**

```sql
SELECT A.*
FROM tableA A
INNER JOIN tableB B
  ON A.id = B.id
  AND A.name = B.name
  AND A.Roll_no = B.Roll_no;
```

#### Q4. Does Hive support `MINUS`/`EXCEPT`? How do I achieve the equivalent?

No, Hive does not support **`MINUS`** or **`EXCEPT`**.

**✅ Solution: Using a `LEFT JOIN` and filtering for `NULL`**

```sql
-- Find all records in table A that do NOT exist in table B
SELECT A.*
FROM tableA A
LEFT JOIN tableB B
  ON (A.x = B.x AND A.y = B.y AND A.z = B.z) -- Match on all columns
WHERE B.x IS NULL; -- The record only exists in A
```

#### Q5. Does Hive support subqueries in the `SELECT` clause?

No, Hive traditionally does not support subqueries in the `SELECT` list.

**❌ Non-Working Example:**

```sql
SELECT customer_num, (SELECT MAX(ship_charge) FROM orders) AS max_ship_chg
FROM customer;
```

**✅ Solution: Using an `INNER JOIN` (Cross Join)**

```sql
SELECT
    C.customer_num,
    M.ship_chg
FROM customer C
INNER JOIN (
    SELECT MAX(ship_charge) AS ship_chg
    FROM orders
) M ON 1=1;
```

#### Q6. How to generate a Surrogate Key / Sequence Number in Hive?

Use SQL analytical functions or unique ID generators:

  * **Window Function:** **`ROW_NUMBER() OVER (PARTITION BY col1 ORDER BY col2)`**
  * **UUID:** The **`UUID()`** function.
  * **Others:** Custom UDFs or `monotonically_increasing_id()` (in Spark/Hive environments).

#### Q7. Inner Join behavior with `NULL` keys.

When performing an `INNER JOIN` on a column that contains **`NULL`** values, the `NULL` values **will not match**. SQL treats `NULL` as an "unknown" value, and two unknown values are not considered equal.

  * 🚫 **Example Result:** If `Table A` and `Table B` both have a row where the join key is `NULL`, the join result will **not** include a merged row for these records.

-----

### 💾 III. Data Ingestion and Table Properties

#### Q8. How to skip header/footer rows in a Hive table?

Use **`TBLPROPERTIES`**:

  * **Skip Header:** `TBLPROPERTIES("skip.header.line.count"="2");`
  * **Skip Footer:** `TBLPROPERTIES("skip.footer.line.count"="1");`

#### Q9. How to read fixed-width data in Hive?

1.  **Using `RegexSerDe` (Regular Expression Serializer/Deserializer)**:
    ```sql
    CREATE EXTERNAL TABLE customers (...)
    ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'
    WITH SERDEPROPERTIES ("input.regex" = "(.{10})(.{10})" ) -- Regex for fixed width
    ...
    ```
2.  **Using `SUBSTR` and `TRIM`**: Load the fixed-width file as a single string column, then parse it:
    ```sql
    SELECT
        TRIM(SUBSTR(single_line_col, 1, 50)) AS name,
        CAST(TRIM(SUBSTR(single_line_col, 51, 3)) AS INT) AS age
    FROM temp_single_column_table;
    ```

#### Q10. How to prevent accidental data duplication on subsequent data loads?

Use the **Immutable Table** property for static or fixed dimension data.

  * 🛡️ **Mechanism:** **`INSERT INTO`** is disallowed if any data is already present. The first `INSERT INTO` succeeds, but successive ones fail.
  * 📌 **Note:** **`INSERT OVERWRITE`** is still allowed even if the table is immutable.
  * **Syntax:** `TBLPROPERTIES ("immutable"="true");`

#### Q11. How to Change table from `EXTERNAL` to `MANAGED` and vice versa?

Change the **`EXTERNAL`** property in the table's metadata:

  * **External to Managed:** `ALTER TABLE customers3 SET TBLPROPERTIES('EXTERNAL'='FALSE');`
  * **Managed to External:** `ALTER TABLE customers3 SET TBLPROPERTIES('EXTERNAL'='TRUE');`

#### Q12. How to load data into Hive without first copying it to HDFS?

Use the **`LOAD DATA LOCAL INPATH`** command.

  * **Example:** `LOAD DATA **LOCAL** INPATH ‘/local/file/path’ INTO TABLE tbl_name;`

#### Q13. Does Hive support record-level DML (`INSERT`, `UPDATE`, `DELETE`)?

  * **Record-level `INSERT`** is supported using `INSERT INTO tablename VALUES();`.
  * **`UPDATE` and `DELETE`** are **not supported by default**, but can be enabled if the table is set up for **ACID transactions**.

#### Q14. What are the constraints in Hive?

Hive supports constraints (**`PRIMARY KEY`**, **`NOT NULL`**, etc.) for **representation purposes only** (metadata), not for **enforcement**.

-----

### ✨ IV. Data Quality and Deduplication

#### Q15. How to remove duplicate data from a Hive table?

**Option 1: Low Volume (`DISTINCT` / `GROUP BY`)**
Use **`INSERT OVERWRITE`** with **`SELECT DISTINCT`**.

```sql
INSERT OVERWRITE TABLE oldtable
SELECT DISTINCT id, name, phone, ts FROM oldtable;
```

**Option 2: High Volume (Window Function - Recommended)**
Use the **`ROW_NUMBER()`** analytical function to keep only the latest/first record (`rnk = 1`):

```sql
CREATE TABLE newtbl AS
SELECT id, name, phone, ts
FROM (
    SELECT
        id, name, phone, ts,
        ROW_NUMBER() OVER(PARTITION BY id, name, phone ORDER BY ts DESC) AS rnk
    FROM oldtable
) ranked_data
WHERE rnk = 1;

-- Then, drop the original table and rename the new one.
DROP TABLE oldtable;
ALTER TABLE newtbl RENAME TO oldtable;
```

#### Q16. Reinstated Data Load Scenario (Handling SCD/Updates)

**Scenario:** Daily feed contains 1 week of data (with updates) to load into a target table containing 3 years of partitioned data.

**Solution:** Use **`INSERT OVERWRITE`** but strategically limit the scope to the specific **partitions** being updated.

  * **Target Table Type:** Partitioned table based on the data date/timestamp.
  * **Logic:** The query must explicitly overwrite *only* those partitions (e.g., `WHERE PARTITION_COL IN ('2021-12-26', '2022-01-02')`).

-----

### 🧱 V. Partitioning and Data Management

#### Q17. How to manage data files in multiple HDFS subdirectories under a single Hive table?

Enable recursive directory listing and subdirectory support:

```sql
SET mapred.input.dir.recursive=true;
SET hive.mapred.supports.subdirectories=true;
```

#### Q18. How to discover partitions created by external systems (e.g., Spark/Sqoop)?

External tools load data into HDFS but don't update the Hive Metastore. Run a repair command:

  * **Repair Command:** **`MSCK REPAIR TABLE <db_name>.<table_name>;`**

#### Q19. How to automatically configure partition discovery and retention?

  * 🔄 **Automatic Discovery:** Set `metastore.partition.management.task.frequency` and enable on the table:
    ```sql
    SET metastore.partition.management.task.frequency=600;
    ALTER TABLE exttbl SET TBLPROPERTIES ('discover.partitions' = 'true');
    ```
  * 🗑️ **Partition Retention (Purging):** Configure a period after which data and metadata are dropped:
    ```sql
    ALTER TABLE customer SET TBLPROPERTIES ('partition.retention.period'='365d');
    ```

#### Q20. Maximum Dynamic Partition Limits

Limits prevent resource exhaustion from creating too many partitions in one operation:

  * **Per Node Limit:** Default is **100**. **Configuration:** `SET hive.exec.max.dynamic.partitions.pernode = <value>`
  * **Total Limit:** Default is **1000**. **Configuration:** `SET hive.exec.max.dynamic.partitions = <value>`

#### Q21. Dropping Tables Permanently (Bypassing Trash)

Use the **`PURGE`** option with the `DROP TABLE` statement:

```sql
DROP TABLE [IF EXISTS] table_name PURGE;
```

This skips the HDFS trash directory.

#### Q22. Effect of Renaming a Hive Table

  * **Managed Table:** HDFS location **is** automatically changed.
  * **External Table:** HDFS location is **not** changed.

#### Q23. Effect of Changing Partition Location (`ALTER TABLE ... SET LOCATION`)

The **data is not automatically moved**. The data must be moved manually in HDFS.

-----

### ⚡ VI. Performance and Optimization

#### Q24. How to resolve `OutOfMemoryError: Java heap space` when running a UDF on large data?

Increase the memory allocated to the tasks:

```sql
SET mapreduce.map.memory.mb=4096m;
SET mapreduce.map.java.opts=-Xmx3686m; -- JVM heap size (e.g., 90% of memory.mb)
SET mapreduce.reduce.memory.mb=4096m;
SET mapreduce.reduce.java.opts=-Xmx3686m;
```

#### Q25. How to fix a "vertex error" during a Hive query?

A common fix is to switch the execution engine from Tez/Spark to MapReduce:

```sql
SET hive.execution.engine=mr;
```

#### Q26. Data Skew Handling (Using `hive.skewjoin.key` properties)

Data skew bottlenecks a single reducer. Hive's optimization splits the large key's work across multiple reducers.

1.  **Detect Skew:** `SET hive.skewjoin.exist = true;`
2.  **Enable Optimization:** `SET hive.optimize.skewjoin = true;`
3.  **Mechanism:** Skewed keys are split by adding a random suffix, allowing multiple smaller tasks to run in parallel.

#### Q27. Steps to Handle the "Small Files Issue" in Hive

Small files cause NameNode overload and inefficient MapReduce.

1.  **Clean Zero-Byte Files:** Remove `_SUCCESS` and `_FAILURE` files.
2.  **Archiving (HAR):** Logically group files. **Benefit:** It **reduces the number of files stored**.
3.  **Merge Files (On Overwrite):** Use `INSERT OVERWRITE` with merge properties:
    ```sql
    SET hive.merge.mapfiles=true;
    SET hive.merge.smallfiles.avgsize=104857600; -- e.g., 100 MB
    INSERT OVERWRITE TABLE table1 SELECT * FROM table1;
    ```
4.  **Compaction (For ACID Tables):** Enable and schedule **Major** and **Minor** compactions for transactional tables.

-----

### 🏗️ VII. Data Architecture and Lifecycle

#### Q28. End-to-end Data Management & Storage Layers

| Layer | Purpose | Key Technology |
| :--- | :--- | :--- |
| **Raw/Transient** | Initial Ingestion | HDFS/S3/GCS/Kafka |
| **Curated Layer** | ETL/ELT, Batch Aggregation | **Hive** (using ORC/Snappy), Presto, Impala, BigQuery |
| **Discovery/Presentation** | Real-time (RT) Aggregation | **NoSQL** (Cassandra, HBase, ES) for low-latency random read/write |

#### Q29. How to write parameterized Hive queries without logging into the prompt?

Use **`hivevar`** or **`hiveconf`** flags when executing the script:

```bash
hive -hivevar db_name='prodretaildb' -hivevar load_dt='2021-12-26' -f /home/hduser/xyz.hql

# Inside the HQL file:
SELECT * FROM ${hivevar:db_name}.tablename
WHERE loaddt=${hivevar:load_dt};
```

#### Q30. Views vs. Materialized Views

| Feature | View | Materialized View (MV) |
| :--- | :--- | :--- |
| **Data Storage** | No (Logical Query/Stored Statement). | Yes (Pre-computed, Cached Data). |
| **Primary Use** | Security (data hiding/masking), Query Simplification. | Performance for BI/dashboard queries. |
| **Schema Evolution** | Only sees new columns if created with `SELECT *`. | N/A |

-----

### ❓ VIII. Additional Interview Questions

| Question | Answer/Solution |
| :--- | :--- |
| **Max size of `STRING` datatype?** | **2 GB**. |
| **Pivot Array column to rows?** | Use the **`EXPLODE`** function. |
| **Connect to Hive?** | **Beeline CLI** (replaces Hive CLI), JDBC/ODBC, Spark.sql(). |
| **Multi-line comment supported?** | **No**. |
| **Partition data present, but no metadata?** | **D. No result are returned**. |
| **Archiving a Partition Benefit?** | **D. reduces the number of files stored**. |
| **Hive most suitable for?** | Data warehouse applications with **static/incremental data** where **fast response time is not required**. |