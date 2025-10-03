## Bucketed Table: Definition and Benefits

Bucketing is a technique used to **divide, group, or distribute data into a defined number of files based on one or more columns with high cardinality**.

## Advantages of Bucketing 🚀

Bucketing offers several key benefits:

  * **Improved Join Performance**: It significantly enhances the speed of join operations.
  * **Faster Filter Clause Performance**: Queries with filter clauses run more efficiently.
  * **Aids Random Sampling**: Makes it easier to perform random sampling of data.

**Example**: Columns like `customer_id` or `product_id` in a sales table are good candidates for bucketing due to their high cardinality.

## How Bucketing Works ⚙️

1.  **Fixed Number of Buckets**: Bucketing divides data into a fixed number of buckets based on the hash value of a chosen column (the **bucket key**).

    **Example**: Let's consider a `column_value` (e.g., `1, 2, 3, 4, 5, 6, 8, 9`) and `no_of_buckets` (e.g., `3`).
    The `bucket key` is calculated using the modulo operator: `mod(column_value / no_of_buckets)`.

      * $mod(1 / 3) = 1$ -\> 1st bucket
      * $mod(2 / 3) = 2$ -\> 2nd bucket
      * $mod(3 / 3) = 0$ -\> 0th bucket
      * $mod(4 / 3) = 1$ -\> 1st bucket
      * $mod(5 / 3) = 2$ -\> 2nd bucket
      * $mod(6 / 3) = 0$ -\> 0th bucket
      * $mod(8 / 3) = 2$ -\> 2nd bucket
      * $mod(9 / 3) = 0$ -\> 0th bucket

2.  **Separate Files**: Each bucket is stored as a separate file within the partition directories (if partitioning is also applied).

## Bucketing Concepts & Join Improvement

### Bucketing Concepts

Bucketing improves joins in the following ways:

  * If tables are joined **without bucketing**, the join operation typically involves reading entire datasets and performing a shuffle across the network to bring matching keys together, which can be inefficient for large datasets.
  * If tables are joined **with bucketing**, and certain conditions are met, a highly optimized join method like **Sort Merge Bucket (SMB) Join** can be used.

### Bucketing Improves Joins: Thumb Rules for SMB Join 👍

For optimal join performance using bucketing (specifically SMB join), adhere to these rules; otherwise, a normal table join will be performed:

1.  **Same Bucket Columns**: Both tables involved in the join must use the **same bucket columns**.
2.  **Same Number of Buckets**: Both tables must be created with the **same number of buckets**.

## Partitioning vs. Bucketing: A Comparison 📊

| Feature           | Partitioning                                          | Bucketing                                                 |
| :---------------- | :---------------------------------------------------- | :-------------------------------------------------------- |
| **Primary Goal** | Organizing data for efficient filtering               | Distributing data evenly for efficient joins and sampling |
| **Column Type** | Low cardinality columns (e.g., `date`, `country`)     | High or low cardinality columns                           |
| **Hierarchy** | Sub-partitions possible inside a partition            | Multiple `clustered by` columns possible; combined for modulo logic |
| **Storage** | Creates folders                                       | Creates files                                             |
| **Performance** | Improves `WHERE`/`FILTER` clause performance          | Improves `JOIN` and `WHERE`/`FILTER` clause performance   |

## Syntax for Creating a Bucketed Table 📝

```sql
CREATE [EXTERNAL] TABLE [db_name.]table_name
[(col_name data_type [COMMENT col_comment], ...)]
CLUSTERED BY (col_name data_type [COMMENT col_comment], ...) [SORTED BY (col_name [ASC|DESC], ...)]
INTO N BUCKETS;
```

## Examples of Bucketing in Action 🧑‍💻

First, let's ensure we have a database to work with:

```sql

-- Enable useful Hive settings
SET hive.cli.print.current.db = true;
SET hive.cli.print.header = true;

-- Create the database if it doesn't exist
CREATE DATABASE IF NOT EXISTS retail_analytics;
USE retail_analytics;
```

