[![Releases](https://img.shields.io/badge/Releases-Download-blue?logo=github&style=for-the-badge)](https://github.com/Saulalvaa/sql-data-analytics-project/releases)

# SQL Data Analytics Project — Queries & Patterns for BI Teams 🧩📊

![data-sql](https://images.unsplash.com/photo-1555949963-aa79dcee981d?auto=format&fit=crop&w=1400&q=80)

A curated set of SQL scripts and patterns for common analytics tasks: changes over time, cumulative measures, performance metrics, segmentation, and part-to-whole analysis. Use these queries as building blocks for dashboards, reports, and data products.

Badges
- Topics: business-analytics, business-intelligence, data-analysis, data-analytics, data-engineering, data-science, query, reporting, sql, sql-queries, sql-server, window-functions  
- License: MIT
- Language: SQL

Quick links
- Releases (download and execute): https://github.com/Saulalvaa/sql-data-analytics-project/releases
- Repo: https://github.com/Saulalvaa/sql-data-analytics-project

What you get
- Ready-to-run SQL scripts that cover common BI scenarios.
- Examples that show window functions, aggregations, joins, and CTES.
- Small sample datasets and schema definitions to test queries locally.
- Readable patterns you can copy into your reports.

Table of contents
- Features
- About the examples
- File layout
- Setup and run
- Common query patterns (with examples)
  - Changes over time
  - Cumulative totals
  - Performance metrics
  - Segmentation and cohorts
  - Part-to-whole and share-of
- Sample schema
- Best practices
- Contributing
- License

Features
- Focused examples. Each script targets a single analysis pattern.
- Cross-platform queries. Scripts run on major SQL engines with minimal edits.
- Window functions. Many examples use OVER, PARTITION BY, and ordered windows.
- Performance-aware patterns. Indexing and execution hints appear in comments.
- Reusable pieces. Build larger queries from the provided CTEs and subqueries.

About the examples
Each folder contains:
- schema.sql — table definitions and sample rows.
- data.sql — INSERTs for small sample data.
- analysis_X.sql — analysis scripts for the pattern.
- readme.md — pattern explanation and expected output.

File layout (sample)
- /schema/
  - schema_orders.sql
  - schema_customers.sql
- /data/
  - sample_orders.sql
  - sample_customers.sql
- /queries/
  - changes_over_time.sql
  - cumulative_totals.sql
  - performance_metrics.sql
  - segmentation_cohorts.sql
  - part_to_whole.sql
- /examples/
  - dashboard_example.sql
- LICENSE
- README.md

Setup and run

1) Grab the release package
- Visit the Releases page and download the release asset. The release contains schema files and runnable SQL scripts.
- Releases: https://github.com/Saulalvaa/sql-data-analytics-project/releases

2) Load the schema and data
- For SQL Server (sqlcmd)
  - Open a command prompt and run:
    ```bash
    sqlcmd -S your_server -d your_db -U your_user -P your_pass -i schema/schema_orders.sql
    sqlcmd -S your_server -d your_db -U your_user -P your_pass -i data/sample_orders.sql
    ```
- For PostgreSQL (psql)
  - Run:
    ```bash
    psql -h host -U user -d db -f schema/schema_orders.sql
    psql -h host -U user -d db -f data/sample_orders.sql
    ```

3) Execute analysis scripts
- Run the queries in a client or via command line. Example:
  ```bash
  sqlcmd -S your_server -d your_db -U user -P pass -i queries/changes_over_time.sql
  ```

Common query patterns (examples below use ANSI SQL and window functions)

Changes over time (trend detection)
- Problem: Get period-over-period change and percent change.
- Pattern: Use window functions to access the previous period, then compute the delta.

Example:
```sql
WITH sales_by_month AS (
  SELECT
    DATE_TRUNC('month', order_date) AS month,
    SUM(total_amount) AS total_sales
  FROM orders
  GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
  month,
  total_sales,
  LAG(total_sales) OVER (ORDER BY month) AS prev_month_sales,
  total_sales - LAG(total_sales) OVER (ORDER BY month) AS change,
  CASE
    WHEN LAG(total_sales) OVER (ORDER BY month) = 0 THEN NULL
    ELSE (total_sales::numeric / LAG(total_sales) OVER (ORDER BY month) - 1) * 100
  END AS pct_change
FROM sales_by_month
ORDER BY month;
```

