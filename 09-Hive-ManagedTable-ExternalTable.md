### Managed (Container) Table & External (Referrer) Tables: Important

**Managed Table**
a. **Syntax (must):** `CREATE TABLE tblname (colname datatype);`
b. **Location:** Is not used in managed table usually because Hive manages it, but it's possible to use.
c. **Semantics (must):** If you `DROP TABLE`, both the table structure/schema and data will be dropped.
d. **Data Loading and Access:** Data is loaded and accessed by Hive itself.
e. **Operations:** Managed tables can be `TRUNCATED`, and you can create `CTAS` (Create Table As Select).

**External Table**
a. **Syntax (must):** `CREATE EXTERNAL TABLE tblname (colname datatype) LOCATION '/somelocation/somelocation';`
b. **Location:** Is mentioned usually because users manage it, but it's possible to not use.
c. **Semantics (must):** If you `DROP TABLE`, only the table structure/schema will be dropped, and the data will **not** be dropped.
d. **Data Loading and Access:** Data is loaded and accessed by external systems; Hive can also load.
e. **Operations:** External tables cannot be `TRUNCATED`, and you cannot create `CTAS`.

**Conversion between Table Types:**
We can change the table from external to managed or vice versa using table properties option.
**(Important)** `ALTER TABLE saravanakumar SET TBLPROPERTIES('EXTERNAL'='FALSE');`

`SET hive.cli.print.current.db=true;`
`SET hive.cli.print.header=true;`

`DROP DATABASE IF EXISTS retail CASCADE;`
`CREATE DATABASE retail;`
`USE retail;`

`DROP DATABASE IF EXISTS retail_raw;`
`DROP DATABASE IF EXISTS retail_curated;`
`DROP DATABASE IF EXISTS retail_analytics;`

`CREATE DATABASE retail_raw;`
`CREATE DATABASE retail_curated;`
`CREATE DATABASE retail_analytics;`

`CREATE TABLE retail_raw.txnrecords (txnno INT, txndate STRING, custno INT, amount DOUBLE, category STRING, product STRING, city STRING, state STRING, spendby STRING)`
`ROW FORMAT DELIMITED FIELDS TERMINATED BY ','`
`STORED AS TEXTFILE;`

**Position to Naming Convention (EL - Extract Load):**
`LOAD DATA LOCAL INPATH '/home/hduser/hive/data/txns' INTO TABLE retail_raw.txnrecords;`

Example Data:
`00000000,06-26-2011,4007024,040.33,Exercise & Fitness,Cardio Machine Accessories,Clarksville,Tennessee,credit`

`SELECT * FROM retail_raw.txnrecords LIMIT 10;`
`SELECT * FROM retail_raw.txnrecords ORDER BY txnno LIMIT 10;`

`CREATE EXTERNAL TABLE retail_curated.externaltxnrecords (`
  `txnno INT,`
  `custno INT,`
  `amount DOUBLE,`
  `category STRING,`
  `product STRING,`
  `city STRING,`
  `state STRING,`
  `spendby STRING,`
  `txndate DATE,`
  `txnts TIMESTAMP`
`)`
`ROW FORMAT DELIMITED FIELDS TERMINATED BY ','`
`STORED AS TEXTFILE`
`LOCATION '/user/hduser/hiveexternaldata';`

**Naming to Naming Convention (ETL - Extract Transform Load):**
`INSERT INTO TABLE retail_curated.externaltxnrecords (txnno, custno, amount, category, product, city, state, spendby, txndate, txnts)`
`SELECT`
  `txnno,`
  `custno,`
  `amount,`
  `category,`
  `product,`
  `city,`
  `state,`
  `spendby,`
  `FROM_UNIXTIME(UNIX_TIMESTAMP(txndate, 'MM-dd-yyyy'), 'yyyy-MM-dd'),`
  `FROM_UNIXTIME(UNIX_TIMESTAMP(txndate, 'MM-dd-yyyy'))`
`FROM retail_raw.txnrecords;`

**Date/Time Conversion Example:**
SOURCE UNIX HIVE
`tamil -> translator (english) -> hindi/malayalam`
`06-26-2011 -> SOMENUMBER (1309026600) -> 2011-06-26`

`SELECT FROM_UNIXTIME(UNIX_TIMESTAMP('06-26-2011','MM-dd-yyyy'),'yyyy-MM-dd');`
`SELECT FROM_UNIXTIME(UNIX_TIMESTAMP('06-26-2011','MM-dd-yyyy'),'yyyy-MM-dd HH:mm:ss');`

**Temporary Table:**
1. A temporary table is a session table, i.e., the life of the table will be available until the session is active.
2. We use this table for some temporary or interim purposes, where if we require the table to be dropped after it is used.
3. Temp table can be created as managed or external too.
4. When we quit out of Hive, this temp table will be dropped automatically.
`DROP TABLE temptable;`
`QUIT;`