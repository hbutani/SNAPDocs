<!-- --- title: SNAP Config Parameters -->

| Name\(prefix with spark.sparklinedata.spmd, unless specified\) | Description | Default | Bytes Unit |
| :--- | :--- | :--- | :--- |
| local.segment.cache | Local Folder\(s\) to use to cache index files. The index files will be copied to one of these locations and then memory-mapped into Spark Executors. | {"storageLocations" : \[{"path" : "/tmp/olapcache", "maxSize" : 10000000 }\], "columnCacheSizeBytes" : 0, "avgSizePerCacheFile" : 524288000, "LocalUnzipFileSizeFactor" : 4, "shareCacheAcrossExecutors" : true} |  |
| segment.query.cache | Control Query Caching | {"useCache" : false,"sizeInMBytes" : 1024,"expireAfterSeconds" : 60,"resultSizeMax" : 20000} |  |
| avgsizeperpartition | 0 | Used by subsequent Indexing Jobs as the avgSizePerPartition setting for the partitions being indexed. Usually this should be set in the Index parameters once during create olap index. | ByteUnit.BYTE |
| preferredsegmentsize | 0 | Used by subsequent Indexing Jobs as the preferredSegmentSize setting for the partitions being indexed. Usually this should be set in the Index parameters once during create olap index. | ByteUnit.BYTE |
| sizereductionpercent | 0.25 | An estimate in size reduction of the SPMD format compared to the orginal data. Ideally this is set by indexing a representatve sample and recording the size difference from the original datasize. By default this is set to 0.25 |  |
| indexing.rowFlushBoundary |  | The row batchsize used during indexing, this impacts the memory footprint of an indexing task; by default this is based on the value set in the Index Options, but can be override using this session level parameter. If this is set to a non-zero value, this value takes precedence over the value in the Index Options. |  |
| select.query.buffersize |  | Preferred size\( in bytes\) of the Pagesize when running an Index Select Query, should be 1-10s of MB for optimal performance |  |
| num\_partitions\_indexed | 0 | Number of partitions being indexed in subsequent Indexing Jobs |  |
| gByEngine.offheapsize | 1gb | Off Heap Pool used by each instance of the Index GroupBy Engine; there is 1 for every core assigned to an Executor. | ByteUnit.MiB |
| selectquery.pagesize | 10000 | Num. of rows fetched on each invocation of SNAP Index Select Query |  |
| enable.segmentcachemanager | true | If true, SegmentCacheManager is used to track segment locations, and influence olap Query locations |  |
| indexing.memory.percore | 1gb | Heap Space to Use for Indexing. Number of Concurrent Indexing Merge operations "is restricted such that the total memory needed doesn't exceed this value \* the number of spark cores | ByteUnit.MiB |
| spark.sparklinedata.use.snapwritercontainer | true | replace dynamiccontainerwriter with snapwritercontainer; for snap generated insert plans sorting of data in the dynamicwriter is not need, as this is done by Repartition/repartitionExpression operators added to the Plan. Only turn this off if you are directly writing to the SNAP Index. |  |
| spark.sparklinedata.indexing.default.rowbatch.memory | 200mb | If the memory footprint for a rowbatch cannot be inferred, than this value is used. | ByteUnit.MiB |




