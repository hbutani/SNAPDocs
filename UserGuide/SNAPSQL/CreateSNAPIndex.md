<!-- --- title: Create SNAP Index -->

### Grammar
```text
createOlapIndex : CREATE OLAP INDEX qualifiedId (opt(OF |ON) qualifiedId)
                    fieldInfos
                    opt(dimensionsOrNot)
                    opt(METRICS stringLit) 
                    opt(OPTIONS "(" options ")")
                    opt(partitionBy)

fieldInfos : rep(fieldInfo)
fieldInfo : indexDimensionInfo | indexTimestampDimensionInfo | indexMetricInfo
indexDimensionInfo : DIMENSION qualifiedCol avgSize 
                      (
                        (IS NULLABLE NULLVALUE stringLit) | 
                        opt(IS NOT NULLABLE)
                      )
indexTimestampDimensionInfo : TIMESTAMP DIMENSION qualifiedCol avgSize
                                opt(SPARK TIMESTAMPFORMAT stringLit)
                                opt(IS INDEX TIMESTAMP) 
                                (
                                  (IS NULLABLE NULLVALUE stringLit) | 
                                  opt(IS NOT NULLABLE)
                                )
indexMetricInfo : METRIC ident aggregator avgSize 
                      (
                        (IS NULLABLE NULLVALUE stringLit) | 
                        opt(IS NOT NULLABLE)
                      )
dimensionsOrNot : opt(IGNORE) DIMENSIONS stringLit
options : rep1sep(option, opt(","))
option : ident stringLit
partitionBy : PARTITION BY rep1sep(qualifiedCol, ",")

avgSize : opt(AVERAGE SIZE numericLit

qualifiedCols : repsep(qualifiedCol, ",")
qualifiedCol : ident opt("." ident) ~ opt("." ident)

```

### Description
Command to *create* a **SNAP Index** on a StarSchema.
The definition is stored as a serialized `PersistedESSInfo` object graph in the table properties of
the Fact Table. This command creates a Spark DataSource Table for the SNAP Index with a SNAP FIleFormat.
It associates the FactTable with this SNAP Index by adding a metadata property to both tables that links 
one to the other.

###Checks
- A StarSchema can only have 1 SNAP Index. If no StarSchema exists on the FactTable, then a single table StarSchema is created on the FactTable.
- Dimension Column DataTypes must be convertible to String
- for metrics specified by name, the datatype must be Integral or Float; for other metrics specify it using a `IndexFieldInfo` object.


### IndexFieldInfo Definition
The user can specify `IndexFieldInfos` as a json string to get full control of the definition of Index fields
or when he wants access to the newer features not exposed in the grammar.

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
                           avgSize : Option[Int],
                          extractInfo : Option[IndexMetricExtractorInfo] = None,
                           sparkTimestampFormat: Option[String] = None)

// IndexAggregators
CountAggregator
LongMinAggregator
LongMaxAggregator
LongSumAggregator
TimestampAggregator
DoubleMinAggregator
DoubleMaxAggregator
DoubleSumAggregator
case class JavaScriptAggregator(fieldNames : List[String],
                                fnAggregate : String,
                                fnReset  : String,
                                fnCombine : String)
case class HistogramAggregator(breaksList : List[Float])
case class HyperUniquesAggregator(name : String)
case class CardinalityAggregator(fieldNames : List[String],
                                 byRow : Boolean)
DoublePrecisionSumAggregator
case class BigDecimalSumAggregator(precision : Int, scale : Int)
StringFirstAggregator

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

### Metric Extractors
An **Extractor** associated with a *Metric* enables potential pushdown of filter and group expressions on the Metric.
The pushed down can have dramtic impact on query performance even if only a partial push down is achieved.
For e.g. an expression ```longMetric > 15.0``` on *longMetric* with a binning extractor will get pushed down as 
a filter that reduce the number of rows scanned from the Index: the reduction percentage depends on how the metric bins
are setup. In case timestamp elemnets(Year, Month, Quarter etc) the speedup can be even more dramatic.

An Extractor is defined by an ```ExtractorInfo```, and provides the following interface:

```scala
  def componentNames : Seq[String]
  
  def addMetricDimensions(iRow : MMap[String, AnyRef]) : Unit

  def pushDownFilterCond(fC : Expression) : Option[(DimFilter, Boolean)]

  def pushDownProjection(pE : Expression) : Option[(DimensionSpec, DataType)]
```

- ComponentNames:
    - each *component* is a many-to-one mapping from the metric's value space onto another 
      (hopefully much smaller) space.
    - for example components for timestamps are *Year*, *Month*, *Quarter* etc.
    - each specified *component* is added to the SNAP Index as a Dimension
