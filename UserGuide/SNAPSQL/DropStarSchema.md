<!-- --- title: Drop Star Schema -->

### Grammar
```text
dropStarSchema : DROP STAR SCHEMA opt(ON) qualifiedId
qualifiedCol : ident opt("." ident) ~ opt("." ident)
```

### Description
Command to *drop* a **StarSchema** on a FactTable.

###Checks
- StarSchema cannot be dropped, if there are `SNAP Indexes` defined on the Fact Table.

### Examples
```sql
drop star schema on lineitem_small
```