## Approaching a SNAP project 

### Overview

- Planning a SNAP project 
- The starting point of any SNAP project is to determine the datasets you will be working with. 
	- **Location/Path** : Where is the data?
		- Is it in Hive/S3 or some other database? 
		- What is the format of the various datasets?
			- CSV, Parquet, Orc etc
		- Is there one table or multiple tables represented in a Star/Snowflake schema
		   -If there is no denormalized dataset or  a star/snowflake then the first exercise is to take the datsets and use cases and make then into a star/snowflake where there is a fact and set of dimensions. Each SNAP Qube is defined on one set of StarSchema(Facts+0 or more dimensions)
Once you have identified your fact tables and dimension tables that will be part of the Qube
			- Star Schema or Snow flake?
			- Are the join keys known and defined. 
			- Validation to make sure that joins don’t result in row explosion( number of rows in the fact should be = number of rows on the join) ( Use analyze OLAP table to find column attributes and values ) 
When the star Schema is well defined and validated we now move on define the OLAP Index. ( Note for a flat table you don’t need to define a starSchema in SNAP)
Basic definitions
		- Decide which columns across all the tables in the join graph should be added to the SNAP Qube.
		- Defining Dimensions
			- Typically most string columns will be dimensions. These are columns you will have in your SELECT or WHERE clause ( Projections and Filters)
			- Most ID columns will be dimensions( even if the datatype is INT)
				- Note: Very high cardinality dimensions can be defined a Metric - See Advanced Metric Definitions
		- Defining Metrics
			- Columns that can be aggregated. ( columns that can be summed, averaged etc)
		- Defining timestamp dimensions
			- Columns that are of datatype timestamp or string columns containing date/time in an ISO format
		       - Which columns are time stamp and need extractions in the form of Year, Month, Day etc. 
			- What is the NDV of each dimension column? Are there columns that have over a million unique values?
				- If so model them as metrics 
				- Decide on a Binning strategy( Metric bands that can speed up filtering of data) for Metric columns. 
				- Do you have unique ID , Foreign key columns that you want in the cube?
					- If Yes model them as metrics. 
