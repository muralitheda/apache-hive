
# Hive Realtime Interview Questions 2023


## 1. Difference between Internal and External tables in Hive

* **Internal Table**:

  * Data is stored in Hive warehouse directory.
  * Dropping the table deletes both data and metadata.

* **External Table**:

  * Data is stored externally (HDFS location provided).
  * Dropping the table deletes only metadata, not data.

---

## 2. What is Partitioning in Hive?

Partitioning helps in dividing large datasets into smaller, manageable parts.

* **Static Partitioning** – partitions are manually specified.
* **Dynamic Partitioning** – partitions are created dynamically based on column values.

```sql
create table sales (id int, name string)
partitioned by (country string, year string)
row format delimited fields terminated by ',';
```

---

## 3. Difference between Partitioning and Bucketing

* **Partitioning** divides table data into directories.
* **Bucketing** distributes data into fixed number of files based on hash of a column.

```sql
create table emp_bucketed(id int, name string)
clustered by (id) into 4 buckets
row format delimited fields terminated by ',';
```

---

## 4. Difference between Managed and External tables

* Managed = Hive controls both data + metadata.
* External = Only metadata controlled by Hive. Data remains in external location.

---

## 5. What is Metastore in Hive?

* Central repository that stores Hive metadata.
* Can use **Derby (default)** or **MySQL/Oracle/Postgres**.

---

## 6. Explain Different File Formats Supported by Hive

* **TEXTFILE**
* **SEQUENCEFILE**
* **ORC (Optimized Row Columnar)**
* **PARQUET**
* **AVRO**

Example:

```sql
create table employee_orc (id int, name string)
stored as orc;
```

---

## 7. What are Hive SerDes?

SerDe = **Serializer/Deserializer**

* Used to read/write different data formats.
* Examples: `RegexSerDe`, `OpenCSVSerde`, `JsonSerDe`.

---

## 8. How to Optimize Hive Queries?

* Use **ORC/Parquet** formats.
* Enable **vectorization**.
* Use **partitioning & bucketing**.
* Apply **map-side joins**.
* Tune **Tez execution engine**.

---

## 9. Difference between Hive and RDBMS

| Feature      | Hive                           | RDBMS        |
| ------------ | ------------------------------ | ------------ |
| Storage      | HDFS                           | Disk         |
| Schema on    | Read                           | Write        |
| Transactions | Limited (ACID since Hive 0.14) | Full support |
| Latency      | High (batch processing)        | Low (OLTP)   |

---

## 10. Can Hive Perform Updates and Deletes?

* Prior to Hive 0.14 → Not supported.
* Since Hive 0.14 → **ACID transactions** (with ORC + transactional property).

```sql
update employee set salary=50000 where id=101;
delete from employee where id=105;
```

---

## 11. Difference between Hive Internal and External Table with Example

```sql
-- Internal Table
create table emp_internal(id int, name string)
row format delimited fields terminated by ',';

-- External Table
create external table emp_external(id int, name string)
row format delimited fields terminated by ','
location '/user/hive/warehouse/emp_external';
```

* Dropping `emp_internal` → deletes data + metadata.
* Dropping `emp_external` → deletes only metadata, data remains in HDFS.

---

## 12. Difference between Local and Distributed Mode in Hive

* **Local Mode** → Queries run in a single JVM (for testing/small data).
* **Distributed Mode** → Queries run on Hadoop cluster (production use).

---

## 13. What is Dynamic Partitioning in Hive?

Enables automatic creation of partitions at runtime.

```sql
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table sales partition(country, year)
select id, name, country, year from sales_staging;
```

---

## 14. When to use Partitioning vs Bucketing?

* Use **partitioning** when filter column has **low cardinality** (few distinct values).
* Use **bucketing** when filter column has **high cardinality**.

---

## 15. What is Vectorization in Hive?

* Executes multiple rows together (batch processing).
* Improves performance on ORC/Parquet formats.

Enable with:

```sql
set hive.vectorized.execution.enabled = true;
set hive.vectorized.execution.reduce.enabled = true;
```

---

## 16. Difference between Hive on MR, Hive on Tez, Hive on Spark

* **Hive on MR** → Traditional, slow.
* **Hive on Tez** → DAG-based execution, faster.
* **Hive on Spark** → Uses Spark execution engine, highly optimized.

---