### Table with Only Buckets

```sql
-- Table with only Buckets
CREATE TABLE retail_analytics.customer_transactions_bucketed_by_id(
    id INT,
    name STRING,
    city STRING
)
CLUSTERED BY (id) SORTED BY (id) INTO 3 BUCKETS
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

INSERT INTO retail_analytics.customer_transactions_bucketed_by_id VALUES
(1,'Alice','Chennai'),
(2,'Bob','Hyderabad'),
(3,'Charlie','Mumbai'),
(4,'David','Chennai'),
(5,'Eve','Mumbai'),
(6,'Frank','Chennai'),
(7,'Grace','Chennai'),
(8,'Heidi','Mumbai'),
(9,'Ivan','Chennai');
```

To view the created bucket files:

```bash
dfs -ls /user/hive/warehouse/retail_analytics.db/customer_transactions_bucketed_by_id/*;
```

**Output**:

```
-rwxrwxr-x 1 hduser supergroup 24 2025-01-21 16:42 /user/hive/warehouse/retail_analytics.db/customer_transactions_bucketed_by_id/000000_0
-rwxrwxr-x 1 hduser supergroup 24 2025-01-21 16:42 /user/hive/warehouse/retail_analytics.db/customer_transactions_bucketed_by_id/000001_0
-rwxrwxr-x 1 hduser supergroup 24 2025-01-21 16:42 /user/hive/warehouse/retail_analytics.db/customer_transactions_bucketed_by_id/000002_0
```

Content of each bucket file:

```bash
dfs -cat /user/hive/warehouse/retail_analytics.db/customer_transactions_bucketed_by_id/000000_0;
```

**Output**:

```
3,Charlie,Mumbai
6,Frank,Chennai
9,Ivan,Chennai
```

```bash
dfs -cat /user/hive/warehouse/retail_analytics.db/customer_transactions_bucketed_by_id/000001_0;
```

**Output**:

```
1,Alice,Chennai
4,David,Chennai
7,Grace,Chennai
```

```bash
dfs -cat /user/hive/warehouse/retail_analytics.db/customer_transactions_bucketed_by_id/000002_0;
```

**Output**:

```
2,Bob,Hyderabad
5,Eve,Mumbai
8,Heidi,Mumbai
```

When querying a specific ID:

```sql
SELECT * FROM retail_analytics.customer_transactions_bucketed_by_id WHERE id=3;

EXPLAIN EXTENDED SELECT * FROM retail_analytics.customer_transactions_bucketed_by_id WHERE id=3;
```

**Explanation**:  
Out of 9 rows,  
Hive (mappers) filtered 1 file (bucket) out of 3 files (buckets) and   
performed 2 searches out of 3 rows to  
find 1 row as output because the data is sorted within the bucket.  

### Table with Both Partition & Buckets

```sql
-- Table with both Partition & Buckets
CREATE TABLE retail_analytics.customer_transactions_partitioned_and_bucketed(
    id INT,
    name STRING
)
PARTITIONED BY (city STRING)
CLUSTERED BY (id) SORTED BY (id) INTO 3 BUCKETS
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

INSERT INTO retail_analytics.customer_transactions_partitioned_and_bucketed PARTITION(city)
SELECT id,name,city FROM retail_analytics.customer_transactions_bucketed_by_id;
```

Querying with both partition and bucket conditions:

```sql
SELECT * FROM retail_analytics.customer_transactions_partitioned_and_bucketed WHERE city='Chennai' AND id=4;

EXPLAIN EXTENDED SELECT * FROM retail_analytics.customer_transactions_partitioned_and_bucketed WHERE city='Chennai' AND id=4; 
```

**Explanation**: Hive will first search the partition folder for `city='Chennai'`, and then, within that partition, it will look inside the relevant bucket to find `(4,David)`.

