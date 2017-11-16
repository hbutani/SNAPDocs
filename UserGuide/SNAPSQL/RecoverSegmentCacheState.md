<!-- --- title: Recover Segment Cache State -->

### Grammar
```text
recover olap index cache state <indexName>
```

### Description
Used to recover information about segment files cached on cluster nodes.
Issue this command on a restart of the *Sparklinedata* thriftserver.
You can issue a `select * from snap$cachesegments` to see what is in the cluster node caches.

- `indexName` is the name of the index whose state should be recovered.


## Examples
```sql
recover olap index cache state tpch_index_part_oyear
```