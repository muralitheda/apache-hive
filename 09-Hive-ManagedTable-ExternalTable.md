
# Managed vs External Tables in Hive

## 🗃️ Managed Table (Container)

* **Syntax**:

  ```sql
  create table tblname (colname datatype);
  ```
* **Location**:
  Not required as Hive manages it (though you *can* specify it).
* **Drop Behavior**:
  Dropping the table deletes **both schema and data**.
* **Data Access**:
  Hive loads and accesses the data directly.
* **Support**:

  * Can be **TRUNCATED**
  * Can use **CTAS (Create Table As Select)**

---

## 📂 External Table (Referrer)

* **Syntax**:

  ```sql
  create external table tblname (colname datatype) LOCATION '/some/location';
  ```
* **Location**:
  Usually specified as the user manages the location (though optional).
* **Drop Behavior**:
  Dropping the table deletes **only schema**, not the data.
* **Data Access**:
  Data can be accessed by Hive and external systems.
* **Support**:

  * **Cannot be TRUNCATED**
  * **CTAS is not allowed** directly

---

## 🔁 Changing Table Type (Managed ⇄ External)

```sql
alter table saravanakumar set tblproperties('EXTERNAL'='FALSE');
```

---

## 🔧 Hive Settings & Database Setup

```sql
set hive.cli.print.current.db=true;
set hive.cli.print.header=true;

drop database if exists retail cascade;
create database retail;
use retail;
```

### Create Required Databases

```sql
drop database if exists retail_raw;
drop database if exists retail_curated;
drop database if exists retail_analytics;

create database retail_raw;
create database retail_curated;
create database retail_analytics;
```

---

## 📥 Managed Table: Load and Query Data

```sql
create table retail_raw.txnrecords (
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
row format delimited
fields terminated by ','
stored as textfile;
```

### Load Data

```sql
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/txns'
INTO TABLE retail_raw.txnrecords;
```

**Sample Record:**

```
00000000,06-26-2011,4007024,040.33,Exercise & Fitness,Cardio Machine Accessories,Clarksville,Tennessee,credit
```

### Query Data

```sql
select * from retail_raw.txnrecords limit 10;
select * from retail_raw.txnrecords order by txnno limit 10;
```

---

## 📤 External Table: Create and Transform

```sql
create external table retail_curated.externaltxnrecords (
  txnno INT,
  custno INT,
  amount DOUBLE,
  category STRING,
  product STRING,
  city STRING,
  state STRING,
  spendby STRING,
  txndate DATE,
  txnts TIMESTAMP
)
row format delimited
fields terminated by ','
stored as textfile
location '/user/hduser/hiveexternaldata';
```

---

## 🔄 Insert Transformed Data

```sql
insert into table retail_curated.externaltxnrecords (
  txnno, custno, amount, category, product, city, state, spendby, txndate, txnts
)
select 
  txnno,
  custno,
  amount,
  category,
  product,
  city,
  state,
  spendby,
  from_unixtime(unix_timestamp(txndate,'MM-dd-yyyy'),'yyyy-MM-dd'),
  from_unixtime(unix_timestamp(txndate,'MM-dd-yyyy'))
from retail_raw.txnrecords;
```

---

## ⏳ Date Format Explanation

| Source     | UNIX Format | Hive Format |
| ---------- | ----------- | ----------- |
| 06-26-2011 | 1309026600  | 2011-06-26  |

```sql
select from_unixtime(unix_timestamp('06-26-2011','MM-dd-yyyy'),'yyyy-MM-dd');
select from_unixtime(unix_timestamp('06-26-2011','MM-dd-yyyy'),'yyyy-MM-dd HH:mm:ss');
```

---

## 🧪 Temporary Tables

* Temporary tables live only for the **duration of the session**.
* Useful for **interim or scratch data**.
* Can be **managed** or **external**.
* Dropped automatically after session ends.

```sql
drop table temptable;
quit;
```

