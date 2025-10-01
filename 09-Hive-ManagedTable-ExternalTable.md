
# 🗂️ Managed vs External Tables in Hive

| Feature | Managed (Internal) Table | External Table |
| :--- | :--- | :--- |
| **Data Ownership** | Hive has full ownership and control of both the metadata and the data files. | Hive only manages the metadata. The data is owned and controlled by the user. |
| **Data Deletion** | Dropping the table deletes both the metadata from the Hive Metastore and the data files from HDFS. | Dropping the table only deletes the metadata. The data files remain untouched in their original location. |
| **Data Location** | Data is stored within Hive's default warehouse directory, typically `/user/hive/warehouse/`. | The user specifies the data's location using the `LOCATION` clause, which can be any valid HDFS or external storage path. |
| **Ideal Use Case** | Use for temporary or intermediate data where Hive is the sole application accessing the data. | Use for data that needs to be shared with other tools (e.g., Spark, Pig) or needs to persist even if the table is dropped. |

## 🧾 Managed Table (Default - Internal)

A **Managed Table** in Hive is **controlled entirely by Hive**—both metadata and data. It's ideal when Hive is the only system accessing the data.

### ✅ Key Characteristics:

* **Syntax:**

  ```sql
  CREATE TABLE table_name (column_name data_type);
  ```

* **Location:**
  Automatically managed by Hive; location can be specified, but usually not needed.

* **Behavior on DROP:**
  Drops both the **table definition** and the **data**.

* **Data Access:**
  Data is read/written directly by Hive.

* **Allowed Operations:**

  * Supports `TRUNCATE`
  * Supports `CTAS` (Create Table As Select)

---

## 🌐 External Table (Referenced - External Data)

An **External Table** allows Hive to reference data **stored outside its control**, meaning the data **remains untouched** even if the table is dropped.

### ✅ Key Characteristics:

* **Syntax:**

  ```sql
  CREATE EXTERNAL TABLE table_name (column_name data_type)
  LOCATION '/external/path/';
  ```

* **Location:**
  Must be explicitly defined by the user.

* **Behavior on DROP:**
  Drops only the **table metadata**; data remains safe.

* **Data Access:**
  Data can be loaded externally; Hive just queries it.

* **Allowed Operations:**

  * Cannot be `TRUNCATED`
  * `CTAS` not allowed

---

## 🔁 Switching Between Table Types

You can **convert** between managed and external table types using `TBLPROPERTIES`:

```sql
ALTER TABLE table_name SET TBLPROPERTIES('EXTERNAL'='FALSE');  -- to convert to managed
```

---

# 🧱 Hive Environment Setup & Best Practices

### 📌 Setup Hive CLI Environment

```sql
SET hive.cli.print.current.db = true;
SET hive.cli.print.header = true;
```

---

### 🏗️ Create & Organize Databases

```sql
-- Clean up old if any
DROP DATABASE IF EXISTS retail CASCADE;

-- Create Main Database
CREATE DATABASE retail;
USE retail;

-- Multi-Layered Architecture
DROP DATABASE IF EXISTS retail_raw CASCADE;
DROP DATABASE IF EXISTS retail_curated CASCADE;
DROP DATABASE IF EXISTS retail_analytics CASCADE;

CREATE DATABASE retail_raw;
CREATE DATABASE retail_curated;
CREATE DATABASE retail_analytics;
```

---

## 📥 Managed Table Example – `retail_raw`

```sql
CREATE TABLE retail_raw.txnrecords (
  txnno     INT,
  txndate   STRING,
  custno    INT,
  amount    DOUBLE,
  category  STRING,
  product   STRING,
  city      STRING,
  state     STRING,
  spendby   STRING
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE;
```

### 📌 Load Data (EL: Extract & Load)

```sql
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/txns'
INTO TABLE retail_raw.txnrecords;
```

Example Row:

```
00000000,06-26-2011,4007024,040.33,Exercise & Fitness,Cardio Machine Accessories,Clarksville,Tennessee,credit
```

---

## 📊 Query Data

