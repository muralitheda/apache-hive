
# Modern Hive: Working with Complex Data Types

In a modern data lakehouse architecture, handling semi-structured data efficiently is crucial. Hive's complex data types (ARRAY, STRUCT, MAP) provide powerful ways to represent and query nested data structures, often encountered in JSON, Avro, or other semi-structured sources.

## Understanding Complex Data Types

### ARRAY

An `ARRAY` is an **ordered sequence of similar elements**. It maintains an index to access its elements. This is analogous to a list in Python or an array in Java.

**Key Characteristics**:

  * Ordered: Elements retain their insertion order.
  * Indexed: Elements are accessed by their numerical position (0-based index).
  * Homogeneous: All elements within an array must be of the same data type.

**Example 1**: Array of strings for `day` elements
`day` =\> `['Sunday', 'Monday', 'Tuesday', 'Wednesday']`

  * `day[0]` =\> `Sunday`
  * `day[1]` =\> `Monday`
  * `day[2]` =\> `Tuesday`
  * `day[3]` =\> `Wednesday`

**Example 2**: Array of integers for `product_ids`
`product_ids` =\> `[101, 205, 310, 422]`

  * `product_ids[0]` =\> `101`
  * `product_ids[2]` =\> `310`

### STRUCT

A `STRUCT` is a **record type that holds a set of named fields (key-value pairs) that can be of different primitive or complex data types**. It's similar to a row in a relational table, a struct in C, or a dictionary/object with fixed keys.

**Key Characteristics**:

  * Named Fields: Each element within a struct has a name and a type.
  * Heterogeneous: Fields can be of different data types.
  * Accessed by Name: Elements are accessed using dot notation (`.`).

**Example 1**: `address` struct
`address` =\> `STRUCT {city STRING; state STRING, postalcode INT}`
Accessing fields:

  * `address.city`
  * `address.state`
  * `address.postalcode`

**Example 2**: `employee_info` struct
`employee_info` =\> `STRUCT {emp_id INT; emp_name STRING; emp_dept STRING}`
Accessing fields:

  * `employee_info.emp_id`
  * `employee_info.emp_name`

### MAP

A `MAP` data type contains **key-value pairs**. In a Map, elements are accessed using their associated keys. It is similar to a dictionary in Python or a hash map in Java.

**Key Characteristics**:

  * Key-Value Pairs: Each entry consists of a unique key and a corresponding value.
  * Accessed by Key: Elements are accessed using square brackets `[]` with the key.
  * Homogeneous Key/Value Types: All keys must be of the same data type, and all values must be of the same data type.

**Example 1**: `name` map
`name` =\> `'firstname': 'John' & 'last_name': 'roy'`
Accessing elements: `name['firstname']`

**Example 2**: `product_prices` map
`product_prices` =\> `'Laptop': '1200' & 'Mouse': '25' & 'Keyboard': '75'`
Accessing elements:

  * `product_prices['Laptop']`
  * `product_prices['Keyboard']`

-----

## Data Preparation for Demonstration

We'll start by preparing a raw CSV file that contains data with delimited elements suitable for parsing into complex types.

Create a file named `agent_complex_data.csv` in your Hive data directory:

```bash
vi ~/hive/data/agent_complex_data.csv
```

**Content for `agent_complex_data.csv`**:

```
1,Cliff,100000,SWE$Analyst$SAnalyst$PL,4$Castlepoint blvd$Piscatway$NJ$10043,Marital Status#M$Education#MS$Language#Eng,French
2,Joe,120000,SWE$Analyst$SAnalyst$PM,8$Camaron 3rd street$Queens$NY$20088,Marital Status#M$Education#BS$Language#Eng
3,Aravind,130000,SWE$Analyst$SAnalyst$PM,3$Doveton street$Purasai$CHN$600022,Marital Status#U$Education#BE$Dept#IT
4,Gokul,80000,SWE$Analyst,31$Mint Street$Broadway$CHN$600001,Marital Status#U$Education#BSc$Bond#Yes
```

-----

## Modern Hive Table Creation & Data Loading Workflow

A common modern practice is to load raw data into a staging table (often in text format) and then transform/insert it into an optimized, columnar format table (like ORC or Parquet) for better query performance.

### 1\. Database Setup

First, let's create and use a dedicated database for our analytics tables.

```sql
-- Create the database if it doesn't exist
CREATE DATABASE IF NOT EXISTS data_lakehouse;
USE data_lakehouse;

-- Set a property to show headers in query results (useful for CLI)
SET hive.cli.print.header=true;
```

### 2\. Create a Staging Table (Text Format)

This table will temporarily hold the raw CSV data.

```sql
CREATE TABLE data_lakehouse.raw_agents_csv
(
    agent_id INT,
    name STRING,
    sal BIGINT,
    desig_raw STRING,        -- Store raw string for ARRAY parsing
    address_raw STRING,      -- Store raw string for STRUCT parsing
    other_info_raw STRING,   -- Store raw string for MAP parsing
    misc_field STRING        -- To capture any unmapped trailing data
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';
```

### 3\. Load Data into Staging Table

```bash
LOAD DATA LOCAL INPATH '/home/hduser/hive/data/agent_complex_data.csv' OVERWRITE INTO TABLE data_lakehouse.raw_agents_csv;
```

### 4\. Create an Optimized Table with Complex Data Types (ORC/Parquet)

