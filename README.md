# CompanyX_ETL

SQL Server Integration Services (SSIS) project for loading the CompanyX
source database into the CompanyX_DW data warehouse.

The project contains dimension and fact-loading packages, plus a MasterLoad
package that runs the complete ETL process in the correct order.

## Project contents

- CompanyX_ETL.slnx
  Visual Studio solution file.

- CompanyX_ETL.dtproj
  SSIS project file using the Project Deployment Model.

- LoadProductDim.dtsx
  Loads product and product-price history data into dw.ProductDim.
  Product history is handled as an SCD Type 2 dimension.

- LoadCustomerDim.dtsx
  Loads customer data into dw.CustomerDim.
  Customer data is handled as an SCD Type 1 dimension.

- LoadOrderLineFact.dtsx
  Loads shipped order lines into dw.OrderLineFact.
  Product dimension keys are resolved using the order date.

- MasterLoad.dtsx
  Runs the three load packages in this order:

  1. LoadProductDim
  2. LoadCustomerDim
  3. LoadOrderLineFact
  4. AdvanceWatermark

- Project.params
  SSIS project parameter file.

## Prerequisites

Install or have access to:

- SQL Server Developer edition or another compatible SQL Server edition.
- The CompanyX source database.
- The CompanyX_DW warehouse database.
- Visual Studio.
- SQL Server Integration Services Projects extension for Visual Studio.
- SQL Server Management Studio (SSMS), recommended for database setup and
  validation.
- Microsoft OLE DB Driver for SQL Server (MSOLEDBSQL).

The source database must contain the CompanyX tables referenced by the
packages, including tables in the Sales and Production schemas.

The CompanyX and CompanyX_DW databases are expected to be available on the
same SQL Server instance.

## Database connections

The packages currently use Windows integrated authentication with these
connections:

- CompanyX
  - Server: localhost
  - Database: CompanyX

- CompanyX_DW
  - Server: localhost
  - Database: CompanyX_DW

If SQL Server is running on another server or named instance, update the
connection managers in the SSIS project before running the packages.

The Windows account running Visual Studio or SSISDB must have permission to
read the CompanyX database and read/write the CompanyX_DW database.

Do not commit database passwords or other credentials to this repository.

## Database setup

Before running the packages, create and prepare the warehouse database using
the SQL scripts in the parent repository:

1. 01_create_CompanyX_DW.sql
2. 02_staging_and_productdim_query.sql
3. 03_staging_and_customerdim.sql
4. 04_staging_and_orderlinefact.sql

The scripts create the CompanyX_DW database, schemas, dimension tables,
staging tables, fact tables, and ETL control objects.

The SQL scripts are located in the parent repository's data_warehouse_script
directory.

The warehouse should be available before opening or executing the SSIS
packages.

## Open the project

1. Open Visual Studio.
2. Install the SQL Server Integration Services Projects extension if it is
   not already installed.
3. Open CompanyX_ETL.slnx or CompanyX_ETL.dtproj.
4. Review the connection managers for CompanyX and CompanyX_DW.
5. Change the server name if SQL Server is not running on localhost.
6. Build the project.

## Run individual packages

Packages can be run individually for testing or troubleshooting.

Recommended order:

1. Open and run LoadProductDim.dtsx.
2. Open and run LoadCustomerDim.dtsx.
3. Open and run LoadOrderLineFact.dtsx.

The order is important because LoadOrderLineFact depends on the product and
customer dimensions already being available.

Each package truncates and reloads its staging table before loading the target
warehouse object.

## Run the complete ETL

To run the complete process:

1. Open MasterLoad.dtsx.
2. Confirm the CompanyX_DW connection manager is configured correctly.
3. Run the package.

MasterLoad executes the packages in this order:

LoadProductDim -> LoadCustomerDim -> LoadOrderLineFact -> AdvanceWatermark