## 17. Difference between ORDER BY, SORT BY, DISTRIBUTE BY, CLUSTER BY

* **ORDER BY** → Global ordering, single reducer.
* **SORT BY** → Sort within reducer, faster.
* **DISTRIBUTE BY** → Controls partitioning.
* **CLUSTER BY** → Same as `distribute by` + `sort by`.

---

## 18. What is Indexing in Hive?

* Used to improve query performance.
* Creates metadata for faster lookups.

```sql
create index emp_index on table employee(id)
as 'compact' with deferred rebuild;
```

---

## 19. Difference between Hive and Spark SQL

| Feature          | Hive SQL         | Spark SQL            |
| ---------------- | ---------------- | -------------------- |
| Execution Engine | MR/Tez/Spark     | Spark Engine         |
| Latency          | Higher           | Lower                |
| ACID Support     | Yes (since 0.14) | Limited (Delta/Lake) |

---

## 20. How to Handle Skewed Data in Hive?

* Use **skewed tables**.
* Use **map-side joins**.
* Apply **salting technique**.

```sql
create table sales_skewed(id int, name string)
skewed by (id) on (1,2,3) stored as directories;
```

---


## 21. What is Map-side Join in Hive?

* Avoids reducer stage by performing join in the mapper itself.
* Works when one of the tables is **small enough to fit in memory**.

```sql
set hive.auto.convert.join=true;
```

---

## 22. What is SMB Join (Sort-Merge-Bucket Join) in Hive?

* Efficient join when both tables are **bucketed and sorted** on the join key.
* No shuffling required.

```sql
select /*+ mapjoin(emp) */
e.id, d.dept_name
from emp e
join dept d
on e.dept_id = d.id;
```

---

## 23. What is Skew Join in Hive?

* Handles **data skew** where some keys have very high frequency.
* Hive redistributes skewed keys into separate tasks.

```sql
set hive.optimize.skewjoin=true;
```

---

## 24. Difference between Static and Dynamic Partition

* **Static** → Partitions specified manually.
* **Dynamic** → Partitions created automatically from data.

---

## 25. What is the role of Hive Driver?

* Compiles query.
* Generates execution plan.
* Submits job to execution engine.
* Returns results to client.

---

## 26. What is the role of Hive Compiler?

* Parses SQL query.
* Converts to **logical plan**.
* Optimizes with **rule-based optimizer**.

---

## 27. What is the role of Hive Metastore?

* Stores table metadata: schema, location, partitions.
* External RDBMS (MySQL/Derby) used to persist data.

---

## 28. What is the difference between External Table and View?

* **External Table** → Physical table, stores metadata.
* **View** → Virtual table, stores only query definition.

```sql
create view emp_view as
select id, name from employee where dept_id=10;
```

---

## 29. What is Explain Plan in Hive?

* Used to check query execution plan.

```sql
explain select * from employee where dept_id=10;
```

---

## 30. What is Difference between Subquery and CTE in Hive?

* **Subquery** → Nested query inside main query.
* **CTE (WITH clause)** → Temporary named result set reused multiple times.

```sql
with dept_count as (
   select dept_id, count(*) as cnt
   from employee
   group by dept_id
)
select * from dept_count where cnt > 10;
```

---


## 31. How does Hive handle Schema on Read?

* Hive does not enforce schema at data load.
* Schema applied only at **query time**.
* Makes Hive flexible for semi-structured/unstructured data.

---

## 32. What are Different Modes in Hive?

* **Local Mode** → Runs queries in a single JVM, for small datasets.
* **Distributed Mode** → Runs queries on Hadoop cluster.

---

## 33. Difference between Hive CLI, Beeline, and Hiveserver2

* **Hive CLI** – Deprecated, directly connects to metastore.
* **Beeline** – JDBC client for HiveServer2.
* **HiveServer2** – Multi-client support, authentication, concurrency.

---

## 34. What is the Significance of `hive.exec.reducers.bytes.per.reducer`?

* Determines reducer count based on input size.
* Default: 256 MB per reducer.
* Example: If input = 1 GB, reducers = 1GB/256MB = 4.

---

## 35. What is `mapreduce.input.fileinputformat.split.maxsize`?

* Controls maximum split size for mapper.
* Helps in parallelism tuning.

---

## 36. What are ACID Properties in Hive?

* **Atomicity, Consistency, Isolation, Durability**.
* Supported for ORC transactional tables since Hive 0.14.