- addMetricDimensions:
    - during indexing this method is called on the Metric Extractor to add in the component dimensions
      for the given input row.
- pushDownProjection:
    - during translation of *Filter* expressions the Planner gives the Extractor a chance to pushdown the 
      Expression as an Index Filter. The second output argument tells the Planner if the pushdown
      was such that the original filter need or need not be applied on the rows coming from the
      Index Scan.
        - for e.g. an expressin like```longMetric > 5``` is pushed down as a *Range* filter on the Index
          that may return rows with ```longMetric <= 5```, hence the original filter needs to be applied
          on top of the Index Scan.
- pushDownProjection:
    - during translation of *Group* Expressions the Planner gives the Extractor a chance to pushdown the
      Expression as an Index Group Expression.
        - For example a timestamp metric with an extraction of Year can have a ```group by year(tsMetric)```
          be evaluated as a Group Expression of the Year component of the timestamp metric.

#### Supported Extractors
##### Timestamp Extractor

This defined by a ```TimestampMetricExtractorInfo``` that is given a set of ```TimeElement```

```scala
// TimeElements:
  val Year = Value("year")
  val DayOfYear = Value("dayofyear")
  val DayOfMonth = Value("dayofmonth")
  val Month = Value("month")
  val Quarter = Value("quarter")
  val Hour = Value("hour")
  val Minute = Value("minute")
  val Second = Value("second")
  
case class TimestampMetricExtractInfo(val timeElems : Seq[TimeElement.Value])
```

For example:
```json
[
   {
      "jsonClass":"IndexMetricInfo",
      "column":"tsMetric",
      "isNullable":true,
      "nullValue":"2017-06-29T17:15:13.546Z",
      "aggregator":"TimestampAggregator",
      "extractInfo":{
         "jsonClass":"TimestampMetricExtractInfo",
         "timeElems":[
            "year",
            "dayofmonth",
            "dayofyear",
            "quarter",
            "month",
            "hour",
            "minute",
            "second"
         ]
      }
   }
]
```

Each specified component is added as a dimension to the Metric. The naming convention is
`` `${metricname}_snap_${componentname}` ``. 
For example tsMetric_snap_year
Also a `${metricname}_snap_nullflag` dimension is added to track null values.

###### Pushdowns
- Comparison and In Filters(`<,<=,=,!=,>,>=,in, not in`) on the specified TimeElements are pushed 
  as Dimension Filters
    - For example `year(tsMetric) > 2015`, `month(cast(tsMetric as datetime)) > month("2015-03-12")`
- `Is Null` and `Not Is Null` are pushed down as Dimension Filters on the nullflag dimension
- Some Comparison and In Filters(`<,<=,=,>,>=, in`) on the timestamp column are pushed as Dimension Filters
  on the Year TimeElemt if it is present.
- Group By Expression on on the specified TimeElements are pushed as Group Expressions
    - - For example `group by year(tsMetric)`, `group by month(cast(tsMetric as datetime))`

##### Binning Extractor
Works by mapping the metric's value space onto a set of bins. The metric dimension is used to capture the bin a
row belongs to; and hence a comparison filter on the metric can be translated to some filter on the metric bin dimension.
The dimension is named: "${metricname}_snap_bin_dim", for example "longmetric_snap_bin_dim"

Specified by a `BinningExtractInfo`. There are 2 kinds:  `ExplicitBinsMetricInfo` and 
`NumBinsMetricInfo`. ExplicitBinsMetricInfo can be specified for Long, Double, DoublePrecision and String metrics,
whereas NumBinsMetricInfo can be specified for Long, Double and DoublePrecision metrics.

- ExplicitBinsMetricInfo:
    - as the name suggests the bins are given by the user. The bins are given by their starting values.
- NumBinsMetricInfo:
    - is specified as `minValue, maxValue, numBins`

