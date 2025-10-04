
## 🚀 Hive Real-time Scenario Questions & Scenarios

### I. Core Hive Query Execution and Configuration

#### Q1. Why does a simple `SELECT *` query not run a MapReduce/Tez/Spark job in Hive?

[cite\_start]This is due to the **Hive Fetch Task Optimization**[cite: 1]. [cite\_start]The property `hive.fetch.task.conversion` lowers the latency overhead of MapReduce/Tez/Spark by skipping the job execution for simple queries[cite: 1].

  * [cite\_start]**Explanation:** Queries that only involve fetching, filtering, or limiting data (e.g., `SELECT`, `FILTER`, `LIMIT`) can bypass the distributed execution engine[cite: 1].
  * [cite\_start]**Values:** `none`, `minimal`, and `more`[cite: 1].
  * **Example Query (Bypasses MapReduce):**
    ```sql
    SELECT * FROM table_name LIMIT 10;
    ```
  * [cite\_start]**Check the Default Value:** `SET hive.fetch.task.conversion;` [cite: 2]

#### Q2. How to change Hive configurations?

[cite\_start]Configurations can be changed at multiple levels[cite: 3]:

| Level | Method | Example |
| :--- | :--- | :--- |
| **Current Session/Query** | Use the `SET` command. | [cite\_start]`SET hive.exec.engine=mr;` [cite: 4] |
| **Cluster/Job Level (User)** | [cite\_start]Change in `hive-site.xml` and submit via Oozie or similar job scheduler[cite: 5]. | N/A |
| **Cluster Level (Admin)** | [cite\_start]Change in cluster management tools (Ambari, Cloudera Manager, EMR, Dataproc)[cite: 6]. | N/A |

-----

### II. Advanced HiveQL and Logic

#### Q3. Can I use a multi-column subquery in the `WHERE` clause in Hive?

[cite\_start]Hive typically does not support multi-column subqueries using the `IN` clause (e.g., `WHERE (col1, col2) IN (SELECT col1, col2...)`)[cite: 7].

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

[cite\_start]**✅ Solution 2: Using an `INNER JOIN` (Preferred Method)** [cite: 9]

```sql
SELECT A.*
FROM tableA A
INNER JOIN tableB B
  ON A.id = B.id
  AND A.name = B.name
  AND A.Roll_no = B.Roll_no;
```

#### Q4. Does Hive support `MINUS`/`EXCEPT`? How do I achieve the equivalent?

[cite\_start]No, Hive does not support `MINUS`/`EXCEPT`[cite: 122].

[cite\_start]**✅ Solution: Using a `LEFT JOIN` and filtering for `NULL`** [cite: 124]

```sql
-- Find all records in table A that do NOT exist in table B
SELECT A.*
FROM tableA A
LEFT JOIN tableB B
  ON (A.x = B.x AND A.y = B.y AND A.z = B.z) -- Match on all columns
WHERE B.x IS NULL; -- The record only exists in A
```

#### Q5. Does Hive support subqueries in the `SELECT` clause?

[cite\_start]No, Hive traditionally does not support subqueries in the `SELECT` list[cite: 125].

**❌ Non-Working Example:**

```sql
SELECT customer_num, (SELECT MAX(ship_charge) FROM orders) AS max_ship_chg
FROM customer;
```

[cite\_start]**✅ Solution: Using an `INNER JOIN` (Cross Join)** [cite: 125]

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

[cite\_start]Use SQL analytical functions or unique ID generators[cite: 119]:

  * **Window Function:** `ROW_NUMBER() OVER (PARTITION BY col1 ORDER BY col2)`
  * **UUID:** The `UUID()` function.
  * **Others:** Custom UDFs or `monotonically_increasing_id()` (in Spark/Hive environments).

#### Q7. Inner Join behavior with `NULL` keys.

[cite\_start]When performing an `INNER JOIN` on a column that contains `NULL` values, the `NULL` values **will not match**[cite: 133]. SQL treats `NULL` as an "unknown" value, and two unknown values are not considered equal.

  * [cite\_start]**Example Result:** If `Table A` and `Table B` both have a row where the join key is `NULL`, the join result will **not** include a merged row for these records[cite: 133].

-----

### III. Data Ingestion and Table Properties

#### Q8. How to skip header/footer rows in a Hive table?

[cite\_start]Use `TBLPROPERTIES`[cite: 11]:

  * **Skip Header:** `TBLPROPERTIES("skip.header.line.count"="2");`
  * **Skip Footer:** `TBLPROPERTIES("skip.footer.line.count"="1");`

