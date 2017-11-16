# Connecting to SNAP 
In order to start your project you should have your SNAP environment setup and running ( see SNAP installer ) 

Once SNAP is running, connect to SNAP through beeline or any SQL UI like Squirrel or Notebooks like Jupyter/Zeppelin using 

`jdbc:hive2://<address of the machine where SNAP is running:PORT/default;auth=noSasl`

The noSasl option on our default demo installer. For various authentication options please refer to ::logging into SNAP.::

Once you are connected you are ready to issue commands to setup a Qube/Index in SNAP.

Setting up a Qube in a 3 step process

1. Define external tables to the datasets 
2. Define a star schema
3. Determine partitioning scheme
4. Define an OLAP index

## Defining external table to your datasets
SNAP can work with data in s3, HDFS or any filesystem. We can read any formats that Spark can read such as csv, parquet, Orc, JSON etc. 

Let us assume your Fact table is orders and data for orders exists in an s3 bucket name s3://data/orders. Typically your facts are partitioned as well and lets us assume it is partitioned by YYYYMM. Let us assume the dataset is in csv format

Use the schema for the dataset orders and define an external table as follows
```
`create table orders_fact 
( order_id integer,
order_date timestamp,
customer_id integer,
order_quantity integer,
discount double, 
netprice double
)
using csv 
options ( path “s3://data/orders” ) 
partitioned by year_month
`
```
Let us also assume that the goal is to build a SNAP Qube combining order facts with a customer dimension table. Data for the customer table is in s3://data/customers. Format is csv

Now create an external table on customers as follows

```
create table customers_dim
(
customer_id integer, 
customer_name string, 
customer_country string, 
customer_state string, 
customer_zip string, 
customer_balance double
)
using csv
options ( path "s3://data/customers" ) 
```

At this point we have created all the external tables required to be part of the SNAP Qube.

## Creating star schemas
Next let us establish the relationship between the fact and dimension table by defining the joins using the 

CREATE starschema statement

```
create star schema  on orders_fact
as many_to_one join of orders_fact with customers_dim on orders_fact.customer_id  = customers_dim.customer_id
```

The star schema identifies the tables that will be part of the Qube and the relationship between the tables. 



## Creating an OLAP Index
The final step for defining a Qube is to define the INDEX and choose the columns from the tables in the star schema that will be part of the Index. This is a logical modeling exercise that has to be done with care. 

1. You can choose all columns in all tables to be part of the Index.

```
create olap index orders_snap on orders_fact
**dimension** order_id is nullable nullvalue "0"
**timestamp dimension** order_date spark timestampformat "YYYYMMDD" is nullable nullvalue "20000101",
dimension orders_fact.customer_id is nullable nullvalue "0",
dimension customer_name is nullable nullvalue "NA", 
dimension customer_country is nullable nullvalue "NA",
dimension customer_state is nullable nullvalue "NA", 
dimension customer_zip is nullable nullvalue "NA", 
metric customer_balance aggregator doubleSum is nullable nullValue "0.0"
metric order_quantity aggregator  longSum is nullable nullValue "0",
metric discount aggregator doubleSum is nullable nullValue "0.0", 
netprice metric order_quantity aggregator  longSum is nullable nullValue "0"
```

2. You can choose subset of the columns to be part of the index. 
```
create olap index orders_snap on orders_fact
**timestamp dimension** order_date spark timestampformat "YYYYMMDD" is nullable nullvalue "20000101",
dimension customer_name is nullable nullvalue "NA", 
dimension customer_country is nullable nullvalue "NA",
dimension customer_state is nullable nullvalue "NA", 
dimension customer_zip is nullable nullvalue "NA", 
metric order_quantity aggregator  longSum is nullable nullValue "0",
metric discount aggregator doubleSum is nullable nullValue "0.0", 
netprice metric order_quantity aggregator  longSum is nullable nullValue "0"
```


Things to note: 

* dimensions are typically columns you group by or filter by 
* metrics : columns with integer or float that will be aggregated in queries. 

## Populating the Qube.

Now that we have defined the Qube ( Source table, Star Schema and Index) we are ready to insert data into the Qube. 

Rebuilding the Qube with no partitions. 

In some situations your facts are mainly snapshots and you may not partition the data. In this case you would insert data into the Qube as follows 

```
insert overwrite OLAP index **orders_snap** **of** **orders_fact**
```

Inserting or Updating the Qube as new data arrives or data is updated. 

New facts arrive periodically ( new orders in order_fact) or old orders can be updated ( change in order quantity ).
Dimensions can also change ( customer is now in a different zip code)

In these situations the Qube is updated as follows. 

1. New data in orders_fact 
	1. The fact table orders_fact is partitioned by YYYYMM . As data arrives everyday we will be overwriting the YYYYMM partition in the Qube. For example lets say you have new data in the original orders_fact table for 20170909 the command to insert data in to the Qube is as follows. 
	2. `insert overwrite OLAP index **orders_snap** **of** **orders_fact** partitions year_month='201709`
	3. You may wonder why we are rebuilding the Qube for the entire month even though data has changed only for one day ( 20170909). Since the fact partition and the Qube partition is on month basis we will rebuild the Qube for the month. 
	4. NOTE: This operation is a few minutes even for 100s of millions of rows and so a rebuild is not an expensive operation. 
2. Updates to old facts or dimensions 
		1. When a dimension column changes such as zip code for the customer there are two ways to handle this.
			1. Type 1 changes
				1. Updates are made on the dimension table and no history is maintained. 
				2. If any changes to customer zip code should be reflected in reports for past facts then the index has to be rebuilt
					1. OR 
					2. The column should not be part of the index.  ( ::see section on joining Qubes to other tables:: )
			2. Normal changes ( Type 2 )
				1. When a zip code for a customer changes any new facts that come in for that customer will be associated with that zip code and old facts for that customer will be associated with the old zip code. 
		2. When old facts are updated 
			1. Sometimes facts are restated. For example in our case if the current month partition is 201709 and if there is an update to old data from 201706 this can be processed by re processing the partition for the Qube as follows 
			. `insert overwrite OLAP index **orders_snap** **of** **orders_fact** partitions year_month='201706`