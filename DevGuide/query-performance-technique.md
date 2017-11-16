<!-- --- title: Analyzing Query Performance -->

MDFormat Test framework provides tooling to run a Index Query directly against Druid and through our layer.

For eg. see: druidexts/src/test/scala/com/sparklinedata/druid/testing/FlightsTest.scala and src/test/scala/com/sparklinedata/mdformat/FlightsQueryTest.scala

This testing a GBy query on a segment in the Flights Index. The process involves:

```
- identify a Segment Index
- download it and unzip it locally
- identify a query
- use Druid API to programmatically express the Query, for example:
val basicQuery: GroupByQuery = {
      val dims = List(
        new DefaultDimensionSpec("UniqueCarrier", "UniqueCarrier"),
        new DefaultDimensionSpec("CRSDepTime", "CRSDepTime")
      )
      val aggs = Array[AggregatorFactory](new LongSumAggregatorFactory("Distance", "Distance")).toList
      GroupByQuery.builder.
        setDataSource("flights").
        setQuerySegmentSpec(new LegacySegmentSpec("1970/3000")).
        setGranularity(QueryGranularities.ALL).
        setDimensions(dims).
        setAggregatorSpecs(aggs).build
    }

- for druid this is enough to query against the segment, see FlightsTest:t1 for an example
- for SNAP, define the DDL for the spmd table, for example:
val indexInfos = SPLMDFUtils.asJson(Seq(
      IndexDimensionInfo("year", true, "NA", None, None),
      IndexDimensionInfo("Month", true, "NA", None, None),
      IndexDimensionInfo("DayofMonth", true, "NA", None, None),
      IndexDimensionInfo("DayOfWeek", true, "NA", None, None),
      IndexDimensionInfo("UniqueCarrier", true, "NA", None, None),
      IndexDimensionInfo("FlightNum", true, "NA", None, None),
      IndexDimensionInfo("TailNum", true, "NA", None, None),
      IndexDimensionInfo("Origin", true, "NA", None, None),
      IndexDimensionInfo("Dest", true, "NA", None, None),
      IndexDimensionInfo("Cancelled", true, "NA", None, None),
      IndexDimensionInfo("CancellationCode", true, "NA", None, None),
      IndexDimensionInfo("Diverted", true, "NA", None, None),
      IndexDimensionInfo("DepTime", true, "NA", None, None),
      IndexDimensionInfo("CRSDepTime", true, "NA", None, None),
      IndexDimensionInfo("ArrTime", true, "NA", None, None),
      IndexDimensionInfo("CRSArrTime", true, "NA", None, None),
      IndexMetricInfo("ActualElapsedTime", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("CRSElapsedTime", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("AirTime", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("ArrDelay", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("Distance", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("TaxiIn", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("TaxiOut", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("CarrierDelay", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("WeatherDelay", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("NASDelay", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("SecurityDelay", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("LateAircraftDelay", true, "0.0", LongSumAggregator, None),
      IndexMetricInfo("ArrDelay", true, "0.0", LongSumAggregator, None)
    ))

    val cT =
      s"""CREATE TABLE if not exists flights_index(
          Month string,
         | DayofMonth string,
         | DayOfWeek string,
         | DepTime bigint,
         | CRSDepTime bigint,
         | ArrTime bigint,
         | CRSArrTime bigint,
         | UniqueCarrier string,
         | FlightNum string,
         | TailNum string,
         | ActualElapsedTime bigint,
         | CRSElapsedTime bigint,
         | AirTime bigint,
         | ArrDelay bigint,
         | Origin string,
         | Dest string,
         | Distance float,
         | TaxiIn bigint,
         | TaxiOut bigint,
         | Cancelled string,
         | CancellationCode string,
         | Diverted string,
         | CarrierDelay bigint,
         | WeatherDelay bigint,
         | NASDelay bigint,
         | SecurityDelay bigint,
         | LateAircraftDelay bigint,
         | year string
          )
      USING spmd
      OPTIONS (
        path "/Users/hbutani/SPLGoogleDrive/SparklineDocs/Development/SNAP/TestScenarios/flights/flights_snap/year=1988/index_attempt_201704300939_0009_m_000000_0_00000.zip",
        indexFieldInfos '$indexInfos',
        avgSizePerPartition "10mb",
        avgNumRowsPerPartition "10000",
        preferredSegmentSize "100mb"
      )""".stripMargin
      
- now you can run queries against it, see FlightsQueryTest:q1 for an example
```



