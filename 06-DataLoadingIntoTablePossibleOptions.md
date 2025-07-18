It looks like the code to convert your Hive data loading methods into Markdown had a formatting issue. Here's the corrected **Markdown version** of your content:

---

## Possible Ways of Loading Data into Hive Tables

### 1. Using `LOAD DATA` Command

*Hive is simply copying the data to the underlying location.*

```sql
CREATE TABLE somehivetbl (
  id INT,
  name STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '|';
```

-- Sample data file
$ vi sampldata3
10|ranjith  
11|ragavi  
12|ragavan  


```sql
LOAD DATA LOCAL INPATH '/home/hduser/sampldata' INTO TABLE somehivetbl;

LOAD DATA LOCAL INPATH '/home/hduser/sampldata' OVERWRITE INTO TABLE somehivetbl;

LOAD DATA INPATH '/user/hduser/sampldata' OVERWRITE INTO TABLE somehivetbl;
-- Data will be moved to Hive table and source location data will be removed.
```

---

### 2. Manually Copying Data Using `hadoop fs -put`

```bash
-- Identify the table HDFS location
hive> DESCRIBE FORMATTED somehivetbl;

-- Copy file into Hive warehouse location
$ hadoop fs -put /home/hduser/sampldata3 /user/hive/warehouse/hivedatabase.db/somehivetbl/sampldata3
```

---

### 3. Creating External Table with Location Clause

```sql
CREATE EXTERNAL TABLE somehivetbl_2 (
  id STRING,
  name STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '|'
LOCATION '/user/hive/warehouse/hivedatabase.db/somehivetbl';
```

---

### 4. Using CTAS / Insert-Select

#### CTAS (Create Table As Select)

```sql
-- Create table and copy data
CREATE TABLE somehivetbl_3
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
AS SELECT * FROM somehivetbl;
```

```sql
-- Create table with same structure, no data (not preferred)
CREATE TABLE somehivetbl2 AS SELECT * FROM somehivetbl WHERE 1=2;
```

```sql
-- Preferred: Duplicate structure without copying data
CREATE TABLE somehivetbl3 LIKE somehivetbl;

-- Insert-Select: Copy data
INSERT INTO somehivetbl3 SELECT * FROM somehivetbl;
```

---

### 5. Using Tools (Shell Scripts, ETL Tools, Sqoop, Spark)

```bash
# Sqoop example 1
sqoop import --connect jdbc:mysql://127.0.0.1/custdb --username root --password Root123$ \
--table customer --driver com.mysql.cj.jdbc.Driver \
--hive-import --hive-table hivedatabase.somehivetbl4 -m 1 --delete-target-dir

# Sqoop example 2
sqoop import --connect jdbc:mysql://00.00.00.00.10/core_banking --username izusername --password izpassword \
--table payments --driver com.mysql.cj.jdbc.Driver \
--hive-import --hive-table hivedatabase.payments -m 1 --delete-target-dir
```

---

### 6. Using Insert Statements (Rare in Hive)

```sql
-- Rarely used due to performance and small file issue
INSERT INTO tablename VALUES (1, 'user1'), (2, 'user2');
```

---

Would you like this appended to the existing Markdown file or downloaded separately?
