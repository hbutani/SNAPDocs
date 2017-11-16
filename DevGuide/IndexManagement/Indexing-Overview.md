<!-- --- title: Indexing Overview -->
An OLAP Index is a representation of the Data that provides significant boost to Analytics when individual Queries are constrained to small slices of the entire multi-dimensional space of a DataSet. In order to get the maximum benefit of an OLAP Index you need to pay attention to certain aspects of its physical layout, and you must be aware of the limited ways in which an Index can be updated:

* At the most basic level, the Index is divided into **Segments **which contain subsets of Source Dataset facts. You want Segments to be a few 100 MBytes containing up to a few million facts.
* The Index is typically also **logically partitioned **\(for example by a Time Period\(Year or Month or Day\) or Geography\(for example by Region or Country\). It is not uncommon to have multiple layers of Partitions: say by Year and Month, or by Year and Supplier.
* It is not possible to support update of individual facts in an OLAP Index, or to add new facts to existing Segments. The operations possible are: to recreate a Segment or to add new Segments. **At least for the first release, in a subsequent release we plan to provide a Change-Data overlay file mechanism that will automatically overcome these limitations**. Hence the choice of Partition scheme is a delicate balance between boost to Query Performance and the Index Update process needed. For example partition by supplier maybe the best solution from a Query time Partition Pruning perspective, but this may mean having to recreate the Index daily when new/update facts that become available.
* But by its nature an OLAP Index is less impacted by how facts are Partitioned: even without partition pruning kicking in for Queries, the way in which the data is processed is very efficient for Slice-and-Dice queries; so you have more leeway when deciding on the Partition scheme; **you don’t have to make the update process very onerous.**

So when designing your OLAP Index your goals should be: the Indexing process should be reasonably quick and the Index Segment files created should be ‘just right’\(not too small or too large\) for optimal Query performance. You have a lot of options at your disposal; this document is a guide on how to go about deciding on your OLAP Index definition. First of all what do we mean by **reasonably quick **and **just right**?

## Index Segment Size

The core advantage of an OLAP Index is in being able to answer ad-hoc slice-and-dice like Queries very very quickly. There is no magic here, this power comes from how an OLAP Index is structured: Inverted Index like structures for Dimension Values, columnar structure for all columns, encodings that provide significant compression etc. For the OLAP Index to work for a very wide range of Query types\( queries with very wide or narrow constraining predicates, queries that compute lot of or small amounts of output, queries that aggregate or just scan etc\) the Index must be divided into Segments that are neither too small or too large. Roughly, Query processing happens in 2 stages: each Segment is processed to get an intermediate answer and then all intermediate results are combined.

Now if you divide your Index into a lot of small Segments than the second stage can dominate Query processing, significantly diluting the value of an OLAP Index. Whereas if your Segment sizes are too large, certain Queries may cause significant memory pressure on the runtime, and GC issues can start to dominate Query processing. Hence there is a **Goldilocks size** for Index Segments, this is in the range of a few 100 MBs with the additional condition that each Segment not have more than a few million facts.

Another important aspect that has a huge impact on Index Segment layout, is the logical partitioning scheme to be employed. So if you know that a lot of Queries are constrained to a time period than it makes sense to partition your data by Day\(or Week or Quarter or Year\) so that during Query Processing **Partition Pruning **kicks in and in a typical case only a few Segments are processed.

## Indexing performance

The creation of an OLAP Index is a computationally intensive process. It is very similar to creating an Inverted Index, and the performance is impacted by the number of dimensions being indexed and their individual cardinalities\( for example large cardinality dimensions can put significant memory pressure on Indexing tasks, which can cause huge GC overheads\).

The OLAP Index is physically organized into **Segments**, each Task\(core\) during Indexing produces 1 or more Segments. The optimal concurrency of an Indexing Job depends on the total size of the Input and on the Logical Partitioning scheme of the data\(we talk more about this in the next section on Index Segment Size\). Given your cluster’s capacity available for Indexing, one goal of Indexing is to parallelize Segment creation to the maximum. Based on the number and cardinalities of your dimensions, and roughly the number of rows per Segment, you can get a good estimate of how fast Indexing should go. For example when Indexing the TPCH dataset when we index all dimensions except the comment fields we observe an Indexing speed per core of around 300-400MB/min on an AWS R3 class machine.

The memory footprint of each Indexing Thread can be controlled by the **rowFlushBoundary **parameter. Set this to a value\(unit is in number of rows\) such that every so many rows the accumulated Index rows are flushed into a temporary segment on disk.

_**\(See the section on Index Process Tuning, for more information on the Indexing Process\) **_

## Index Partitioning Scheme

Ideally you want the Index Partition scheme to be solely based on the boost this can provide via Partition Pruning to your Analytic Workload. But this implicitly assumes that the cost of updating an Index matches the amount of data being updated. But since Index Segments can only be recreated or new ones added, this can easily not be the case: as an extreme example, requiring that the Index be recreated on every batch update event is usually a non starter design.

At the Index **Segment **level the operations possible are: to recreate a Segment or to add new Segments. So as stated for release 1, updates involve recreating Segments, which can entail re-indexing the entire Dataset if the Index has an arbitrary partition scheme.

> ```
> In a future release we plan to support a Change-Data-Capture process that will allow OLAP Indexes to be
>  given Change Files containing Insert-Update-Delete information about Facts.
> ```

Even with the current restrictions, if we design the Index partition scheme appropriately most scenarios can be well supported. It is not uncommon to disallow restating of facts\(update of existing facts\) but to handle all change as Type II changes. In this case the Source Partition scheme is based on how new facts are received into the Analytic Warehouse. Typically this is a Time-based primary partition and optionally a secondary partition based on other business factors like your Supply-chain, your IT infrastructure etc. For example, in a Sales scenario you could be receiving new facts every day from each region. Each batch of Facts maybe arriving on a different schedule. Occasionally you would need to redo a batch because of errors along the ETL pipeline. So the partition columns you choose has a major impact on 3 aspects of OLAP Indexing:

##### Segment Size

using columns that have a very high cardinality has a major drawback: this may lead to large numbers of very small Index Segments. In this case Query performance will further be affected by the time it takes to do Partition Pruning on large number of partitions.

##### Index Update

periodically new facts need to be indexed, and also old facts may have to be restated. Currently we only support addition of new facts. The case we are focusing on is when the Raw data is primarily partitioned by say Day and periodically new partitions are added to the Source DataSet. The pipeline that adds new partitions to the Source Dataset should be extended to trigger indexing of these partitions.

##### Indexing Performance

When Indexing a non-partitioned dataset, it is relatively easy to get a sense of the optimal parallelism and hence distribution of data to utilize the cluster resources optimally. Any Task can be given any slice of the Source Dataset, we only need to distribute the data evenly among the Tasks. When indexing multiple partitions of a partitioned dataset, a Task needs to handle each partitions data as a separate output bucket\(it will be helpful here if you read about Dynamic Partitioning in Spark\). This can cause small Segment files and/or significant memory pressure\(based on how many dynamic partition a task has to handle\). **We attempt to avoid this, by ensuring Tasks get data in an orderly fashion \(partition by partition\)**

To conclude we don’t impose any restriction on the partition scheme you choose for the OLAP Index, but the only operations we allow today are: recreating Segments or adding new Segments\(by adding new Partitions\). Further these have to be manually managed by your DBA team. These have to be done so that Segment sizes are optimal and Indexing time is acceptable. Inspite of these restrictions, most real-world scenarios should be covered reasonably well: by giving a significant boost to Query Performance, with increase in management burden being relatively small.

Hopefully you are getting a sense of what aspects are important when designing an OLAP Index: the Indexing performance, your Index update strategy and Index Segment sizes. Next we walk through a detailed example that we think is a fairly typical scenario; we also provide commentary on if and how you can handle scenarios that deviate from this ‘typical’ scenario.





