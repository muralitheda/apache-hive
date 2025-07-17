Hive Query(HQL):				
$	hive			
				
hive>	set hive.cli.print.current.db=true;			
				
hive>	set hive.cli.print.header=true;			
				
hive>	"show databases;
show databases like '*data*';"			
				
hive>	"create database if not exists newdatabase_1 
comment 'batch database' 
location '/user/hduser/newdatabase_1' 
with dbproperties ('created_by' = 'jack', 'when created' = '28-DEC-2024');"			
				
hive>	"drop database newdatabase_1;
drop database newdatabase_1 cascade;"			
				
hive>	use newdatabase_1;			
				
hive>	"desc database newdatabase_1;
desc database extended newdatabase_1;"			
				
hive>	show tables;			
				
$	hadoop fs -ls /user/hduser/newdatabase_1			
				
hive>	"#ETL
#Semi Structured Data => {""id"":1,""age"":""thirty three"",""name"":""udhayakumar""}
#Load Time Parsing (converting semi structure data to structure)
#Load Time Cleaning/reformatting (cleaning the udhayakumar as udhayakuma, transforming thirty three to 33)

create table we45tbl (id int, name varchar(10), age smallint);

insert into we45tbl (id,name,age) values (1,""udhayakuma"",30);

insert into we45tbl (id,name,age) values (2,""udhayakumar"",""thirty three""); ---- Hive accept invalid datatype data as well

select * from we45tbl;
1        udhayakuma        30
2        udhayakuma        NULL"			
hive>	"#EL T(Query time)
#Semi Structured Data => {""id"":1,""age"":""thirty three"",""name"":""udhayakumar""}
#Load as it is and do query time parsing (converting semi from structure)

CREATE EXTERNAL TABLE we45jsontbl(id int,name varchar(10),age smallint)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES (""case.insensitive"" = ""false"")
STORED AS TEXTFILE
LOCATION '/user/hduser/newdatabase_1/we45jsontbl';

LOAD DATA LOCAL INPATH ""/home/hduser/jsondata.json"" INTO TABLE we45jsontbl;
$hadoop fs -cat /user/hduser/newdatabase_1/we45jsontbl/jsondata.json

SELECT * FROM we45jsontbl; --ERROR due to data format is different

#Iteration 1 (same file)
CREATE EXTERNAL TABLE we45jsontbl_1(id int,name varchar(100),age smallint,city string)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES (""case.insensitive"" = ""false"")
STORED AS TEXTFILE
LOCATION '/user/hduser/newdatabase_1/we45jsontbl';

SELECT * FROM we45jsontbl_1; --ERROR due to data format is different

#Iteration 2 (same file)
CREATE EXTERNAL TABLE we45jsontbl_2(id int,name varchar(100),age varchar(10),city string)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES (""case.insensitive"" = ""false"")
STORED AS TEXTFILE
LOCATION '/user/hduser/newdatabase_1/we45jsontbl';

SELECT * FROM we45jsontbl_2; --PARTIAL success due to data format is matched

#Iteration 3 (same file)
CREATE EXTERNAL TABLE we45jsontbl_3(id int,name varchar(100),age varchar(100),city string)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES (""case.insensitive"" = ""false"")
STORED AS TEXTFILE
LOCATION '/user/hduser/newdatabase_1/we45jsontbl';

SELECT * FROM we45jsontbl_3; --FULLsuccess due to data format is matched"			
				
Tables	Tables are container/referrer to data in a form of rows and columns			
Sample data	"1,
irfan,
43,
i am an IT professional with 2 kids residing in chennai with an experience of 20 years...,
139 ~ 5th main road ~ AGS Colony ~ Velachery,
TN,
5.11,
102,
1982-01-29,
2024-12-28 11:08:01,
true,
110000000123410"			
				
	create database if not exists we45database;			
	use we45database;			
				
Data Discovery	"create table custinfo (
id int,
name varchar(100),
age tinyint,
comments string,
address string,
statecd char(2),
height float,
weight smallint,
dob date,
login_ts timestamp,
isactive boolean,
acct_no bigint)
row format delimited fields terminated by ',';"			
				
	load data local inpath '/home/hduser/datatypedata.txt' into table custinfo;			
				
Regularly used datatype	"create table custinfo_simple (
id int,
name string,
age int,
comments string, 
address string,
statecd string,
height float,
weight int,
dob date,
login_ts timestamp,
isactive string,
acct_no bigint)
row format delimited fields terminated by ',';"			
				
Simple Syntax	"create table tablename (colname datatype,colname datatype ...)
row format delimited fields terminated by '|';

show create table tablename;"			
				
	(If we learn then we can say we completed hive learning)			
Complete Syntax	"CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.] table_name
[(col_name data_type [COMMENT col_comment], ...)]
[COMMENT table_comment]
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
[CLUSTERED BY (col_name, col_name, ...) [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
[ROW FORMAT row_format]
[STORED AS file_format] | STORED BY 'storage.handler.class.name' [WITH SERDEPROPERTIES (...)]]
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
[AS select_statement];"			"*External Table - will be used to refer the other hadoop location.

*Column/Table level COMMENT will be helpful for data governance and sensitivity masking.

*Default row_format delimeter is Control A.

*Stored By option will be used to connect different sql dbs and nosql dbs to get the data."
				
	"#Syntax to create a table if not exist in the database we45database with column comments, table comments, delimiter as ~, 
#Storage/retrieval format as orc with data pointing/refer to a location created by some tool/some other hive table...
#We can use location to refer some data already in place or if the data is expected in the given location later"			
	"CREATE TABLE IF NOT EXISTS we45database.sampletable5
(col_id1 int COMMENT 'this column holds id value',col_name2 string COMMENT 'this column holds name value')
COMMENT 'This table holds customer info'
row format delimited fields terminated by '~'
stored as orc
location '/user/hduser/warehouse/we45database.db/sampletable5';"			
	"INSERT INTO sampletable5 VALUES (1, 'MURALI');
INSERT INTO sampletable5 VALUES (2,'JACK');"			
				
	"CREATE TABLE IF NOT EXISTS we45database.sampletable4
(col_id1 int COMMENT 'this column holds id value',col_name2 string COMMENT 'this column holds name value')
COMMENT 'This table holds customer info'
row format delimited fields terminated by '~'
stored as textfile
location '/user/hduser/warehouse/we45database.db/sampletable4';"			
				
"When do we use, 
what caluse options:"	"schema/database_name - prefix if we want to create/access the table present in some database like absolute or relative path concept in linux.
IF NOT EXISTS - for automation script running on daily basis
COMMENT - info/details about table/column for data governance/stewardship/discovery/catalog/classification/business glossaries etc.,
ROW FORMAT delmited - Help us make hive understand (columns wise data need split while reading and writing) the pattern of data wrt columns, lines and complex datatypes also (we learn later)
STORED AS file_format - by default it uses textfile, but if we want hive to store and read the data in some special formal - orc/parquet/avro/rc/seq/json/xml/html/yaml...
LOCATION - Used for loading/refering/point the data is already present in a location or used for creating a table pointing to some external location where we expect data later."			