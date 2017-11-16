<!-- --- title: Create Star Schema -->

### Grammar
```text
createOrAlterStarSchema : createOrAlter STAR SCHEMA ON qualifiedId
                           opt(WITH STATS) 
                           opt(AS 
                               repsep(
                                  (tableInfo | joinInfo | columnEqInfo), 
                                  opt(",")
                                  )
                               )

createOrAlter : CREATE ~ opt(IF ~ NOT ~ EXISTS) |
                ALTER

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
Command to *create* or *alter* a **StarSchema** on a FactTable.
The definition is stored as a serialized `PersistedESSInfo` object graph in the table properties of
the Fact Table.

###Checks
- ALter must be called if StarSchema exists
- StarSchema cannot be altered, if there are `SNAP Indexes` defined on the Fact Table.
- Validates table and column existence
- All Table Keys are specified or can be inferred
- Any join with the `Fact Table` must be on the Dimension Table `key` columns
- Any join condition must have at least 1-side that involves key columns
- A Dimension Table can only 1 parent-child hierarchy

### Examples
```sql
-- tpch star schema
create star schema on lineitembase
as many_to_one join of lineitembase with orders on l_orderkey = o_orderkey
   many_to_one join of lineitembase with partsupp on
          l_partkey = ps_partkey and l_suppkey = ps_suppke
   many_to_one join of partsupp with part on ps_partkey = p_partkey
   many_to_one join of partsupp with supplier on ps_suppkey = s_suppkey
   many_to_one join of orders with customer on o_custkey = c_custkey
   many_to_one join of customer with custnation on c_nationkey = cn_nationkey
   many_to_one join of custnation with custregion on cn_regionkey = cr_regionkey
   many_to_one join of supplier with suppnation on s_nationkey = sn_nationkey
   many_to_one join of suppnation with suppregion on sn_regionkey = sr_regionkey
;

-- tpcds store_sales cube star schema
create star schema on store_sales as
   many_to_one join of store_sales with time_dim
     on  ss_sold_time_sk = t_time_sk
   many_to_one join of store_sales with store
     on ss_store_sk = s_store_sk
   many_to_one join of store_sales with date_dim
     on ss_sold_date_sk = d_date_sk
   many_to_one join of store_sales with item
     on ss_item_sk = i_item_sk
   many_to_one join of store_sales with promotion
     on ss_promo_sk = p_promo_sk
   many_to_one join of store_sales with customer_demographics
     on ss_cdemo_sk = cd_demo_sk
   many_to_one join of store_sales with customer
     on ss_customer_sk = c_customer_sk
   many_to_one join of store_sales with customer_address
     on ss_addr_sk = ca_address_sk
   many_to_one join of store_sales with household_demographics
     on ss_hdemo_sk = hd_demo_sk
   many_to_one join of store with date_dim
     on s_closed_date_sk = d_date_sk
   many_to_one join of promotion with date_dim
     on p_start_date_sk = d_date_sk
   many_to_one join of promotion with date_dim
     on p_end_date_sk = d_date_sk
   many_to_one join of promotion with item
     on p_item_sk = i_item_sk
   many_to_one join of customer with date_dim
     on c_customer_sk = d_date_sk
   many_to_one join of customer with customer_demographics
     on c_current_cdemo_sk = cd_demo_sk
   many_to_one join of customer with customer_address
     on c_current_addr_sk = ca_address_sk
   many_to_one join of customer with household_demographics
     on c_current_hdemo_sk = hd_demo_sk
   many_to_one join of household_demographics with income_band
     on hd_income_band_sk = ib_income_band_sk
   table date_dim
     d_date_id determines d_date_sk
     d_date determines d_date_sk
     d_day_name determines d_weekend
     level hierarchy d_week_seq, d_month_seq, d_quarter_seq
   table time_dim
     t_time_id determines t_time_sk
     t_time determines t_time_sk
     t_hour determines t_am_pm
   table store
      s_division_id determines s_division_name
      s_company_id determines s_company_name
   table promotion
     p_promo_id determines p_promo_sk
   table customer
     c_customer_id determines c_customer_sk
   table customer_address
     ca_address_id determines ca_address_sk
```