#### Q9. How to read fixed-width data in Hive?

1.  [cite\_start]**Using `RegexSerDe` (Regular Expression Serializer/Deserializer)**[cite: 21]:
    ```sql
    CREATE EXTERNAL TABLE customers (...)
    ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'
    WITH SERDEPROPERTIES ("input.regex" = "(.{10})(.{10})" ) -- Regex for fixed width
    ...
    ```
2.  [cite\_start]**Using `SUBSTR` and `TRIM`**[cite: 20]: Load the fixed-width file as a single string column, then parse it:
    ```sql
    SELECT
        TRIM(SUBSTR(single_line_col, 1, 50)) AS name,
        CAST(TRIM(SUBSTR(single_line_col, 51, 3)) AS INT) AS age
    FROM temp_single_column_table;
    ```

#### Q10. How to prevent accidental data duplication on subsequent data loads?

[cite\_start]Use the **Immutable Table** property for static or fixed dimension data (e.g., calendar, geography)[cite: 26, 27].

  * [cite\_start]**Mechanism:** `INSERT INTO` is disallowed if any data is already present[cite: 29]. [cite\_start]The first `INSERT INTO` succeeds, but successive ones fail, preventing duplication[cite: 27].
  * [cite\_start]**Note:** `INSERT OVERWRITE` is still allowed even if the table is immutable[cite: 30].
  * [cite\_start]**Syntax:** `TBLPROPERTIES ("immutable"="true");` [cite: 28]

#### Q11. How to Change table from `EXTERNAL` to `MANAGED` and vice versa?

[cite\_start]Change the `EXTERNAL` property in the table's metadata[cite: 32]:

  * **External to Managed:** `ALTER TABLE customers3 SET TBLPROPERTIES('EXTERNAL'='FALSE');`
  * **Managed to External:** `ALTER TABLE customers3 SET TBLPROPERTIES('EXTERNAL'='TRUE');`

#### Q12. How to load data into Hive without first copying it to HDFS?

[cite\_start]Use the **`LOAD DATA LOCAL INPATH`** command[cite: 158].

  * [cite\_start]**Example:** `LOAD DATA **LOCAL** INPATH ‘/local/file/path’ INTO TABLE tbl_name;` [cite: 158]

#### Q13. Does Hive support record-level DML (`INSERT`, `UPDATE`, `DELETE`)?

  * [cite\_start]**Record-level `INSERT`** is supported using `INSERT INTO tablename VALUES();`[cite: 162].
  * [cite\_start]**`UPDATE` and `DELETE`** are **not supported by default**, but can be enabled if the table is set up for **ACID transactions** (Atomicity, Consistency, Isolation, and Durability)[cite: 163].

#### Q14. What are the constraints in Hive?

[cite\_start]Hive supports constraints (`PRIMARY KEY`, `NOT NULL`, `UNIQUE`, `FOREIGN KEY`) for **representation purposes only**, not for enforcement[cite: 173]. [cite\_start]This is because enforcing constraints would degrade performance and is not practical for the high-volume, non-sequential nature of a Big Data platform[cite: 173, 174].

-----

### IV. Data Quality and Deduplication

#### Q15. How to remove duplicate data from a Hive table?

**Option 1: Low Volume (`DISTINCT` / `GROUP BY`)**
[cite\_start]Use `INSERT OVERWRITE` with `SELECT DISTINCT`[cite: 23].

```sql
INSERT OVERWRITE TABLE oldtable
SELECT DISTINCT id, name, phone, ts FROM oldtable;
```

**Option 2: High Volume (Window Function - Recommended)**
[cite\_start]Use the `ROW_NUMBER()` analytical function to keep only the latest/first record (`rnk = 1`)[cite: 24]:

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

**Scenario:** You receive a daily feed that contains 1 week of data, which includes potential updates/deletes for that week. You need to load it into a target table containing 3 years of partitioned data.

**Solution:** Use an `INSERT OVERWRITE` approach, but strategically limit the scope of the overwrite to the partitions being updated (the 1 week of data).

  * [cite\_start]**Target Table Type:** Partitioned table based on the data date/timestamp[cite: 131].
  * **Logic:** The `INSERT OVERWRITE` query should only run against the specific partitions being refreshed. If the source data covers partitions `2021-12-26` to `2022-01-02`, the query must explicitly overwrite *only* those partitions. [cite\_start]This avoids deleting data outside that date range[cite: 96].

-----

### V. Partitioning and Data Management

#### Q17. How to manage data files in multiple HDFS subdirectories under a single Hive table?

