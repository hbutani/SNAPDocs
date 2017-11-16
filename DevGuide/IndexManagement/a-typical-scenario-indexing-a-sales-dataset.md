<!-- --- title: A Typical Scenario: indexing a sales dataset -->
Consider that you have a DataSet about Sales Facts, where for each Sales Entry has detailed information about the Part, Supplier, Customer and Order. For those aware of theTPCH Benchmark the example schema here is built from that DataSet. Assume that you get a monthly dump of Sales Facts, and that your starting point is Data for the past 5 years. Based on your Analytics workloads you have decided that SparklineData SNAP platform has the potential to provide significant acceleration and management benefits. So you want to try it out. Let’s assume your Source Dataset has the following structure

```
CREATE TABLE if not exists sales_fact(
      o_orderkey integer, o_orderstatus string, o_clerk string, 
      o_shippriority integer,  o_totalprice double, 

      l_returnflag string, l_linestatus string, l_shipdate string, 
      l_commitdate string, l_receiptdate string, l_shipinstruct string, 
      l_shipmode string, l_quantity double, l_extendedprice double, 
      l_discount double, l_tax double,

      p_name string,p_mfgr string, p_brand string, 
      p_type string, p_size integer, p_container string,
      p_retailprice double,

      s_phone string, s_acctbal double, s_nation string,
      s_region string,  ps_availqty integer, ps_supplycost double, 

      c_name string , c_address string , c_phone string , 
      c_acctbal double,

      ship_year string, ship_month string
)
USING com.databricks.spark.csv
OPTIONS (path "sales_fact",
         header "false", 
         delimiter "|"
        )
partitioned by (shipYear, shipMonth)
```

The Dataset:

* is at the **grain **of individual Sales LineItems.
* Each LineItem has information about the Order, LineItem, Part, Supplier and Customer and several Timestamps about this Sales Item.
* we are capturing the Price, Quantity, Discount, Tax and other metrics about each LineItem.
* we also have recorded other metrics with this Fact, that are measurements about the Dimension/Context of this Fact: like the Size and Price of a Part, the Account Balance of a Supplier etc. This is a common technique of flattening all information about a Fact to the grain of the Fact.
* The DataSet is partitioned by ShipYear and ShipMonth. The primary reasoning a partition scheme is to facilitate the process to consuming new Facts. In our scenario assume we get a dump of new Facts at the end of every month that were shipped in that month. We could have secondary partition schemes such as further partitioning by Supplier in our case. The secondary partition scheme is driven by a combination of how data is received and of impact on Query Performance\(using Partition Pruning\).

**Alternate Scenarios/Not Covered Details**

```
1. Though we will not talk about it in our  example, it is ok to have the data further partitioned by say
   supplier. The decision should be driven by how you want to design your Index update process.

2. In our example we are describing the Indexing of 'Flattened' Dataset. We support Indexing of a
   Star/SnowFlake Schema. You need to describe the Star Schema before creating the OLAP Index. We have
   several examples of this on our wiki. In our example here we describe the Indexing processes for a
   Flattened Dataset, because the Star Schema doesn't add any new conceptual points but makes our SQLs
   very long.
```

## Defining the OLAP Index.

Now let’s get into the details of defining an OLAP Index for this DataSet.

### What to Index

The first thing to decide is: What is the Grain of your Index and what information is included in the Index. We require that the Index be at the same grain as the Facts\(support for different Indexing Grains will be provided through our Cube Management system in a future release\).

You can choose to exclude attributes/columns from the Index: a good example of this is a **Comment** field; slice-and-dice workloads typically don’t involve analysis of comments. But removal of a field should not change the grain of the Index. For example removing the partName from the Index in our example will mean that multiple lineItems in an order Parts with the same brand will be combined into a single row. **We don’t enforce this restriction: that the Index is at the grain of Facts, but Query Rewrites assume this is true.**

### The Index Partition scheme

Let us go over a couple of Scenarios for our example:

**Year based Indexing**

For our example scenario you realize that Index Segments at the Month level are not optimal. So you setup the Index to have a shipYear partition scheme.

* the initial Indexing goes great because the data for the initial years are not broken down by Month, and hence the Segments have good sizes, the Queries are running fast.
* now as new data arrives every Month, the new Segments are small and this smallness effect can accumulate over time.
  * we recommend you periodically, say every Quarter re-index the current Year’s data; so that the Segments are created with optimal size. In a future release we will provide automation tools for such a process, but for now you need to manage this on your known.
  * apart from manual management, the other potential issue is handling restated facts: what if you realize that the data from 3 months back was incorrect and had to be reloaded into the system. Now the state of the Index may require you to re-index a Year’s worth of Data.

**Supplier based Indexing**

Based on your Query Workload you realize that most Analysis is done at a Supplier and Geographic level, so partitioning the Index by Supplier Region has a huge benefit on Query Processing. You also check that there is no issue with small Segments, for each Region we get large amounts of Sales Facts every month. You could setup your Index partitioned by supplier-region; now every month each supplier-region index partition receives new Segments.

