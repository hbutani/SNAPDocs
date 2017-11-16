<!-- --- title: Export Model -->

### Grammar
```text
exportModel : EXPORT MODEL opt(COMPONENTS) rep(ident)
      ON qualifiedId opt(TO stringLit)

qualifiedId : ident opt("." ident)

```

### Description
Command to *export* a **SNAP Model**.
- *Components* can be `starschema`, `snapindex`, `stats` or `all`; default is `all`
- The user can optionally specify the root folder to export to; if not specified the `spark.sparklinedata.spmd.model.folder` setting controls the root folder location.
- Model components are exported to `db/table` folder under the root folder. There is a separate json file for each component.

### Examples
```sql
-- export all components
export model on lineitem_small to '/tmp';

-- export starschema
export model components starschema on lineitem_small to '/tmp'

-- export snapindex
export model snapindex on lineitem_small to '/tmp'
```