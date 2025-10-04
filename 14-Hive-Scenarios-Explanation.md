
## 🚀 Hive Real-time Interview Questions & Scenarios

-----

### I. Core Hive Query Execution and Configuration

| Question | Explanation & Properties |
| :--- | :--- |
| **Why does a simple `SELECT *` query not run a MapReduce/Tez/Spark job in Hive?** | This is due to the **Hive Fetch Task Optimization**. The `hive.fetch.task.conversion` property allows Hive to skip the distributed execution engine for simple queries like `SELECT`, `FILTER`, or `LIMIT`, significantly reducing latency. |
| **How to see the default value set for a config?** | Use the `SET` command: `SET hive.fetch.task.conversion;` |
| **How to change configs (session/job level)?** | Use `SET <property>=<value>;` in the query session. For job/cluster level changes, modify `hive-site.xml` or use cluster management tools. |

-----

### II. Advanced HiveQL and Logic

#### Q1. Multi-Column Subquery in `WHERE` Clause

Older Hive versions or certain syntaxes do not support multi-column subqueries using `IN`.

  * **❌ Doesn't Work:** `WHERE (A.id, A.name) IN (SELECT Id, name FROM table B)`
  * **✅ Solution 1: Use `CONCAT`**
    ```sql
    SELECT * FROM tableA WHERE CONCAT(A.id, A.name) IN (SELECT CONCAT(Id, name) FROM tableB);
    ```
  * **✅ Solution 2: Use an `INNER JOIN` (Recommended)**
    ```sql
    SELECT A.* FROM tableA A INNER JOIN tableB B ON A.id = B.id AND A.name = B.name;
    ```

#### Q2. `MINUS`/`EXCEPT` Equivalent

Hive does not support `MINUS` or `EXCEPT`.

  * **✅ Solution: Use `LEFT JOIN` and check for `NULL`**
    ```sql
    -- Find all records in A that do NOT exist in B
    SELECT A.*
    FROM tableA A
    LEFT JOIN tableB B
      ON (A.x = B.x AND A.y = B.y)
    WHERE B.x IS NULL;
    ```

#### Q3. Subqueries in `SELECT` Clause

Hive traditionally does not support correlated subqueries in the `SELECT` list.

  * **✅ Solution: Use an `INNER JOIN` (Cross Join) on the aggregate subquery**
    ```sql
    SELECT C.customer_num, M.max_ship_chg
    FROM customer C
    INNER JOIN (SELECT MAX(ship_charge) AS max_ship_chg FROM orders) M ON 1=1;
    ```

#### Q4. Inner Join behavior with `NULL` keys

When performing an `INNER JOIN` on a column with `NULL` values, the `NULL` values **will not match**. SQL treats `NULL` as an unknown value, and thus `NULL = NULL` is false.

#### Q5. Generating a Surrogate Key / Sequence Number

  * **Window Function:** `ROW_NUMBER() OVER (PARTITION BY col1 ORDER BY col2)`
  * **Unique ID:** `UUID()` function.

-----

### III. Data Ingestion and Table Properties

| Question | Explanation & Properties |
| :--- | :--- |
| **How to skip header/footer rows?** | Use `TBLPROPERTIES`: `TBLPROPERTIES("skip.header.line.count"="2");` or `TBLPROPERTIES("skip.footer.line.count"="1");` |
| **How to read fixed-width data?** | 1. Use **`RegexSerDe`** with a regular expression pattern. 2. Load data as a single `STRING` column and use **`SUBSTR`** and **`TRIM`** to parse fields. |
| **Prevent accidental duplication on load?** | Use the **Immutable Table** property. `TBLPROPERTIES ("immutable"="true");`. This blocks successive `INSERT INTO` operations (but not `INSERT OVERWRITE`). |
| **Change `EXTERNAL` to `MANAGED`?** | `ALTER TABLE customers SET TBLPROPERTIES('EXTERNAL'='FALSE');` (and vice-versa). |
| **Load data without copying to HDFS first?**| Use **`LOAD DATA LOCAL INPATH`**. Example: `LOAD DATA LOCAL INPATH '/local/file/path' INTO TABLE tbl_name;` |
| **Is record-level DML (`UPDATE`/`DELETE`) supported?** | **No, not by default**. It is only supported if the table is enabled for **ACID transactions** (Atomicity, Consistency, Isolation, and Durability) and typically stored in **ORC** format with bucketing. |
| **What are the constraints in Hive?** | Constraints (`PRIMARY KEY`, `NOT NULL`) are supported for **representation purposes only** (metadata), not for enforcement, to maintain performance. |

