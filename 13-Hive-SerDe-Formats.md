## Serialization & Deserialization Formats (SerDes) and Data Types in Hive

## Introduction to SerDes

1.  Serialization and deserialization formats are popularly known as **SerDes**.
2.  Hive allows the framework to **read or write** data in a particular format.
3.  These formats parse the **structured, semi-structured, or unstructured data bytes** stored in HDFS in accordance with the schema definition of Hive tables.
4.  Hive provides a set of **in-built SerDes** and also **allows the user to create custom SerDes** based on their data definition.

## Common Hive SerDes

  * Json Serde
  * Lazy Simple Serde
  * Csv Serde
  * Regex Serde
  * **Avro Serde**
  * ORC Serde
  * Parquet Serde

-----

## JSON SerDe Example

Let's demonstrate using the `JsonSerDe` to read JSON data directly from HDFS. This is often used for landing raw semi-structured data.

### 1\. Create Data File

Create a JSON file with a series of JSON objects (one per line).

```bash
vi ~/hive/data/agents.json
```

**Content for `agents.json`**:

```json
{"agent_id":1,"name":"Cliff","sal":100000,"desig":["SWE","Analyst","SAnalyst","PL"],"address":{"dno":4,"street":"Castlepoint blvd","area":"Piscatway","city":"NJ","pin":10043},"other_info":{"Marital Status":"M","Education":"MS","Language":"Eng"}}
{"agent_id":2,"name":"Joe","sal":120000,"desig":["SWE","Analyst","SAnalyst","PM"],"address":{"dno":8,"street":"Camaron 3rd street","area":"Queens","city":"NY","20088},"other_info":{"Marital Status":"M","Education":"BS","Language":"Eng"}}
{"agent_id":3,"name":"Aravind","sal":130000,"desig":["SWE","Analyst","SAnalyst","PM"],"address":{"dno":3,"street":"Doveton street","area":"Purasai","city":"CHN","pin":600022},"other_info": {"Marital Status":"U","Education":"BE","Dept":"IT"}}
{"agent_id":4,"name":"Gokul","sal":80000,"desig":["SWE","Analyst"],"address":{"dno":31,"street":"Mint Street","area":"Broadway","city":"CHN","pin":600001},"other_info":{"Marital Status":"U","Education":"BSc","Bond":"Yes"}}
```

### 2\. Database Setup

First, ensure you are using the correct database context.

```sql
CREATE DATABASE IF NOT EXISTS raw_landing_zone;
USE raw_landing_zone;
```

### 3\. Create External Table with JSON SerDe

We create an `EXTERNAL` table, meaning Hive manages the schema but the data files remain in their specified HDFS location. The `ROW FORMAT SERDE` clause specifies how Hive should interpret the data.

```sql
CREATE EXTERNAL TABLE raw_landing_zone.agents_json_raw_external
(
    agent_id INT,
    name STRING,
    sal BIGINT,
    desig ARRAY<STRING>,
    address STRUCT<dno:INT,street:STRING,area:STRING,city:STRING,pin:BIGINT>,
    other_info MAP<STRING,STRING>
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES ("case.insensitive" = "false")
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/raw_landing_zone.db/agents_json_raw_external'; -- Standardized HDFS location
```

### 4\. Load Data into the External Table

```bash
LOAD DATA LOCAL INPATH '~/hive/data/agents.json' OVERWRITE INTO TABLE raw_landing_zone.agents_json_raw_external;
```

### 5\. Show Create Table (Verification)

You can verify the table's DDL using `SHOW CREATE TABLE`.

```sql
SHOW CREATE TABLE raw_landing_zone.agents_json_raw_external;
```

**Expected Output of `SHOW CREATE TABLE` (with standardized names and paths)**:

```sql
CREATE EXTERNAL TABLE `agents_json_raw_external`(
  `agent_id` int COMMENT 'from deserializer',
  `name` string COMMENT 'from deserializer',
  `sal` bigint COMMENT 'from deserializer',
  `desig` array<string> COMMENT 'from deserializer',
  `address` struct<dno:int,street:string,area:string,city:string,pin:bigint> COMMENT 'from deserializer',
  `other_info` map<string,string> COMMENT 'from deserializer')
ROW FORMAT SERDE
  'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES (
  'case.insensitive'='false')
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://localhost:8020/user/hive/warehouse/raw_landing_zone.db/agents_json_raw_external' -- HDFS port might vary (e.g., 8020 default)
TBLPROPERTIES (
  'transient_lastDdlTime'='<timestamp>') -- Timestamp will vary
```

