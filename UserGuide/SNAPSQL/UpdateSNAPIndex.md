<!-- --- title: Update SNAP Index -->

### Grammar
```text
insertOrUpdateOlapIndex :
    UPDATE OLAP INDEX ~ qualifiedId OF qualifiedId 
      opt(PARTITIONS partitionsPredicates) opt(LISTONLY)

partitionsPredicates : rep1sep(partPredicate, opt(","))
partPredicate : ident "=" stringLit
qualifiedId : ident opt("." ident)

```

### Description
Command to *update* a **SNAP Index**.
- figures out what partitions need to be **indexed**: because they are new or the source partition has changed. Uses the `lastModTime` value of files within each partition to infer this.
  - for non-partitioned tables, the index is recreated if there exists a source file such that its  `lastModTime` exceeds that of any index segment file.
- the `listOnly` option can be used to see what partitions will be indexed, without actually triggering the index operation.

### Examples
```sql
-- update of a non-partitioned table
update olap index tpch_flat_index of orderLineItemPartSupplierBase;

-- listOnly command
update olap index tpch_flat_index of
 orderLineItemPartSupplierBase list_only;

-- update specific partitions
update olap index tpch_flat_part_index
of tpch_flat_small_part partitions shipYear="1994", shipMonth="12";

```