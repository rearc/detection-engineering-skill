# Grouped Percentiles

Compute percentile distributions of a numeric column per entity or segment. Use for threshold evidence — understanding where the 90th or 99th percentile sits before proposing a detection threshold.

**Important:** `value_col` must be an existing numeric table column, not an aggregate output from a previous query. Do not pass `count_result` or other derived values.

## Template

```sql
SELECT
    {group_col1},
    {group_col2},  -- add or remove group columns as needed
    PERCENTILE(CAST({value_col} AS BIGINT), 0.50) AS p50,
    PERCENTILE(CAST({value_col} AS BIGINT), 0.75) AS p75,
    PERCENTILE(CAST({value_col} AS BIGINT), 0.90) AS p90,
    PERCENTILE(CAST({value_col} AS BIGINT), 0.95) AS p95,
    PERCENTILE(CAST({value_col} AS BIGINT), 0.99) AS p99,
    COUNT(*)                                       AS event_count
FROM {table}
WHERE {filters}
GROUP BY {group_col1}, {group_col2}
ORDER BY {group_col1}, {group_col2}
LIMIT 100;
```

## Single-Population Percentiles (No Grouping)

When you need the overall distribution without segmentation:

```sql
SELECT
    PERCENTILE(CAST({value_col} AS BIGINT), 0.50) AS p50,
    PERCENTILE(CAST({value_col} AS BIGINT), 0.75) AS p75,
    PERCENTILE(CAST({value_col} AS BIGINT), 0.90) AS p90,
    PERCENTILE(CAST({value_col} AS BIGINT), 0.95) AS p95,
    PERCENTILE(CAST({value_col} AS BIGINT), 0.99) AS p99,
    COUNT(*)                                       AS event_count
FROM {table}
WHERE {filters};
```

## Notes

- Use `CAST(col AS BIGINT)` — `PERCENTILE` in Databricks SQL requires a numeric type; string-stored numbers need explicit casting.
- Adjust the percentile list to match the evidence needed. For detection thresholds, p90/p95/p99 are the most useful.
- If the p99 value is close to the p95 value, the distribution is relatively uniform at the high end. If p99 is much larger, a small number of outliers dominate — which is often the right anchor for a detection threshold.
- Cap `group_by_columns` at 4; more than that makes results hard to interpret and query planning slow.
