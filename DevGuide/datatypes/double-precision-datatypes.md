<!-- --- title: Double Precision DataTypes -->

We support 2 double precision types for the Index:

1. **IDoublePrecisionType: **whose internal representation is that of a java double.
2. **IBigDecimalType**
   * is similar to Spark's Decimal Type
   * user needs to specify the precision and scale
   * internally these are stored in a fashion similar to Spark, and is based on the precision of the Type: for precision of up to 18, values are stored as Long, for values with higher precision they are stored as byte arrays.
   * The precision can be at most 38 digits

Here is an example of how to define an Index with these types:

```
CREATE TABLE if not exists datatypes_table(
          o_orderkey int,
          float_value float,
          double_value double,
          double_value2 double,
          bigdecimal_long_scale decimal(18,14),
          bigdecimal_long_scale2 decimal(18,14),
          bigdecimal_large_scale decimal(38,30),
          bigdecimal_large_scale2 decimal(38,30)
          )
      USING csv
      OPTIONS (path "$datatypesTableFolder",
      header "false", delimiter "|")

val indexInfos = SPLMDFUtils.asJson(Seq(
      IndexMetricInfo("double_value", false, "0.0", DoublePrecisionSumAggregator, None),
      IndexMetricInfo("double_value2", false, "0.0", DoublePrecisionSumAggregator, None),
      IndexMetricInfo("bigdecimal_long_scale", false, "0.0", BigDecimalSumAggregator(18,14), None),
      IndexMetricInfo("bigdecimal_long_scale2", false, "0.0", BigDecimalSumAggregator(18,14), None),
      IndexMetricInfo("bigdecimal_large_scale", false, "0.0", BigDecimalSumAggregator(38,30), None),
      IndexMetricInfo("bigdecimal_large_scale2", false, "0.0", BigDecimalSumAggregator(38,30), None)
    )).replaceAll("\n", "")

create olap index datatypes_table_index on datatypes_table
      dimensions "o_orderkey"
      metrics "float_value"
      OPTIONS (
        path "${datatypesTableFolderIndex}",
        indexFieldInfos '$indexInfos',
        nonAggregateQueryHandling "push_project_and_filters",
        allowTopNRewrite "true"
        preferredSegmentSize "200mb"
      ) 
```

As of this writing, the internal **indexFieldInfos **parameter containing the json of the MetricInfos has to be used; parser support has not been added.

### Caveats, Notes about Behavior

1. **Sum Aggregation**
   * The Sum Aggregator for the Index behaves differently from Spark's Sum Aggregation Function. 
   * In a $$sum(decimalCol)$$ where the precision of 'decimalCol' is say 10, Spark sets the precision of the output as 20, i.e. Spark increases the output precision by up to 10 digits. 
   * But the Index Sum Aggregator doesn't change the output precision. The reason for this is that the Aggregator is called even during Ingestion, if we increase the output values precision in this case then the storage cost may increase significantly.
   *  _A work around is to set the precision in the Metric definition to account for this; but bare in mind\(especially at the 18 digit boundary, this can have an impact on storage\)._
2. _**Javascript Support**_
   * We don't support JavaScript for the **IBigDecimalType. **So any expressions on these will cause Fact rows to be pulled out of the Index and operated on in Spark Operators.



