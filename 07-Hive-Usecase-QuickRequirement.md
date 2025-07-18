
## Quick Requirement

### Business Case

**Source Data Location**:
`/home/hduser/hive/data/custs`

Sample data:

```
4000001,Kristina,Chung,55,Pilot
4000002,Paige,Chen,77,Teacher
4000003,Sherri,Melton,34,Firefighter
4000004,Gretchen,Hill,66,Computer hardware engineer
```

**Requirements**:

a. From the raw data, extract only records with profession = "Pilot" in **UPPERCASE** with `id`, `age`, and `prof`, delimited by `|`, and store in HDFS path:
`hdfs:///user/hduser/extdata`

Example output:
`4000001|55|PILOT`

b. Load the same three columns into a Hive table for analytical use in optimized (row-column) format.

---

## Solution for Above Problem

### 1. Data Discovery & Initial Load

Perform data discovery:

* Type of data
* Field mapping
* Column names, datatypes
* Delimiter identification

**Create raw table**:

```sql
CREATE TABLE raw_table (
  id INT,
  fname STRING,
  lname STRING,
  age INT,
  prof STRING
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',';
```

**Load data**:

```sql
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/custs' 
INTO TABLE raw_table;
```

---

### 2. Create Final External Table (Pipe Delimited)

**Create external table with required 3 columns**:

```sql
CREATE EXTERNAL TABLE final_table (
  id INT,
  age INT,
  prof STRING
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY '|'
LOCATION '/user/hduser/extdata';
```

---

### 3. Perform ETL in Hive (Transformation)

**Transform and load data (meeting client requirement 1)**:

```sql
INSERT INTO final_table
SELECT id, age, UPPER(prof)
FROM raw_table
WHERE prof = 'Pilot';
```

---

### 4. Create ORC Table for Analytics (Consumer Table)

**Create analytical Hive table in ORC format**:

```sql
CREATE TABLE consumer_table (
  id INT,
  age INT,
  prof STRING
)
STORED AS ORC;
```

---

### 5. Load Data into Consumer Table

**Insert transformed data into analytical table**:

```sql
INSERT INTO consumer_table
SELECT id, age, prof
FROM raw_table;
```

**Client queries**:

```sql
SELECT * FROM consumer_table WHERE prof = 'Pilot';

SELECT id, age FROM consumer_table WHERE prof = 'Teacher';
```

---

### 6. Cleanup (Optional)

```sql
DROP TABLE raw_table;       -- Optional: not needed after transformation is complete

DROP TABLE final_table;     -- Only metadata is dropped (external table)
```

---
