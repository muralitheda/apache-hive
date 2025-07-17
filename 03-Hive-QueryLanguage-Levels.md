## 🐝 Hive Query (HQL) Levels

Hive organizes and processes data through several hierarchical levels:

1. **Databases/Schemas**
2. **Tables**

   * Columns, Data Types, Table Types (Managed/External), SerDe Libraries
3. **Partitions**
4. **Buckets**

---

### 1. **Database**

A **database** in Hive is a container or namespace that holds a collection of related tables (and other objects like views and functions).

---

### 2. **Tables**

A **table** is a structured collection of records represented in **rows and columns**.

* **Managed Tables**: Hive manages both data and metadata.
* **External Tables**: Hive manages only metadata; data resides externally.

---

### 3. **Partitioned Tables**

Partitioning is a **horizontal division** of a table's data into **separate folders**, based on column(s) of **low cardinality** (i.e., few unique values). This improves performance of `WHERE` clause filtering by reducing the data scanned.

#### 📁 Example: Partition by `country` column

```
/user/hive/warehouse/sales/
    ├── country=USA/
    ├── country=India/
    └── country=Canada/
```

```sql
SELECT * FROM sales WHERE country = 'USA';
```

---

### 4. **Bucketed Tables**

Bucketing is the **distribution** of data into a fixed number of files (buckets) based on the **hash value** of a **high-cardinality** column. It optimizes:

* **Join performance**
* **WHERE clause filtering**
* **Random sampling of data**

#### 🧮 Example: Bucket by `customer_id` or `product_id`

How bucketing works:

1. Choose a bucket column (e.g., `customer_id`).
2. Define a number of buckets (e.g., 10).
3. Hive hashes the column’s value and assigns the row to a bucket.
4. Each bucket becomes a separate file (inside partition directories if partitioned).

---
