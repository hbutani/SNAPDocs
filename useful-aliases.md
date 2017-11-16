# ivy
alias ivy-fix="rm -fr ~/.ivy2/cache/org.apache.ivy/ ~/.ivy2/cache/org.scala-lang.modules/ ~/.ivy2/cache/jline/"

# snap
alias snap-build-test="build/sbt compile druidexts/test snapformat/test \"test-only *OLAPIndexTestSuite\""

alias snap-clean-build-test="build/sbt clean compile druidexts/test snapformat/test \"test-only *OLAPIndexTestSuite\""

alias snap-rm-test-indexes="rm -fr /Users/hbutani/sparkline/snap/snapformat/src/test/resources/spmd /Users/hbutani/sparkline/snap/src/test/resources/spmd/"

alias snap-assembly="build/sbt clean compile -Dproject.version=\"2.0.0-SNAPSHOT\" assembly"

alias snap-s3put="s3cmd put /Users/hbutani/sparkline/snap/target/scala-2.11/snap-assembly-2.0.0-SNAPSHOT.jar s3://splsnap/"

alias snap-start="sbin/start-sparklinedatathriftserver.sh \"/Users/hbutani/sparkline/snap/target/scala-2.11/snap-assembly-2.0.0-SNAPSHOT.jar\" --properties-file ~/hcluster/conf/spark.slicestore.properties --master "

alias snap-stop="sbin/stop-sparklinedatathriftserver.sh"

alias snap-beeline="./beeline -u 'jdbc:hive2://localhost:10000'"

alias snap-sh="./snap-shell /tmp/snap --properties-file ~/hcluster/conf/spark.slicestore.properties --master "