As described here this is what we encountered in *Constellation Brands*, but the technique is generic.

Consider a *Fact Source* with multiple *id* columns, wlog say `dist_id, account_id, eff_id, simp_id`. Also assume the dataset is for multiple years and there is a `year` column to identify the year of the fact. The dataset has many more dimensions and metrics, but for this use case we can ignore those details.

A **key analysis** that is performed is the `count distinct` of ids, for e.g. the change in number of accounts year-over-year. To quote the customer:
> This query surfaces in a *Mobile App* to e-staff and hence must be fast(1-second response time)

So consider a representative query:
```sql
select dist_id, count(distinct account_id)
from fact_source
where year = 2015
```

Also consider the following statistics of the data
```text
- 3 years worth of data
- each year is about 35 million facts
```

A typical starting point is to set up a **SNAP Index** partitioned by year, and so in this case going down this modeling path we set up Segments with around `625K` rows and `135` segments for 2 years worth of data.

But the above query was taking several seconds to perform. 

**First of all, you need to understand the reason for is this?**

- A `count distinct` is computed as a 2-level aggregation, with the first level doing a distinct at the column grain, so in our e.g. the inner aggregation is a `distinct dist_id, account_id`. The second level aggregation is a count on the count distinct column.
- The `distinct` query's performance and output size is highly dependent on the `number-of-distinct`(ndv) values of the `distinct` expression; in our case the number of distinct combinations of `dist_id, account_id` was over `2 million`
- So initial SNAP Segment level Group-By operations where outputting `100s of thousands` of rows and the reduction of these `135` sets into 1 distinct set was a key reason for the time taking by the query.

Stepping back, note that this is a common pattern: performing set operations (count distinct, set difference) on `ids`; and if the id-set is large these can be very expensive.

**So is there a way around this?**

The trick is to think of flipping the partitioning/work splitting on its head: instead of dividing the data based by time(year in this case) try to divide it by the high ndv id column; doing this means we can do set operations on small subspaces of the overall id-space. 

In this case we observed that the `acct_id` is probably functionally determined by `dist_id`. The `ndv` of the combination `dist_id, acct_id` was almost the same as the `acct_id`; this implied that `acct_ids` were not shared across `dist_ids`

So we partitioned the dataset by acct_id into `100` segments. The expression used was: `pmod( hash( acct_id), 100 )`. Why `100`? We wanted to divide the `acct_id` space such that each *SNAP Segment* group-by operation only output a few `10s` of thousands of rows. Given `2 million` ndv and spreading over `100` partitions we get around `20k` ids. 

**So, the difference is:**
- each Segment task outputs around `20k` rows as opposed to `100s of thousands of rows`
- the reduction into 1 set is much simpler: the size of individual mappers is much smaller. Also each mapper is ending rows to a small(ideally 1) number of reducers.
- the smaller query output sizes at the segment level means these are more amiable to query caching.


> Note, this is a simplification of the CBrands use-case
> CBrands query involved 3 count-distincts in 1 query
> but the ids were all correlated to the technique worked by partition on the 
> highest cardinality id column
