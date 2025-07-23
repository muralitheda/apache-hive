
## 🧩 Table Partitioning in Hive

### Overview

Partitioning is the horizontal division of data into smaller parts based on column values.

In Hive:

* Each partition is a **subfolder under the table directory**.
* This helps in **better organization and efficient data retrieval**.
* Hive applies **partition pruning** to scan only relevant partitions.

---

### ✅ Advantages of Hive Partitioning

* Efficient execution by **skipping irrelevant partitions**.
* Helps **scale horizontally** and manage large datasets.
* Improves query performance by reducing full table scans.

---

### ❌ Disadvantages

* Too many partitions = too many HDFS directories.
* Excessive partitions can slow down performance and **increase catalog overhead**.

---

### 📘 Base Table Creation and Load

```sql
-- Enable useful Hive settings
SET hive.cli.print.current.db = true;
SET hive.cli.print.header = true;

-- Create and use demo database
CREATE DATABASE IF NOT EXISTS retail_demo;
USE retail_demo;

-- Create raw base table
CREATE TABLE raw_customer_data (
  id INT,
  fname STRING,
  lname STRING,
  age INT,
  prof STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

-- Load data
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/custs' INTO TABLE raw_customer_data;
```

---

### 🔁 Dynamic Partitioning Example

```sql
-- Enable dynamic partitioning
SET hive.exec.dynamic.partition.mode=nonstrict;
SET hive.exec.max.dynamic.partitions=10000;
SET hive.exec.max.dynamic.partitions.pernode=10000;

-- Create dynamic partitioned external table
CREATE EXTERNAL TABLE curated_customer_part_dyn (
  id INT,
  fname STRING,
  lname STRING
)
PARTITIONED BY (prof STRING, age INT)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/user/hduser/retail_demo/curated_customer_part_dyn';

-- Load partitions dynamically
INSERT INTO curated_customer_part_dyn PARTITION(prof, age)
SELECT id, fname, lname, prof, age FROM raw_customer_data;

-- View partitions
SHOW PARTITIONS curated_customer_part_dyn;
```

---

### 📘 Static Partitioning Example

```sql
-- Reuse partition settings
SET hive.exec.dynamic.partition.mode=nonstrict;

-- Create static partitioned external table
CREATE EXTERNAL TABLE curated_customer_part_static (
  id INT,
  fname STRING,
  lname STRING
)
PARTITIONED BY (prof STRING, age INT)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/user/hduser/retail_demo/curated_customer_part_static';

-- Insert into a specific static partition
INSERT INTO curated_customer_part_static PARTITION(prof='Pilot', age=35)
SELECT id, fname, lname FROM raw_customer_data WHERE prof='Pilot' AND age=35;

-- Overwrite existing partition
INSERT OVERWRITE TABLE curated_customer_part_static PARTITION(prof='Pilot', age=35)
SELECT id, fname, lname FROM raw_customer_data WHERE prof='Pilot' AND age=35;

-- Export partition locally
INSERT OVERWRITE LOCAL DIRECTORY '/home/hduser/customer_prof_Teacher_age_71'
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
SELECT id, fname, lname, age, prof FROM raw_customer_data WHERE prof='Teacher' AND age=71;

-- Load exported data back into Hive
LOAD DATA LOCAL INPATH '/home/hduser/customer_prof_Teacher_age_71/customer_prof_Teacher_age_71.csv'
OVERWRITE INTO TABLE curated_customer_part_static PARTITION(prof='Teacher', age=71);

-- Group two professions into one partition
INSERT OVERWRITE TABLE curated_customer_part_static PARTITION(prof='Pilot_Teacher', age=50)
SELECT id, fname, lname FROM raw_customer_data WHERE prof IN ('Pilot','Teacher') AND age=50;
```

---

### ❓ Interview Question: Drop External Table and Its Data

```sql
-- Option 1: Convert to managed and drop
ALTER TABLE curated_customer_part_static SET TBLPROPERTIES('EXTERNAL'='FALSE');
DROP TABLE curated_customer_part_static;

-- Option 2: Drop and manually clean HDFS
DROP TABLE curated_customer_part_static;
DFS -rm -r /user/hduser/retail_demo/curated_customer_part_static;
```

---

### 📊 Static vs Dynamic Partitioning Comparison

| Feature                   | Static Partitions               | Dynamic Partitions                 |
| ------------------------- | ------------------------------- | ---------------------------------- |
| Partition Value Specified | Manually in query               | Derived from data                  |
| Performance               | Generally faster                | Slower with high partition volume  |
| Control                   | High (explicit partition logic) | Low (data-driven)                  |
| Flexibility               | Limited                         | Highly flexible                    |
| Use Cases                 | Known partition keys            | Unknown or changing partition keys |

---

### 🧰 Example: Date & Region-Based Partitioning

```sql
-- Create new partitioned table
CREATE TABLE retail_demo.txn_data_by_date_region (
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

-- Load partitions
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/txns_20201212_PADE'
OVERWRITE INTO TABLE retail_demo.txn_data_by_date_region
PARTITION (datadt='2020-12-12', region='PADE');

LOAD DATA LOCAL INPATH '/home/hduser/hive/data/txns_20201212_NY'
OVERWRITE INTO TABLE retail_demo.txn_data_by_date_region
PARTITION (datadt='2020-12-12', region='NY');

-- Show partition folders
DFS -ls -R /user/hive/warehouse/retail_demo.db/txn_data_by_date_region/datadt=2020-12-12/;

-- Query using partition filter
SELECT COUNT(1) FROM retail_demo.txn_data_by_date_region
WHERE datadt='2020-12-12' AND region IN ('PADE', 'NY');

-- View all partitions
SHOW PARTITIONS retail_demo.txn_data_by_date_region;
```

---

### ➕ Manage Partitions

#### Add Partition

```sql
ALTER TABLE retail_demo.txn_data_by_date_region ADD IF NOT EXISTS
PARTITION (datadt='2020-12-13', region='PADE')
LOCATION '/user/hive/warehouse/retail_demo.db/txn_data_by_date_region/datadt=2020-12-13/region=PADE/';
```

#### Rename Partition

```sql
ALTER TABLE retail_demo.txn_data_by_date_region PARTITION (datadt='2020-12-13', region='PADE')
RENAME TO PARTITION (datadt='2020-12-14', region='NY');
```

#### Drop Partition

```sql
ALTER TABLE retail_demo.txn_data_by_date_region DROP PARTITION (datadt='2020-12-14', region='NY');
```

---

### 🔄 Repair Partition Metadata (MSCK REPAIR)

```bash
-- Copy file and create partition folder
!cp /home/hduser/hive/data/txns /home/hduser/hive/data/txns_20201214_PADE
DFS -mkdir -p /user/hive/warehouse/retail_demo.db/txn_data_by_date_region/datadt=2020-12-14/region=PADE/
DFS -put /home/hduser/hive/data/txns_20201214_PADE /user/hive/warehouse/retail_demo.db/txn_data_by_date_region/datadt=2020-12-14/region=PADE/

-- Repair metadata to recognize new partitions
SET hive.msck.path.validation=ignore;
MSCK REPAIR TABLE retail_demo.txn_data_by_date_region;
```

---

### 🔎 Notes on Metadata & HDFS Overhead

* **Metadata Overhead**: Each partition adds entries in the Hive Metastore. Too many can slow down planning and queries.
* **HDFS Overhead**: Each partition = directory in HDFS. Too many small files or directories cause NameNode memory issues and slow directory listing.
* **Best Practice**: Use low-cardinality fields (e.g., date, region) for partitioning. Avoid high-cardinality fields like user ID or timestamps.

