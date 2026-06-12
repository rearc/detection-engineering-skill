# Column Summarization

Profile 1–5 schema-confirmed columns. Do not guess column names — run `DESCRIBE TABLE` first.

Run these queries per column. Replace `{col}`, `{table}`, and `{filters}` with actual values.

## Step 1: Summary Stats

```sql
SELECT
    COUNT(*)                                                       AS total_row_count,
    SUM(CASE WHEN {col} IS NULL THEN 1 ELSE 0 END)                AS null_count,
    SUM(CASE WHEN {col} IS NULL THEN 1 ELSE 0 END)
        / NULLIF(COUNT(*), 0)                                      AS null_rate,
    COUNT(DISTINCT {col})                                          AS distinct_count,
    COUNT({col})                                                   AS non_null_count,
    COUNT(TRY_CAST({col} AS DOUBLE))                               AS numeric_cast_count
FROM {table}
WHERE {filters};
```

**Interpret results:**
- `null_rate >= 0.5` → column is null-heavy; warn the expert before using it as a filter or group key
- `distinct_count > 20` → high-cardinality; top-values query will be a partial view
- `numeric_cast_count < non_null_count` → column has mixed types; `TRY_CAST` silently drops non-numeric values

## Step 2: Top Values

```sql
SELECT
    CAST({col} AS STRING) AS value,
    COUNT(*)              AS count_result
FROM {table}
WHERE {col} IS NOT NULL
  AND {filters}
GROUP BY {col}
ORDER BY count_result DESC, value ASC
LIMIT 20; -- default value, increase if needed
```

## Step 3: Numeric Stats (only if `numeric_cast_count > 0`)

```sql
SELECT
    MIN(TRY_CAST({col} AS DOUBLE))    AS minimum,
    MAX(TRY_CAST({col} AS DOUBLE))    AS maximum,
    AVG(TRY_CAST({col} AS DOUBLE))    AS average,
    STDDEV(TRY_CAST({col} AS DOUBLE)) AS standard_deviation
FROM {table}
WHERE {filters};
```

## Step 4: Sample Values

```sql
SELECT DISTINCT CAST({col} AS STRING) AS sample_value
FROM {table}
WHERE {col} IS NOT NULL
  AND {filters}
ORDER BY sample_value ASC
LIMIT 5; -- default value, increase if needed
```

## Notes

- Run Step 3 only if Step 1 shows `numeric_cast_count > 0`; skip for string-only columns.
- Cap the number of columns you profile at 5 per call to keep results readable.
- For multi-column profiling, repeat the four steps per column rather than batching them.