```sql
create table emp_acid(id int, name string)
stored as orc
tblproperties ("transactional"="true");
```

---

## 37. What are the Limitations of Hive?

* High latency (batch processing).
* Not suitable for OLTP.
* Limited transaction support.
* Complex query optimizations slower than RDBMS.

---

## 38. Union vs Union All in Hive

* **UNION** → Removes duplicates.
* **UNION ALL** → Retains duplicates.

---

## 39. How to Pick Max Salary Employee from Each Department?

```sql
select dept_id, max(salary)
from employee
group by dept_id;
```

---

## 40. Where do you see Pivot Functionality in Hive?

* Hive doesn’t have native pivot.
* Achieved using `lateral view`, `explode`, or Spark SQL pivot.

```sql
select dept, concat_ws(',', collect_list(emp_name))
from employee
group by dept;
```

---


## 41. Difference between `count(*)` and `count(1)` in Hive

* **count(*)** → Counts all rows including NULLs.
* **count(1)** → Also counts all rows, but slightly faster since it doesn’t check each column.

---

## 42. First Value and Last Value Functions in Hive

* **first_value()** → Returns the first value in a window.
* **last_value()** → Returns the last value in a window.

```sql
select emp_id,
       first_value(salary) over(partition by dept_id order by salary) as min_sal,
       last_value(salary) over(partition by dept_id order by salary) as max_sal
from employee;
```

---

## 43. Lead and Lag in Hive

* **lead()** → Fetches next row’s value.
* **lag()** → Fetches previous row’s value.

```sql
select emp_id, salary,
       lag(salary,1) over(partition by dept_id order by salary) as prev_sal,
       lead(salary,1) over(partition by dept_id order by salary) as next_sal
from employee;
```

---

## 44. Questions Related to Window Functions

* `row_number()`
* `rank()`
* `dense_rank()`
* `sum()`, `avg()`, `count()`
* `lead()`, `lag()`
* `first_value()`, `last_value()`

---

## 45. Benefits of WITH Clause (CTE)

* Makes queries **readable**.
* Avoids repeated subqueries.
* Acts like a temporary table within a query.

```sql
with sales_cte as (
   select cust_id, sum(amount) as total_sales
   from sales
   group by cust_id
)
select * from sales_cte where total_sales > 5000;
```

---

## 46. How to Fetch Latest Record?

* Using **row_number()**:

```sql
select * from (
   select row_number() over(partition by cust_id order by txn_date desc) as rn, t.*
   from transactions t
) tmp
where rn = 1;
```

* Using **max()**:

```sql
select * from transactions
where txn_date = (select max(txn_date) from transactions);
```

---

## 47. Join Scenario with NULLs

**Table A** → (Null, James, Null)
**Table B** → (James, Null, Robert)

* Inner join will exclude NULL keys.

```sql
create table t1(id int, name string);
create table t2(id int, name string);

insert into t1 values(1,'James'),(2,null),(3,'Robert');
insert into t2 values(1,'James'),(2,null),(3,'Smith');

select a.*, b.* from t1 a inner join t2 b on a.name=b.name;
```

---

## 48. Changing Date Format in Hive

```sql
select from_unixtime(unix_timestamp('12-31-2023','MM-dd-yyyy'),'yyyy-MM-dd');
```

---

## 49. Can Hive Use Temp Tables?

* Yes, using **temporary tables** within session.

```sql
create temporary table temp_sales as
select * from sales where amount > 5000;
```

---

## 50. How to Handle Slowly Changing Dimensions (SCD) in Hive?

* Implement using **ORC + ACID transactions**.
* Type 2 (history preservation):

```sql
insert into customer_dim
select cust_id, name, start_date, end_date, current_flag
from customer_dim
where current_flag='Y';
```

---

## 51. How to Improve Performance of Hive Queries?

* Use **ORC/Parquet** file formats.
* Apply **vectorization**.
* Use **partitioning & bucketing**.
* Enable **map-side joins**.
* Limit use of `order by` (use `sort by` or `distribute by`).
* Tune parameters (`hive.exec.reducers.bytes.per.reducer`, `mapreduce.input.fileinputformat.split.maxsize`).

---

## 52. Can Hive Handle Unstructured Data?

* Hive is best for **structured & semi-structured** data.
* With **custom SerDe**, Hive can handle JSON, XML, logs.

