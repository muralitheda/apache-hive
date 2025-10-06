

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

* These settings last **only for the current HQL session**.

### 2. Job / Oozie Level

* Modify `hive-site.xml` directly and submit the job via Oozie or other workflow tools.
* Settings are applied **for jobs using that config**.

### 3. Cluster / Admin Level (Permanent)

* Admin can change configs cluster-wide using:

  * `hive-site.xml` on all nodes
  * Cluster management tools like **Ambari, Cloudera Manager, EMR Manager, Dataproc Manager**

---

