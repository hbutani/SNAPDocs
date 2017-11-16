<!-- --- title: Segment Cache Notes -->
* Each Executor maintains a Local Cache of OLAP Segments
* Each OLAP Index Segment File is cached in a small subset of Executor Local Caches
* Spark Task Scheduling is influenced by providing the Segment cached locations as the preferred locations for an OLAP Query Task on that Segment.
* The **SegmentCacheManager **maintains the cache locations of all Segments.
* * It listens on Executor Lifecycle events, Segment Load Events and Index Insert Events.
  * When asked to **pick preferred task locations **for an OLAP Segment Task, it calculates this from the known cached locations and current active executors.
* The **OLAP Index Format **generates Segment Lifecycle Events that help the SegmentCacheManager maintain the cluster cache state:
* * When a Segment is Loaded into an Executor’s Local Cache, a Segment Load Event is posted.
  * When a new Segment is created \(because of a initial Insert or a Insert Overwrite\), a Partition Insert Event is posted.
* **Segment Load Events **are processed by the SegmentCacheManager once the Task in which the Event was posted is finished.
* **Partition Insert Events **are processed by the SegmentCacheManager once the SQL Insert Statement in which the Event was posted is finished. This ensures that Queries running while an Insert is going on continue to be scheduled based on the old state.