---

## 53. Difference between Schema on Write vs Schema on Read

* **Schema on Write** (RDBMS) → Schema applied before loading data.
* **Schema on Read** (Hive) → Schema applied at query time.

---

## 54. How to Find Top N Records in Hive?

```sql
select * from employee order by salary desc limit 5;
```

---

## 55. Difference between Row_Number, Rank, and Dense_Rank

* **row_number()** → Sequential numbering.
* **rank()** → Same rank for ties, skips next numbers.
* **dense_rank()** → Same rank for ties, no skipping.

---

## 56. What is the Difference between Collect_Set and Collect_List?

* **collect_set()** → Returns unique elements.
* **collect_list()** → Returns all elements (including duplicates).

```sql
select collect_set(dept_id), collect_list(dept_id)
from employee;
```

---

## 57. Difference between Hive and Pig

| Feature  | Hive (SQL-like)      | Pig (Scripting)        |
| -------- | -------------------- | ---------------------- |
| Language | HiveQL (SQL style)   | Pig Latin (procedural) |
| Audience | Analysts             | Programmers            |
| Ease     | Easy (SQL knowledge) | Steeper learning       |

---

## 58. What are Complex Types in Hive?

* **ARRAY**
* **MAP**
* **STRUCT**

Example:

```sql
create table customer(
   id int,
   name string,
   address struct<street:string, city:string, state:string>,
   phone array<string>,
   preferences map<string,string>
)
row format delimited fields terminated by ',';
```

---

## 59. How to Explode Array Data in Hive?

```sql
select cust_id, phone_number
from customer lateral view explode(phone) p as phone_number;
```

---

## 60. How to Parse JSON Data in Hive?

* Use **JsonSerDe**.

```sql
create table json_table(
   id int,
   data string
)
row format serde 'org.apache.hive.hcatalog.data.JsonSerDe';
```

---


## 61. How to Read CSV Files in Hive?

```sql
create external table emp_csv(
   id int,
   name string,
   salary double
)
row format serde 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
with serdeproperties(
   "separatorChar" = ",",
   "quoteChar"     = "\""
)
location '/user/hive/emp_csv';
```

---

## 62. How to Handle Null Values in Hive?

* Replace with default value:

```sql
select nvl(salary,0) from employee;
```

* Filter nulls:

```sql
select * from employee where salary is not null;
```

---

## 63. How to Find Duplicate Records in Hive?

```sql
select id, count(*)
from employee
group by id
having count(*) > 1;
```

---

## 64. Difference between HiveQL and SQL

* **HiveQL** → Schema on Read, batch queries on HDFS.
* **SQL** → Schema on Write, OLTP, real-time.

---

## 65. Can We Join Tables from Different Databases in Hive?

Yes, by prefixing the schema name:

```sql
select a.id, b.name
from db1.emp a
join db2.dept b
on a.dept_id = b.id;
```

---

## 66. What is the Use of `explode` in Hive?

Converts array/map into multiple rows.

```sql
select id, phone
from customer lateral view explode(phone) t as phone;
```

---

## 67. Difference between `explode` and `posexplode`

* **explode** → Breaks array into rows.
* **posexplode** → Breaks array + returns position of each element.

```sql
select posexplode(array('a','b','c'));
```

---

## 68. How to Create Surrogate Key in Hive?

```sql
select row_number() over(order by id) as surrogate_key, *
from employee;
```

---

## 69. Difference between Hive Internal Join and Left Semi Join

* **Inner Join** → Returns matching rows with columns from both tables.
* **Left Semi Join** → Returns rows from left table if at least one match exists (no columns from right).

```sql
select e.*
from employee e
left semi join dept d
on e.dept_id=d.id;
```

---

## 70. How to Load Data into Hive Table?

```sql
load data local inpath '/home/user/emp.txt'
into table employee;

load data inpath '/user/hadoop/emp.txt'
overwrite into table employee;
```

---

## 71. Difference between Internal Table and External Table (Data Management)

* **Internal Table** → Data managed by Hive, deleted on `drop table`.
* **External Table** → Only metadata deleted on `drop table`, data remains in HDFS.

---

## 72. Can We Delete Data from Hive Table?

* Yes, if **ACID transactions** are enabled.

```sql
delete from employee where id=101;
```

---

## 73. How to Update Data in Hive?

```sql
update employee
set salary = 50000
where id = 101;
```