### 6\. Query the JSON Data

You can now query the `agents_json_raw_external` table, accessing its complex data types just like a regular Hive table.

```sql
SELECT
    agent_id,
    name,
    sal,
    desig[0] AS first_desig,
    desig[1] AS second_desig,
    address.dno,
    address.city,
    address.pin,
    other_info["Marital Status"] AS maritalstatus,
    other_info["Language"] AS language,
    other_info["Dept"] AS dept,
    other_info["Bond"] AS in_bond
FROM
    raw_landing_zone.agents_json_raw_external;
```

-----

## ORC SerDe Example (Optimized Columnar Format)

ORC (Optimized Row Columnar) is Hive's highly recommended and most efficient file format. It's a columnar storage format with built-in indexing and compression, leading to significant performance improvements for analytical queries. When you specify `STORED AS ORC`, Hive implicitly uses the `OrcSerde` for serialization and deserialization.

### 1\. Create ORC Table

We'll create an internal Hive table with the same schema as our JSON table, but specify `STORED AS ORC`.

```sql
CREATE TABLE raw_landing_zone.agents_profiles_orc
(
    agent_id INT,
    name STRING,
    sal BIGINT,
    desig ARRAY<STRING>,
    address STRUCT<dno:INT,street:STRING,area:STRING,city:STRING,pin:BIGINT>,
    other_info MAP<STRING,STRING>
)
STORED AS ORC
TBLPROPERTIES ('orc.compress'='SNAPPY'); -- SNAPPY is a good default for balancing compression and performance
```

**Key Difference**: You don't explicitly declare `ROW FORMAT SERDE 'org.apache.hadoop.hive.ql.io.orc.OrcSerde'` because `STORED AS ORC` handles it implicitly and correctly configures the table properties for ORC.

### 2\. Load Data into ORC Table

The most efficient way to load data into an ORC table from an existing Hive table is using `INSERT OVERWRITE` or `INSERT INTO`. This operation will trigger a MapReduce/Tez job that reads data from the source table (our JSON external table), serializes it into ORC format, and writes it to the new ORC table.

```sql
INSERT OVERWRITE TABLE raw_landing_zone.agents_profiles_orc
SELECT
    agent_id,
    name,
    sal,
    desig,
    address,
    other_info
FROM
    raw_landing_zone.agents_json_raw_external;
```

### 3\. Query the ORC Data

Now, query the `agents_profiles_orc` table. You should observe much faster query execution times compared to the `TEXTFILE` table for larger datasets, especially with filters or aggregations.

```sql
SELECT
    agent_id,
    name,
    sal,
    desig[0] AS first_designation,
    address.city AS live_city,
    other_info["Marital Status"] AS marital_status
FROM
    raw_landing_zone.agents_profiles_orc
WHERE
    sal > 100000;
```

-----

## Avro SerDe Example (Schema-Driven Format)

Avro is a row-oriented, schema-driven binary data format. It's excellent for data serialization, remote procedure calls, and robust schema evolution. Hive's AvroSerde allows you to read and write Avro data, leveraging the Avro schema for validation and efficiency.

### 1\. Create Avro Table

When creating an Avro table, Hive's `AvroSerde` needs to know the Avro schema. You can define the schema directly in the DDL (`avro.schema.literal`) or point to an external `.avsc` file (`avro.schema.url`). For simplicity in this example, we'll let Hive generate an inferred schema based on the DDL.

```sql
CREATE TABLE raw_landing_zone.agents_avro_profile
(
    agent_id INT,
    name STRING,
    sal BIGINT,
    desig ARRAY<STRING>,
    address STRUCT<dno:INT,street:STRING,area:STRING,city:STRING,pin:BIGINT>,
    other_info MAP<STRING,STRING>
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.avro.AvroSerde'
STORED AS AVRO;
```

**Note**: When `STORED AS AVRO` is used, Hive will try to automatically derive the Avro schema from the Hive table's DDL. For more complex scenarios or explicit control, you can use `TBLPROPERTIES ('avro.schema.literal'='{"type":"record",...}')` or `TBLPROPERTIES ('avro.schema.url'='hdfs:///path/to/your_schema.avsc')`.