To list all files recursively:

```bash
dfs -ls -R /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed;
```

**Output**:

```
drwxrwxr-x - hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Chennai
-rwxrwxr-x 1 hduser supergroup 8 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Chennai/000000_0
-rwxrwxr-x 1 hduser supergroup 12 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Chennai/000001_0
-rwxrwxr-x 1 hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Chennai/000002_0
drwxrwxr-x - hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Hyderabad
-rwxrwxr-x 1 hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Hyderabad/000000_0
-rwxrwxr-x 1 hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Hyderabad/000001_0
-rwxrwxr-x 1 hduser supergroup 4 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Hyderabad/000002_0
drwxrwxr-x - hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Mumbai
-rwxrwxr-x 1 hduser supergroup 4 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Mumbai/000000_0
-rwxrwxr-x 1 hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Mumbai/000001_0
-rwxrwxr-x 1 hduser supergroup 8 2025-01-21 17:04 /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Mumbai/000002_0
```

Content of selected bucket files within partitions:

```bash
dfs -cat /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Chennai/000000_0;
```

**Output**:

```
6,Frank
9,Ivan
```

```bash
dfs -cat /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Chennai/000001_0;
```

**Output**:

```
1,Alice
4,David
7,Grace
```

```bash
dfs -cat /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Hyderabad/000002_0;
```

**Output**:

```
2,Bob
```

```bash
dfs -cat /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Mumbai/000000_0;
```

**Output**:

```
3,Charlie
```

```bash
dfs -cat /user/hive/warehouse/retail_analytics.db/customer_transactions_partitioned_and_bucketed/city=Mumbai/000002_0;
```

**Output**:

```
5,Eve
8,Heidi
```

## Interview Questions on Bucketing 🧠

### 1\. Can we modify the number of buckets after some time due to data volume increase without changing the entire dataset?

**Answer**: Not directly. You cannot simply `ALTER TABLE` to change the number of buckets and expect the existing data to be re-bucketed. You need to create a new table with the desired number of buckets and then move the data.

**Example**:

```sql
CREATE TABLE retail_analytics.customer_transactions_bucketed_by_id_tmp(
    id INT,
    name STRING,
    city STRING
)
CLUSTERED BY (id) SORTED BY (id) INTO 10 BUCKETS -- New number of buckets
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

INSERT INTO retail_analytics.customer_transactions_bucketed_by_id_tmp
SELECT id,name,city FROM retail_analytics.customer_transactions_bucketed_by_id; -- Re-bucket data

DROP TABLE retail_analytics.customer_transactions_bucketed_by_id;

ALTER TABLE retail_analytics.customer_transactions_bucketed_by_id_tmp RENAME TO customer_transactions_bucketed_by_id; -- Rename to replace original table
```

### 2\. When will skewness happen in bucketing?

**Answer**: Skewness (imbalanced load) occurs if you choose **poor bucketing columns or an inappropriate number of buckets**. This can lead to one bucket file being heavily loaded while others are underutilized, impacting the performance of data nodes. For instance, if you bucket on a column with very few distinct values, most data will end up in a few buckets.

### 3\. How should we decide the number of buckets when creating tables? Is there any criteria?

**Answer**: The decision for the number of buckets is typically based on:

1.  **Output Volume of Data**: A common guideline is that **each reducer can generate approximately 256 MB of data**. Since each bucket often corresponds to a reducer, this provides a baseline.
2.  **Column Cardinality**: Consider the cardinality of the column(s) chosen for bucketing.