Requires:

* ORC file format.
* `"transactional"="true"` table property.

---

## 74. What is Skewed Data and How to Handle It?

* When some keys have **high frequency** → reducers overloaded.
* Solutions:

  * Use **skewed tables**.
  * Apply **salting technique**.
  * Enable skew join optimization.

---

## 75. How to Find Second Highest Salary in Hive?

```sql
select max(salary) 
from employee
where salary < (select max(salary) from employee);
```

Or using `row_number()`:

```sql
select salary from (
  select salary, row_number() over(order by salary desc) as rn
  from employee
) t where rn=2;
```

---

## 76. What are Different Join Types in Hive?

* Inner Join
* Left/Right Outer Join
* Full Outer Join
* Cross Join
* Left Semi Join
* Anti Join

---

## 77. Difference between `mapjoin` and Regular Join

* **Regular Join** → Both tables go through shuffle.
* **Map Join** → Small table loaded into memory, avoids shuffle.

```sql
set hive.auto.convert.join=true;
```

---

## 78. How to Enable Dynamic Partitioning?

```sql
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
```

---

## 79. How to Create Bucketed Tables?

```sql
create table emp_bucketed(
  id int,
  name string
)
clustered by (id) into 4 buckets
row format delimited fields terminated by ',';
```

---

## 80. Difference between ORC and Parquet

* **ORC** → Better for Hive (row + column compression, vectorization).
* **Parquet** → Widely used across Hadoop ecosystem (Hive, Spark, Impala).
* Both are **columnar storage formats**.

---

### **81. What is the difference between `EXTERNAL` and `MANAGED` tables in Hive?**

**Ans:**

| Feature        | Managed Table                  | External Table                       |
| -------------- | ------------------------------ | ------------------------------------ |
| Data Ownership | Hive owns the data             | User owns the data                   |
| Data Deletion  | Dropping table deletes data    | Dropping table only deletes metadata |
| Location       | Hive default warehouse         | User-defined location                |
| Use Case       | Internal processing, temp data | Shared data across tools             |

---

### **82. What is the difference between `ORC` and `Parquet` file formats?**

**Ans:**

| Feature        | ORC (Optimized Row Columnar)  | Parquet                    |
| -------------- | ----------------------------- | -------------------------- |
| Compression    | Zlib (high compression)       | Snappy (faster read/write) |
| Storage Format | Row + Columnar hybrid         | Columnar                   |
| Splittable     | Yes                           | Yes                        |
| Use Case       | Hive (high performance, ACID) | Spark, Presto, Hadoop      |

---

### **83. What is Hive ACID?**

**Ans:**

* ACID (Atomicity, Consistency, Isolation, Durability) allows **insert, update, delete** operations in Hive tables.
* Requires:

  * `transactional=true` on table
  * ORC format
  * Hive transactional support enabled (`hive.txn.manager`)

---

### **84. What are the limitations of ACID tables in Hive?**

**Ans:**

* Only supports **ORC file format**.
* Requires **Hive 0.14+**.
* Cannot **update/delete partitioned external tables** in older versions.
* Performance overhead for frequent small updates.
* Compaction is needed for long-running tables (minor/major compaction).

---

### **85. What is dynamic partitioning in Hive?**

**Ans:**

* Partitions are automatically created at **run-time** based on column values.
* Useful when you don’t know partition values in advance.
* Syntax:

  ```sql
  SET hive.exec.dynamic.partition=true;
  SET hive.exec.dynamic.partition.mode=nonstrict;
  INSERT INTO table PARTITION(year, month) SELECT ...;
  ```

---

### **86. What are Hive Metastore and its importance?**

**Ans:**

* Hive Metastore stores **metadata** of Hive tables (schema, partition info, location).
* Can be **embedded** (Derby) or **remote** (MySQL/PostgreSQL).
* Enables multiple tools (Spark, Presto) to read Hive metadata.
* Essential for **query compilation and optimization**.

---

### **87. What is the difference between `EXPLAIN` and `DESCRIBE` in Hive?**

**Ans:**

| Command    | Purpose                                  |
| ---------- | ---------------------------------------- |
| `DESCRIBE` | Shows table schema or column information |
| `EXPLAIN`  | Shows execution plan of a query          |

---

### **88. How can you optimize Hive queries?**

**Ans:**