### 2\. Load Data into Avro Table

Similar to the ORC example, we'll load data into the Avro table from our `agents_json_raw_external` table. Hive will handle the serialization from the textual JSON representation into the binary Avro format according to the Avro schema derived from `agents_avro_profile`'s DDL.

```sql
INSERT OVERWRITE TABLE raw_landing_zone.agents_avro_profile
SELECT
    agent_id,
    name,
    sal,
    desig,
    address,
    other_info
FROM
    raw_landing_zone.agents_json_raw_external;
```

### 3\. Query the Avro Data

Querying an Avro table is similar to querying any other Hive table.

```sql
SELECT
    agent_id,
    name,
    sal,
    desig[0] AS first_desig,
    address.area,
    other_info["Education"] AS education_level
FROM
    raw_landing_zone.agents_avro_profile
WHERE
    agent_id IN (1, 3);
```

**Benefits of Avro Format**:

  * **Self-Describing**: Avro files contain their schema within them, making them easy to read by any Avro-compatible application.
  * **Schema Evolution**: Avro provides robust rules for schema evolution, allowing you to add, remove, or modify fields in the schema while maintaining backward and forward compatibility with older and newer data.
  * **Compact Binary Format**: Avro data is stored in a compact binary format, which is efficient for storage and network transfer.
  * **Language Neutrality**: Avro schemas are JSON-based and data is language-agnostic, making it a popular choice for data interchange between systems written in different programming languages.

-----

## Comparison of Data Formats: JSON, Avro, ORC, and Parquet

Choosing the right file format is crucial for performance, storage efficiency, and interoperability in a big data ecosystem. Here's a comparison of the formats discussed:

| Characteristic         | JSON (TextFile)                                     | Avro                                                  | ORC                                                     | Parquet                                                   |
| :--------------------- | :-------------------------------------------------- | :---------------------------------------------------- | :------------------------------------------------------ | :-------------------------------------------------------- |
| **Storage Type** | Row-oriented (Human-readable text)                  | Row-oriented (Binary)                                 | Column-oriented (Binary)                                | Column-oriented (Binary)                                  |
| **Read Performance (Analytical)** | Poor (Requires full row scan, text parsing overhead) | Good for row-by-row, less optimal for analytical scans | **Excellent** (Optimized for analytical queries/scans)  | **Excellent** (Optimized for analytical queries/scans)    |
| **Write Performance** | Fast (Simple text write)                            | Good                                                  | Good (Overhead for columnar organization)               | Good (Overhead for columnar organization)                 |
| **Schema Evolution** | Flexible but schema-on-read; no strict enforcement | **Excellent** (Schema embedded, robust compatibility) | Good (Handles adding/dropping columns well)             | Good (Handles adding/dropping columns well)               |
| **Compression** | File-level (e.g., Gzip); can impact splitability    | Block-level (pluggable codecs)                        | **Column-level** (Highly effective, dictionary encoding) | **Column-level** (Highly effective, dictionary encoding)  |
| **Predicate Pushdown** | No (Requires full record parsing)                   | Limited (Requires deserialization of entire record)   | **Excellent** (Filters applied at storage level)        | **Excellent** (Filters applied at storage level)          |
| **Column Pruning** | No (Reads entire row)                               | No (Reads entire row)                                 | **Yes** (Reads only required columns)                   | **Yes** (Reads only required columns)                     |
| **Splitability** | Yes (with splittable codecs like Bzip2, LZO)        | Yes                                                   | Yes                                                     | Yes                                                       |
| **Data Types** | All (Parsed on read, less strict enforcement)       | All (Strong schema enforcement)                       | All (Optimized for Hive's internal types)               | All (Widely supported across ecosystems)                  |
| **Human Readability** | **High** (Plain text)                               | No (Binary)                                           | No (Binary)                                             | No (Binary)                                               |
| **Typical Use Cases** | Raw data landing, small datasets, debugging         | Data ingest/export, ETL pipelines, inter-process comm. | **Primary Hive storage**, data warehousing, OLAP        | **Primary Spark/Impala storage**, cross-platform data exchange |
| **Hive DDL (Example)** | `ROW FORMAT SERDE '...' STORED AS TEXTFILE`         | `ROW FORMAT SERDE '...' STORED AS AVRO`                 | `STORED AS ORC`                                         | `STORED AS PARQUET`                                       |