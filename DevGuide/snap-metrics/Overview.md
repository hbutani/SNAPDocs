<!-- --- title: SNAP Metrics -->
#### **Metrics available for:**

**Cache Metrics**

* Cache Manager
* Per Executor Segment Cache

#### Enable SparklineData Metric Sources

**In conf/metrics.properties:**

> \# add sparkline Metric Sources to executor and driver
>
> executor.source.splexecutorsegcachemetrics.class=org.apache.spark.sql.mdformat.SegmentCacheMetricsSource
>
> executor.source.splexecutorquerycachemetrics.class=org.apache.spark.sql.execution.columnar.snapcache.QueryCacheMetricsSource

**In spark properties:**

> \# add snap jar to executor extra classpath, so Metric Sources can
>
> \# be initialized when the Executor starts
>
> \# for example
>
> spark.executor.extraClassPath=/Users/hbutani/sparkline/snap/target/scala-2.11/snap-assembly-1.0.0-SNAPSHOT.jar

#### Remote Attach for JConsole

**In spark.executor.extraJavaOptions, spark.driver.extraJavaOptions add:**

> -Dcom.sun.management.jmxremote.port=portNum -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false