The AdvanceWatermark task runs after the three load packages succeed. It
updates the replay watermark and inserts a successful run into etl.RunLog.

If an earlier package fails, the later tasks should not run and the watermark
should not advance.

## Optional replay workflow

The project supports incremental date-window replay for simulating
near-real-time loading.

Configure the replay in the CompanyX_DW database using the SQL script:

05_replay_setup.sql

The replay uses:

- etl.ReplayConfig
  Controls the number of days loaded in each run.

- etl.vReplayWindow
  Shows the current replay window.

- etl.Watermark
  Stores the last successfully loaded date.

- etl.RunLog
  Records completed ETL runs.

- etl.usp_ReplayReset
  Clears the loaded dimensions and fact table and rewinds the replay clock.

To start a replay demonstration:

1. Run the replay setup script in SSMS.
2. Reset the warehouse:

   EXEC etl.usp_ReplayReset;

3. Set the replay window, for example:

   UPDATE etl.ReplayConfig
   SET WindowDays = 7;

4. Run MasterLoad.dtsx from Visual Studio.
5. Check the replay state:

   SELECT *
   FROM etl.vReplayWindow;

6. Repeat the MasterLoad execution until the replay reaches the final source
   date.

The replay should not advance the watermark unless all three load packages
complete successfully.

## Validate the load

Run the following queries in CompanyX_DW after a successful load:

    SELECT COUNT(*) AS ProductVersions
    FROM dw.ProductDim;

    SELECT COUNT(*) AS Customers
    FROM dw.CustomerDim;

    SELECT COUNT(*) AS OrderLines
    FROM dw.OrderLineFact;

    SELECT LastWatermark, RowsLoaded
    FROM etl.Watermark
    WHERE TableName = N'dw.OrderLineFact';

    SELECT TOP (20) *
    FROM etl.RunLog
    ORDER BY FinishedAt DESC;

Expected full-load values are approximately:

- ProductDim: 353 product versions
- CustomerDim: 19,119 customers
- OrderLineFact: 120,147 shipped order lines

Exact values may differ if the source database has changed.

## Data-loading behavior

- Only shipped orders are loaded into OrderLineFact.
- Product price and cost changes are stored as separate ProductDim versions.
- Customer changes overwrite the existing CustomerDim row.
- Staging tables are reloaded for each package execution.
- The fact load uses the order date to select the appropriate product
  dimension version.
- The warehouse keeps anomalous order lines for reconciliation.
- Association-rule mining should exclude lines with null, zero, or negative
  order quantities or prices.

## Troubleshooting

### Connection errors

- Confirm that SQL Server is running.
- Confirm that the server name is correct.
- Confirm that CompanyX and CompanyX_DW exist.
- Confirm that Windows integrated authentication is enabled.
- Confirm that the current Windows account has the required permissions.
- Test both connections in SSMS before running the package.

### Missing table or schema errors

Run the warehouse SQL scripts from the parent repository in the documented
order. Make sure the CompanyX source database contains the expected Sales and
Production schemas.

### LoadOrderLineFact fails

Make sure LoadProductDim and LoadCustomerDim completed successfully first.
The fact load depends on both dimensions.

### Replay watermark does not advance

Check the package execution result, etl.vReplayWindow, etl.Watermark, and
etl.RunLog. The watermark is updated only by the final AdvanceWatermark task
in MasterLoad.

### Unexpected row counts

Review the validation queries included in the warehouse SQL scripts. Also
check whether the source data has changed or whether the package is being run
in replay mode rather than full-load mode.

## Relationship to the parent repository

This repository is included as a Git submodule by the main BI repository.

From the parent repository, initialize the submodule with:

    git submodule update --init --recursive

To clone the parent repository and initialize this project in one command:

    git clone --recurse-submodules <parent-repository-url>

Changes to this SSIS project must be committed in this repository first.
The parent repository then needs a separate commit updating the submodule
pointer to the new commit.
