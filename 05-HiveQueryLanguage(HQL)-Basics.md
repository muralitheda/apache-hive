# Hive Query Language (HQL) Guide

## Starting Hive Shell

```bash
$ hive
```

## Basic Settings

```sql
set hive.cli.print.current.db=true;  
set hive.cli.print.header=true;  
```

## Database Operations

```sql
-- Show all databases
show databases;  

-- Show databases with name pattern
show databases like '*hive*';  

-- Create a new database
CREATE DATABASE IF NOT EXISTS hive_batch_db
COMMENT 'Hive batch training database'
LOCATION '/user/hduser/hive_batch_db'
WITH DBPROPERTIES ('created_by' = 'trainer', 'created_on' = '2024-12-28');

-- Drop database
DROP DATABASE hive_batch_db; -- This command attempts to drop an empty database
DROP DATABASE hive_batch_db CASCADE; -- This command is used to forcefully drop a database and all its contents (tables)

-- Use database
USE hive_batch_db;

-- Describe database
DESC DATABASE hive_batch_db;
DESC DATABASE EXTENDED hive_batch_db;

-- List tables
SHOW TABLES;

-- HDFS check
$ hadoop fs -ls /user/hduser/hive_batch_db
```

## ETL vs ELT in Hive

### ETL (Load-time parsing and cleaning)

```sql
-- Create a table
CREATE TABLE hive_tbl (
  id INT,
  name VARCHAR(10),
  age SMALLINT
);

-- Insert valid data
INSERT INTO hive_tbl (id, name, age) VALUES (1, 'udhayakuma', 30);

-- Insert semi-structured/invalid data (Hive allows it)
INSERT INTO hive_tbl (id, name, age) VALUES (2, 'udhayakumar', 'thirty three');

-- Query
SELECT * FROM hive_tbl;
-- Output:
-- 1  udhayakuma  30
-- 2  udhayakuma  NULL
```

### ELT (Query-time parsing)

```sql
-- Create External Table for JSON
CREATE EXTERNAL TABLE hive_json_tbl(
  id INT,
  name VARCHAR(10),
  age SMALLINT
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES ("case.insensitive" = "false")
STORED AS TEXTFILE
LOCATION '/user/hduser/hive_batch_db/json_tbl';

-- Load data
LOAD DATA LOCAL INPATH "/home/hduser/jsondata.json" INTO TABLE hive_json_tbl;

-- Check raw file
$ hadoop fs -cat /user/hduser/hive_batch_db/json_tbl/jsondata.json

-- SELECT fails due to data mismatch
SELECT * FROM hive_json_tbl;

-- Iteration 1
CREATE EXTERNAL TABLE hive_json_tbl_1(
  id INT,
  name VARCHAR(100),
  age SMALLINT,
  city STRING
)
...

-- Iteration 2
CREATE EXTERNAL TABLE hive_json_tbl_2(
  id INT,
  name VARCHAR(100),
  age VARCHAR(10),
  city STRING
)
...

-- Iteration 3
CREATE EXTERNAL TABLE hive_json_tbl_3(
  id INT,
  name VARCHAR(100),
  age VARCHAR(100),
  city STRING
)
...
```

## Table Structure

Hive tables are structured in rows and columns.

### Sample Record

```text
1,
Jack,
43,
i am an IT professional with 3 kids residing in chennai with an experience of 20 years...,
139 ~ 53th main road ~ AED Colony ~ Velachery,
TN,
5.11,
102,
1982-02-22,
2024-12-28 11:08:01,
true,
110000000123410
```

### Data Discovery Table

```sql
CREATE TABLE custinfo (
  id INT,
  name VARCHAR(100),
  age TINYINT,
  comments STRING,
  address STRING,
  statecd CHAR(2),
  height FLOAT,
  weight SMALLINT,
  dob DATE,
  login_ts TIMESTAMP,
  isactive BOOLEAN,
  acct_no BIGINT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

LOAD DATA LOCAL INPATH '/home/hduser/datatypedata.txt' INTO TABLE custinfo;
```

### Simpler Version

```sql
CREATE TABLE custinfo_simple (
  id INT,
  name STRING,
  age INT,
  comments STRING,
  address STRING,
  statecd STRING,
  height FLOAT,
  weight INT,
  dob DATE,
  login_ts TIMESTAMP,
  isactive STRING,
  acct_no BIGINT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';
```

## Basic Table Creation Syntax

```sql
CREATE TABLE tablename (
  colname datatype, colname datatype
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '|';

SHOW CREATE TABLE tablename;
```

## Complete Table Syntax (Reference)

```sql
CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name
(
  col_name data_type [COMMENT col_comment], ...
)
[COMMENT table_comment]
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
[CLUSTERED BY (...) SORTED BY (...) INTO n BUCKETS]
[ROW FORMAT row_format]
[STORED AS file_format]
[LOCATION hdfs_path]
[TBLPROPERTIES (...)]
[AS select_statement];
```

### Notes:

- External Table: Refers to data stored outside Hive warehouse.
- COMMENT: Useful for data discovery and governance.
- Default delimiter is `\001` (Ctrl+A).
- `STORED BY` can connect Hive with other data sources like RDBMS/NoSQL.

## Example: Create Table with Metadata

```sql
CREATE TABLE IF NOT EXISTS hive_batch_db.sample_table5 (
  col_id1 INT COMMENT 'this column holds id value',
  col_name2 STRING COMMENT 'this column holds name value'
)
COMMENT 'This table holds customer info'
ROW FORMAT DELIMITED FIELDS TERMINATED BY '~'
STORED AS ORC
LOCATION '/user/hduser/warehouse/hive_batch_db.db/sample_table5';

INSERT INTO sample_table5 VALUES (1, 'MURALI');
INSERT INTO sample_table5 VALUES (2, 'JACK');
```

## When to Use Which Option

| Feature                | Purpose                                            |
| ---------------------- | -------------------------------------------------- |
| `schema.table`         | Access tables in different DBs                     |
| `IF NOT EXISTS`        | Avoid errors in automated scripts                  |
| `COMMENT`              | Data governance and discovery                      |
| `ROW FORMAT DELIMITED` | Helps Hive understand how to read the file         |
| `STORED AS`            | Specify file format (textfile, orc, parquet, etc.) |
| `LOCATION`             | Use existing HDFS data or stage new data           |

---