**Examples**:

  * **Scenario 1**: You have 25 states' data and are creating buckets based on the state.

      * **Approximate number of maximum buckets**: Around 10-25 (matching the number of distinct states or a factor thereof). For 25 states, you might choose 25 buckets to ensure each state gets its own bucket, or a smaller number if data per state is minimal.

  * **Scenario 2**: You have 10 lakh (`1,000,000`) customer IDs with 40 GB of data, and you're creating buckets based on `custid`.

      * **Guideline**: `1 bucket = 1 reducer` = \~256 MB.

      * **Calculation**:

          * 1 GB = 1024 MB / 256 MB = 4 buckets.
          * For 40 GB: 40 GB \* 4 buckets/GB = 160 buckets.

      * **Analysis**:

          * 10 buckets (4 GB/bucket) is generally not a good idea as it leads to large bucket files.
          * 80 buckets (512 MB/bucket) is acceptable, as it allows for 80 reducers to run concurrently.
            * SET hive.exec.reducers.bytes.per.reducer=512m;   -- Reducer size
          * 160 buckets (256 MB/bucket) is often considered the best choice as it aligns with the 256 MB per reducer guideline, though running 160 reducers might be too many for some clusters, leading to overhead.
          * If each bucket contains approximately 2GB of data, this implies too few buckets and might lead to large file sizes, negating some benefits.

  * **Scenario 3**: How to calculate the required number of reducers?

    ```sql
    -- Create Table
    CREATE TABLE default.txns (
      txn_id        STRING,
      txn_date      STRING,
      cust_id       STRING,
      amount        DOUBLE,
      category      STRING,
      subcategory   STRING,
      city          STRING,
      state         STRING,
      payment_type  STRING
    )
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    STORED AS TEXTFILE;
    
    -- Load Data
    LOAD DATA LOCAL INPATH '/home/hduser/txns_1gb' 
    OVERWRITE INTO TABLE default.txns;
    
    -- Check Table Size
    dfs -du -h /user/hive/warehouse/txns;
    
    -- Reducer Configuration
    SET hive.exec.reducers.bytes.per.reducer=268435456;
    SET hive.exec.reducers.max=1009;
    
    -- What Happens
    -- Total data size ≈ 1.2 GB
    -- Reducer size = 256 MB
    -- No of reducers = total map output size / hive.exec.reducers.bytes.per.reducer
    --                 = 1.2 GB / 256 MB ≈ 5
    
    -- Example Query (EXPLAIN)
    EXPLAIN 
    SELECT state, COUNT(*), SUM(amount) 
    FROM txns 
    GROUP BY state;
    
    --Now it should calculate reducer count correctly (≈ 5 for  1.2 GB dataset).
    ```

## Bucketing Improves Join Performance: Example 🤝

To reiterate, for bucketing to significantly improve join performance, both tables must:

1.  Have the **same bucket columns**.
2.  Have the **same number of buckets**.

Let's create two similar bucketed tables for a join demonstration:

```sql
CREATE TABLE retail_analytics.orders_bucketed_by_customer_id_v1(
    customer_id INT,
    product_name STRING,
    order_city STRING
)
CLUSTERED BY (customer_id) SORTED BY (customer_id) INTO 3 BUCKETS
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

INSERT INTO retail_analytics.orders_bucketed_by_customer_id_v1 VALUES
(1,'Laptop','Chennai'),
(2,'Mouse','Hyderabad'),
(3,'Keyboard','Mumbai'),
(4,'Monitor','Chennai'),
(5,'Webcam','Hyderabad'),
(6,'Printer','Chennai'),
(7,'Headphones','Chennai'),
(8,'Speaker','Mumbai'),
(9,'Microphone','Chennai');


CREATE TABLE retail_analytics.orders_bucketed_by_customer_id_v2(
    customer_id INT,
    product_name STRING,
    order_city STRING
)
CLUSTERED BY (customer_id) SORTED BY (customer_id) INTO 3 BUCKETS
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

INSERT INTO retail_analytics.orders_bucketed_by_customer_id_v2 VALUES
(1,'External Hard Drive','Chennai'),
(2,'SSD','Hyderabad'),
(3,'USB Drive','Mumbai'),
(4,'Router','Chennai'),
(5,'Modem','Hyderabad'),
(6,'Switch','Chennai'),
(7,'Cable','Chennai'),
(8,'Adapter','Mumbai'),
(9,'Hub','Chennai');
```

