
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

      * `mod(1 / 3) = 1` -\> 1st bucket
      * `mod(2 / 3) = 2` -\> 2nd bucket
      * `mod(3 / 3) = 0` -\> 0th bucket
      * `mod(4 / 3) = 1` -\> 1st bucket
      * `mod(5 / 3) = 2` -\> 2nd bucket
      * `mod(6 / 3) = 0` -\> 0th bucket
      * `mod(8 / 3) = 2` -\> 2nd bucket
      * `mod(9 / 3) = 0` -\> 0th bucket

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

| Feature                    | Partitioning                                       | Bucketing                                          |
| :------------------------- | :------------------------------------------------- | :------------------------------------------------- |
| **Primary Goal** | Organizing data for efficient filtering            | Distributing data evenly for efficient joins and sampling |
| **Column Cardinality** | Low cardinality/difference columns (e.g., `date`, `country`) | High or low cardinality columns                   |
| **Hierarchy** | Sub-partitions are possible inside a partition     | Multiple `clustered by` columns are possible. If multiple bucket columns, they are combined (ASCII value) before applying the modulo logic. |
| **Storage Structure** | Creates folders                                    | Creates files                                      |
| **Performance Impact** | Improves `WHERE`/`FILTER` clause performance       | Improves `JOIN` and `WHERE`/`FILTER` clause performance |



## Syntax for Creating a Bucketed Table 📝

```sql
CREATE [EXTERNAL] TABLE [db_name.]table_name
[(col_name data_type [COMMENT col_comment], ...)]
CLUSTERED BY (col_name data_type [COMMENT col_comment], ...) [SORTED BY (col_name [ASC|DESC], ...)]
INTO N BUCKETS;
```



## Examples of Bucketing in Action 🧑‍💻

### Table with Only Buckets

```sql
# Table with only Buckets
create table retail.sample_bucket1(id int,name string,city string)
clustered by (id) sorted by (id) into 3 buckets
row format delimited fields terminated by ',';

insert into retail.sample_bucket1 values
(1,'a','chn'),
(2,'b','hyd'),
(3,'c','mum'),
(4,'d','chn'),
(5,'e','mum'),
(6,'f','chn'),
(7,'g','chn'),
(8,'h','mum'),
(9,'i','chn');
```

To view the created bucket files:

```bash
dfs -ls /user/hive/warehouse/retail.db/sample_bucket1/*;
```

**Output**:

```
-rwxrwxr-x 1 hduser supergroup 24 2025-01-21 16:42 /user/hive/warehouse/retail.db/sample_bucket1/000000_0
-rwxrwxr-x 1 hduser supergroup 24 2025-01-21 16:42 /user/hive/warehouse/retail.db/sample_bucket1/000001_0
-rwxrwxr-x 1 hduser supergroup 24 2025-01-21 16:42 /user/hive/warehouse/retail.db/sample_bucket1/000002_0
```

Content of each bucket file:

```bash
dfs -cat /user/hive/warehouse/retail.db/sample_bucket1/000000_0;
```

**Output**:

```
3,c,mum
6,f,chn
9,i,chn
```

```bash
dfs -cat /user/hive/warehouse/retail.db/sample_bucket1/000001_0;
```

**Output**:

```
1,a,chn
4,d,chn
7,g,chn
```

```bash
dfs -cat /user/hive/warehouse/retail.db/sample_bucket1/000002_0;
```

**Output**:

```
2,b,hyd
5,e,mum
8,h,mum
```

When querying a specific ID:

```sql
select * from sample_bucket where id=3;
```

**Explanation**: Out of 9 rows, Hive (mappers) filtered 1 file (bucket) out of 3 files (buckets) and performed 2 searches out of 3 rows to find 1 row as output because the data is sorted within the bucket.



### Table with Both Partition & Buckets

```sql
# Table with both Partition & Buckets
create table retail.sample_part_bucket(id int,name string)
partitioned by (city string)
clustered by (id) sorted by (id) into 3 buckets
row format delimited fields terminated by ',';

insert into retail.sample_part_bucket partition(city)
select id,name,city from retail.sample_bucket1;
```

Querying with both partition and bucket conditions:

```sql
select * from retail.sample_part_bucket where city='chn' and id=4;
```

**Explanation**: Hive will first search the partition folder for `city='chn'`, and then, within that partition, it will look inside the relevant bucket to find `(4,d)`.

To list all files recursively:

```bash
dfs -ls -R /user/hive/warehouse/retail.db/sample_part_bucket;
```

**Output**:

```
drwxrwxr-x - hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=chn
-rwxrwxr-x 1 hduser supergroup 8 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=chn/000000_0
-rwxrwxr-x 1 hduser supergroup 12 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=chn/000001_0
-rwxrwxr-x 1 hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=chn/000002_0
drwxrwxr-x - hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=hyd
-rwxrwxr-x 1 hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=hyd/000000_0
-rwxrwxr-x 1 hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=hyd/000001_0
-rwxrwxr-x 1 hduser supergroup 4 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=hyd/000002_0
drwxrwxr-x - hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=mum
-rwxrwxr-x 1 hduser supergroup 4 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=mum/000000_0
-rwxrwxr-x 1 hduser supergroup 0 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=mum/000001_0
-rwxrwxr-x 1 hduser supergroup 8 2025-01-21 17:04 /user/hive/warehouse/retail.db/sample_part_bucket/city=mum/000002_0
```

