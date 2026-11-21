Data Warehouse with Databricks

An end-to-end data warehouse built on Databricks using PySpark, SQL and Delta tables. Raw CSV exports from source systems (CRM and ERP) are loaded, cleaned and modeled into a star schema, following the medallion architecture (bronze, silver, gold).

Architecture
CSV files (Unity Catalog Volume)
        |
        v
   BRONZE  -> raw data, loaded as-is
        |
        v
   SILVER  -> cleaned and standardized data
        |
        v
    GOLD   -> star schema (dimensions + fact table) for reporting
Layer	Schema	Purpose
Bronze	workspace.bronze	Raw copies of the source files, no transformations
Silver	workspace.sliver	Cleaned, standardized tables
Gold	workspace.gold	Business-ready star schema
Tech stack
Databricks (serverless compute, Unity Catalog, Workflows)
PySpark and Spark SQL
Delta Lake
Power BI (reporting on top of the gold layer)
Data sources

Raw CSV files are stored in a Unity Catalog Volume and come from two source systems:

CRM: customer information, product information, sales details
ERP: customer demographics (cust_az12), customer locations (loc_a101), product categories (px_cat_g1v2)
Layers in detail
Bronze

Each CSV is read with the header option and schema inference, then saved as a Delta table with overwrite mode, so the load can be re-run safely.

Tables: cust_info, cust_az12, loc_a101, prd_info, prd_info2, px_cat_g1v2, sales_details

Silver

Each silver notebook reads from bronze and writes to silver, so re-running never depends on earlier output. Typical cleaning steps:

Trim leading and trailing spaces from all text columns
Standardize codes into readable values (for example M becomes MALE, F becomes FEMALE, M/D marital status becomes MARRIED/DIVORCED, anything else becomes N/A)
Split the ERP customer ID into a prefix and a key so it matches the CRM customer key
Remove dashes from the location customer ID
Derive a category key from the first 5 characters of the product key to join with product categories

Tables: cust_info, cust_az12, loc_a101, prod_info, prd_info_short, px_cat_g1v2, sales_details

Gold

Star schema built from the silver tables:

Table	Type	Description
dim_customers	Dimension	Customer details combined from CRM and ERP sources
dim_products	Dimension	Products joined with their category, with a generated surrogate product_key
fact_sales	Fact	Sales transactions joined to both dimensions through their keys
Job orchestration

The pipeline runs as a Databricks Job with task dependencies:

bronze
  |
  +--> customer_az12 --+
  +--> customer_info --+--> dim_Customers --+
  +--> loc_a101 -------+                     |
  +--> product_info ---+--> dim_products ----+--> fact_sales
  +--> px_cat_g1v2 ----+                     |
  +--> sales -----------------------------> -+

Gold tasks only start after all the silver tasks they depend on have finished successfully.

How to run
Upload the source CSV files to the Unity Catalog Volume.
Import the notebooks into your Databricks workspace (or use this repo as a Git folder).
Create a Job with the tasks and dependencies shown above, using serverless compute.
Click Run now.
Lessons learned
Always read from bronze in silver notebooks, so a notebook can run any number of times with the same result.
When a write changes a table's columns, use .option("overwriteSchema", "true") with overwrite mode.
Check df.columns before saving to catch duplicate column names.
Variables only live in the current session, so create a DataFrame in the same notebook that uses it.
Future improvements
Add data quality checks (null keys, duplicates, row counts) between layers
Add a schedule with a sensible frequency
Build Power BI dashboards on the gold tables
Rename the sliver schema to silver
