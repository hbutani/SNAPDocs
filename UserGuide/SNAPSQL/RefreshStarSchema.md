<!-- --- title: Refresh Star Schema -->

### Grammar
```text
refreshStarSchema : REFRESH STAR SCHEMA opt(ON) qualifiedId
qualifiedCol : ident opt("." ident) ~ opt("." ident)
```

### Description
Use to persist existing *SNAP 2.0* StarSchemas as *Extended StarSchemas*. As a best practise, run this
command on every StarSchema when upgrading to SNAP2.1

### Examples
```sql
refresh star schema lineitem_small
```
