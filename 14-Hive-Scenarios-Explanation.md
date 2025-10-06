

## Q1. How to see the default value set for a config in Hive?

Run the `SET` command with the property name:

```sql
set hive.fetch.task.conversion;
```

**Answer:**

* It shows the current value (default or overridden).
* If not explicitly set in `hive-site.xml` or via `SET`, Hive returns the default value.

---

## Q2. Why does a `SELECT *` query in Hive not run MapReduce/Tez/Spark?

**Answer:**

* Controlled by `hive.fetch.task.conversion`.
* For simple queries (`SELECT`, `FILTER`, `LIMIT`), Hive skips MapReduce/Tez/Spark and directly fetches data.

**Values:**

* **none** – Always use MapReduce/Tez/Spark.
* **minimal** – Optimization only for very simple queries.
* **more** – Extended optimization for filters/projections (default).

---

## Q3. How to change Hive configs permanently within the current HQL session / cluster / job level?

**Answer:**

### 1. Session / Query Level (Temporary)

* Set properties for the current session only:

```sql
set hive.exec.engine=mr;
set hive.map.size=1024m;
set hive.vectorized.exec.enabled=true;
```

* Hive CLI / Script Level: `SET` command or `-hiveconf`
```bash
hive -hiveconf hive.exec.engine=mr -f my_query.hql
```

* These settings last **only for the current HQL session**.

### 2. Job / Oozie Level (Temporary)

* If you are using a job scheduler (like Oozie) that packages and executes your job, you can include a *custom `hive-site.xml`* containing only the necessary overrides.
* Settings are applied **for jobs using that config**.

### 3. Cluster / Admin Level (Permanent)

* Admin can change configs cluster-wide using:

  * `hive-site.xml` on all nodes
  * Cluster management tools like **Ambari, Cloudera Manager, EMR Manager, Dataproc Manager**

---

## Q4. Can I use multiple columns in a subquery `WHERE ... IN` clause in Hive?

**Answer:**

* **Hive does not support multi-column `IN` subqueries** like:

```sql
SELECT * FROM tableA
WHERE (A.id, A.name, A.Roll_no) IN (
    SELECT Id, name, roll_no FROM tableB
);
```

* **Workarounds:**

**Option 1: Use `CONCAT`**

```sql
SELECT * FROM tableA
WHERE CONCAT(A.id, A.name, A.Roll_no) IN (
    SELECT CONCAT(Id, name, roll_no) FROM tableB
);
```

**Option 2: Use `JOIN`**

```sql
SELECT a.*
FROM tableA a
INNER JOIN tableB b
ON a.id = b.id
AND a.name = b.name
AND a.rollno = b.rollno;
```

**Key:** Multi-column matching is supported via `JOIN` or `CONCAT`, not direct multi-column `IN`.

---

## Q5. How to skip header/footer rows from a table in Hive?

**Answer:**

* Use **table properties** when creating or altering the table:

```sql
-- Skip first 2 header rows
TBLPROPERTIES ("skip.header.line.count"="2");

-- Skip last 1 footer row
TBLPROPERTIES ("skip.footer.line.count"="1");
```

* Applies when reading data from **text/CSV files**.
* Useful for **ignoring file headers or summary/footer lines**.

**Key:** Only works for **external or managed tables with text-based formats**.

**Example:**

```sql
vi /home/hduser/students.csv
Name,Age,Class
John,15,10
Alice,14,9
Bob,16,11
Summary: 3 rows

hadoop fs -mkdir /user/hive/warehouse/students/
hadoop fs -put /home/hduser/students.csv /user/hive/warehouse/students/

--hive

DROP TABLE IF EXISTS default.students;

CREATE EXTERNAL TABLE default.students(
    Name STRING,
    Age INT,
    Class STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/students/'
TBLPROPERTIES (
    "skip.header.line.count"="1",
    "skip.footer.line.count"="1"
);

SELECT * FROM students;
John    15    10
Alice   14    9
Bob     16    11

```

✅ The header (`Name,Age,Class`) and footer (`Summary: 3 rows`) are **skipped automatically**.




---
