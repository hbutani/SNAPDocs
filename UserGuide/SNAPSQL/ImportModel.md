<!-- --- title: Import Model -->

### Grammar
```text
importModel : IMPORT MODEL opt(COMPONENTS) rep(ident)
      ON qualifiedId opt(FROM stringLit)

qualifiedId : ident opt("." ident)

```

### Description
Command to *export* a **SNAP Model**.
- *Components* can be `starschema`, `snapindex`, `stats` or `all`; default is `all`
- The user can optionally specify the root folder to export to; if not specified the `spark.sparklinedata.spmd.model.folder` setting controls the root folder location.
- Model components are imported from `db/table` folder under the root folder. There should be a separate json file for each component.
- When importing the `SNAPIndex`, if an Index exists it is dropped and recreated.
- An `StarSchema` import is only allowed if there is no Index on the FactTable; in cases when there is an Index, import both the `StarSchema` and the `SNApIndex` in one import statement.

### Examples
```sql
-- import starschema, snapindex components
import model starschema snapindex on lineitem_small from '/tmp';

-- import starschema
export model components starschema on lineitem_small to '/tmp'

-- import snapindex
import model snapindex on lineitem_small to '/tmp'
```