* Use **partitioning** and **bucketing**.
* Enable **vectorized query execution**.
* Use **columnar formats** like ORC/Parquet.
* Avoid `SELECT *`; only select required columns.
* Use **map-side joins** for small tables.
* Enable **cost-based optimizer (CBO)**.

---

### **89. What is a Hive UDF?**

**Ans:**

* User-Defined Function allows custom operations in Hive queries.
* Examples: string manipulation, custom aggregations.
* Written in Java and registered in Hive:

  ```sql
  CREATE FUNCTION myUDF AS 'com.example.MyUDF';
  SELECT myUDF(col) FROM table;
  ```

---

### **90. What are Hive Indexes and when should you use them?**

**Ans:**

* Hive Index improves **query performance** by reducing the data scanned.
* Types: **Compact Index**, **Bitmap Index**.
* Usually used on **columns frequently used in WHERE clauses**.
* Syntax:

  ```sql
  CREATE INDEX idx_name ON TABLE table_name (col)
  AS 'COMPACT' WITH DEFERRED REBUILD;
  ```

---

### **91. What is the difference between `MAPJOIN` and regular join in Hive?**

**Ans:**

* **MapJoin (Broadcast Join):** Small table is **loaded into memory** on each mapper to avoid shuffle.

  * Faster for small-large table joins.
* **Regular Join:** Data from both tables may be shuffled to reducers.
* Use `/*+ MAPJOIN(small_table) */` hint for explicit map joins.

---

### **92. What is the difference between `CLUSTERED BY` and `DISTRIBUTE BY` in Hive?**

**Ans:**

| Feature   | CLUSTERED BY                  | DISTRIBUTE BY                                     |
| --------- | ----------------------------- | ------------------------------------------------- |
| Purpose   | Defines **buckets** for table | Controls how rows are distributed across reducers |
| Used with | Bucketing tables              | Queries (SELECT)                                  |
| Sorting   | Can combine with `SORTED BY`  | No implicit sorting                               |

---

### **93. What is Hive bucketing?**

**Ans:**

* Bucketing divides a table into **fixed number of files (buckets)** based on **hash of a column**.
* Benefits: Optimized **joins**, **sampling**, and **query performance**.
* Example:

  ```sql
  CREATE TABLE employee(id INT, name STRING)
  CLUSTERED BY (id) INTO 4 BUCKETS;
  ```

---

### **94. What is the difference between `UNION` and `UNION ALL` in Hive?**

**Ans:**

| Feature     | UNION                     | UNION ALL                      |
| ----------- | ------------------------- | ------------------------------ |
| Duplicates  | Removes duplicates        | Keeps all records              |
| Performance | Slower (extra step)       | Faster                         |
| Use Case    | When you need unique rows | When duplicates are acceptable |

---

### **95. What are Hive views and their types?**

**Ans:**

* **Views** are **virtual tables** defined by a query.
* Types:

  1. **Virtual View:** Only query stored; data is not stored.
  2. **Materialized View:** Data is **precomputed and stored**, improving query performance.
* Syntax:

  ```sql
  CREATE VIEW view_name AS SELECT ...;
  CREATE MATERIALIZED VIEW mv_name AS SELECT ...;
  ```

---

### **96. How do you handle NULL values in Hive?**

**Ans:**

* Use functions:

  * `COALESCE(col1, col2, ...)` → returns first non-null value.
  * `NVL(col, default_value)` → replaces NULL with a value.
  * `IFNULL(col, value)` → similar to NVL.
* Avoid NULLs in keys for joins to prevent unexpected results.

---

### **97. What is Hive SerDe?**

**Ans:**

* **SerDe = Serializer/Deserializer**
* Controls **how Hive reads/writes data**.
* Example:

  * `LazySimpleSerDe` → Default CSV/TSV format.
  * `ORC` or `Parquet` SerDe → For columnar storage.
* Example syntax:

  ```sql
  CREATE TABLE t1(...) ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe';
  ```

---

### **98. What is Hive’s `EXTERNAL` table vs `TEMPORARY` table?**

**Ans:**

| Feature       | EXTERNAL TABLE                    | TEMPORARY TABLE             |
| ------------- | --------------------------------- | --------------------------- |
| Scope         | Stored in Hive warehouse          | Only session-level          |
| Data Lifetime | Persistent after session ends     | Deleted after session       |
| Metadata      | Metadata stored in metastore      | Metadata lost after session |
| Use Case      | Shared data across sessions/tools | Temporary computations      |

