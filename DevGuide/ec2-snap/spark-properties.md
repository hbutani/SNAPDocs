```
fs.default.name=local
spark.sql.warehouse.dir=/tmp

#spark.executor.instances=2
#spark.executor.cores=2

#spark.dynamicAllocation.enabled=true

#spark.shuffle.service.enabled    true
#spark.shuffle.consolidateFiles=true
#spark.shuffle.sort.bypassMergeThreshold 5

spark.serializer=org.apache.spark.serializer.KryoSerializer
#spark.sql.autoBroadcastJoinThreshold=100000000
spark.sql.autoBroadcastJoinThreshold=-1
#spark.sql.planner.externalSort=true

#spark.rdd.compress=true
#spark.externalBlockStore.baseDir=/data1,/data2
#spark.local.dir=/data1,/data2

spark.app.name=SNAP

spark.scheduler.mode=FAIR
spark.scheduler.pool=sparkline
spark.sql.thriftserver.scheduler.pool=sparkline
spark.scheduler.allocation.file=/home/ec2-user/spark-2.0.2-bin-hadoop2.7/conf/fairscheduler.xml

spark.executor.memory=14g
spark.driver.memory=4g
#spark.executor.cores=2

spark.sql.codegen.enabled=true
spark.sql.tungsten.enabled=true
spark.sql.unsafe.enabled=true
#spark.shuffle.manager=tungsten-sort

spark.storage.memoryFraction=0.5
spark.shuffle.memoryFraction=0.1

spark.shuffle.consolidateFiles=true

spark.sql.shuffle.partitions=10
spark.default.parallelism=10

spark.driver.extraJavaOptions=-XX:+UseG1GC -verbose:gc
#spark.driver.extraJavaOptions=-XX:+UseG1GC -Dhdp.version=2.3.4.0-3485 -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,address=4000,server=y,suspend=y
spark.executor.extraJavaOptions=-verbose:gc -XX:MaxDirectMemorySize=9g -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:OnOutOfMemoryError='kill -9 %p'
#spark.driver.extraJavaOptions=-DsocksProxyHost=localhost -DsocksProxyPort=25000
#spark.driver.extraJavaOptions=-Djava.net.useSystemProxies=true -DsocksProxyHost=localhost -DsocksProxyPort=25000 -Dhttp.proxyHost=localhost -Dhttp.proxyPort=25000


spark.sparklinedata.druid.max.connections=200
spark.sparklinedata.druid.max.connections.per.route=50

#spark.sparklinedata.modules=org.apache.spark.sql.hive.sparklinedata.MySqlCompatibilityModule
spark.sparklinedata.enable.druid.query.history=true

#olap indexing
spark.sparklinedata.spmd.local.segment.cache={"storageLocations" : [{"path" : "/mnt/olapcache", "maxSize" : 100000000000 }], "columnCacheSizeBytes" : 200000, "avgSizePerCacheFile" : 52428800, "LocalUnzipFileSizeFactor" : 2, "shareCacheAcrossExecutors" : true }
spark.sql.files.maxPartitionBytes=2147483648

spark.sparklinedata.spmd.gByEngine.offheapsize=256mb
spark.sql.sources.schemaStringLengthThreshold=1000
spark.sparklinedata.druid.option.allowTopN=true
spark.sparklinedata.druid.option.topNMaxThreshold=1000

#AWS
spark.hadoop.fs.s3a.access.key=<ur key>
spark.hadoop.fs.s3a.secret.key=<ur secret key>
#spark.jars.packages=net.java.dev.jets3t:jets3t:0.9.0,com.google.guava:guava:16.0.1,com.amazonaws:aws-java-sdk:1.7.4,org.apache.hadoop:hadoop-aws:2.7.1
spark.jars.packages=com.amazonaws:aws-java-sdk:1.7.4,org.apache.hadoop:hadoop-aws:2.6.3
#spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem
```



