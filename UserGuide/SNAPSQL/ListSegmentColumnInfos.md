<!-- --- title: List SNAP Segment Details -->

### Grammar
```text

show olap index segments [indexName] [of|on] [tableName] [partitionSpec]

partitionSpec ::
   partitions (col_name = string_literal)+

```

### Description
- index and/or table must be specified. At least one of them must be specified.
- if partition spec is not specified information is returned for all partitions.
- output has the shape:
```text
 segmentFile: String
 columnName: String
 dictionarySize: Int
 bitmapSize: Int
 columnSize: Int
 totalSize: Int
 numDistincts: Int
 numRows: Int
 minValue: String
 maxValue: String
 valueHistogram: String
 bitMapSizeHistogram: String
```
- for numeric metrics, value historgram is an indicative histogram of upto 10 buckets
- for dimensions, bitmapSize histogram gives the distribution of sizes of position bitmaps; 
  this is an indication of the number of rows per unique value.
  
### Examples
```sql

-- all segments
show olap index segments tpch_index_part_ryear of lineitem_tiny;

-- partition clause
show olap index segments tpch_index_part_ryear of lineitem_tiny
   partitions receiptYear = '1996';
   
-- fully qualified index name
show olap index segments default.tpch_index_part_ryear of lineitem_tiny
   partitions receiptYear = '1996';

--  based on table name
show olap index segments of lineitem_tiny
   partitions receiptYear = '1996';
   
-- based on index name
show olap index segments default.tpch_index_part_ryear
   partitions receiptYear = '1996';
```