This is the final table where data will be stored efficiently using columnar formats, which are highly recommended for performance in modern Hive.

```sql
CREATE TABLE data_lakehouse.agents_profiles_orc
(
    agent_id INT,
    name STRING,
    sal BIGINT,
    desig ARRAY<STRING>,
    address STRUCT<dno:INT,street:STRING,area:STRING,city:STRING,pin:BIGINT>,
    other_info MAP<STRING,STRING>
)
STORED AS ORC
TBLPROPERTIES ('orc.compress'='SNAPPY'); -- Recommended: Use compression
```

**Explanation of `STORED AS ORC`**:

  * **Columnar Storage**: ORC (Optimized Row Columnar) stores data column by column, which is highly efficient for analytical queries as it only reads necessary columns.
  * **Compression**: `SNAPPY` is a fast, lightweight compression codec that balances compression ratio and CPU usage, ideal for big data.
  * **Performance**: ORC (and Parquet) tables support predicate pushdown and vectorization, significantly speeding up queries compared to text formats.

### 5\. Insert Data from Staging to Optimized Table

Now, we transform and load the data from the raw CSV staging table into our optimized ORC table, applying the complex data type parsing during the `INSERT`.

```sql
INSERT OVERWRITE TABLE data_lakehouse.agents_profiles_orc
SELECT
    agent_id,
    name,
    sal,
    SPLIT(desig_raw, '\\$') AS desig, -- Parse ARRAY from '$' delimited string
    NAMED_STRUCT(
        'dno', CAST(SPLIT(address_raw, '\\$')[0] AS INT),
        'street', SPLIT(address_raw, '\\$')[1],
        'area', SPLIT(address_raw, '\\$')[2],
        'city', SPLIT(address_raw, '\\$')[3],
        'pin', CAST(SPLIT(address_raw, '\\$')[4] AS BIGINT)
    ) AS address, -- Parse STRUCT from '$' delimited string
    STR_TO_MAP(other_info_raw, '\\$', '#') AS other_info -- Parse MAP from '$' and '#' delimited string
FROM
    data_lakehouse.raw_agents_csv;
```

**Note on `SPLIT` and `NAMED_STRUCT`**:

  * `SPLIT(string, delimiter)`: This built-in Hive function is used to convert a delimited string into an ARRAY. We use `\\$` to escape the `$` character in regex.
  * `NAMED_STRUCT(key1, value1, key2, value2, ...)`: This function is used to construct a STRUCT on the fly, naming its fields and assigning values.
  * `STR_TO_MAP(string, pair_delimiter, key_value_delimiter)`: This function converts a string into a MAP by specifying delimiters for pairs and for key-value within each pair.

-----

## Querying Complex Data Types

Once the data is in the `agents_profiles_orc` table, you can query its complex data types directly.

```sql
SELECT
    agent_id,
    name,
    sal,
    desig[0] AS first_designation,  -- Access ARRAY element by index
    desig[1] AS second_designation,
    address.dno AS house_number,    -- Access STRUCT field by name
    address.city AS live_city,
    address.pin AS postal_code,
    other_info["Marital Status"] AS marital_status, -- Access MAP value by key
    other_info["Language"] AS primary_language,
    other_info["Dept"] AS department,
    other_info["Bond"] AS in_bond
FROM
    data_lakehouse.agents_profiles_orc
WHERE
    agent_id = 1; -- Example filter
```

### Observing the Physical Storage

While modern Hive users interact mostly via SQL, it's good to understand how data is stored. After the `INSERT OVERWRITE` operation, you will find ORC files in your table directory.

```bash
dfs -ls -h /user/hive/warehouse/data_lakehouse.db/agents_profiles_orc/;
```

You'll see `.orc` files (e.g., `000000_0`, `000001_0_copy_1`, etc.) rather than simple text files. The number of files depends on the degree of parallelism (number of reducers) during the insert operation.

-----

## Modern Considerations and Best Practices

1.  **File Formats (ORC/Parquet)**: Always prefer columnar file formats like ORC or Parquet for tables storing large datasets. They offer:

      * **High Compression**: Saves storage space.
      * **Faster Reads**: Predicate pushdown (filtering at the storage level) and column pruning (reading only necessary columns) significantly reduce I/O.
      * **Vectorization**: Processes data in batches, leading to CPU efficiency.
      * **Schema Evolution**: Easier to add, remove, or modify columns without breaking existing queries.

2.  **Compression**: Always enable compression (e.g., Snappy, Zlib, Gzip) for your data files. This saves disk space and network I/O, improving overall performance.

3.  **Schema Evolution with Complex Types**: Complex types naturally support schema evolution. For example:

      * **Adding a new field to a STRUCT**: Older data will simply have a `NULL` for the new field.
      * **Adding a new key to a MAP**: New data will include it; older data will not.
      * **Appending to an ARRAY**: New data can have more elements.

4.  **Query Engines**: Hive tables with complex types and ORC/Parquet formats can be directly queried by other modern big data processing engines like Apache Spark SQL, Presto/Trino, and Impala, enabling a unified data access layer.

5.  **Small Files Problem**: Be mindful of generating too many small files, especially with text formats or certain bucketing/partitioning strategies. Columnar formats and proper data loading can help mitigate this.

By leveraging complex data types alongside optimized file formats, Hive remains a powerful component in a modern data analytics stack for handling diverse data structures.