---

### **99. What is Hive transaction log?**

**Ans:**

* Hive transaction log tracks **insert, update, delete operations** on ACID tables.
* Stored in `hive.txn.manager` (default: `DbTxnManager`).
* Helps with **rollback, commit, and recovery**.
* Operations:

  * `BEGIN TRANSACTION` → start
  * `COMMIT` → save
  * `ROLLBACK` → undo

---

### **100. How does Hive handle schema evolution?**

**Ans:**

* Hive allows adding or replacing columns without rewriting the entire table:

  * `ALTER TABLE table_name ADD COLUMNS (col_name data_type)`
  * `REPLACE COLUMNS` (overwrites schema, data must match new structure)
* Compatible with **Parquet/ORC formats**.
* Enables smooth **data pipeline changes** without breaking queries.

---


### **101. What is the difference between `INSERT OVERWRITE` and `INSERT INTO` in Hive?**

**Ans:**

| Feature       | INSERT INTO                     | INSERT OVERWRITE                     |
| ------------- | ------------------------------- | ------------------------------------ |
| Data Handling | Appends data to the table       | Replaces existing data in the table  |
| Use Case      | Incremental data load           | Full refresh of table or partition   |
| Performance   | Can be slower if table is large | Efficient for overwriting partitions |

---

### **102. What are Hive partitions and why are they used?**

**Ans:**

* Partitioning splits a table into **sub-directories** based on column values.
* Benefits:

  * Reduces the data scanned during queries (`WHERE partition_col = value`).
  * Improves query performance.
* Example:

  ```sql
  CREATE TABLE sales(amount INT, date STRING)
  PARTITIONED BY (year INT, month INT);
  ```

---

### **103. Difference between static and dynamic partitioning in Hive?**

**Ans:**

| Feature          | Static Partitioning             | Dynamic Partitioning                   |
| ---------------- | ------------------------------- | -------------------------------------- |
| Partition Values | Known at query time             | Determined at runtime                  |
| Syntax           | Specify partition column values | Use SELECT query to determine values   |
| Flexibility      | Less flexible                   | More flexible, auto-creates partitions |

---

### **104. How does Hive handle large datasets efficiently?**

**Ans:**

* Using **columnar storage formats** (ORC, Parquet)
* **Partitioning and bucketing**
* **Vectorized query execution**
* **Map-side joins** for small tables
* **Cost-based optimizer (CBO)** enabled

---

### **105. What is Hive LLAP?**

**Ans:**

* **LLAP = Live Long and Process**
* A **daemon service** for low-latency analytical queries.
* Features:

  * In-memory caching
  * Persistent daemons for faster query execution
  * Supports multi-user concurrency

---

### **106. What is Hive’s Metastore DB?**

**Ans:**

* Stores **metadata**: table schema, partitions, storage locations.
* Can use: **Derby (embedded)**, **MySQL/PostgreSQL (remote)**
* Required for **query planning and optimization**.

---

### **107. Difference between Hive and traditional RDBMS?**

| Feature      | Hive                   | RDBMS                       |
| ------------ | ---------------------- | --------------------------- |
| Schema       | Schema-on-read         | Schema-on-write             |
| Transactions | Limited ACID support   | Full ACID                   |
| Indexing     | Optional, limited      | Fully supported             |
| Storage      | HDFS (distributed)     | Local/distributed storage   |
| Use Case     | OLAP, Big Data queries | OLTP, small-medium datasets |

---

### **108. Difference between internal table and external table in Hive?**

**Ans:** Already covered in **Q81**, but summary:

* **Internal (Managed):** Hive owns data, dropping table deletes data.
* **External:** User owns data, dropping table deletes only metadata.

---

### **109. What is the difference between Hive and Impala?**

| Feature         | Hive                          | Impala                         |
| --------------- | ----------------------------- | ------------------------------ |
| Query Execution | Batch-oriented                | Low-latency, real-time         |
| Engine          | MapReduce / Tez / Spark       | Native MPP engine              |
| Performance     | Slower for small queries      | Faster for interactive queries |
| Compatibility   | Good with Hive formats & ACID | Works well with Parquet/ORC    |

---

### **110. How do you enable Hive ACID tables?**

**Ans:**

