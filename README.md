# Bank of Canada USD/CAD Exchange Rate Pipeline

A portfolio project by **Shaquille Thomas** that retrieves daily USD/CAD exchange rates from the Bank of Canada Valet API, loads them into Azure SQL Database, and presents the results in Power BI.

The project demonstrates API ingestion, SQL validation, rejection logging, duplicate prevention, and data freshness checks.

**Tools:** Azure Data Factory · Azure SQL Database · T-SQL · REST API · Power BI

## Dashboard Preview

![Power BI dashboard showing the USD/CAD exchange rate trend and latest loaded observation](docs/images/Updated%20Dashboard.png)

[Open the dashboard image](docs/images/Updated%20Dashboard.png) · [Download the Power BI report](Bank%20of%20Canada%20USDCAD.pbix)

## Architecture

1. **Bank of Canada Valet API** supplies observations from the `FXUSDCAD` series.
2. **Azure Data Factory** requests a supplied date range and passes the observations and pipeline run ID to SQL.
3. **Azure SQL Database** validates and loads the observations, logs rejected records, and checks freshness when enabled.
4. **Power BI Desktop** imports the stored observations for reporting.

Exchange rates are expressed as **Canadian dollars per US dollar**.

[View the Bank of Canada API endpoint](https://www.bankofcanada.ca/valet/observations/FXUSDCAD/json)

## Data Loading and Validation

- `dbo.LoadUsdCad` validates dates and positive numeric exchange rates.
- Rejected records are logged with the pipeline run ID.
- Valid observations are inserted or updated.
- A primary key on `(SeriesCode, ObservationDate)` prevents duplicate observations when the same date range is reloaded.
- When `CheckFreshness` is enabled, the freshness procedure compares the latest stored observation date with the expected publication date.
- Freshness calculations account for weekends and the holidays included in the reviewed calendar.

## Repository Contents

| Location | Contents |
|---|---|
| [ADF](ADF/) | Pipeline and daily trigger JSON exports |
| [SQL Setup](SQL%20Setup/) | Table definitions, stored procedures, and execution grants |
| [SQL Validation](SQL%20Validation/) | Test load, invalid-rate, backfill, and freshness checks |
| [Dashboard images](docs/images/) | Power BI dashboard screenshots |
| [Power BI report](Bank%20of%20Canada%20USDCAD.pbix) | Downloadable Power BI Desktop report |

## Reproduce the Pipeline

### 1. Create the Azure resources

Create an Azure SQL database and an Azure Data Factory with a managed identity.

### 2. Create the database objects

In the target database, run the SQL Setup scripts in this order:

1. Script **1** — create the exchange-rate table.
2. Script **2** — create the rejection table.
3. Script **3** — create the loading procedure.
4. Script **9** — create the freshness procedure.

### 3. Grant execution permissions

Replace the example factory identity name in scripts **4** and **10** with your own.

Run script **4** and the `GRANT` statement in script **10** to give the factory identity permission to execute both procedures.

The `EXEC` statement in script **10** is a separate manual test that requires historical data.

### 4. Configure Data Factory

Create an Azure SQL linked service named `ls_azure_sql_boc_fx` using the factory's managed identity.

Use the JSON exports in the `ADF` folder to recreate:

- `pl_load_usd_cad`
- `tr_daily_usd_cad`

Configure the linked service for your database and set the trigger start date for your deployment.

### 5. Run the historical backfill

Use these pipeline parameters:

| Parameter | Value |
|---|---|
| `StartDate` | `2025-11-16` |
| `EndDate` | `2026-11-15` |
| `CheckFreshness` | `false` |

To reproduce the small test load used by validation scripts **5** and **6**, also load January 2–6, 2025 with freshness disabled.

### 6. Test before enabling the schedule

Run the validation scripts and inspect the Data Factory activity results.

The exported daily trigger is configured for **6:00 p.m. Eastern**. It supplies:

- The scheduled local date as `EndDate`.
- Six calendar days earlier as `StartDate`.
- `CheckFreshness = true`.

The requested range includes both dates.

## Validation Results

| Script | Check | Observed result |
|---|---|---|
| 5 | Small test load | Three observations for January 2, 3, and 6, 2025 |
| 6 | Invalid-rate test | `bad-rate` was logged, while the existing January 2 rate remained `1.4418` |
| 7 | Historical backfill | 248 observations from November 17, 2025 through November 13, 2026 |
| 8 | Freshness date logic | A holiday, a regular weekday, and a Sunday returned `Fresh` |
| 10 | Direct stored-procedure check | The November 15, 2026 check completed successfully |

Script **6** writes a test rejection each time it runs. Run it in a development database after the small test load to compare the preserved valid rate with the rejected value.

## Execution History and Troubleshooting

### Missing date parameters

An initial Data Factory Debug run failed with SQL error `50002` when freshness checking was enabled.

The activity input showed `CheckDate = null`, and the API request contained blank date parameters. The freshness activity mapped `CheckDate` to the pipeline's `EndDate`.

Supplying both pipeline dates in `yyyy-MM-dd` format resolved the issue. A subsequent Debug run succeeded across all four activities, including the freshness check.

### SQL connection timeout

The daily trigger fired on **November 18, 2026 at 6:00 p.m. Eastern**, but the SQL load failed with this connection timeout:

> Server provided routing information, but timeout already expired.

A manual rerun initially failed with the same error. A subsequent linked-service connection test passed, and a later manual rerun on **November 19** succeeded across all four activities, including the freshness check.

The underlying cause of the temporary connectivity failure was not established.

Both SQL stored-procedure activities were configured for up to **three retries at 60-second intervals**. These settings were published to Azure and saved in the pipeline JSON.

**Verification status:** A successful manual execution has been verified. A successful unattended scheduled execution remains to be verified.

## Power BI Report

The report imports `dbo.ExchangeRate` from Azure SQL and filters on:

- Series: `FXUSDCAD`
- Dates: November 16, 2025 onward

The lower date bound excludes the January test load.

The report displays the historical exchange-rate trend, latest loaded observation date, and corresponding rate.

After the documented refresh, the latest observation was **November 18, 2026**, at **1.4326 CAD per USD**. This is a recorded project result, not a live exchange-rate quote.

To refresh the report:

1. Open the `.pbix` file in Power BI Desktop.
2. Update the Azure SQL data-source settings for your environment.
3. Select **Refresh** after loading new observations.

### Earlier Dashboard View

![Earlier Power BI dashboard overview](docs/images/Dashboard%20Overview.png)

## Calendar Maintenance

The freshness procedure currently accepts check dates from **November 11 through December 31, 2026**.

Extend the reviewed holiday calendar and supported date range before using freshness checks for 2027.

Check dates outside the supported range raise SQL error `50003`.

## Scope and Limitations

- The implemented pipeline covers USD/CAD only.
- The dashboard reflects the data loaded at its last refresh.
- Power BI Desktop refresh is manual in this documented implementation.
- Successful scheduled execution remains to be verified.
- Freshness checking depends on the supported calendar dates described above.

## Data Attribution

Source: [Bank of Canada Valet API](https://www.bankofcanada.ca/valet/docs), FXUSDCAD series.

Exchange-rate observations are used for an independent portfolio analysis. Data preparation and visualizations are by Shaquille Thomas.

This is an independent portfolio project using public Bank of Canada data. It is not affiliated with or endorsed by the Bank of Canada.