Content of selected bucket files within partitions:

```bash
dfs -cat /user/hive/warehouse/retail.db/sample_part_bucket/city=chn/000000_0;
```

**Output**:

```
6,f
9,i
```

```bash
dfs -cat /user/hive/warehouse/retail.db/sample_part_bucket/city=chn/000001_0;
```

**Output**:

```
1,a
4,d
7,g
```

```bash
dfs -cat /user/hive/warehouse/retail.db/sample_part_bucket/city=hyd/000002_0;
```

**Output**:

```
2,b
```

```bash
dfs -cat /user/hive/warehouse/retail.db/sample_part_bucket/city=mum/000000_0;
```

**Output**:

```
3,c
```

```bash
dfs -cat /user/hive/warehouse/retail.db/sample_part_bucket/city=mum/000002_0;
```

**Output**:

```
5,e
8,h
```

## Interview Questions on Bucketing 🧠

### 1\. Can we modify the number of buckets after some time due to data volume increase without changing the entire dataset?

**Answer**: Not directly. You cannot simply `ALTER TABLE` to change the number of buckets and expect the existing data to be re-bucketed. You need to create a new table with the desired number of buckets and then move the data.

**Example**:

```sql
create table retail.sample_bucket1_tmp(id int,name string,city string)
clustered by (id) sorted by (id) into 10 buckets # New number of buckets
row format delimited fields terminated by ',';

insert into retail.sample_bucket1_tmp select id,name,city from retail.sample_bucket1; # Re-bucket data

drop table retail.sample_bucket1;

alter table retail.sample_bucket1_tmp rename to sample_bucket1; # Rename to replace original table
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

      * **Guideline**: 1 bucket = 1 reducer = \~256 MB.

      * **Calculation**:

          * 1 GB = 1024 MB / 256 MB = 4 buckets.
          * For 40 GB: 40 GB \* 4 buckets/GB = 160 buckets.

      * **Analysis**:

          * 10 buckets (4 GB/bucket) is generally not a good idea as it leads to large bucket files.
          * 80 buckets (512 MB/bucket) is acceptable, as it allows for 80 reducers to run concurrently.
          * 160 buckets (256 MB/bucket) is often considered the best choice as it aligns with the 256 MB per reducer guideline, though running 160 reducers might be too many for some clusters, leading to overhead.
          * If each bucket contains approximately 2GB of data, this implies too few buckets and might lead to large file sizes, negating some benefits.


## Bucketing Improves Join Performance: Example 🤝

To reiterate, for bucketing to significantly improve join performance, both tables must:

1.  Have the **same bucket columns**.
2.  Have the **same number of buckets**.

Let's create two similar bucketed tables:

```sql
create table retail.sample_bucket2(id int,name string,city string)
clustered by (id) sorted by (id) into 3 buckets
row format delimited fields terminated by ',';

insert into retail.sample_bucket2 values
(1,'a','chn'),
(2,'b','hyd'),
(3,'c','mum'),
(4,'d','chn'),
(5,'e','hyd'),
(6,'f','chn'),
(7,'g','chn'),
(8,'h','mum'),
(9,'i','chn');

create table retail.sample_bucket3(id int,name string,city string)
clustered by (id) sorted by (id) into 3 buckets
row format delimited fields terminated by ',';

insert into retail.sample_bucket3 values
(1,'a','chn'),
(2,'b','hyd'),
(3,'c','mum'),
(4,'d','chn'),
(5,'e','hyd'),
(6,'f','chn'),
(7,'g','chn'),
(8,'h','mum'),
(9,'i','chn');
```

Now, perform a join operation:

```sql
select b1.*,b3.* from retail.sample_bucket1 b1 join retail.sample_bucket3 b3 on b1.id=b3.id;
```

**Output**:

```
b1.id b1.name b1.city b3.id b3.name b3.city
3     c       mum     3     c       mum
6     f       chn     6     f       chn
9     i       chn     9     i       chn
1     a       chn     1     a       chn
4     d       chn     4     d       chn
7     g       chn     7     g       chn
2     b       hyd     2     b       hyd
5     e       mum     5     e       hyd
8     h       mum     8     h       mum
```

In this scenario, because `sample_bucket1` and `sample_bucket3` (note: the original prompt used `sample_bucket1` for the join, assuming it also meets the bucket criteria) have the same bucket column (`id`) and the same number of buckets (`3`), Hive can perform an optimized **Sort Merge Bucket Join**. This means that instead of a full shuffle, it can join corresponding buckets directly, leading to significant performance gains.