The issue arises on how to handle ‘restating’ events. If we realize that a particular Month’s data needs to be reloaded, there is no way to re-index only these facts. At least based on the SQL verbs we provide\(insert olap index\). These restating facts events would be very costly from an Index maintenance perspective. We should point out here that a key point of an OLAP index is to support arbitrary slice-and-dice: so the physical partition scheme should not have a major Query penalty/boost on any Query performance: i.e. even without Partition Pruning Query Performance should be very good if the Query is on a small slice of the Dataset. Net-net choosing an Index partition scheme is possible, but beware of Index maintenance cost and the boost from a custom scheme may not be as much as you think.

**For now we decide to go with shipYear, shipMonth partitioning; matching the partition scheme of the Source Dataset.**

### Index Segment Size and Indexing Job Performance

Ok now that we have settled on what to Index and how to partition the Index, let’s get into the details of the Indexing Jobs. We need the Index Segments to be reasonably sized and we want both the initial Indexing Job and periodic Jobs to utilize the available cluster resources well.

**Deciding on Index Segment Sizes**

So let’s say we have 5 years worth of data. We need to gather the following information:

###### storage size

This can be gathered by issuing a

`hdfs dfs -count`

command. Find out the total size and the per partition size.

###### number of rows

Record the total number of rows and the avg. number of rows per partition.

###### reduction in size when indexed

This is a measure of the size of the index relative to the source size. The best way to do get this is to index a sample of the source, say around 1-5GB of data with at least a million rows.

So say in our case, the parameters have the following values:

```
total_storage_size=120GB
avg_storage_per_partition=2GB
total_number_rows=120 million
avg_rows_per_partition=2 million
index_reduction_percent=30%
```

Given this, you can estimate:

```
total_index_size=84GB
avg_index_size_per_partition=1.4GB

source_per_row_size=1000 bytes

-- assuming segments of about 350mb
total_number_of_segments=240
avg_number_segments_per_partition=4
number_of_rows_per_segment=500K
```

Given these estimates, we configure an Indexing Job like this:

* based on the insert statement’s partition specification\(or lack of it\) we know how many partitions are being indexed.
* from this we can estimate the number of segments that are being generated.
* this is set as the parallelism of the job.
* before the Indexing creation Stage\(the last stage\) in the Job the data is redistributed based on the partition columns of the Index, and is ordered by them and the timestamp column of the Index\(if the Index has a timestamp\). This ensures an Indexing Task handles only a few\(ideally 1\) segment creation. The ordering ensures that at any given time the Index creation task is only processing 1 segment, so the memory footprint per task is fairly constant for its entire duration.

So when defining an OLAP Index you should provide the`avgSizePerPartition,avgNumRowsPerPartition,preferredSegmentSize`If you don’t specify these, we will try to estimate these based on the source storage size and an estimate of avg. row size based on datatypes. But this will be very error prone, so we strongly recommend that the above parameters be set during index creation.

![](/assets/Indexing.png)

In addition during each insert statement\(Indexing Job\) you should set a system parameter:`spark.sparklinedata.spmd.num_partitions_indexed`to indicate how many Index partitions are being inserted into. Again we will try to estimate this from the insert partition clause and the source partitions, but this will assume that the index has the same partitions as the source DataSet. In cases this is not true, you should set this parameter or the Index Segments may not have optimal sizes.

![](/assets/IndexingJob.png)

Finally the`create olap index`is defined as:

```
create olap index sales_fact_index on sales_fact
       timestamp dimension l_shipdate spark timestampformat "iso"
                 is index timestamp
                 is nullable nullvalue "1992-01-01T00:00:00.000"
      dimensions "o_orderkey,o_orderstatus,o_clerk,o_shippriority,l_returnflag,l_linestatus,l_commitdate,l_receiptdate,l_shipinstruct,l_shipmode,p_name,p_mfgr,p_brand,..."
      metrics "o_totalprice,l_quantity,l_extendedprice,l_discount,l_tax,..."
      OPTIONS (
        path "sales_fact_index",
        nonAggregateQueryHandling "push_project_and_filters",
        avgSizePerPartition  "1.4gb",
        avgNumRowsPerPartition "500000",
        preferredSegmentSize "350mb",
        rowFlushBoundary "100000"
      )
      partition by (shipYear, shipMonth)
```

Now when we issue the initial indexing job, set the number of partitions to =60=\(assuming there is 5 years worth of data to index\).

```
set spark.sparklinedata.spmd.num_partitions_indexed=60;
insert olap index sales_fact_index  of sales_fact;
```

Subsequently when indexing for some TimePeriod\(Month or Year\) worth of data set the partitions appropriately:

```
set spark.sparklinedata.spmd.num_partitions_indexed=1;
insert olap index sales_fact_index  of sales_fact
   partitions shipYear="2016" shipMonth="12"
;

-- or to re-index a year issue
 set spark.sparklinedata.spmd.num_partitions_indexed=12;
insert overwrite olap index sales_fact_index  of sales_fact
   partitions shipYear="2016"
;
```

**Alternate Scenarios/Not Covered Details**

* no partitioning
* Star Schema