Now, perform a join operation:

```sql
SELECT
    t1.customer_id AS customer_id_v1,
    t1.product_name AS product_name_v1,
    t1.order_city AS order_city_v1,
    t2.customer_id AS customer_id_v2,
    t2.product_name AS product_name_v2,
    t2.order_city AS order_city_v2
FROM
    retail_analytics.orders_bucketed_by_customer_id_v1 t1
JOIN
    retail_analytics.orders_bucketed_by_customer_id_v2 t2
ON
    t1.customer_id = t2.customer_id;
```

**Output**:

```
customer_id_v1 product_name_v1 order_city_v1 customer_id_v2 product_name_v2 order_city_v2
3              Keyboard        Mumbai        3              USB Drive       Mumbai
6              Printer         Chennai       6              Switch          Chennai
9              Microphone      Chennai       9              Hub             Chennai
1              Laptop          Chennai       1              External Hard Drive Chennai
4              Monitor         Chennai       4              Router          Chennai
7              Headphones      Chennai       7              Cable           Chennai
2              Mouse           Hyderabad     2              SSD             Hyderabad
5              Webcam          Hyderabad     5              Modem           Hyderabad
8              Speaker         Mumbai        8              Adapter         Mumbai
```

In this scenario, because `orders_bucketed_by_customer_id_v1` and `orders_bucketed_by_customer_id_v2` have the same bucket column (`customer_id`) and the same number of buckets (`3`), Hive can perform an optimized **Sort Merge Bucket Join**. This means that instead of a full shuffle, it can join corresponding buckets directly, leading to significant performance gains.

```sql
--Explain Plain Long
EXPLAIN EXTENDED SELECT
    t1.customer_id AS customer_id_v1,
    t1.product_name AS product_name_v1,
    t1.order_city AS order_city_v1,
    t2.customer_id AS customer_id_v2,
    t2.product_name AS product_name_v2,
    t2.order_city AS order_city_v2
FROM
    retail_analytics.orders_bucketed_by_customer_id_v1 t1
JOIN
    retail_analytics.orders_bucketed_by_customer_id_v2 t2
ON
    t1.customer_id = t2.customer_id;
    
--Explain Plain Short
EXPLAIN  SELECT
    t1.customer_id AS customer_id_v1,
    t1.product_name AS product_name_v1,
    t1.order_city AS order_city_v1,
    t2.customer_id AS customer_id_v2,
    t2.product_name AS product_name_v2,
    t2.order_city AS order_city_v2
FROM
    retail_analytics.orders_bucketed_by_customer_id_v1 t1
JOIN
    retail_analytics.orders_bucketed_by_customer_id_v2 t2
ON
    t1.customer_id = t2.customer_id;
    
  
--Explain Plain long   
EXPLAIN EXTENDED SELECT
    t1.customer_id AS customer_id_v1,
    t1.product_name AS product_name_v1,
    t1.order_city AS order_city_v1,
    t2.customer_id AS customer_id_v2,
    t2.product_name AS product_name_v2,
    t2.order_city AS order_city_v2
FROM
    retail_analytics.orders_bucketed_by_customer_id_v1 t1
JOIN
    retail_analytics.orders_bucketed_by_customer_id_v2 t2
ON
    t1.customer_id = t2.customer_id
WHERE t1.customer_id= 1;
```

---

# Step-by-step: understand a Hive join plan

## 0) Quick concept recap

* **MapJoin (broadcast/map-side join)** = one table is small → broadcast to mappers → **no reducers**. Fast for small+big joins.
* **Shuffle (common) join** = both sides are shuffled by join key to reducers → **reducers do the join**. Used for large tables.
* **Bucketed/bucket map join** = both tables are bucketed on same key (and bucket counts align) → can avoid full shuffle or do bucket-wise joins.

