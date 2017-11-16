
# This notebook is based on sample data to illustrate metric binning

## Connect to SNAP


```python
%load_ext sql
```


```python
%%sql 
hive://ec2-34-204-193-242.compute-1.amazonaws.com:10000/default?auth=NOSASL
```




    u'Connected: None@default'



## Defining Bins when creating the SNAP Qube

### First lets refresh our memory on how metrics are defined in a standard Qube


```python
create olap index ssb_star_index_p10 on lineorder_sp_10
timestamp dimension lo_orderdate spark timestampformat "YYYYMMDD"
dimension p_name is nullable nullValue "NA"
dimension p_mfgr is nullable nullValue "NA"
dimension p_category is nullable nullValue "NA"
..
..
** Metrics are defined on a line with the "metric" keyword. **
metric lo_quantity aggregator doubleSum is nullable nullValue "0.0"
metric lo_extendedprice aggregator doubleSum is nullable nullValue "0.0"
metric lo_discount aggregator doubleSum is nullable nullValue "0.0"
metric lo_revenue aggregator longSum is nullable nullValue "0"
metric lo_supplycost aggregator longSum is nullable nullValue "0"
metric lo_tax aggregator doubleSum is nullable nullValue "0.0"
```

### Next lets see how we define bins

###### Structure of Bins

<h5> Dimensions as binned metric </h5>
<p> In the original table PART, the field p_size is defined as an integer. The field p_size has 50 values. This
field can be defined as dimension but for illustration purposes we have defined this as a binned metric. 
In this case we have defined "explicit bins by bucketing the 50 values in 4 buckets [ 0-10, 11-25, 26-40, 41- ]
</p>
<p>
{
    "jsonClass": "IndexMetricInfo",
    "column": "p_size",
    "isNullable": true,
    "nullValue": "-1",
    "aggregator": "LongSumAggregator",
    "extractInfo": {
      "jsonClass": "LongExplicitBinsMetricInfo",
      "startValues": [
        0,
        10,
        25,
        40
      ]
    }
  }
 </p>
<h5> Measures as binned metric </h5>
<p> Now lets us look at a typical measure, in this case ExtendedPrice as defined by the field lo_extendedprice. First we want to see the range of values for this field in the original dataset. 
We can do that using the following command in SNAP 
"analyze olap table lineorder_base 50 percent columns lo_extendedprice"

![image.png](attachment:image.png)

We then take the min and max and plug into the JSON as follows

<p>  {
    "jsonClass": "IndexMetricInfo",
    "column": "lo_extendedprice",
    "isNullable": true,
    "nullValue": "0.0",
    "aggregator": "DoubleSumAggregator",
    "extractInfo": {
      "jsonClass": "DoubleNumBinsMetricInfo",
      "minValue": 50000,
      "maxValue": 15000000,
      "numBins": 5
    }
  }
 </p>

<h4> Putting it all together </h4>
<p> Now that we know the structure of defining the metric bins we can put them all together as comma separated JSONs as in the next cell. We can then plug this in to the "options" section of the CREATE OLAP INDEX COMMAND where instead of explicitly specifying the metrics as before, they will be structured as a JSON we defined in an option names indexFieldInfo</p>

