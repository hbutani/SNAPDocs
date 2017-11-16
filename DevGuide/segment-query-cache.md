<!-- --- title: Segment Query Cache -->

This feature caches Query Results on a per Segment basis. Use the `spark.sparklinedata.spmd.segment.query.cache` setting to control Query Cache behavior. For example, here is a sample setting:

```text

spark.sparklinedata.spmd.segment.query.cache={"useCache" : true, "sizeInMBytes" : 1024,"expireAfterSeconds" : 60,"resultSizeMax" : 20000}
```

| Parameter | Description |
| :--- | :--- |
| useCache | turn on or off caching |
| sizeInMBytes | total size of the Query Cache in MB |
| expireAfterSeconds | the duration after which a query result is evictable |
| resultSizeMax | for GroupBy and Search Queries only if the number of rows in the result is below this size are they cached. The result size is estimated in a very conservative way: as the product of the cardinalities of the dimensions involved. |

* under the covers the [Caffeine Library](https://github.com/ben-manes/caffeine) is used for caching
  * Caffeine is configured to evict based on query usage and the Query Result memory footprint.
* Spark's **CachedBatch** mechanism is used to represent Query results in the cache.
* Currently only GroupBy, Search and Timeseries Queries are considered for caching.

#### Metrics

The following metrics are exposed:

| Metric | Description |
| :--- | :--- |
| Hit-Count | number of times Result found in Cache |
| Miss-Count | number of times Result not found in Cache |
| Hit-Rate | Percentage of Hits |
| Eviction-Count | number of times a Result eviction happened because of memory pressure |
| Eviction-Weight | total memory size of Results evicted. |
| Result Serializes | Time to serialize Query Results, the histogram of the time taken, and the Histogram of the Query Result Size |
| Result Deserializes | Time to deserialize Query Results, the histogram of the time taken. |



