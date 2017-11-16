<!-- --- title: Per Executor Cache Metrics -->


Name Prefix: &lt;app-id&gt;.sparklinedata.executor.segmentcache

| Name | Type | Description |
| :--- | :--- | :--- |
| SegmentRequest | Timer | Counts and Times for Segment Request calls |
| SegmentLoad | Timer | Counts and Times for when a Segment is Loaded from Deep Store. |
| SegmentSizePerRequest | Histogram | Size distribution of Segments being requested from local-cache. |
| SegmentSizePerLoad | Histogram | Size distribution of Segments being copied from deep store. |
| numCacheHits | Meter | Count of Requests that are serviced by locally cache segments |
| cacheHitRatio | Ratio | Ratio of numCacheHits to SegmentRequests |
| makeSpaceRequest | Timer | Counts and Times when a Segment Request required space to be made in local cache. |
| cacheTotalSpace | Guage | Total Space of Local Cache |
| cacheFreeSpace | Guage | Current Free Space in Local Cache |

#### 



