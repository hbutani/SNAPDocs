<!-- --- title: SNAP FileFormat Options -->


* Options are specified in the create olap index command.

* But many options can be overridden by  system level settings. This way indexing and query behavior can be tuned on per statement basis.

| Name | Description | Default Value | Session Level  Override |
| :--- | :--- | :--- | :--- |
| path | The deep store location where this index s stored. | None | None |
| ignoreInvalidRows | When indexing if invalid values are encountered, should we silently drop these rows and continue indexing | false |  |
| dimensions | comma separated list of source columns that are indexed as dimensions. These can be qualified names. These dimensions will have default indexing attributes. See the section below on Index Field Information. | None |  |
| Metrics | comma separated list of source columns are indexed as metrics. These can be qualified names. These will have default Indexing attributes. See the section below on Index Field information | None |  |
| indexFieldInfos | A json string interpreted as a list of Index Field Infos. See below on how to specify Index Fields | None |  |
| defaultTimestampValue | If a Dataset doesn't have a timestamp column, the Index rows have this value set as the timestamp. Can be any arbitrary value, you shouldn't need to change the default. | 2016-11-02T09:50:00-08:00 \(time when Cubs won the World Series\) |  |
| columnsAreNullByDefault | the default behavior for nulls in source columns. If true then we assume source columns can have null | true |  |
| defaultNullValueInIndex | for nullable columns, if the null value to insert into the index is not specified, this value is used | "" |  |
| selectQueryBufferSize | Preferred size\(in bytes\) of the SelectEngine Pagesize; should be 1-10s of MB for optimal performance. | 4MB | spark.sparklinedata.spmd.select.query.buffersize |
| addCountMetric | Add Count Metric to the SNAP Index; if source rows are aggregated during indexing\(because of duplicates\) this metric tracks the original count. | false |  |
| **Indexing Options:** |  |  |  |
| rowFlushBoundary | During Indexing the number rows processed in a batch in memory. Every batch of  rowFlushBoundary rows is written to a temp file on disk | 10000 | spark.sparklinedata.spmd.indexing.rowFlushBoundary |
| preferredSegmentSize | The Size of each Index Segment File. This should be a few hundred MBytes, with upto a few million rows. |  |  |
| avgSizePerPartition | An estimate of the size of an input\(flattened star schema\) partition. If this is not specified, we attempt to estimate this from the Fact tables files and factor in the increase in size of input tors when flattening. |  | spark.sparklinedata.spmd.avgsizeperpartition |
| numPartitionsBeingIndexed | If this is not specified, we try to infer it from the Fact table partitions and the partition clause given for the insert statement. Since this can be different for each insert execution, we recommend this be specified as a session setting. |  | spark.sparklinedata.spmd.num\_partitions\_indexed |
| indexSizeReduction | A fraction value that is an estimate of how much smaller the index size will be relative to the source size. |  | spark.sparklinedata.spmd.sizereductionpercent Default value for this option is 0.25 |

### Index Field Infos:

There are several kinds of Field Infos:

* DimensionInfo
* MetricInfo
* TimestampDimensionInfo
* SpatialDimensionInfo and SpatialComponentDimensionInfo

#### Dimension Info

```java
case class IndexDimensionInfo(column: String,
                              isNullable: Boolean,
                              nullValue: String,
                              hllMetric : Option[String] = None,
                              avgSize : Option[Int])
```

You specify the Source column that is mapped to a DImension, and null mapping: is the source column nullable and if yes how should it be mapped to the Index. Optionally you can have a HyperLogLog sketch be maintained for this Dimension in the Index. This enables doing approximate counts in the Index.

#### Metric Info

```java
case class IndexMetricInfo(column: String,
                           isNullable: Boolean,
                           nullValue: String,
                           aggregator: IndexAggregator,
                           avgSize : Option[Int])
```

You specify the Source column that is mapped to a Metric, and null mapping

#### Timestamp Dimension

```java
case class IndexTimestampDimensionInfo(column: String,
                                       isNullable: Boolean,
                                       nullValue: String,
                                       sparkTimestampFormat: Option[String],
                                       isIndexTimestamp: Boolean,
                                       avgSize : Option[Int])
```

For Timestamp Dimensions you must specify how to extract DateTime values if the source table is a String, so a sparkTimestampFormat must be given. In addition you can set one of the Timestamp Dimensions as the Timestamp of the fact, by setting isIndexTimestamp=true.

#### Spatial Dimension

```java
case class IndexSpatialDimensionInfo(name: String,
                                     spatialCoords: Seq[SpatialComponentDimensionInfo])

case class SpatialComponentDimensionInfo(column: String,
                                         isNullable: Boolean,
                                         nullValue: String,
                                         minValue : Option[Double],
                                         maxValue : Option[Double],
                                         avgSize : Option[Int])
```

A Spatial Dimension definition includes the details of each component\(axis\) of the Spatial point.

#### Specifying Field Infos in the DDL

Field Infos are specified in the indexFieldInfos parameter as a JSON array, for example:

```java
create olap index zipCodesI on zipCodesBaseS
        dimensions "zip_code,state,county"
      OPTIONS (
        path "/Users/hbutani/sparkline/snap/src/test/resources/spmd/zipCodes",
        indexFieldInfos '[{"jsonClass":"IndexTimestampDimensionInfo","column":"record_date","isNullable":true,"nullValue":"1992-01-01T00:00:00.000","sparkTimestampFormat":"iso","isIndexTimestamp":true},{"jsonClass":"IndexSpatialDimensionInfo","name":"coordinates","spatialCoords":[{"column":"latitude","isNullable":true,"nullValue":"0.0","minValue":-90.0,"maxValue":90.0},{"column":"longitude","isNullable":true,"nullValue":"0.0","minValue":-180.0,"maxValue":180.0}]},{"jsonClass":"IndexDimensionInfo","column":"city","isNullable":true,"nullValue":"","hllMetric":"unique_city"}]',
        nonAggregateQueryHandling "push_project_and_filters",
        avgSizePerPartition "10mb",
        avgNumRowsPerPartition "10000",
        preferredSegmentSize "100mb"
      )
```