### Sample Output
```text
+-------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+--------------+----------+----------+---------+------------+-------+--------+--------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------+
|segmentFile                                                                                                                                            |columnName     |dictionarySize|bitmapSize|columnSize|totalSize|numDistincts|numRows|minValue|maxValue|valueHistogram                                                                                                                                                                                                                                             |bitMapSizeHistogram                                                                                                                                             |
+-------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+--------------+----------+----------+---------+------------+-------+--------+--------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------+
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|count          |0             |0         |282       |282      |0           |99     |null    |null    |[(1, 1) -> 99]                                                                                                                                                                                                                                             |null                                                                                                                                                            |
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|l_extendedprice|0             |0         |672       |672      |0           |99     |null    |null    |[(1230.2, 10831.2) -> 98, (10831.2, 20432.2) -> 0, (20432.2, 30034.2) -> 0, (30034.2, 39635.2) -> 0, (39635.2, 49237.2) -> 0, (49237.2, 58838.2) -> 0, (58838.2, 68440.2) -> 0, (68440.2, 78041.2) -> 0, (78041.2, 87643.2) -> 0, (87643.2, 97245.12) -> 1]|null                                                                                                                                                            |
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|l_quantity     |0             |0         |527       |527      |0           |99     |null    |null    |[(1.0, 5.0) -> 97, (5.0, 10.0) -> 0, (10.0, 15.0) -> 0, (15.0, 20.0) -> 0, (20.0, 25.0) -> 0, (25.0, 30.0) -> 0, (30.0, 35.0) -> 0, (35.0, 40.0) -> 0, (40.0, 45.0) -> 0, (45.0, 50.0) -> 2]                                                               |null                                                                                                                                                            |
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|p_name         |4007          |2562      |375       |6944     |99          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-48, -42) -> 1, (-42, -35) -> 0, (-35, -29) -> 0, (-29, -22) -> 0, (-22, -15) -> 0, (-15, -9) -> 0, (-9, -2) -> 0, (-2, 4) -> 0, (4, 11) -> 0, (11, 18) -> 97]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|s_address      |3173          |2562      |374       |6109     |99          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-48, -42) -> 1, (-42, -35) -> 0, (-35, -29) -> 0, (-29, -22) -> 0, (-22, -15) -> 0, (-15, -9) -> 0, (-9, -2) -> 0, (-2, 4) -> 0, (4, 11) -> 0, (11, 18) -> 97]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|sn_name        |377           |786       |375       |1538     |25          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-58, -50) -> 22, (-50, -41) -> 0, (-41, -32) -> 0, (-32, -23) -> 0, (-23, -14) -> 0, (-14, -6) -> 0, (-6, 3) -> 0, (3, 12) -> 0, (12, 21) -> 0, (21, 30) -> 2]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|sn_nationkey   |240           |786       |375       |1401     |25          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-62, -53) -> 22, (-53, -44) -> 0, (-44, -35) -> 0, (-35, -26) -> 0, (-26, -16) -> 0, (-16, -7) -> 0, (-7, 2) -> 0, (2, 11) -> 0, (11, 20) -> 0, (20, 30) -> 2]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|s_nationkey    |240           |786       |375       |1401     |25          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-62, -53) -> 22, (-53, -44) -> 0, (-44, -35) -> 0, (-35, -26) -> 0, (-26, -16) -> 0, (-16, -7) -> 0, (-7, 2) -> 0, (2, 11) -> 0, (11, 20) -> 0, (20, 30) -> 2]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|s_suppkey      |1177          |2562      |375       |4114     |99          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-48, -42) -> 1, (-42, -35) -> 0, (-35, -29) -> 0, (-29, -22) -> 0, (-22, -15) -> 0, (-15, -9) -> 0, (-9, -2) -> 0, (-2, 4) -> 0, (4, 11) -> 0, (11, 18) -> 97]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|p_mfgr         |110           |306       |369       |785      |5           |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-122, -86) -> 3, (-86, -49) -> 0, (-49, -12) -> 0, (-12, 25) -> 0, (25, 62) -> 1]                                                                             |
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|s_name         |2574          |2562      |375       |5511     |99          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-48, -42) -> 1, (-42, -35) -> 0, (-35, -29) -> 0, (-29, -22) -> 0, (-22, -15) -> 0, (-15, -9) -> 0, (-9, -2) -> 0, (-2, 4) -> 0, (4, 11) -> 0, (11, 18) -> 97]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|l_linenumber   |63            |354       |373       |790      |7           |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-120, -95) -> 5, (-95, -69) -> 0, (-69, -43) -> 0, (-43, -18) -> 0, (-18, 8) -> 0, (8, 34) -> 0, (34, 60) -> 1]                                               |
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|l_partkey      |1332          |2562      |375       |4269     |99          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-48, -42) -> 1, (-42, -35) -> 0, (-35, -29) -> 0, (-29, -22) -> 0, (-22, -15) -> 0, (-15, -9) -> 0, (-9, -2) -> 0, (-2, 4) -> 0, (4, 11) -> 0, (11, 18) -> 97]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|p_partkey      |1332          |2562      |375       |4269     |99          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-48, -42) -> 1, (-42, -35) -> 0, (-35, -29) -> 0, (-29, -22) -> 0, (-22, -15) -> 0, (-15, -9) -> 0, (-9, -2) -> 0, (-2, 4) -> 0, (4, 11) -> 0, (11, 18) -> 97]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|l_suppkey      |1177          |2562      |375       |4114     |99          |99     |null    |null    |null                                                                                                                                                                                                                                                       |[(-48, -42) -> 1, (-42, -35) -> 0, (-35, -29) -> 0, (-29, -22) -> 0, (-22, -15) -> 0, (-15, -9) -> 0, (-9, -2) -> 0, (-2, 4) -> 0, (4, 11) -> 0, (11, 18) -> 97]|
|file:///Users/hbutani/sparkline/snap/src/test/resources/spmd/tpch_star_part_ryear/receiptYear=1996/index_attempt_201707212309_0013_m_000004_0_00000.zip|receiptYear    |12            |222       |275       |509      |1           |99     |null    |null    |null                                                                                                                                                                                                                                                       |null                                                                                                                                                            |
+-------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+--------------+----------+----------+---------+------------+-------+--------+--------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------+


```