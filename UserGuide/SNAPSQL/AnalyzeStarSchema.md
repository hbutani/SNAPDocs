<!-- --- title: Analyze Star Schema -->

### Grammar
```text
analyzeStarSchema : ANALYZE STAR SCHEMA ON qualifiedId
                     opt(
                        AS 
                        repsep(
                          (tableInfo | joinInfo | columnEqInfo), 
                          opt(",")
                         )
                       )

tableInfo : TABLE qualifiedCol
             repsep(
                keyColumns | functionalDep | levelHierarchy | parentChildHierarchy, 
                opt(",")
              )
joinInfo : opt(starRelationType) opt(INDEX) 
             JOIN OF qualifiedId WITH qualifiedId ON repsep(joinEqualityCond, AND)
joinEqualityCond : qualifiedCol "=" qualifiedCol
columnEqInfo : COLUMNS qualifiedCol "=" qualifiedCol

functionalDep : opt(COLUMNS) qualifiedCols DETERMINES opt(COLUMNS) qualifiedCols
levelHierarchy : LEVEL ~ HIERARCHY qualifiedCols
parentChildHierarchy : PARENT CHILD HIERARCHY qualifiedCol opt(",") qualifiedCol


keyColumns : KEY ~ opt(COLUMNS) qualifiedCols


qualifiedCols : repsep(qualifiedCol, ",")
qualifiedCol : ident opt("." ident) ~ opt("." ident)


```

### Description
Use *analyze* command to profile the columns in the Tables involved in the Star Schema.
This command computes the `distinct_count, count, percentMissing and percentUnique` values for each column.

###Checks
- Validates table and column existence
- All Table Keys are specified or can be inferred
- Any join with the `Fact Table` must be on the Dimension Table `key` columns
- Any join condition must have at least 1-side that involves key columns
- A Dimension Table can only 1 parent-child hierarchy

### Examples
```sql
-- tpch star schema
analyze star schema on lineitem_small
as many_to_one join of lineitem_small with orders on l_orderkey = o_orderkey
   many_to_one join of lineitem_small with partsupp on l_partkey = ps_partkey and l_suppkey = ps_suppkey
   many_to_one join of partsupp with part on ps_partkey = p_partkey
   many_to_one join of partsupp with supplier on ps_suppkey = s_suppkey
   many_to_one join of orders with customer on o_custkey = c_custkey
   many_to_one join of customer with custnation on c_nationkey = cn_nationkey
   many_to_one join of custnation with custregion on cn_regionkey = cr_regionkey
   many_to_one join of supplier with suppnation on s_nationkey = sn_nationkey
   many_to_one join of suppnation with suppregion on sn_regionkey = sr_regionkey
;
```

### Sample Output
```text
+-------------------------------+----+--------------+-------+--------------+--------------------+
|ColumnName                     |Type|Distinct Count|Count  |PercentMissing|PercentUnique       |
+-------------------------------+----+--------------+-------+--------------+--------------------+
|default.orders.o_orderkey      |INT |1500000       |1500000|0.0           |100.0               |
|default.partsupp.ps_suppkey    |INT |10000         |800000 |0.0           |1.25                |
|default.partsupp.ps_partkey    |INT |200000        |800000 |0.0           |25.0                |
|default.part.p_partkey         |INT |200000        |200000 |0.0           |100.0               |
|default.partsupp.ps_partkey    |INT |200000        |800000 |0.0           |25.0                |
|default.supplier.s_suppkey     |INT |10000         |10000  |0.0           |100.0               |
|default.partsupp.ps_suppkey    |INT |10000         |800000 |0.0           |1.25                |
|default.customer.c_custkey     |INT |150000        |150000 |0.0           |100.0               |
|default.orders.o_custkey       |INT |99996         |1500000|0.0           |6.6664              |
|default.custnation.cn_nationkey|INT |25            |25     |0.0           |100.0               |
|default.customer.c_nationkey   |INT |25            |150000 |0.0           |0.016666666666666666|
|default.custregion.cr_regionkey|INT |5             |5      |0.0           |100.0               |
|default.custnation.cn_regionkey|INT |5             |25     |0.0           |20.0                |
|default.suppnation.sn_nationkey|INT |25            |25     |0.0           |100.0               |
|default.supplier.s_nationkey   |INT |25            |10000  |0.0           |0.25                |
|default.suppregion.sr_regionkey|INT |5             |5      |0.0           |100.0               |
|default.suppnation.sn_regionkey|INT |5             |25     |0.0           |20.0                |
+-------------------------------+----+--------------+-------+--------------+--------------------+
```