<p>
CREATE OLAP INDEX .......
...
options (path "s3a://snap-samples/ssb10/snap/",  
         nonaggregatequeryhandling "push_project_and_filters",
         avgsizeperpartition  "20000mb",
        preferredsegmentsize "300mb",
         indexFieldInfos '[{"jsonClass":"IndexMetricInfo","column":"p_size","isNullable":true,"nullValue":"-1","aggregator":"LongSumAggregator","extractInfo":{"jsonClass":"LongExplicitBinsMetricInfo","startValues":[0,10,25,40]}},{"jsonClass":"IndexMetricInfo","column":"lo_revenue","isNullable":true,"nullValue":"-1","aggregator":"LongSumAggregator","extractInfo":{"jsonClass":"LongNumBinsMetricInfo","minValue":0,"maxValue":1000000,"numBins":10}},{"jsonClass":"IndexMetricInfo","column":"lo_supplycost","isNullable":true,"nullValue":"0","aggregator":"LongSumAggregator","extractInfo":{"jsonClass":"LongNumBinsMetricInfo","minValue":0,"maxValue":200000,"numBins":10}},{"jsonClass":"IndexMetricInfo","column":"lo_discount","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}},{"jsonClass":"IndexMetricInfo","column":"lo_ordtotalprice","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}},{"jsonClass":"IndexMetricInfo","column":"lo_extendedprice","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}},{"jsonClass":"IndexMetricInfo","column":"lo_tax","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}},{"jsonClass":"IndexMetricInfo","column":"lo_quantity","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}}]',
        rowflushboundary "50000")

</p>


```python
[
  {
    "jsonClass": "IndexMetricInfo",
    "column": "p_size",
    "isNullable": true,
    "nullValue": "-1",
    "aggregator": "LongSumAggregator",
    "extractInfo": {
      "jsonClass": "LongExplicitBinsMetricInfo",
      "startValues": [
        0,
        10,
        25,
        40
      ]
    }
  },
  {
    "jsonClass": "IndexMetricInfo",
    "column": "lo_revenue",
    "isNullable": true,
    "nullValue": "-1",
    "aggregator": "LongSumAggregator",
    "extractInfo": {
      "jsonClass": "LongNumBinsMetricInfo",
      "minValue": 50000,
      "maxValue": 11494850,
      "numBins": 10
    }
  },
  {
    "jsonClass": "IndexMetricInfo",
    "column": "lo_supplycost",
    "isNullable": true,
    "nullValue": "0",
    "aggregator": "LongSumAggregator",
    "extractInfo": {
      "jsonClass": "LongNumBinsMetricInfo",
      "minValue": 50000,
      "maxValue": 130000,
      "numBins": 10
    }
  },
  {
    "jsonClass": "IndexMetricInfo",
    "column": "lo_discount",
    "isNullable": true,
    "nullValue": "0.0",
    "aggregator": "DoubleSumAggregator",
    "extractInfo": {
      "jsonClass": "DoubleNumBinsMetricInfo",
      "minValue": 0,
      "maxValue": 10,
      "numBins": 5
    }
  },
  {
    "jsonClass": "IndexMetricInfo",
    "column": "lo_ordtotalprice",
    "isNullable": true,
    "nullValue": "0.0",
    "aggregator": "DoubleSumAggregator",
    "extractInfo": {
      "jsonClass": "DoubleNumBinsMetricInfo",
      "minValue": 80000,
      "maxValue": 56290557,
      "numBins": 5
    }
  },
  {
    "jsonClass": "IndexMetricInfo",
    "column": "lo_extendedprice",
    "isNullable": true,
    "nullValue": "0.0",
    "aggregator": "DoubleSumAggregator",
    "extractInfo": {
      "jsonClass": "DoubleNumBinsMetricInfo",
      "minValue": 90000,
      "maxValue": 10494950,
      "numBins": 5
    }
  },
  {
    "jsonClass": "IndexMetricInfo",
    "column": "lo_tax",
    "isNullable": true,
    "nullValue": "0.0",
    "aggregator": "DoubleSumAggregator",
    "extractInfo": {
      "jsonClass": "DoubleNumBinsMetricInfo",
      "minValue": 0,
      "maxValue": 8,
      "numBins": 1
    }
  },
  {
    "jsonClass": "IndexMetricInfo",
    "column": "lo_quantity",
    "isNullable": true,
    "nullValue": "0.0",
    "aggregator": "DoubleSumAggregator",
    "extractInfo": {
      "jsonClass": "DoubleNumBinsMetricInfo",
      "minValue": 1,
      "maxValue": 50,
      "numBins": 5
    }
  }
]
```