[cite\_start]You must enable recursive directory listing and subdirectory support[cite: 15]:

```sql
SET mapred.input.dir.recursive=true;
SET hive.mapred.supports.subdirectories=true;
```

#### Q18. How to discover partitions created by external systems (e.g., Spark/Sqoop)?

[cite\_start]External tools load data into HDFS but don't update the Hive Metastore[cite: 58]. [cite\_start]You must run a repair command[cite: 60]:

  * [cite\_start]**Repair Command:** `MSCK REPAIR TABLE <db_name>.<table_name>;` [cite: 60]

#### Q19. How to automatically configure partition discovery and retention?

  * [cite\_start]**Automatic Discovery:** Set up partition synchronization to occur periodically, e.g., every 10 minutes (600 seconds)[cite: 64]:
    ```sql
    SET metastore.partition.management.task.frequency=600;
    ALTER TABLE exttbl SET TBLPROPERTIES ('discover.partitions' = 'true');
    ```
  * [cite\_start]**Partition Retention (Purging):** Configure a retention period, after which the partition metadata and data are automatically dropped[cite: 65, 66]:
    ```sql
    ALTER TABLE customer SET TBLPROPERTIES ('partition.retention.period'='365d');
    ```

#### Q20. Maximum Dynamic Partition Limits

[cite\_start]By default, there are limits on how many dynamic partitions can be created in a single operation[cite: 93]:

  * [cite\_start]**Per Node Limit:** Default is **100**[cite: 93].
      * [cite\_start]**Configuration:** `SET hive.exec.max.dynamic.partitions.pernode = <value>` [cite: 94]
  * [cite\_start]**Total Limit:** Default is **1000**[cite: 93].
      * [cite\_start]**Configuration:** `SET hive.exec.max.dynamic.partitions = <value>` [cite: 94]

#### Q21. Dropping Tables Permanently (Bypassing Trash)

[cite\_start]Use the `PURGE` option with the `DROP TABLE` statement[cite: 105]:

```sql
DROP TABLE [IF EXISTS] table_name PURGE;
```

[cite\_start]This skips the HDFS trash directory, meaning the data cannot be recovered[cite: 107].

#### Q22. Effect of Renaming a Hive Table

  * [cite\_start]**Managed Table:** Yes, the underlying HDFS location is automatically changed to match the new table name (if a custom location was not specified)[cite: 112].
  * [cite\_start]**External Table:** No, the underlying HDFS location is **not** changed because Hive does not own the location[cite: 113].

#### Q23. Effect of Changing Partition Location (`ALTER TABLE ... SET LOCATION`)

[cite\_start]If you change a partition location using `ALTER TABLE ... SET LOCATION`, the **data is not automatically moved**[cite: 89]. [cite\_start]The data must be moved manually to the new location[cite: 89].

-----

### VI. Performance and Optimization

#### Q24. How to resolve `OutOfMemoryError: Java heap space` when running a UDF on large data?

[cite\_start]Increase the memory allocated to the MapReduce/Tez tasks[cite: 16]:

```sql
SET mapreduce.map.memory.mb=4096m;
SET mapreduce.map.java.opts=-Xmx3686m; -- JVM heap size (e.g., 90% of memory.mb)
SET mapreduce.reduce.memory.mb=4096m;
SET mapreduce.reduce.java.opts=-Xmx3686m;
```

#### Q25. How to fix a "vertex error" during a Hive query?

[cite\_start]A common fix is to switch the execution engine from Tez/Spark to MapReduce, which can sometimes bypass specific engine-related bugs or resource issues[cite: 19]:

```sql
SET hive.execution.engine=mr;
```

#### Q26. Data Skew Handling (Using `hive.skewjoin.key` properties)

Data skew occurs when one or a few keys have a disproportionately large number of records, leading to a single mapper/reducer processing most of the data and creating a bottleneck.

1.  **Detect Skew:** Set `hive.skewjoin.exist` to `true`.
2.  **Enable Skew Join Optimization:** Set `hive.optimize.skewjoin` to `true`.
3.  **Mechanism:** When Hive detects a skewed key:
      * It splits the processing of the large key into multiple smaller tasks (by adding a random suffix to the skewed key).
      * It processes the small keys separately, resulting in a **split phase join** that improves parallelization.

#### Q27. Steps to Handle the "Small Files Issue" in Hive

[cite\_start]Small files are inefficient and cause NameNode overhead[cite: 71].

