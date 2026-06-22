# Retail Order Analysis — ETL Pipeline with Python & SQL

An end-to-end data pipeline that takes raw retail order data from Kaggle, cleans and transforms it with Python (Pandas), loads it into SQL Server, and then answers real business questions using window functions and CTEs.

## What this project does

Most "ETL projects" online stop at cleaning a CSV. This one goes the full distance — extract from Kaggle's API, transform with Pandas, load into a SQL Server database, and then actually *use* that data to answer the kind of questions a retail business or sales team would ask: which products make the most money, which regions are performing well, and how does growth look year-over-year.

## Dataset

[Retail Orders dataset](https://www.kaggle.com/datasets/ankitbansal06/retail-orders) by Ankit Bansal on Kaggle — order-level retail transaction data including product category, sub-category, region, ship mode, list price, cost price, discount percent, and order dates.

## Pipeline overview

**1. Extract**
- Pulled the dataset directly via the Kaggle API (`kaggle datasets download`) rather than a manual download, so the pipeline is reproducible end-to-end.
- Unzipped the file programmatically with Python's `zipfile`.

**2. Transform (Pandas)**
- Handled inconsistent null representations at load time — values like `'Not Available'` and `'unknown'` were mapped to proper `NaN` using `na_values` in `pd.read_csv`, instead of cleaning them after the fact.
- Standardized column names: lowercased and replaced spaces with underscores (e.g. `Ship Mode` → `ship_mode`) so the columns map cleanly to SQL conventions.
- Engineered new business metrics that weren't in the raw data:
  - `discount` = `list_price * discount_percent / 100`
  - `sale_price` = the actual price after discount
  - `profit` = `sale_price - cost_price`
- Converted `order_date` from a plain string/object to a proper `datetime` type, which is what makes the later month/year SQL aggregations possible.
- Dropped the now-redundant raw columns (`list_price`, `cost_price`, `discount_percent`) once the derived metrics were in place, keeping the final table lean.

**3. Load**
- Pushed the cleaned DataFrame into a SQL Server table (`df_orders`) using SQLAlchemy's `create_engine` with the `mssql` dialect, connecting to a local SQL Server instance via Windows authentication.
- Used `df.to_sql(..., if_exists='append')` so the pipeline can be re-run without dropping and recreating the schema each time.

**4. Analyze (SQL)**
With clean data sitting in SQL Server, I wrote five analytical queries to answer specific business questions:

| Question | Technique used |
|---|---|
| Top 10 highest revenue-generating products | `SUM` + `GROUP BY` + `ORDER BY ... TOP` |
| Top 5 best-selling products *per region* | `ROW_NUMBER() OVER (PARTITION BY region ORDER BY sales DESC)` |
| Month-over-month sales comparison, 2022 vs 2023 | CTE + conditional aggregation (`SUM(CASE WHEN year = 2022 ...)`) to pivot years into columns |
| Best-performing month for each product category | CTE + `ROW_NUMBER()` partitioned by category, filtered to rank 1 |
| Sub-category with the highest profit growth, 2023 vs 2022 | Two-stage CTE: aggregate by year, pivot to columns, then rank by `(sales_2023 - sales_2022)` |

These mirror the kinds of questions a sales/category manager would actually ask in a retail business review — not just "can you write a window function."

## Tech stack

- **Python**: Pandas (cleaning, transformation, feature engineering)
- **SQLAlchemy**: database connection and load
- **SQL Server (T-SQL)**: window functions, CTEs, conditional aggregation
- **Kaggle API**: automated dataset extraction

## Key takeaway

This project isn't just "clean a CSV and call it ETL" — it covers the full loop: pulling data programmatically, transforming it into business-ready metrics that didn't exist in the raw source, persisting it into a relational database, and then writing production-style SQL (CTEs, window functions, pivoting with conditional aggregation) to extract decision-ready insights.

## Possible next steps

- Parameterize the SQL Server connection string (currently hardcoded to a local instance) using environment variables for portability.
- Add a Power BI dashboard on top of `df_orders` to visualize the regional and category trends from these queries.
- Wrap the queries into stored procedures or a lightweight dbt model for repeatable reporting.
