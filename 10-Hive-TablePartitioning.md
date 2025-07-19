# Hive Table Partitioning: Concepts, Syntax, and Examples

---

## 🔄 What is Table Partitioning?

**General Definition:**

> Partitioning is the **horizontal division** of a table's data into segments based on column values, enabling **better organization**, **faster access**, and **optimized data management**.

**In Hive:**

> Partitioning physically separates data into **sub-folders** in HDFS, improving query performance by avoiding full table scans.

---

## 👉 Hive Partitioning Pros and Cons

### Advantages:

* Efficient query execution with **limited data scans**
* **Faster query** performance due to reduced I/O
* Logical **data separation** using folder-like structure

### Disadvantages:

* Risk of creating **too many small partitions** (i.e., directory explosion)
* Requires **careful design** to balance granularity and manageability

---

## ✏️ Creating Raw Table (Unpartitioned)

```sql
CREATE TABLE raw_table (
  id INT,
  fname STRING,
  lname STRING,
  age INT,
  prof STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

LOAD DATA LOCAL INPATH '/home/hduser/hive/data/custs'
INTO TABLE raw_table;
```

---

## 🚀 Dynamic Partitioning

Dynamically create partitions based on **source data values**.

### Setup:

```sql
SET hive.exec.dynamic.partition.mode=nonstrict;
SET hive.exec.max.dynamic.partitions=10000;
SET hive.exec.max.dynamic.partitions.pernode=10000;
```

### Create Target Table:

```sql
CREATE EXTERNAL TABLE curated_part_tbl4 (
  id INT,
  fname STRING,
  lname STRING
)
PARTITIONED BY (prof STRING, age INT)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/user/hduser/curated_part_tbl4';
```

### Insert Data:

```sql
-- Insert with auto-generated partitions
INSERT INTO curated_part_tbl4 PARTITION(prof, age)
SELECT id, fname, lname, prof, age FROM raw_table;

-- Conditional insert
INSERT INTO curated_part_tbl4 PARTITION(prof, age)
SELECT id, fname, lname, prof, age
FROM raw_table WHERE prof IN ('Pilot','Teacher');
```

---

## ✅ Static Partitioning

Manually specify the partition values during insert.

### Setup:

```sql
SET hive.exec.dynamic.partition.mode=nonstrict;
```

### Create Target Table:

```sql
CREATE EXTERNAL TABLE curated_part_tbl5 (
  id INT,
  fname STRING,
  lname STRING
)
PARTITIONED BY (prof STRING, age INT)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/user/hduser/curated_part_tbl5';
```

### Insert Using Static Values:

```sql
-- Appending to partitions
INSERT INTO curated_part_tbl5 PARTITION(prof='Pilot', age=35)
SELECT id, fname, lname FROM raw_table WHERE prof='Pilot' AND age=35;

-- Overwriting partitions
INSERT OVERWRITE TABLE curated_part_tbl5 PARTITION(prof='Pilot', age=35)
SELECT id, fname, lname FROM raw_table WHERE prof='Pilot' AND age=35;
```

### Insert Using Local File:

```sql
LOAD DATA LOCAL INPATH '/home/hduser/customer_prof_Teacher_age_71/customer_prof_Teacher_age_71.csv'
OVERWRITE INTO TABLE curated_part_tbl5 PARTITION(prof='Teacher', age=71);
```

---

## 📌 Static vs Dynamic Partitioning Comparison

| Feature         | Static Partitioning       | Dynamic Partitioning         |
| --------------- | ------------------------- | ---------------------------- |
| Partition Value | Manually specified        | Automatically determined     |
| Speed           | Faster for small datasets | May be slower for large data |
| Control         | More control              | Less control                 |
| Flexibility     | Less flexible             | More flexible                |
| Use Case        | Known partition values    | Unknown at runtime           |

---

## ➕ Additional Partition Options

### Create Partitioned Table by Date & Region

```sql
CREATE TABLE retail.txnrecsbydtreg (
  txnno INT,
  txndate STRING,
  custno INT,
  amount DOUBLE,
  category STRING,
  product STRING,
  city STRING,
  state STRING,
  spendby STRING
)
PARTITIONED BY (datadt DATE, region STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

### Load Partitioned Data:

```sql
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/txns_20201212_PADE'
OVERWRITE INTO TABLE retail.txnrecsbydtreg
PARTITION (datadt='2020-12-12', region='PADE');
```

---

## ✍️ Partition Maintenance Commands

### 1. Add Partition:

```sql
ALTER TABLE retail.txnrecsbydtreg ADD IF NOT EXISTS
PARTITION (datadt='2020-12-13', region='PADE')
LOCATION '/user/hive/warehouse/retail.db/txnrecsbydtreg/datadt=2020-12-13/region=PADE/';
```

### 2. Rename Partition:

```sql
ALTER TABLE retail.txnrecsbydtreg
PARTITION (datadt='2020-12-13', region='PADE')
RENAME TO PARTITION (datadt='2020-12-14', region='NY');
```

### 3. Drop Partition:

```sql
ALTER TABLE retail.txnrecsbydtreg
DROP PARTITION (datadt='2020-12-14', region='NY');
```

### 4. Automate Partition Repair:

```sql
SET hive.msck.path.validation=ignore;
MSCK REPAIR TABLE retail.txnrecsbydtreg;
```

---

## ❓ Interview Tip: Dropping External Table + Data

### Method 1:

```sql
ALTER TABLE curated_part_tbl4 SET TBLPROPERTIES ('EXTERNAL'='FALSE');
DROP TABLE curated_part_tbl4;
```

### Method 2:

```sql
DROP TABLE curated_part_tbl4;
dfs -rm -r /user/hduser/curated_part_tbl4;
```