```json
[
   {
      "jsonClass":"IndexMetricInfo",
      "column":"strMetric",
      "isNullable":true,
      "nullValue":"",
      "aggregator":{
         "maxBytes":1
      },
      "extractInfo":{
         "jsonClass":"StringExplicitBinsMetricInfo",
         "startValues":[
            "BC",
            "BW",
            "Bq",
            "CM",
            "Cg",
            "DC",
            "DW",
            "Dq",
            "EM",
            "Eg"
         ]
      }
   },
   {
      "jsonClass":"IndexMetricInfo",
      "column":"longNumBinMetric",
      "isNullable":true,
      "nullValue":"-1",
      "aggregator":"LongSumAggregator",
      "extractInfo":{
         "jsonClass":"LongNumBinsMetricInfo",
         "minValue":0,
         "maxValue":1000,
         "numBins":10
      }
   },
   {
      "jsonClass":"IndexMetricInfo",
      "column":"dblNumBinMetric",
      "isNullable":true,
      "nullValue":"-200.0",
      "aggregator":"DoubleSumAggregator",
      "extractInfo":{
         "jsonClass":"DoubleNumBinsMetricInfo",
         "minValue":-100.0,
         "maxValue":100.0,
         "numBins":5
      }
   },
   {
      "jsonClass":"IndexMetricInfo",
      "column":"longExBinMetric",
      "isNullable":true,
      "nullValue":"-1",
      "aggregator":"LongSumAggregator",
      "extractInfo":{
         "jsonClass":"LongExplicitBinsMetricInfo",
         "startValues":[
            25,
            150,
            500,
            750
         ]
      }
   },
   {
      "jsonClass":"IndexMetricInfo",
      "column":"dblExBinMetric",
      "isNullable":true,
      "nullValue":"-200.0",
      "aggregator":"DoubleSumAggregator",
      "extractInfo":{
         "jsonClass":"DoubleExplicitBinsMetricInfo",
         "startValues":[
            -100.0,
            -2.0,
            45.0,
            75.0
         ]
      }
   }
]
```

###### Pushdowns
- Some Comparison and In Filters(`<,<=,=,>,>=,in`) on the specified TimeElements are pushed 
  as Dimension Filters. The original Filters are retained in the Query Plan, because the pushed filter's
  range is a superset of the original filter's range. 
    - For example `longMetric > 53.0`, `dblMetric < 1.0 + 5`
    - `!= and not in` filters are not pushed down. If these are common predicates in the Query workload, binning
      will not help.
- `Is Null` and `Not Is Null` are pushed down as Dimension Filters on the nullflag dimension

### SNAP Index Options


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


