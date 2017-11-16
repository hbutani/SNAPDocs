<!-- --- title: Index Process Tuning -->

#### Overall Indexing Process\(from the perspective of CPU and Memory consumption\)

Indexing is a memory and computationally intensive process:

* Spark Rows are converted to a Index Row form
* In an initial pass, rows are ingested into an **Index Ingestion Form**
* On persist, the Index Ingestion Form is converted into a **Queryable Index Form**

#### Contributors to CPU and Memory consumption

Both the **Index Ingestion Form ** and the **Queryable Index Form **take up a lot of Memory and create a lot of ephemeral objects. Some of the main contributors to Object creations are:

* The conversion of Spark UTF8 Strings to java Strings
* Formatting of Spark Columns\(String or Timestamp or Long\) into an ISO format for the Index

#### Mini-Segments based on rowFlushBoundary: some speed-up, some level of memory consumption regulation

The number of rows in the Segment is obviously also a huge factor in the Memory footprint during Indexing, it also has a significant impact on inserting into the **Index Ingestion Form **and the persist operation. For this reason, the **rowFlushBoundary** can be set on an OLAP Index; this controls the batching of rows into mini-segments during Ingestion. The last step of Segment creation then requires the mini-segments to be merged into the final Segment.

This certainly helps when only 1 Indexing Task is running in a JVM; the final merge step across mini-segments still requires considerable memory, as all mini-segments must be in memory for the merge to happen.

The Golidlocks Rule applies to the **rowFlushBoundary **setting:

* too low and you get, too many segments to merge, which has slows down indexing because a lot more Queryable Index Forms are created, and the final merge step has to merge a lot more mini-segments
* too high, and the efficacy of mini-segments is nullified.
* Our experience is a rowFlushBoundary around 100k is generally a good number; though the usual cautionary statement applies here: the optimal setting depends quite a lot on the number of index columns that their cardinalities.

A big issue not handled by just the **rowFlushBoundary** setting is what happens many Indexing task run in a JVM. The number of Objects created and the Memory consumption can quickly get out of hand. We recommend that you start with 6-8 GB per core of on-heap memory for Executors, this setting is for Executors that handle either Indexing or Querying workloads. Event with this amount of memory, the GC overhead can be significant because of the number of objects being created.

#### Regulating Concurrent Merge Operations in a JVM, via the use of IndexMergePermits

An additional level of safety is built into indexing via the _spark.sparklinedata.indexing.memory.percore _setting, which by default is set to 1gb. Whenever an Indexing task wants to do a mini-segment conversion to the **Queryable Index Form**, or to do a final merge of all mini-segments, it requests for a certain number of MergePermits. The number requested is based on the size being operated on. It proceeds with these operations only if it has received the Permits to do so. 

##### An Example

Say you have 2 Indexing Tasks running in an Executor JVM, and the total Permits available = `1gb * 2 = 2gb`. So if at time _t1_ the Task T1 requests 1.5GB of permits to perform an operation, and Task T2 requests 1GB at time_ t1 + 1_, Task T2 will have to wait for Task T1 to finish its operation and release the permits.

##### Calculation of Number Permits required and mapping this to Memory

In case of the final merge, we know the file sizes of each of the mini-segments, we use this as estimate of the permits required. During mini-segment creation, we know the file sizes of prior mini-segments, we use the average across the mini-segments created so far. For the first mini-segment, we start with an arbitrary value of `200mb`. The actual memory permits is **5x** the amount of the file sizes, this accounts for the internal processing time data structures created to do the operation\(conversion to **Queryable Index Form **or** Mini-Segment merge**\)