* Steps to enable ACID:

  1. Set `hive.txn.manager = org.apache.hadoop.hive.ql.lockmgr.DbTxnManager`
  2. Enable `hive.support.concurrency = true`
  3. Use `transactional = true` when creating tables:

     ```sql
     CREATE TABLE t1(id INT, name STRING)
     STORED AS ORC
     TBLPROPERTIES ('transactional'='true');
     ```


---

### **111. What is Hive vectorization?**

**Ans:**

* Vectorization allows Hive to **process a batch of rows together** instead of one row at a time.
* Improves query performance by reducing CPU cycles.
* Enabled via:

  ```sql
  SET hive.vectorized.execution.enabled = true;
  SET hive.vectorized.execution.reduce.enabled = true;
  ```

---

### **112. What is Hive compaction?**

**Ans:**

* Compaction merges **small ACID files** into larger ones to improve query performance.
* Types:

  * **Minor Compaction:** Merges small delta files.
  * **Major Compaction:** Merges base + delta files into a new base.
* Triggered automatically or manually:

  ```sql
  ALTER TABLE table_name COMPACT 'MAJOR';
  ```

---

### **113. What are Hive constraints?**

**Ans:**

* Hive supports **primary key, foreign key, not null, unique**, but constraints are **not enforced** (mostly metadata only).
* Example:

  ```sql
  CREATE TABLE t1(
    id INT PRIMARY KEY,
    name STRING NOT NULL
  );
  ```

---

### **114. What is the difference between Hive `TRANSFORM` and UDF?**

**Ans:**

| Feature      | TRANSFORM                          | UDF                                |
| ------------ | ---------------------------------- | ---------------------------------- |
| Type         | Executes **custom scripts**        | Executes **custom Java functions** |
| Input/Output | Can handle multiple rows & columns | Operates row by row                |
| Language     | Any (Python, Shell, Perl)          | Java only                          |

---

### **115. What is Hive’s map-side join?**

**Ans:**

* Map-side join is **performed at the mapper stage** to avoid shuffling.
* Small table is **broadcasted to all mappers**.
* Faster than regular join, reduces network I/O.
* Example hint:

  ```sql
  SELECT /*+ MAPJOIN(small_table) */ * FROM large_table JOIN small_table ON ...
  ```

---

### **116. What is Hive’s cost-based optimizer (CBO)?**

**Ans:**

* CBO uses **table statistics** (size, cardinality) to **choose optimal execution plans**.
* Steps to enable:

  ```sql
  SET hive.cbo.enable = true;
  ANALYZE TABLE table_name COMPUTE STATISTICS;
  ANALYZE TABLE table_name COMPUTE STATISTICS FOR COLUMNS;
  ```

---

### **117. How do you perform incremental data load in Hive?**

**Ans:**

* Use **partitioning** or **timestamp columns**.
* Example using partition:

  ```sql
  INSERT INTO TABLE sales PARTITION (year, month)
  SELECT * FROM staging_sales WHERE year=2025 AND month=10;
  ```
* Helps **load only new data**, avoiding full table scan.

---

### **118. What is Hive replication?**

**Ans:**

* Hive replication allows **copying tables/partitions** from one cluster to another.
* Useful for **backup, DR, or multi-cluster setups**.
* Supports: **full replication** and **incremental replication**.
* Commands example:

  ```sql
  REPL DUMP source_db;
  REPL LOAD target_db FROM '/path';
  ```

---

### **119. What is the difference between `ORC` and `RCFile` in Hive?**

| Feature     | ORC (Optimized Row Columnar)   | RCFile (Record Columnar File)     |
| ----------- | ------------------------------ | --------------------------------- |
| Compression | High (Zlib)                    | Moderate (Default, Zlib optional) |
| Splittable  | Yes                            | Yes                               |
| Performance | Faster for read/write and ACID | Slower for large queries          |
| Support     | Hive ACID tables               | Older Hive versions               |

---

### **120. How does Hive handle skewed data?**

**Ans:**

* Skewed data occurs when **some partitions or keys have significantly more data** than others.
* Handling methods:

  * Use **`SKEWED BY`** clause while creating table:

    ```sql
    CREATE TABLE t1(a INT, b STRING)
    SKEWED BY (a) ON (1,2,3)
    STORED AS DIRECTORIES;
    ```
  * Enable **map-side join** for small skewed tables.
  * Use **salting techniques** for join keys to distribute data evenly.

---