Cumulative totals (running totals)
- Problem: Show running aggregation to date.
- Pattern: Use SUM() OVER (ORDER BY ...) with a frame that accumulates rows.

Example:
```sql
SELECT
  order_date,
  SUM(total_amount) AS daily_sales,
  SUM(SUM(total_amount)) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders
GROUP BY order_date
ORDER BY order_date;
```

Performance metrics (latency, throughput)
- Problem: Compute average and P95 latency per service or endpoint.
- Pattern: Use percentile functions and filtered aggregates.

Example (Postgres):
```sql
SELECT
  service,
  COUNT(*) AS requests,
  AVG(response_time_ms) AS avg_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time_ms) AS p95_ms
FROM service_logs
WHERE event_time >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY service
ORDER BY requests DESC;
```
For SQL Server use APPROX_PERCENTILE or use window-based percentile approximations.

Segmentation and cohorts
- Problem: Group users by behavior or acquisition cohort.
- Pattern: Assign cohort based on first event, then compare retention or value across cohorts.

Example:
```sql
WITH first_order AS (
  SELECT user_id, MIN(order_date) AS first_order_date
  FROM orders
  GROUP BY user_id
),
cohort_orders AS (
  SELECT
    f.user_id,
    DATE_TRUNC('month', f.first_order_date) AS cohort_month,
    DATE_TRUNC('month', o.order_date) AS order_month,
    SUM(o.total_amount) AS order_value
  FROM first_order f
  JOIN orders o ON o.user_id = f.user_id
  GROUP BY f.user_id, cohort_month, order_month
)
SELECT
  cohort_month,
  order_month,
  COUNT(DISTINCT user_id) AS users,
  SUM(order_value) AS revenue
FROM cohort_orders
GROUP BY cohort_month, order_month
ORDER BY cohort_month, order_month;
```

Part-to-whole analysis (share of total)
- Problem: Compute contribution of categories to totals and show top N + others.
- Pattern: Use window SUM() to compute total and then TOP N with UNION for rest.

Example:
```sql
WITH sales_by_category AS (
  SELECT category, SUM(total_amount) AS revenue
  FROM orders
  GROUP BY category
),
ranked AS (
  SELECT
    category,
    revenue,
    RANK() OVER (ORDER BY revenue DESC) AS rnk,
    SUM(revenue) OVER () AS total_revenue
  FROM sales_by_category
)
SELECT
  CASE WHEN rnk <= 5 THEN category ELSE 'Other' END AS category_group,
  SUM(revenue) AS revenue,
  SUM(revenue) * 100.0 / MAX(total_revenue) AS pct_of_total
FROM ranked
GROUP BY CASE WHEN rnk <= 5 THEN category ELSE 'Other' END
ORDER BY revenue DESC;
```

Sample schema (short)
- orders (order_id, user_id, order_date, total_amount, category)
- users (user_id, signup_date, country)
- service_logs (log_id, service, event_time, response_time_ms)

Best practices
- Prefer window functions for ordered analytics. They outperform self-joins in many cases.
- Use numeric types that match your scale. Use decimal or numeric for money.
- Add relevant indexes on partition keys and date columns.
- Use small sample data locally to validate logic before running on large tables.
- Test performance with EXPLAIN or execution plans.

Releases and downloads
- The release bundle contains the full set of scripts and sample data.
- Download the asset from the Releases page and execute the SQL files corresponding to your engine.
- Releases: https://github.com/Saulalvaa/sql-data-analytics-project/releases

Contributing
- Fork the repo.
- Add a clear example file and a matching schema/data file.
- Keep scripts focused on a single pattern.
- Open a pull request with a description of the pattern and expected output.

Contact and support
- Use the repository Issues for bug reports, performance notes, and feature requests.
- Provide engine details and sample query for issues that involve performance.

Images and visual assets
- Charts in examples link to static sample images or code for clients like Grafana and Power BI.
- Use provided dashboard_example.sql to export result sets that feed into visual tools.

License
- MIT

Images used
- Unsplash dataset image for header visualization: https://images.unsplash.com/photo-1555949963-aa79dcee981d

End of README entries.