## The full OLAP Index command with Binning 


```python
indexinfo='[{"jsonClass":"IndexMetricInfo","column":"p_size","isNullable":true,"nullValue":"-1","aggregator":"LongSumAggregator","extractInfo":{"jsonClass":"LongExplicitBinsMetricInfo","startValues":[0,10,25,40]}},{"jsonClass":"IndexMetricInfo","column":"lo_revenue","isNullable":true,"nullValue":"-1","aggregator":"LongSumAggregator","extractInfo":{"jsonClass":"LongNumBinsMetricInfo","minValue":0,"maxValue":1000000,"numBins":10}},{"jsonClass":"IndexMetricInfo","column":"lo_supplycost","isNullable":true,"nullValue":"0","aggregator":"LongSumAggregator","extractInfo":{"jsonClass":"LongNumBinsMetricInfo","minValue":0,"maxValue":200000,"numBins":10}},{"jsonClass":"IndexMetricInfo","column":"lo_discount","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}},{"jsonClass":"IndexMetricInfo","column":"lo_ordtotalprice","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}},{"jsonClass":"IndexMetricInfo","column":"lo_extendedprice","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}},{"jsonClass":"IndexMetricInfo","column":"lo_tax","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}},{"jsonClass":"IndexMetricInfo","column":"lo_quantity","isNullable":true,"nullValue":"0.0","aggregator":"DoubleSumAggregator","extractInfo":{"jsonClass":"DoubleNumBinsMetricInfo","minValue":0,"maxValue":150000,"numBins":5}}]'
```


```python

create olap index ssb_star_index_p10 on lineorder_sp_10
timestamp dimension lo_orderdate spark timestampformat "YYYYMMDD"
dimension p_name is nullable nullValue "NA"
dimension p_mfgr is nullable nullValue "NA"
dimension p_category is nullable nullValue "NA"
dimension p_brand1 is nullable nullValue "NA"
dimension p_color is nullable nullValue "NA"
dimension p_container is nullable nullValue "NA"
dimension c_name is nullable nullValue "NA"
dimension c_city is nullable nullValue "NA"
dimension c_nation is nullable nullValue "NA"
dimension c_region is nullable nullValue "NA"
dimension c_mktsegment is nullable nullValue "NA"
dimension s_name is nullable nullValue "NA"
dimension s_city is nullable nullValue "NA"
dimension s_nation is nullable nullValue "NA"
dimension s_region is nullable nullValue "NA"
dimension d_date is nullable nullValue "NA"
dimension d_month is nullable nullValue "NA"
dimension d_dayofweek is nullable nullValue "NA"
dimension d_year is nullable nullValue "NA"
dimension d_yearmonthnum is nullable nullValue "NA"
dimension d_yearmonth is nullable nullValue "NA"
dimension d_daynuminweek is nullable nullValue "NA"
dimension d_daynuminmonth is nullable nullValue "NA"
dimension d_daynuminyear is nullable nullValue "NA"
dimension d_monthnuminyear is nullable nullValue "NA"
dimension d_weeknuminyear is nullable nullValue "NA"
dimension d_sellingseason is nullable nullValue "NA"
dimension d_lastdayinweekfl is nullable nullValue "NA"
dimension d_lastdayinmonthfl is nullable nullValue "NA"
dimension d_holidayfl is nullable nullValue "NA"
dimension d_weekdayfl is nullable nullValue "NA"
dimension lo_orderpriority is nullable nullValue "NA"
dimension lo_shipmode is nullable nullValue "NA"
options (path "s3a://snap-samples/ssb10/snap/",  
         nonaggregatequeryhandling "push_project_and_filters",
         avgsizeperpartition  "20000mb",
        preferredsegmentsize "300mb",
         indexFieldInfos :indexinfo,
        rowflushboundary "50000")
partition by p_year
```


```python

```