---

## 1) Start with table metadata (always)

Commands:

```sql
DESCRIBE FORMATTED db.tbl;
SHOW CREATE TABLE db.tbl;
```

What to check:

* File format (TEXT/ORC/Parquet — columnar formats change size & shuffle cost).
* Partitioning and bucketing (are they *bucketed by* the join key?).
* File count & sizes (many small files can create many splits).

---

## 2) Compute & refresh statistics (very important)

Commands:

```sql
ANALYZE TABLE db.tbl COMPUTE STATISTICS;
ANALYZE TABLE db.tbl COMPUTE STATISTICS FOR COLUMNS customer_id;
```

Why:

* Optimizer uses stats to decide map vs shuffle, and whether mapjoin fits memory. Missing stats → unexpected plans.

---

## 3) Get a readable plan: `EXPLAIN FORMATTED` / `EXPLAIN EXTENDED`

Commands:

```sql
EXPLAIN FORMATTED
SELECT ... FROM t1 JOIN t2 ON t1.k = t2.k;
```

or

```sql
EXPLAIN EXTENDED SELECT ...;
```

What to look for in the plan:

* `TableScan` (TS) for each table — shows columns read and filters.
* `ReduceSink` (RS) operators — **presence of RS = shuffle on that key** (i.e., a shuffle join).
* `MapJoin` or `Map Join Operator` — indicates a map-side (broadcast) join.
* `Join Operator` block — shows join type (INNER/LEFT/RIGHT) and condition mapping (e.g., `Inner Join 0 to 1`).
* Any `Bucket Map Join` lines — indicates bucketed join optimization.

---

## 4) How to *read* the operator tree (annotated example)

Example snippet (simplified):

```
Map Reduce
  Map Operator Tree:
    TableScan (t1)
    Select
    ReduceSink  <-- (RS: key = customer_id)
  Reduce Operator Tree:
    Join Operator  <-- (join on customer_id)
    Select
    FileSink
```

Interpretation:

* RS exists → mappers output keyed records, Hadoop shuffles by `customer_id`, reducers run the join → **shuffle (reduce) join**.
* If you see `MapJoinOperator` or logs mentioning a **HashTable** uploaded to distributed cache → **map join**.

---

## 5) Run the query (not only EXPLAIN) and capture full logs

From shell (capture everything):

```bash
hive -f my_query.sql > hive_full_log.txt 2>&1
# or
hive -e "SELECT ... JOIN ..." 2>&1 | tee hive_full_log.txt
```

Search logs for join indicators:

```bash
grep -i "Map Join" hive_full_log.txt
grep -i "Converting join" hive_full_log.txt
grep -i "Bucket Map Join" hive_full_log.txt
grep -i "HashTable-Stage" hive_full_log.txt   # indicates hash table created for map join
grep -i "Number of reduce tasks" hive_full_log.txt
```

What you'll see:

* `Number of reduce tasks is set to X` (when a reduce phase exists);
* `Uploading HashTable-Stage-3...` or `Add 1 archive to distributed cache` → map join;
* `Using bucket map join` / `Converting join to map join` → bucket/map join behaviors.

---

## 6) Check YARN / JobHistory UI for actual runtime behavior

Open the tracking URL from logs (something like `http://<RM>:8088/proxy/application_xxx/`) and look for:

* Number of map and reduce tasks.
* Shuffle bytes (how much data moved).
* Task durations and counters (HDFS read/write).
  This confirms what actually ran (EXPLAIN is planner view; YARN shows runtime).

---

## 7) Quick rules-of-thumb to identify join type

* **If plan contains `ReduceSink` and `Join` in reduce tree → Shuffle join**.
* **If plan or logs show "HashTable-Stage" or `MapJoinOperator` or "Add archive file to distributed cache" → Map join**.
* **If plan mentions bucket map join / sorted merge → bucketed join optimized**.