### Examples
```sql
-- 1. index on single, non partitioned fact table
create olap index tpch_flat_index on orderLineItemPartSupplierBase
      dimension p_name is not nullable
      dimension ps_comment is nullable nullvalue ""
      timestamp dimension l_shipdate spark timestampformat "yyyy-MM-dd"
                 is index timestamp
                 is nullable nullvalue "1992-01-01"
      timestamp dimension o_orderdate
      timestamp dimension l_commitdate
          is nullable nullvalue "1992-01-01T00:00:00.000Z"
      timestamp dimension l_receiptdate
          is not nullable
      metric o_totalprice aggregator doubleSum
      dimensions "o_orderkey,o_custkey,o_orderstatus,o_orderpriority,o_clerk,..."
      metrics "l_quantity,l_extendedprice,l_discount,l_tax,ps_availqty,...."
      OPTIONS (
        path "src/test/resources/spmd/tpch_flat"
        preferredSegmentSize "200mb"
      )
;

-- 2. index on single, partitioned fact table
create olap index tpch_flat_part_index on tpch_flat_small_part
dimension p_name is not nullable
dimension ps_comment is nullable nullvalue ""
timestamp dimension l_shipdate spark timestampformat "yyyy-MM-dd"
                 is index timestamp
                 is nullable nullvalue "1992-01-01"
timestamp dimension o_orderdate
timestamp dimension l_commitdate
          is nullable nullvalue "1992-01-01T00:00:00.000Z"
timestamp dimension l_receiptdate
          is not nullable
metric o_totalprice aggregator doubleSum
dimensions "o_orderkey,o_custkey,o_orderstatus,o_orderpriority,o_clerk,..."
metrics "l_quantity,l_extendedprice,l_discount,l_tax,ps_availqty,...."
      OPTIONS (
        path "src/test/resources/spmd/tpch_part"
        preferredSegmentSize "200k"
)
partition by shipYear, shipMonth
;

-- 3. index on single, non-partitioned view
create view tpch_flat_view as
  select p_name,s_name,s_address,s_phone,s_comment,s_nation,s_region,
         avg(l_quantity) as l_quantity, avg(l_extendedprice) as l_extendedprice
  from orderLineItemPartSupplierBase
  group by p_name,s_name,s_address,s_phone,s_comment,s_nation,s_region
;
create olap index tpch_flat_view_index on tpch_flat_view
dimensions "p_name,s_name,s_address,s_phone,s_comment,s_nation,s_region"
OPTIONS(
        path "src/test/resources/spmd/tpch_view_index"
        indexFieldInfos '$indexInfos'
        preferredSegmentSize "200mb"
        avgSizePerPartition "10mb"
        avgNumRowsPerPartition "100000"
        )
;

-- 4. index on star-schema, non-partitioned fact table
create star schema on lineitem_small
as many_to_one join of lineitem_small with orders on l_orderkey = o_orderkey
   many_to_one join of lineitem_small with partsupp on
          l_partkey = ps_partkey and l_suppkey = ps_suppkey
   many_to_one join of partsupp with part on ps_partkey = p_partkey
   many_to_one join of partsupp with supplier on ps_suppkey = s_suppkey
   many_to_one join of orders with customer on o_custkey = c_custkey
   many_to_one join of customer with custnation on c_nationkey = cn_nationkey
   many_to_one join of custnation with custregion on cn_regionkey = cr_regionkey
   many_to_one join of supplier with suppnation on s_nationkey = sn_nationkey
   many_to_one join of suppnation with suppregion on sn_regionkey = sr_regionkey
;
create olap index tpch_star_flat_index on lineitem_small
dimension p_name is not nullable
dimension ps_comment is nullable nullvalue ""
timestamp dimension l_shipdate spark timestampformat "iso"
                 is index timestamp
                 is nullable nullvalue "1992-01-01T00:00:00.000Z"
timestamp dimension o_orderdate
timestamp dimension l_commitdate
          is nullable nullvalue "1992-01-01T00:00:00.000Z"
timestamp dimension l_receiptdate
          is not nullable
metric o_totalprice aggregator doubleSum
dimensions "o_orderstatus,o_orderpriority,o_clerk,..."
metrics "l_quantity,l_extendedprice,l_discount,l_tax,ps_availqty,..."
      OPTIONS (
        path "src/test/resources/spmd/tpch_star_flat"
        indexSizeReduction "0.8"
        preferredSegmentSize "200mb"
)
;

-- 5. index on hive metastore table, non-partitioned
create table notExternal_t1(x string, y string, m long);
create olap index notExternal_t1_idx on notExternal_t1
dimensions "x, y"
metrics "m"
;

-- 6. tables in multiple databases
create database idx_db1;
create database fact_db1;
use fact_db1;
create table multidb_fact(x string, y string, m long)
     USING com.databricks.spark.csv
;
"use idx_db1;
create olap index multidb_fact_idx on fact_db1.multidb_fact
       ignore dimensions "x"
       metrics "m"
;

-- 7. with metric extractions
val indexInfos = SPLMDFUtils.asJson(Seq(
    IndexMetricInfo("strMetric", true, "", StringFirstAggregator(), None,
      Some(StringExplicitBinsMetricInfo(strBinStartVals))
    ),
    IndexMetricInfo("tsMetric", true, "2017-06-29T17:15:13.546Z", TimestampAggregator, None,
      Some(TimestampMetricExtractInfo(Seq(
        TimeElement.Year, TimeElement.DayOfMonth, TimeElement.DayOfYear,
        TimeElement.Quarter, TimeElement.Month, TimeElement.Hour,
        TimeElement.Minute, TimeElement.Second
      )))),
    IndexMetricInfo("longNumBinMetric", true, "-1", LongSumAggregator, None,
      Some(LongNumBinsMetricInfo(0L, 1000L, 10))
    ),
    IndexMetricInfo("dblNumBinMetric", true, "-200.0", DoubleSumAggregator, None,
      Some(DoubleNumBinsMetricInfo(-100.0, 100.0, 5))
    ),
    IndexMetricInfo("longExBinMetric", true, "-1", LongSumAggregator, None,
      Some(LongExplicitBinsMetricInfo(Seq(25L,150L, 500L, 750L)))
    ),
    IndexMetricInfo("dblExBinMetric", true, "-200.0", DoubleSumAggregator, None,
      Some(DoubleExplicitBinsMetricInfo(Seq(-100.0, -2.0, 45.0, 75.0)))
    )
  )).replaceAll("\n", "")

create olap index metrics2_src_table_idx on metrics2_src_table
      dimensions "key"
      OPTIONS (
        path "${indexData}",
        indexFieldInfos '$indexInfos',
        nonAggregateQueryHandling "push_project_and_filters",
        allowTopNRewrite "true"
        preferredSegmentSize "200mb"
        addCountMetric "true"
      )
;

-- 8. timestamp dimensions
create olap index datetime_table_index on datetime_table
dimension date_str is not nullable
timestamp dimension timestamp_str is not nullable
dimension date_col is not nullable
timestamp dimension timestamp_col is not nullable
      OPTIONS (
        path "${datetimeTableFolderIndex}",
        nonAggregateQueryHandling "push_project_and_filters",
        allowTopNRewrite "true"
        preferredSegmentSize "200mb"
      )
;
```