```sql
SELECT * FROM retail_raw.txnrecords LIMIT 10;
SELECT * FROM retail_raw.txnrecords ORDER BY txnno LIMIT 10;
```

---

## 🌐 External Table Example – `retail_curated`

```sql

DROP TABLE IF EXISTS retail_curated.externaltxnrecords;

$ hadoop fs -rm -r /user/hduser/hiveexternaldata

CREATE EXTERNAL TABLE retail_curated.externaltxnrecords (
  txnno     INT,
  custno    INT,
  amount    DOUBLE,
  category  STRING,
  product   STRING,
  city      STRING,
  state     STRING,
  spendby   STRING,
  txndate   DATE,
  txnts     TIMESTAMP
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE
LOCATION '/user/hduser/hiveexternaldata';
```

---

## 🔁 Insert Transformed Data (ETL: Extract, Transform, Load)

```sql
INSERT INTO TABLE retail_curated.externaltxnrecords (
  txnno, custno, amount, category, product, city, state, spendby, txndate, txnts
)
SELECT
  txnno,
  custno,
  amount,
  category,
  product,
  city,
  state,
  spendby,
  FROM_UNIXTIME(UNIX_TIMESTAMP(txndate, 'MM-dd-yyyy'), 'yyyy-MM-dd'),
  FROM_UNIXTIME(UNIX_TIMESTAMP(txndate, 'MM-dd-yyyy'))
FROM retail_raw.txnrecords;


SELECT * FROM retail_curated.externaltxnrecords LIMIT 10;

```

---

## 📅 Date/Time Conversion Logic

**Conceptual Flow:**

```
06-26-2011 (String) → UNIX TIMESTAMP (e.g., 1309026600) → yyyy-MM-dd Format
```

### Examples:

```sql
SELECT FROM_UNIXTIME(UNIX_TIMESTAMP('06-26-2011','MM-dd-yyyy'), 'yyyy-MM-dd');

SELECT FROM_UNIXTIME(UNIX_TIMESTAMP('06-26-2011','MM-dd-yyyy'), 'yyyy-MM-dd HH:mm:ss');
```

---

## ⚡ Temporary Tables in Hive

### ✅ Step-by-Step: Create & Use a Temporary Table in Hive

#### 1. **Select or Create Database**

```sql
CREATE DATABASE IF NOT EXISTS retail_temp_demo;
USE retail_temp_demo;
```

#### 2. **Create a New Temporary Table**

```sql
CREATE TEMPORARY TABLE temp_summary_txn (
    custno INT,
    total_amount DOUBLE
)
STORED AS ORC;
```

🔸 This table is:

* **Session-scoped** – it vanishes when the Hive session ends.
* **Not stored in Hive Metastore** – no metadata persists.
* **No data written to HDFS** – used only in local/scratch area.


#### 3. **Insert Data from Existing Table**

(Assuming `retail_raw.txnrecords` exists)

```sql
INSERT INTO TABLE temp_summary_txn
SELECT custno, SUM(amount)
FROM retail_raw.txnrecords
GROUP BY custno;
```


#### 4. **Query the Temporary Table**

```sql
SELECT * FROM temp_summary_txn LIMIT 10;
```


### 💡 Why Use a Temporary Table?

#### ✅ Key Advantages:

* **Session-based**: Auto-deleted when the session ends — no cleanup required.
* **Fast**: Skips HDFS write/read, useful for staging or intermediate processing.
* **No Hive Metastore Impact**: Reduces load on catalog and avoids clutter.
* **Ideal for ETL steps**, testing, and summary aggregations.


### 🧠 Internal Notes

| Property              | Behavior                                                              |
| --------------------- | --------------------------------------------------------------------- |
| **Persistence**       | Exists only during the session                                        |
| **Storage Location**  | Temporary local directory (`/tmp/hive/`)                              |
| **Metadata Handling** | Not stored in Hive Metastore                                          |
| **Query Speed**       | Faster for reads/writes (no HDFS replication or catalog registration) |
| **Limitations**       | Cannot be shared across sessions or visible to other users            |