-----

### IV. Performance and Optimization

#### Q6. Handling Data Skew (The Missing Point)

Data skew is when one key has significantly more data than others, causing one reducer to become a bottleneck. Hive can mitigate this using a **Skew Join Optimization**.

  * **Enable Skew Detection:** `SET hive.skewjoin.exist = true;`
  * **Enable Skew Join Optimization:** `SET hive.optimize.skewjoin = true;`
  * **Mechanism:** When a highly skewed key is detected, Hive splits the processing for that large key into multiple smaller tasks (by adding a random suffix to the skewed key), allowing the join to be completed by multiple reducers in parallel.

#### Q7. Resolving `OutOfMemoryError: Java heap space`

Increase the memory allocated to the tasks:

```sql
SET mapreduce.map.memory.mb=4096m;      
SET mapreduce.map.java.opts=-Xmx3686m;  -- Must be slightly less than memory.mb
SET mapreduce.reduce.memory.mb=4096m;   
SET mapreduce.reduce.java.opts=-Xmx3686m;
```

#### Q8. Fixing "Small Files Issue" (The Missing Point)

Small files create excessive overhead for the NameNode and inefficient MapReduce jobs.

1.  **Merging:** Use `INSERT OVERWRITE` with merge properties to force a shuffle that combines files:
    ```sql
    SET hive.merge.mapfiles=true;
    SET hive.merge.mapredfiles=true;
    SET hive.merge.smallfiles.avgsize=104857600; -- Target average size (e.g., 100MB)
    INSERT OVERWRITE TABLE table1 SELECT * FROM table1;
    ```
2.  **Archiving (HAR):** Use Hadoop Archives to logically group small files.
      * **Benefit:** It **reduces the number of files stored** (Option D).
3.  **Compaction:** For ACID tables, enable Major/Minor compactions to combine small delta files.

#### Q9. Fixing a "Vertex Error"

If a query fails with a Tez/Spark vertex error, a common fix is to revert to the more stable MapReduce engine for that session:

```sql
SET hive.execution.engine=mr;
```

-----

### V. Partitioning and Data Management

| Question | Explanation & Properties |
| :--- | :--- |
| **Manage data in multiple HDFS subdirectories?** | Enable recursive directory listing: `SET mapred.input.dir.recursive=true;` and `SET hive.mapred.supports.subdirectories=true;` |
| **Discover partitions created externally?** | Update the Metastore with the HDFS structure: **`MSCK REPAIR TABLE <table_name>;`** |
| **Dynamic Partition Limits (The Missing Point)** | To prevent too many partitions, Hive has limits: **Per Node:** `SET hive.exec.max.dynamic.partitions.pernode = 100` (Default). **Total:** `SET hive.exec.max.dynamic.partitions = 1000` (Default). |
| **Automatic Partition Retention/Purging?** | `ALTER TABLE customer SET TBLPROPERTIES ('partition.retention.period'='365d');` |
| **Permanently drop table and data (skip trash)?** | `DROP TABLE [IF EXISTS] table_name PURGE;` |
| **Effect of Renaming an External Table?** | **No**, the underlying HDFS location is **not** changed. |
| **Effect of Changing Partition Location?** | **No**, the data is **not** automatically moved. You must manually move the data in HDFS. |
| **Data present, but no partition metadata?** | **No results are returned**. Hive queries the Metastore first; without the partition entry, the data is invisible. |

-----

### VI. Data Lifecycle and Advanced Features

| Concept | Description/Feature |
| :--- | :--- |
| **Deduplication (High Volume)** | Use **`ROW_NUMBER() OVER(PARTITION BY... ORDER BY...)`** to assign a rank and keep only the latest record (`rnk = 1`). |
| **Data Management Layers** | **Raw/Transient** (Ingestion) $\rightarrow$ **Curated Layer** (**Hive, Presto**) $\rightarrow$ **Discovery/Presentation** (**NoSQL, HBase**) for final use. |
| **Materialized Views** | Pre-computed, cached tables that the CBO automatically uses to speed up aggregate/join queries (e.g., for BI dashboards). |
| **Transaction Processing** | Modern Hive supports **ACID** transactions. ACID tables no longer strictly require bucketing or ORC format for basic DML. |
| **Deprecated Interfaces** | **Hive CLI** (replaced by **Beeline**), **MapReduce** engine (replaced by **Tez**), and **SQL Standard Authorization** (replaced by **Ranger**). |
| **Parameterized Queries** | Use the `-hivevar` flag when running scripts: `hive -hivevar db_name='prod' -f /path/to/script.hql` |