1.  [cite\_start]**Clean Zero-Byte Files:** Remove `_SUCCESS` and `_FAILURE` files that consume NameNode memory[cite: 73].
2.  [cite\_start]**Archiving (HAR):** Use Hadoop Archives (HAR) to logically group small files into one large file, reducing file count[cite: 80, 81].
    ```sql
    ALTER TABLE <table_name> ARCHIVE PARTITION (...);
    ```
      * [cite\_start]**Benefit of Archiving:** It **reduces the number of files stored**[cite: 98].
3.  [cite\_start]**Merge Small Files (On Overwrite):** Set merge properties and run an `INSERT OVERWRITE` query to combine small files into fewer, larger files[cite: 85, 87].
    ```sql
    SET hive.merge.mapfiles=true;
    SET hive.merge.smallfiles.avgsize=104857600; -- e.g., 100 MB
    INSERT OVERWRITE TABLE table1 SELECT * FROM table1;
    ```
4.  [cite\_start]**Compaction (For ACID Tables):** Enable and schedule **Major** and **Minor** compactions on transactional tables to combine small Delta files[cite: 89].
    ```sql
    SET hive.compactor.initiator.on=true;
    ```

-----

### VII. Data Architecture and Lifecycle

#### Q28. End-to-end Data Management & Storage Layers

Data typically flows through three main layers:

| Layer | Purpose | Storage Technologies |
| :--- | :--- | :--- |
| **Raw/Transient** | Initial ingestion, reusable source data | HDFS/S3/GCS/Kafka |
| **Curated Layer** | ETL/ELT, Batch Aggregation, OLAP cubes | **Hive** (using MR/Tez/Spark), Presto, Impala, BigQuery |
| **Discovery/Presentation** | Real-time (RT) Aggregation, frequently changed data | [cite\_start]**NoSQL** (Cassandra, HBase, ES) for low-latency random read/write [cite: 36] |

  * [cite\_start]Final processed data is typically stored in Hive using **ORC file format** with **Snappy compression** for efficiency[cite: 39].

#### Q29. How to write parameterized Hive queries without logging into the prompt?

[cite\_start]Use `hivevar` or `hiveconf` flags when executing the script[cite: 43]:

```bash
# Pass variables 'db_name' and 'load_dt'
hive -hivevar db_name='prodretaildb' -hivevar load_dt='2021-12-26' -f /home/hduser/xyz.hql

# Inside the HQL file:
SELECT * FROM ${hivevar:db_name}.tablename
WHERE loaddt=${hivevar:load_dt};
```

#### Q30. Views vs. Materialized Views

| Feature | View | Materialized View (MV) |
| :--- | :--- | :--- |
| **Data Storage** | [cite\_start]No, it is a logical table/stored query[cite: 136]. | [cite\_start]Yes, it contains pre-computed, cached data[cite: 142]. |
| **Updates** | Changes when base table data is queried (Real-time). | Must be explicitly refreshed/recomputed. |
| **Primary Use** | [cite\_start]Security (data hiding/masking), simplifying complex queries[cite: 137]. | [cite\_start]Performance for BI/dashboard queries by using cached joins/aggregations[cite: 142]. |

  * [cite\_start]**Schema Evolution on View:** If the base table adds a new column, the view will only see it if the view was created using `SELECT *`[cite: 152]. [cite\_start]If it was created using an explicit column list, the new column won't be displayed[cite: 151].

-----

### VIII. Additional Interview Questions

| Question | Answer/Solution |
| :--- | :--- |
| **Max size of `STRING` datatype?** | [cite\_start]The maximum size of the `STRING` data type is **2 GB**[cite: 12]. |
| **Pivot Array column to rows?** | Use the **`EXPLODE`** function. [cite\_start]It takes an array/map and converts its elements into separate table rows[cite: 115]. |
| **Connect to Hive?** | [cite\_start]**Beeline CLI** (Hiveserver2), Hue (UI), Ambari Hive View, SQL tools (via JDBC/ODBC), Spark.sql()[cite: 120]. |
| **Multi-line comment supported?** | [cite\_start]**No**[cite: 161]. |
| **Partition data present, but no metadata?** | **D. [cite\_start]No result are returned**[cite: 104]. Hive relies on the Metastore entry; without it, the data is invisible to the query engine. |
| **Archiving a Partition Benefit?** | [cite\_start]**D. reduces the number of files stored**[cite: 98]. It combines small files into one HAR file. |
| **Hive most suitable for?** | [cite\_start]Data warehouse applications where: 1) The data is relatively **static/incremental** [cite: 176][cite\_start], and 2) **Fast response time is not required**[cite: 176]. |