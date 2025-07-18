
	
	
	1. Create landing pad /home/hduser/retaildata
	"Landing pad creation:
mkdir /home/hduser/retaildata
cp txns /home/hduser/retaildata/retail-txns.csv

Sample data:
00000000,06-26-2011,4007024,040.33,Exercise & Fitness,Cardio Machine Accessories,Clarksville,Tennessee,credit
00000001,05-26-2011,4006742,198.44,Exercise & Fitness,Weightlifting Gloves,Long Beach,California,credit

Data Discovery(column headers):
record_id, transaction_date, customer_id, amount, sports_category, sports_type, city, state, payment_type"
	
	2. Copy the data to Transient/Zone
	"hadoop fs -mkdir /user/hduser/retaildata
hadoop fs -put -f /home/hduser/retaildata/retail-txns.csv /user/hduser/retaildata/retail-txns.csv"
	
	3. Create retail_raw, retail_curated, retail_analytical databases
	"drop database if exists db_retail_raw cascade;
drop database if exists db_retail_curated cascade;
drop database if exists db_retail_analytical cascade;

create database if not exists db_retail_raw comment 'retail raw database' location '/user/hduser/db_retail_raw' with dbproperties ('created_by' = 'murali', 'when created' = '08-JAN-2025');
create database if not exists db_retail_curated comment 'retail curated database' location '/user/hduser/db_retail_curated' with dbproperties ('created_by' = 'murali', 'when created' = '08-JAN-2025');
create database if not exists db_retail_analytical comment 'retail analytical database' location '/user/hduser/db_retail_analytical' with dbproperties ('created_by' = 'murali', 'when created' = '08-JAN-2025');

desc database db_retail_raw;
desc database db_retail_curated;
desc database db_retail_analytical; "
	
	4. Create the respective tables as per the diagram with some proper name in the respective databases and propogate the data till the last table.
	"
Raw Zone:

create table db_retail_raw.tbl_retail_raw_staging (record_id int, transaction_date string, customer_id int, amount double, sports_category string, sports_type string, city varchar(300), state varchar(300), payment_type varchar(300))
row format delimited fields terminated by ','
location '/user/hduser/retaildata';

select * from db_retail_raw.tbl_retail_raw_staging limit 5;

Curated Zone: 

[INFO] Creating it as an external table so that external systems can access this curated data.
create external table db_retail_curated.tbl_retail_curated_final (record_id int, transaction_date date, customer_id int, amount double, sports_category string, sports_type string, city varchar(300), state varchar(300), payment_type varchar(300))
row format delimited fields terminated by '|'
location '/user/hduser/retaildata_curated';

insert into db_retail_curated.tbl_retail_curated_final (record_id, transaction_date, customer_id, amount, sports_category, sports_type, city, state, payment_type)
select record_id, cast(to_date(from_unixtime(unix_timestamp(transaction_date, 'MM-dd-yyyy'))) as date), customer_id, amount, sports_category, sports_type, city, state, payment_type from db_retail_raw.tbl_retail_raw_staging;

select * from db_retail_curated.tbl_retail_curated_final limit 5;

Analytical Zone:

---1. [User Reporting] Table with only 'New York' state records with file format as ORC for faster access

create table db_retail_analytical.tbl_retail_analytical_newyork_state_records stored as orc as select * from db_retail_curated.tbl_retail_curated_final where state='New York'; 
select * from  db_retail_analytical.tbl_retail_analytical_newyork_state_records limit 5;

---2. [User Reporting] Table with amonut > 100 records 

create table db_retail_analytical.tbl_retail_analytical_amount_gt_100_records as select * from db_retail_curated.tbl_retail_curated_final where amount >100; 
select * from  db_retail_analytical.tbl_retail_analytical_amount_gt_100_records  limit 5;

[Error 10070]: CREATE-TABLE-AS-SELECT cannot create external table
create external table db_retail_analytical.tbl_retail_analytical_amount_lt_100_records as select * from db_retail_curated.tbl_retail_curated_final where amount <100; 

[Workaround]: create table first and then load the data
create external table db_retail_analytical.tbl_retail_analytical_amount_lt_100_records like db_retail_curated.tbl_retail_curated_final; 
insert into db_retail_analytical.tbl_retail_analytical_amount_lt_100_records select * from db_retail_curated.tbl_retail_curated_final where amount <100; 

select * from  db_retail_analytical.tbl_retail_analytical_amount_lt_100_records  limit 5;

describe formatted db_retail_analytical.tbl_retail_analytical_amount_lt_100_records;

[Altering Table Properties]
alter table db_retail_analytical.tbl_retail_analytical_amount_lt_100_records set tblproperties('EXTERNAL'='FALSE'); 

--3. [Analytical] Sports Category wise sales report
select sports_category, sum(amount) total_amount from db_retail_curated.tbl_retail_curated_final group by sports_category;

--4. [Analytical] Sports Category wise sales report and write results into hdfs storage
hadoop fs -mkdir /user/hduser/retaildata_analytical

insert overwrite directory '/user/hduser/retaildata_analytical'
row format delimited fields terminated by '|'
select sports_category, sum(amount) total_amount from db_retail_curated.tbl_retail_curated_final group by sports_category;

hadoop fs -cat /user/hduser/retaildata_analytical/*

--5. [Analytical] Sports Category wise sales report and write results into linux storage
mkdir /home/hduser/retaildata_analytical

insert overwrite local directory '/home/hduser/retaildata_analytical'
row format delimited fields terminated by '|'
select sports_category, sum(amount) total_amount from db_retail_curated.tbl_retail_curated_final group by sports_category;

cat /home/hduser/retaildata_analytical/*"
	
	5. Finally, we will do the cleanup work
	"--Since it is a managed table, it will drop the table metadata as well as actual data in hdfs
--Also, this raw table data moved to an external curated table db_retail_curated.tbl_retail_curated_final for any detailed analysis

drop table db_retail_raw.tbl_retail_raw_staging; "