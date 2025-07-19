## Table Partitioning in Hive

### Overview

Partitioning is the horizontal division of data into smaller parts based on column values.

In Hive:

* Each partition is implemented as a subfolder under the table directory.
* This structure allows for better data organization and performance improvements.
* Queries can skip scanning unneeded partitions (partition pruning).

### ✅ Advantages of Hive Partitioning

* Distributes execution load horizontally.
* Enables faster query execution with reduced data scanning.
* Avoids full table scans.

### ❌ Disadvantages

* May create too many small partitions (i.e., too many directories).
* Too many partitions can degrade performance and increase metadata management overhead.

---

### 📘 Example: Table Creation and Loading

```sql
-- Create a raw base table
CREATE TABLE raw_table (
  id INT,
  fname STRING,
  lname STRING,
  age INT,
  prof STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

-- Load data into the raw table
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/custs' INTO TABLE raw_table;
```

### 🔁 Dynamic Partitioning Example

> 💡 Can Hive convert a SQL to a MapReduce job, submit to YARN, read from one Hive table, and dynamically create partitions (as folders) in the target table based on source data?

```sql
-- Enable dynamic partitioning
SET hive.exec.dynamic.partition.mode=nonstrict;
SET hive.exec.max.dynamic.partitions=10000;
SET hive.exec.max.dynamic.partitions.pernode=10000;

-- Create external partitioned table
CREATE EXTERNAL TABLE curated_part_tbl4 (
  id INT,
  fname STRING,
  lname STRING
)
PARTITIONED BY (prof STRING, age INT)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/user/hduser/curated_part_tbl4';

-- Insert data dynamically into partitioned table
INSERT INTO curated_part_tbl4 PARTITION(prof, age)
SELECT id, fname, lname, prof, age FROM raw_table;

-- View partitions
SHOW PARTITIONS curated_part_tbl4;
```

#### ❓ Interview Question:

> How to drop both metadata and data for an external Hive table?

```sql
-- Method 1
ALTER TABLE curated_part_tbl4 SET TBLPROPERTIES('EXTERNAL'='FALSE');
DROP TABLE curated_part_tbl4;

-- Method 2
DROP TABLE curated_part_tbl4;
DFS -rm -r /user/hduser/curated_part_tbl4;
```

---

### 📘 Static Partitioning Example

```sql
-- Static partition load
SET hive.exec.dynamic.partition.mode=nonstrict;
SET hive.exec.max.dynamic.partitions=10000;
SET hive.exec.max.dynamic.partitions.pernode=10000;

CREATE EXTERNAL TABLE curated_part_tbl5 (
  id INT,
  fname STRING,
  lname STRING
)
PARTITIONED BY (prof STRING, age INT)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/user/hduser/curated_part_tbl5';

-- Insert into specific partitions
INSERT INTO curated_part_tbl5 PARTITION(prof='Pilot', age=35)
SELECT id, fname, lname FROM raw_table WHERE prof='Pilot' AND age=35;

-- Overwrite partition data
INSERT OVERWRITE TABLE curated_part_tbl5 PARTITION(prof='Pilot', age=35)
SELECT id, fname, lname FROM raw_table WHERE prof='Pilot' AND age=35;

-- Local export
INSERT OVERWRITE LOCAL DIRECTORY '/home/hduser/customer_prof_Teacher_age_71'
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
SELECT id, fname, lname, age, prof FROM raw_table WHERE prof='Teacher' AND age=71;

-- Load data from file
LOAD DATA LOCAL INPATH '/home/hduser/customer_prof_Teacher_age_71/customer_prof_Teacher_age_71.csv'
OVERWRITE INTO TABLE curated_part_tbl5 PARTITION(prof='Teacher', age=71);

-- Regrouping
INSERT OVERWRITE TABLE curated_part_tbl5 PARTITION(prof='Pilot_Teacher', age=50)
SELECT id, fname, lname FROM raw_table WHERE prof IN ('Pilot','Teacher') AND age=50;
```

#### ❓ Interview Question:

> How to drop the external table and its data?

```sql
-- Method 1
ALTER TABLE curated_part_tbl5 SET TBLPROPERTIES('EXTERNAL'='FALSE');
DROP TABLE curated_part_tbl5;

-- Method 2
DROP TABLE curated_part_tbl5;
DFS -rm -r /user/hduser/curated_part_tbl5;
```

---

### 📊 Static vs Dynamic Partitioning Comparison

| Feature                       | Static Partitions                | Dynamic Partitions                  |
| ----------------------------- | -------------------------------- | ----------------------------------- |
| Partition Value Determination | Manually specified               | Automatically determined by Hive    |
| Data Loading Speed            | Generally faster                 | Can be slower for large datasets    |
| Control                       | More control over data placement | Less control over data placement    |
| Flexibility                   | Less flexible                    | More flexible                       |
| Use Cases                     | When partition values are known  | When partition values are not known |

---

### 🧰 Additional Partitioning Options

```sql
-- Create a partitioned table by date and region
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

-- Load partitions
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/txns_20201212_PADE'
OVERWRITE INTO TABLE retail.txnrecsbydtreg
PARTITION (datadt='2020-12-12', region='PADE');

LOAD DATA LOCAL INPATH '/home/hduser/hive/data/txns_20201212_NY'
OVERWRITE INTO TABLE retail.txnrecsbydtreg
PARTITION (datadt='2020-12-12', region='NY');

-- View partition directory
DFS -ls -R /user/hive/warehouse/retail.db/txnrecsbydtreg/datadt=2020-12-12/;

-- Query by partition
SELECT COUNT(1) FROM retail.txnrecsbydtreg WHERE datadt='2020-12-12' AND region IN ('PADE', 'NY');

-- View partitions
SHOW PARTITIONS retail.txnrecsbydtreg;
```

#### ➕ Add New Partitions

```sql
ALTER TABLE retail.txnrecsbydtreg ADD IF NOT EXISTS
PARTITION (datadt='2020-12-13', region='PADE')
LOCATION '/user/hive/warehouse/retail.db/txnrecsbydtreg/datadt=2020-12-13/region=PADE/';
```

#### ✏️ Rename Partitions

```sql
ALTER TABLE retail.txnrecsbydtreg PARTITION (datadt='2020-12-13', region='PADE')
RENAME TO PARTITION (datadt='2020-12-14', region='NY');
```

#### ❌ Drop Partitions

```sql
ALTER TABLE retail.txnrecsbydtreg DROP PARTITION (datadt='2020-12-14', region='NY');
```

#### 🔄 Automate Partition Discovery

```bash
!cp /home/hduser/hive/data/txns /home/hduser/hive/data/txns_20201214_PADE
DFS -mkdir -p /user/hive/warehouse/retail.db/txnrecsbydtreg/datadt=2020-12-14/region=PADE/
DFS -put /home/hduser/hive/data/txns_20201214_PADE /user/hive/warehouse/retail.db/txnrecsbydtreg/datadt=2020-12-14/region=PADE/

-- Repair metadata
SET hive.msck.path.validation=ignore;
MSCK REPAIR TABLE retail.txnrecsbydtreg;
```

> ℹ️ Partitions not in metastore: txnrecsbydtreg\:datadt=2020-12-14/region=PADE
> ✅ Repair: Added partition to metastore txnrecsbydtreg\:datadt=2020-12-14/region=PADE