---

## 8) Settings that influence join choice (useful to know)

Useful knobs:

```sql
-- allow conversion of joins to map-joins when one side is small
SET hive.auto.convert.join=true;

-- threshold for small table (value in bytes)
SET hive.mapjoin.smalltable.filesize=10000000;  

-- enable bucket map join
SET hive.optimize.bucketmapjoin=true;
SET hive.enforce.bucketing=true;
SET hive.enforce.sorting=true;

-- disable cartesian safety check (careful)
SET hive.strict.checks.cartesian.product=false;
```

(Replace sizes appropriately.) After changing, re-run `ANALYZE` & `EXPLAIN` to see effect.

---

## 9) How to force a particular join type (for testing)

* **Force map join (hint):**

```sql
SELECT /*+ MAPJOIN(t2) */ ... FROM t1 JOIN t2 ON ...
```

* **Disable map joins** (force shuffle):

```sql
SET hive.auto.convert.join=false;
```

Use these to test performance & behavior.

---

## 10) Troubleshooting checklist (if plan is unexpected)

* Are data types of join keys identical? Mismatches can prevent optimizations.
* Are statistics up-to-date? Run `ANALYZE TABLE`.
* Are buckets and bucket counts aligned (if you expect bucket map join)?
* Are table sizes known to the optimizer (small table must be below threshold to choose map join)?
* Any filters pushed down? (predicate pushdown reduces data before join)
* If you see `Cartesian products are disabled...` check whether optimizer misunderstood join keys — double-check SQL and types.

---

## 11) Useful grep keywords to scan logs quickly

```bash
grep -Ei "mapjoin|map join|mapjoinoperator|hashtable|distributed cache|bucket map join|reduce sink|reduce tasks|Number of reduce tasks|Converting join" hive_full_log.txt
```

---

## 12) Minimal workflow you can copy-paste (practical)

1. Check table structure/stats:

```sql
DESCRIBE FORMATTED db.t1;
ANALYZE TABLE db.t1 COMPUTE STATISTICS;
ANALYZE TABLE db.t1 COMPUTE STATISTICS FOR COLUMNS customer_id;
```

2. Get planner view:

```sql
EXPLAIN FORMATTED SELECT ... FROM t1 JOIN t2 ON t1.k = t2.k;
```

3. Run and capture runtime logs:

```bash
hive -e "SELECT ... FROM t1 JOIN t2 ON ..." > run.log 2>&1
grep -i "HashTable" run.log || grep -i "Number of reduce tasks" run.log
```

4. Inspect YARN tracking URL from the logs.

---

## Annotated mini-example (what to look for in `EXPLAIN`)

Imagine this part in `EXPLAIN FORMATTED`:

```
Map Operator Tree:
  TableScan t1
  Select
  ReduceSink  <-- key: customer_id
Reduce Operator Tree:
  Join Operator  <-- keys: t1.customer_id == t2.customer_id
```

* `ReduceSink` present → shuffle.
* `Join Operator` in the reduce tree → reducers perform join.

If instead you saw:

```
Map Operator Tree:
  TableScan t2
  MapJoinOperator  <-- t2 loaded into memory
```

→ it's a **map join**.

---

## TL;DR — Quick 6-step checklist for a dev

1. `DESCRIBE FORMATTED` + `ANALYZE TABLE` (get stats)
2. `EXPLAIN FORMATTED` (read TS / RS / JOIN blocks)
3. Run query and save logs (`hive > log.txt 2>&1`)
4. Grep log for `HashTable`, `MapJoin`, `ReduceSink`, `Number of reduce tasks`
5. Open YARN tracking URL for actual maps/reduces & shuffle metrics
6. Tweak `hive.auto.convert.join`, `hive.mapjoin.smalltable.filesize`, bucket settings and re-run

---

