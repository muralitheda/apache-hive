
## 🧬 Hive Data Types

Hive supports two major categories of data types:

---

### 🔹 1. Primitive Data Types

| **Category**  | **Data Type** | **Size**      | **Range / Description**                                 |
| ------------- | ------------- | ------------- | ------------------------------------------------------- |
| Numeric       | `TINYINT`     | 1 byte        | -128 to 127                                             |
|               | `SMALLINT`    | 2 bytes       | -32,768 to 32,767                                       |
|               | `INT`         | 4 bytes       | -2,147,483,648 to 2,147,483,647                         |
|               | `BIGINT`      | 8 bytes       | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |
|               | `FLOAT`       | 4 bytes       | Single-precision floating point                         |
|               | `DOUBLE`      | 8 bytes       | Double-precision floating point                         |
|               | `DECIMAL`     | Variable      | Fixed precision and scale numbers                       |
| Date/Time     | `DATE`        | 4 bytes       | Calendar date (YYYY-MM-DD)                              |
|               | `TIMESTAMP`   | 8 bytes       | Date and time (with optional timezone)                  |
|               | `INTERVAL`    | Variable      | Duration between dates/times                            |
| String        | `STRING`      | Up to 2 GB    | UTF-8 variable-length string                            |
|               | `VARCHAR(n)`  | Up to n chars | Variable-length string with max length `n`              |
|               | `CHAR(n)`     | Fixed size    | Fixed-length string with length `n`                     |
| Miscellaneous | `BOOLEAN`     | 1 byte        | `TRUE` or `FALSE`                                       |
|               | `BINARY`      | Variable      | Raw bytes (binary data)                                 |

---

### 🔸 2. Collection / Complex Data Types

| **Data Type** | **Description**                                    | **Analogy**                              |
| ------------- | -------------------------------------------------- | ---------------------------------------- |
| `ARRAY<T>`    | Ordered collection (list) of elements of same type | Like an array or list in other languages |
| `MAP<K, V>`   | Unordered set of key-value pairs                   | Like a dictionary or hash map            |
| `STRUCT`      | Collection of fields with different data types     | Like a record or a class                 |
| `UNIONTYPE`   | Holds a value from a set of specified types        | Similar to union in C/C++                |

---

### 📘 Notes

* Use **primitive types** for basic columns and **complex types** for nested or hierarchical data.
* Complex types are especially useful when working with semi-structured data like JSON or XML.
* Hive uses **SerDe (Serializer/Deserializer)** libraries to parse complex types from storage formats like ORC, Parquet, Avro, etc.

---
