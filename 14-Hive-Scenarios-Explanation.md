# Hive Scenarios and Real-Time Questions

## 1. Why does a `SELECT *` query in Hive not run MapReduce/Tez/Spark?

Because of the **`hive.fetch.task.conversion`** property.
It allows Hive to skip MapReduce/Tez overhead for simple queries like `SELECT`, `FILTER`, and `LIMIT`, directly fetching data from storage.

### Values of `hive.fetch.task.conversion`

```sql
SET hive.fetch.task.conversion
```

* **none** – Always use MapReduce/Tez/Spark.
* **minimal** – Fetch optimization for very simple queries.
* **more** – Extended optimization for projections and filters (default in newer versions).

👉 For complex queries (joins, aggregations, UDFs), Hive will still launch MapReduce/Tez/Spark jobs.

